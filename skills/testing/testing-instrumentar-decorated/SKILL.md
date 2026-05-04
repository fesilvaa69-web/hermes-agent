---
name: testing-instrumentar-decorated
description: How to mock and test functions decorated with @instrumentar (functools.wraps gotcha)
triggers:
  - "@instrumentar"
  - "mock decorated function"
  - "__globals__ decorator"
  - "functools.wraps __globals__"
  - "sys.modules mock"
  - "urllib mock test"
  - "radar_live_commands mock"
---

# Testing @instrumentar Decorated Functions

## Contexto

O decorator `@instrumentar` (em `cortex/maintenance/instrumentar.py`) usa `@wraps(func)` do functools. Quando uma função é decorada com `@instrumentar`, o `func.__globals__` NÃO aponta para o módulo original da função — em vez disso, aponta para o módulo ONDE O DECORATOR É DEFINIDO (`instrumentar.py`).

## Problema Prático

Ao tentar mockar funções decoradas com `@instrumentar` para testes unitários:

```python
# ❌ NÃO FUNCIONA — th_module.handle_list NÃO é a mesma referência
# que process_message vê internamente
th_module.handle_list = mock.MagicMock(return_value="MOCK")
```

## Solução — Padrão 1: Decorator Pass-Through

Para módulos decorados com `@instrumentar`, usar mock transparente da função real:

```python
import sys
from unittest.mock import MagicMock

def _instrumentar_mock(tool_id=None, skill_name=None, **kwargs):
    def decorator(func):
        return func
    return decorator

# Setup ANTES do import do módulo
sys.modules['cortex.maintenance.instrumentar'] = MagicMock()
sys.modules['cortex.maintenance.instrumentar'].instrumentar = _instrumentar_mock

# Agora importar — as funções decoradas executam normalmente
import my_module
```

## Solução — Padrão 2: `__wrapped__` Access

Se precisar mockar uma função interna chamada pela função decorada:

```python
def test_despacha_trade(self):
    import services.telegram_handler as th_module

    orig_pm = th_module.process_message.__wrapped__
    orig_g = orig_pm.__globals__

    # Mockar handle_trade nos globals de process_message
    orig_g['handle_trade'] = mock.MagicMock(return_value="✅ OK")

    try:
        result = th_module.process_message("/trade\nmodelo: FVG\nresultado: WIN")
        assert "✅ OK" in result
    finally:
        orig_g['handle_trade'] = th_module.handle_trade  # restaurar
```

## Padrão Completo para Testar

```python
import sys
from pathlib import Path
from unittest.mock import MagicMock
import pytest

WORKSPACE = Path.home() / ".openclaw/workspace"
sys.path.insert(0, str(WORKSPACE))
sys.path.insert(0, str(WORKSPACE / "trading_agent"))  # para módulos nested

# Mock do instrumentar ANTES do import
def _instrumentar_mock(tool_id=None, skill_name=None, **kwargs):
    def decorator(func):
        return func
    return decorator

sys.modules['cortex.maintenance.instrumentar'] = MagicMock()
sys.modules['cortex.maintenance.instrumentar'].instrumentar = _instrumentar_mock

# Importar módulo já decorado
import my_module
```

## Técnica: Mockando Módulos Importados Localmente

Algumas funções fazem `import requests` ou `import feedparser` DENTRO da função (não no topo). Esses módulos precisam ser mockados via `sys.modules` ANTES da chamada:

```python
# Setup para funções com imports locais dentro delas
sys.modules['requests'] = MagicMock()
sys.modules['feedparser'] = MagicMock()
sys.modules['yfinance'] = MagicMock()

# Para radar_live_commands (importado dentro de handle_callback):
mock_rlc = MagicMock()
mock_rlc.cmd_briefing = MagicMock(return_value="📊 Briefing")
mock_rlc.dispatch = MagicMock(return_value="📊 Resultado")
sys.modules['radar_live_commands'] = mock_rlc
```

## Técnica: Mockando urllib.request.urlopen (HTTP)

Para funções que fazem chamadas HTTP via `urllib.request`:

```python
from unittest.mock import MagicMock, patch, mock_open

def test_send_with_buttons():
    mock_resp = MagicMock()
    mock_resp.read.return_value = b'{"ok": true}'
    mock_resp.__enter__ = MagicMock(return_value=mock_resp)
    mock_resp.__exit__ = MagicMock(return_value=False)

    mock_urlopen = MagicMock(return_value=mock_resp)
    mock_config = '{"channels": {"telegram": {"accounts": {"radar": {"botToken": "test_token"}}}}}'

    with patch('urllib.request.urlopen', mock_urlopen):
        with patch('pathlib.Path.open', mock_open(read_data=mock_config)):
            result = my_module.send_with_buttons("Teste", [[{"text": "OK", "callback_data": "ok"}]], "12345")

    assert mock_urlopen.called
    assert mock_urlopen.call_args[0][0].get_method() == 'POST'
```

## Técnica: Mockando leitura de arquivos (token, config)

