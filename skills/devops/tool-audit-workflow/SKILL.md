---
name: tool-audit-workflow
description: Systematic tool analysis and testing workflow for OpenClaw/Hermes ecosystem using DIAGRAM cycles and SIAS 4-layer testing
triggers:
  - "continuar workflow"
  - "auditar ferramenta"
  - "analisar ferramenta"
  - "DIAGRAM"
  - "continuar mapeamento"
---

# Tool Audit Workflow — DIAGRAM Cycle

## Purpose
Systematic analysis and testing of Python tools in the OpenClaw/Hermes ecosystem using the DIAGRAM cycle. Continuous workflow to audit tools one by one, following the SIAS 4-layer testing pattern.

## When to Use
When asked to continue the tool mapping workflow, audit a specific Python tool, or perform systematic QA on the workspace-core/skills directory.

## Workflow

### Phase A — Static Analysis
1. Read the target tool file completely
2. Identify problems by severity (CRITICAL/MEDIUM/LOW)
3. Check for hardcoded credentials, missing tests, code duplication

### Phase B — Test Planning
Plan 4-layer SIAS test structure:
- **SMOKE** (2-4 tests): Module imports, basic functionality exists
- **UNIT** (6-10 tests): Business logic, pure functions
- **INTEGRATION** (2-4 tests): System behavior, file I/O
- **STRESS** (2-4 tests): Edge cases, load testing

### Phase C — Create Tests
Create `test_<tool_name>.py` in the `test/` directory alongside the tool.

### Phase D — Run & Iterate
```bash
cd ~/.openclaw/workspace-core/skills/diario-trade
python3 -m pytest test/test_<tool>.py -v
```

### Phase E — Innovation
Propose improvements:
1. Retry decorators with exponential backoff
2. MessageBus abstraction for multi-channel delivery
3. Prometheus metrics for monitoring
4. Environment-based configuration

## Key Patterns Discovered

### Mocking TOKEN (NOT a module attribute)
TOKEN is defined INSIDE functions, not at module level. Do NOT use:
```python
patch('diario_scheduler.TOKEN')  # FAILS - AttributeError
```
Instead, mock the entire function:
```python
patch('diario_scheduler.enviar_mensagem_telegram')
```

### State File Path
State file is `STATE_FILE = LOG_DIR / "scheduler_state.json"`. Mock `load_state()` directly rather than patching path:
```python
with patch('diario_scheduler.load_state') as mock_load:
    mock_load.return_value = {"last_runs": {}}
    result = load_state()
```

### Imports for Testing
```python
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))
from <module> import function_name
```

## Test Structure Template
```python
"""
test_<tool>.py — Tests for <tool>.py
====================================

SIAS FASE D: Test | Camadas: Smoke + Unit + Integration + Stress
Target: <tool>.py
"""

import pytest
import sys
import os
from unittest.mock import MagicMock, patch, mock_open

sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))
from <module> import (
    function1,
    function2,
    CLASS_NAME,
)

# ═══════════════════════════════════════════════════════════════
# CAMADA 1: SMOKE TESTS
# ═══════════════════════════════════════════════════════════════
class TestSmoke:
    """SMOKE: Validação básica"""

    def test_modulo_importado_com_sucesso(self):
        import <module>
        assert hasattr(<module>, 'function1')

    def test_funcao_principal_existe(self):
        assert callable(function1)

# ═══════════════════════════════════════════════════════════════
# CAMADA 2: UNIT TESTS
# ═══════════════════════════════════════════════════════════════
class TestUnit:
    """UNIT: Lógica de negócio"""

    def test_funcao_retorna_tipo_esperado(self):
        result = function1(arg1="value")
        assert isinstance(result, ExpectedType)

# ═══════════════════════════════════════════════════════════════
# CAMADA 3: INTEGRATION TESTS
# ═══════════════════════════════════════════════════════════════
@pytest.mark.integration
class TestIntegration:
    """INTEGRATION: Comportamento de sistema"""

    def test_integracao_modulos(self):
        ...

# ═══════════════════════════════════════════════════════════════
# CAMADA 4: STRESS TESTS
# ═══════════════════════════════════════════════════════════════
@pytest.mark.stress
class TestStress:
    """STRESS: Edge cases extremos"""

    def test_valor_extremo_nao_quebra(self):
        ...

pytestmark = pytest.mark.<tool_name>
```

## DIAGRAM Score Calculation
| Criteria | Weight | Score |
|----------|--------|-------|
| Tests pass rate | 30% | pass/total × 100 |
| Coverage | 25% | coverage% |
| Problems fixed | 25% | (fixed/total) × 100 |
| Innovation proposed | 20% | quality × 100 |

## Output Format
Generate DIAGRAM report:
```
# 📊 DIAGRAM v1.XX — CYCLE 00X

## 🔍 FERRAMENTA: `tool.py`

| Atributo | Valor |
|----------|-------|
| Caminho | ... |
| Linhas | XXX |
| Propósito | ... |
| Agente | ... |
| Score | XX% |

## 📋 FASE A — ANÁLISE ESTÁTICA
[Problemas table]

## 🧪 FASE B-D — TESTES
[Test results]

## 🎯 FASE E — INOVAÇÃO
[Proposed innovations]

## 📊 RESULTADO
| Métrica | Valor |
|---------|-------|
| Testes | XX |
| Pass Rate | XX% |
| Cobertura | XX% |
| Score | XX% |
```

## Next Tool in Queue
Current queue after `diario_scheduler.py`:
1. `monitor_engine.py` — Scheduler de monitoramento
2. `diario_core.py` — Core do diário
3. `bias_automacao/briefing_generator.py` — Gerador de briefing
4. `proativo/pattern_detector.py` — Detecção de padrões
5. `interactive_handler.py` — Handler interativo
6. `media_storage.py` — Armazenamento de mídia
7. `photo_handler.py` — Handler de fotos

## Pitfalls
1. TOKEN/CHAT_ID are function-scoped, not module-scoped — cannot patch at module level
2. STATE_FILE uses pathlib.Path — mocking needs care
3. Many functions use `diario_core.status_diario()` — mock this for isolated tests
4. APScheduler BlockingScheduler blocks thread — tests should mock scheduler.add_job
