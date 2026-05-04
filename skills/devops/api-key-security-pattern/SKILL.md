---
name: api-key-security-pattern
description: Implementar gerenciamento seguro de chaves API em projeto Python com dotenv — .env.example, setup.py interativo e documentação. Para SaaS white-label multi-deploy.
triggers:
  - "mascarar chaves"
  - "segurança de API keys"
  - "implementar .env para deploy"
  - "tornar genérico para deploy"
  - "API key security"
---

# API Key Security Pattern — Python SaaS

## Trigger

Implementar gerenciamento seguro de chaves API em projeto Python com dotenv — quando Felipe pedir para "mascarar chaves", "tornar genérico para deploy", "implementar segurança de API keys", ou "configurar .env para deploy".

**Também serve para:** preparar repositório Python SaaS white-label para múltiplos mentores/hosts, onde cada deploy tem suas próprias credenciais.

---

## Quando NÃO usar

- Se o projeto já tem `.env.example`, `setup.py` e dotenv integrado → já está feito
- Se é um projeto novo que ainda não usa dotenv → verificar se dotenv já é padrão antes de implementar

---

## Passos (Implementação)

### 1. Mapear variáveis de ambiente em uso

```bash
grep -rn "os.getenv\|load_dotenv" src/ --include="*.py"
```

Identificar TODAS as variáveis que o projeto usa (TELEGRAM_BOT_TOKEN, REDIS_PASSWORD, API_KEY, etc.).

### 2. Verificar .gitignore

```bash
cat .gitignore
grep -c "\.env" .gitignore   # deve retornar >= 1
```

Se não tiver `.env`, adicionar:
```
.env
.env.local
.env.production
*.env
```

### 3. Verificar dotenv integrado

```bash
grep -l "load_dotenv" src/ --include="*.py" -r
```

Se algum módulo principal (bot.py, consumer.py, etc.) não tem `from dotenv import load_dotenv` + `load_dotenv()` no início, adicionar.

### 4. Criar .env.example

Template com TODAS as variáveis do projeto, valores genéricos (ex: `your_telegram_token_here`), sem valores reais. Estrutura por seção (Telegram, Redis, ChromaDB, LLM, etc.) com comentários.

**Importante:** `.env.example` vai na **raiz** do projeto (para visibilidade), não em `config/.env.example` (que pode não existir em todos os repos).

### 5. Criar setup.py (script interativo)

Script interativo que:
1. Lê o `.env.example` para saber quais vars são necessárias
2. Pergunta cada valor ao usuário via `input()` (com defaults)
3. Se `.env` já existe, pergunta se deseja sobrescrever
4. Grava no `.env` local
5. NUNCA exibe ou loga valores reais

Estrutura recomendada:
```python
#!/usr/bin/env python3
from __future__ import annotations
import os, sys

DOTENV_PATH = os.path.join(os.path.dirname(__file__), ".env")
EXAMPLE_PATH = os.path.join(os.path.dirname(__file__), ".env.example")

# Seções por tipo de variável (Telegram, Redis, LLM, etc.)
# Cada campo: (KEY, label, required)
# required=True →Loop até usuário preencher
# default → valor do .env.example

def load_defaults() -> dict[str, str]:
    """Lê valores padrão do .env.example."""
    defaults = {}
    if os.path.exists(EXAMPLE_PATH):
        with open(EXAMPLE_PATH, encoding="utf-8") as f:
            for line in f:
                line = line.strip()
                if "=" in line and not line.startswith("#"):
                    key, val = line.split("=", 1)
                    defaults[key.strip()] = val.strip()
    return defaults

def run():
    if os.path.exists(DOTENV_PATH):
        # Perguntar se sobrescreve
        ...
    defaults = load_defaults()
    # Coletar valores e gravar .env
    ...
```

### 6. Atualizar README.md

Adicionar seção `## ⚙️ Configuração (Primeiro Deploy)` com:
- Instrução: `python setup.py`
- Tabela de variáveis (nome, descrição, obrigatório/opcional)
- Aviso: `.env` nunca deve ser commitado

Manter o Quick Start limpo — `python setup.py` em vez de `cp .env.example .env`.

### 7. Validar

```bash
# Sintaxe
python -c "import setup; print('OK')"

# dotenv carrega
source venv/bin/activate
python -c "
from dotenv import load_dotenv
load_dotenv('config/.env')
import os
print('TELEGRAM_BOT_TOKEN:', bool(os.getenv('TELEGRAM_BOT_TOKEN')))
"

# Imports do projeto
python -c "import src.bot; print('bot import OK')"
```

### 8. Commitar e pushar

```bash
git add .env.example setup.py README.md
git commit -m "feat(security): add .env.example + setup.py interativo + README config section"
git push origin <branch>
```

---

## Validação Final Checklist

- [ ] Nenhum arquivo `.py` contém chave ou token hardcoded
- [ ] `.env` está no `.gitignore`
- [ ] `.env.example` tem todas as variáveis com valores genéricos
- [ ] `setup.py` syntax OK e fluxo interativo funciona
- [ ] `load_dotenv()` está no ponto de entrada do bot
- [ ] README tem seção de configuração
- [ ] Committed e pushed

---

## Complexidade

| Componente | Esforço |
|-----------|---------|
| Mapear variáveis | Baixo |
| Verificar/ajustar .gitignore | Mínimo |
| Criar .env.example | Mínimo |
| Criar setup.py | Baixo/Médio |
| Atualizar README | Baixo |
| Validar | Baixo |
| **Total** | **~1 sprint curto** |

**Uma biblioteca:** `python-dotenv` (já padrão no mercado, sem criptografia customizada necessária).

---

## Notas

- O padrão NÃO exige criptografia — `python-dotenv` + `.env` + `.gitignore` é suficiente para o caso de uso
- Para SaaS white-label, o `setup.py` interativo é o diferencial — cada mentor roda uma vez e tem seu `.env` local
- O `.env` NUNCA vai para o repositório — nem mesmo "vazio" (valores genéricos no `.env.example` são suficientes para novos deploys)