```python
from unittest.mock import mock_open

def test_leitura_token():
    mock_config = '{"token": "abc123"}'
    with patch('pathlib.Path.open', mock_open(read_data=mock_config)):
        result = my_module.get_token()
    assert result == "abc123"
```

## Técnica: Mockando sqlite3.Row (FakeRow)

Quando uma função decorada retorna `sqlite3.Row` objects (ex: de `conn.execute().fetchone()`), `MagicMock` não funciona com `dict()` porque `sqlite3.Row` precisa de `.keys()` para conversão:

```python
# ❌ NÃO FUNCIONA
mock_row = MagicMock()
mock_row.__getitem__ = lambda self, k: {...}.get(k, None)
dict(mock_row)  # → {} (MagicMock não tem .keys())
```

**Solução — FakeRow class:**

```python
class FakeRow:
    def __init__(self, data):
        self._data = data
    def __getitem__(self, key):
        return self._data[key]
    def keys(self):
        return self._data.keys()

# Uso:
mock_conn.execute.return_value.fetchone.return_value = FakeRow({
    'id': 10, 'entrada_plan': 1000.0, 'stop_plan': 990.0,
    'direcao': 'LONG', 'resultado': 'WIN',
})
```

**Também útil para `fetchall()`** — retorna lista de `FakeRow`:

```python
mock_row2 = FakeRow({'id': 1, 'ativo': 'WDOFUT'})
mock_conn.execute.return_value.fetchall.return_value = [mock_row2]
```

## Técnica: sys.path para Módulos Nested

Quando o test file está dentro de um subpacote (ex: `diario_trades/core/test_db_manager.py`) e precisa importar `diario_trades.core.db_manager`:

```python
import sys
from pathlib import Path
WORKSPACE = Path.home() / ".openclaw/workspace"
sys.path.insert(0, str(WORKSPACE))
sys.path.insert(0, str(WORKSPACE / "trading_agent"))  # ← NECESSÁRIO

from diario_trades.core import db_manager  # agora funciona
```

## Técnica: `datetime` Mocking — TRÊS Padrões (do mais simples ao mais difícil)

**Padrão 1 — Sem aritmética:** `patch.object` + attrs preservadas

Quando a função só usa `datetime.now()` (sem `timedelta`, sem `fromisoformat` comparisons):
```python
with patch.object(module, "datetime") as mock_dt:
    mock_dt.now.return_value = datetime(2026, 4, 20, 10, 0, 0)
    mock_dt.fromisoformat = datetime.fromisoformat  # ← se usar fromisoformat
    mock_dt.timedelta = timedelta                   # ← se usar timedelta
    mock_dt.side_effect = lambda *a, **kw: datetime(*a, **kw)
```

**Padrão 2 — Com aritmética (timedelta subtraction): `MagicMock - timedelta` retorna MagicMock (truthy!)**

Quando o código faz `datetime.now() - timedelta(days=N)`:
```python
# ❌ patch.object com MagicMock — FALHA silenciosamente
with patch.object(module, "datetime") as mock_dt:
    mock_dt.now.return_value = fake_now
    # mock_dt - timedelta → MagicMock (sempre truthy!)
    # Comparisons like MagicMock > datetime → TypeError ou truthy errado
```

**✅ SOLUÇÃO — `ControlledDatetime` (subclasse real):**
```python
from datetime import datetime as RealDatetime

class ControlledDatetime(RealDatetime):
    """Subclasse que controla now() mas preserva TODAS as operações aritméticas."""
    _now = RealDatetime(2026, 4, 20, 10, 0, 0)
    @classmethod
    def now(cls, tz=None):
        return cls._now

# Uso — monkeypatch para substituir a classe no módulo:
monkeypatch.setattr(module, "datetime", ControlledDatetime)
# Para modules com BRT timezone:
monkeypatch.setattr(module, "BRT", None)  # previne BRT.timezone() calls
```

**Padrão 3 — `wraps=datetime` com `FakeDatetime` para fromisoformat:**
```python
class FakeDatetime(datetime):
    @staticmethod
    def fromisoformat(s):
        return datetime.fromisoformat(s)

with patch.object(module, "datetime", wraps=datetime) as mock_dt:
    mock_dt.now.return_value = fake_now
    mock_dt.fromisoformat = FakeDatetime.fromisoformat
    # ⚠️ datetime - timedelta ainda retorna MagicMock com wraps!
    # → prefira ControlledDatetime se usar aritmética
```

**Regra de decisão:**
| Situação | Padrão |
|---|---|
| Só `datetime.now()` | Padrão 1 (`patch.object` + attrs) |
| `datetime - timedelta` comparisons | **Padrão 2 (`ControlledDatetime`)** ← mais comum |
| Precisa `wraps` para preservar métodos | Padrão 3 (com `FakeDatetime`) |

**⚠️ ERRO COMUM — `MagicMock - timedelta` é truthy:**
```python
# O que acontece dentro do módulo com MagicMock:
mock_dt = MagicMock()
result = mock_dt - timedelta(days=30)
# result é MagicMock (não um datetime real)
# comparisons: MagicMock > datetime → TypeError
# ou se não der TypeError: MagicMock > datetime → True (sempre!)
# → list comprehensions com filtros por data quebram silenciosamente
```

**✅ CORRETO — ControlledDatetime preserva aritmética:**
```python
class ControlledDatetime(datetime):
    _now = datetime(2026, 4, 20, 10, 0, 0)
    @classmethod
    def now(cls, tz=None):
        return cls._now

# Dentro do módulo:
# now = ControlledDatetime.now() → 2026-04-20 10:00:00
# cutoff = now - timedelta(days=30) → 2026-03-21 10:00:00 ← REAL datetime
# comparison: RealDatetime > RealDatetime → funciona corretamente
```

## Técnica: Ordem de Dados em `_load_raw()` — ARMADILHA

**Problema:** Funções de histórico que usam `_load_raw()` retornam dados em **ordem de inserção (antigo → novo)**. Testes com dados em ordem newest→oldest falham silenciosamente.

```python
# ❌ ORDEM ERRADA — newest first (intuitivo mas errado)
history = [
    {"ts": "2026-04-20T10:00:00", "score": 75.0},  # newest
    {"ts": "2026-04-19T10:00:00", "score": 70.0},
    {"ts": "2026-04-18T10:00:00", "score": 60.0},
    {"ts": "2026-04-17T10:00:00", "score": 55.0},  # oldest
]
# get_score_trend: first_half=[75,70]avg=72.5, second=[60,55]avg=57.5 → WRONG trend!
```

**✅ CORRETO — oldest first (insertion order):**
```python
history = [
    {"ts": "2026-04-17T10:00:00", "score": 55.0},  # oldest FIRST
    {"ts": "2026-04-18T10:00:00", "score": 60.0},
    {"ts": "2026-04-19T10:00:00", "score": 70.0},
    {"ts": "2026-04-20T10:00:00", "score": 75.0},  # newest LAST
]
# first_half=[55,60]avg=57.5, second=[70,75]avg=72.5 → improving ✓
```

**Regra:** Sempre que `load_history`/`_load_raw` for mocked em testes de tendência, dados DEVEM estar em ordem de inserção (oldest→newest) conforme o arquivo JSON real.

## Técnica: Verificar Keys em Dicts de Retorno — Early Returns

**Problema:** Algumas branches de funções retornam dicts incompletos. Exemplo: `get_score_trend` com `insufficient_data` não inclui `emoji` no retorno, mas as outras branches incluem.

```python
# Código:
def get_score_trend(history):
    if len(history) < 4:
        return {"trend": "insufficient_data", "delta": 0.0}  # ← sem emoji!
    # ...
    return {
        "trend": trend,
        "emoji": emoji_map[trend],  # ← só aqui
        # ...
    }

# Teste ❌ FALHA:
assert "emoji" in trend  # AssertionError — não existe na branch early return!

# Teste ✅ CORRETO:
assert trend["trend"] == "insufficient_data"
assert "emoji" not in trend
```

**Regra:** Sempre verificar o código real antes de assumir que todas as branches retornam as mesmas keys. Fazer um `read_file` rápido do módulo para confirmar keys retornadas em cada branch.

## Técnica: `patch.object` vs `patch()` com String Path

**Preferir `patch.object`** para patching no mesmo módulo testado:
- `patch.object(module, "datetime")` → funciona com `sys.modules` já carregados
- `patch("module.datetime")` → às vezes perde atributos (como `fromisoformat`)

```python
# ✅ preferred
with patch.object(sh, "datetime") as mock_dt:
    mock_dt.now.return_value = fake_dt
    mock_dt.fromisoformat = datetime.fromisoformat
    mock_dt.timedelta = timedelta

# ⚠️ works but riskier with complex modules
with patch("trading_agent.bot.core.score_history.datetime") as mock_dt:
    mock_dt.now.return_value = fake_dt
```

## Técnica: Decorator Pass-Through — Diferença de FakeRow

Para módulos **sem** `__wrapped__` (ex: `score_engine.py` — funções puras sem calls internos), usar apenas `_instrumentar_mock`. O decorador é transparente e não interfere.

Para módulos **com** `__wrapped__` (ex: `telegram_handler.py` — funções que chamam outras funções), usar `__wrapped__` para mockar deps internas.

**Regra:** se `hasattr(module.func, '__wrapped__')` → usar `__wrapped__`. Senão → mock transparente basta.

## Quando Não Usar Este Padrão

- Se a função decorada não usa `@wraps` — aí `__globals__` seria o módulo original normalmente
- Se o mock não precisa ser verificado com `assert_called_with` — substituição direta no módulo basta
- Módulos puros de cálculo (sem deps externas) — mock transparente é suficiente

## Referências

- `__wrapped__` attribute: Python functools.wraps preserves reference to original function
- `__globals__` of a wrapper: points to defining module (the decorator's module), NOT the wrapped function's module
- `sys.modules` mocking: standard Python technique for intercepting module imports