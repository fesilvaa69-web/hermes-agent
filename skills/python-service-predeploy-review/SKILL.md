---
name: python-service-predeploy-review
description: "Senior-level pre-deploy OR post-deploy review of a Python service (especially Telegram bots). Systematic validation of architecture, config consistency, security, and operational health. Trigger: when Felipe asks to review, audit, validate, or diagnose a Python service — before first deploy, after deploy problems, or as a periodic health audit. Also use after parallel bot clone deployments to identify configuration gaps."
category: software-development
---

# Senior Pre-Deploy Review — Python Service

## Quando Usar

- Felipe pede "review completo", "validação", "auditoria" de um projeto Python antes de deploy
- Felipe pede "auditoria completa" com multiplos subagentes (verificar filesystem antes de delegar)
- Felipe pede para "diagnosticar por que o bot nao responde" (causa multipla: Redis, Chroma, LLM, allowlist)
- Primeiro deploy em outra máquina
- Antes de merge de branch significativo (generic-white-label, feature branch, etc.)
- Após mudanças estruturais (novo stack, refactoring, mudança de infra)
- Aps um incidente de producao (bot parou, usurios bloqueados, disk full)

## Premissa

- O servico ja existe e esta funcionando no ambiente atual
- O objetivo e validar que vai funcionar em novo ambiente E que nao tem problemas obvios
- **Nao inventar problemas.** Basear tudo no que esta no codigo, config, e runtime state real

## Workflow — 5 Etapas

### Etapa 1 — Ler o Codigo Real

Ler os arquivos criticos, nao apenas confiar no que foi dito:

```
- src/bot.py (ou main entry point)
- src/worker/llm.py (se existir)
- docker-compose.yml
- config/.env / config/.env.example
- requirements.txt
- .gitignore
```

Regra: ler o que esta no disco, nao assumir que corresponde a documentacao.

### Etapa 2 — Verificar Consistência Config × Codigo

Procurar gaps entre variaveis de ambiente definidas e o codigo que as consome:

```bash
# Variáveis definidas no .env mas nunca usadas no codigo
grep -r "os.getenv" src/

# Variáveis lidas pelo codigo mas nao definidas no .env.example
# (comparar .env.example vs todas os.getenv() em src/)
```

Gaps comuns:
- Variavel nomeada de um jeito no codigo, de outro no .env (ex: REDIS_DB_JOB vs REDIS_DB)
- Variavel lida pelo codigo mas ausente do .env.example
- Variavel no .env.example mas nao lida por nenhum codigo
- Configuracao de auth que existe no .env mas nao e passada ao servico (ex: ChromaDB auth password)
- Features flags no codigo sem documentacao no .env.example

**Verificacao de nomes de variaveis — checklist obrigatorio:**
```bash
# 1. Listar todas variaveis lidas pelo codigo
grep -rh "os.getenv\|os.environ\|os\.getenv" src/ --include="*.py" | \
  grep -oE '"[A-Z_]+"' | sort -u | sed 's/"//g' > /tmp/code_vars.txt

# 2. Listar todas variaveis no .env.example
grep -h "^[^#]" config/.env.example | grep "=" | cut -d= -f1 | sort -u > /tmp/env_vars.txt

# 3. Variaveis no codigo mas ausentes do .env.example (PRECISAM ser adicionadas)
comm -23 /tmp/code_vars.txt /tmp/env_vars.txt

# 4. Variaveis no .env.example mas nunca lidas pelo codigo (podem ser removidas)
comm -13 /tmp/code_vars.txt /tmp/env_vars.txt
```

**Verificacao de token exposure no git history:**
```bash
# Tokens reais que entraram no repo (procurar em todos os commits)
git log --all --oneline | while read sha msg; do
  git show $sha --name-only --format="" | grep -q "config/.env" && echo "$sha: $msg"
done

# Verificar se .env real foi commitado em algum momento
git rev-list --all | while read sha; do
  git ls-tree -r $sha -- config/.env 2>/dev/null && echo "EXISTS: $sha"
done | head -5
```

### Etapa 3 — Verificar Consistência Docker × Scripts

```bash
# container_names no docker-compose.yml
docker compose ps
# Comparar nomes gerados vs nomes esperados por scripts

# Scripts que referenciam containers
grep -r "container_name" scripts/
```

Problema tipico: docker-compose.yml cria mentoria-redis, scripts esperam mentorbot-redis.

### Etapa 4 — Verificar .gitignore e Conteudo Sensivel

```bash
# O que esta no repo que NAO deveria estar
git status --short
git ls-files kb/
git ls-files data/

# Verificar credenciais exposure
grep -r "PASSWORD\|API_KEY\|TOKEN" config/.env.example config/.env 2>/dev/null | grep -v "=***" | grep -v "your_"
```

Verificar:
- kb/ esta no .gitignore (conteudo do mentor e privado)
- .env esta no .gitignore
- data/ e volumes nao estao commitados
- Credenciais no .env.example usam *** ou your_...here como placeholder

### Etapa 5 — Testar Integracao Real

```bash
# Health check (se existir)
kill -USR1 <pid>; cat /tmp/mentorbot_health.json

# Docker services
docker compose ps
curl -s http://localhost:8000/api/v2/heartbeat

# Redis
redis-cli -a <password> ping

# Teste de LLM real (1 chamada)
python -c "from src.worker.llm import llm_generate; print(llm_generate([{'role':'user','content':'ping'}], max_tokens=3, temperature=0))"

# ChromaDB collection EXISTE (gap comum: deploy sobe ChromaDB vazio)
python -c "
import chromadb
c = chromadb.HttpClient(host='localhost', port=8000)
coll = c.get_collection('mentorbot_kb')
r = coll.get()
print(f'Collection existe: {len(r[\"ids\"])} docs')
" 2>/dev/null
# Se der CollectionDoesNotExistError → ingest.py nao foi executado pos-deploy
```

### Etapa 6 — Reporte e Fixes

Agrupar achados em 3 categorias:

| Severidade | Significado | Acao |
|------------|-------------|------|
| Critico | Deploy vai falhar ou dados expostos | Fix antes de merge |
| Aviso | Funciona mas problema latente | Fix ou documentar |
| Observacao | Boa pratica, nao urgente | Melhorar se tempo permitir |

**Regra:** Nao inventar complexidade. Se o codigo funciona e o problema e teorico (ex: "e se o ChromaDB precisar de auth?"), marcar como Observacao, nao Critico.

## Armadilha: Subagentes + /tmp/openclaw-backup

Subagentes via `delegate_task` dependem de `/tmp/openclaw-backup` existir. Se não existir, TODAS as operações de arquivo falham com `FileNotFoundError: [Errno 2] No such file or directory: '/tmp/openclaw-backup'`.

** workaround:** Para auditorias completas, fazer o trabalho DIRETAMENTE com `execute_code` em vez de delegar a subagentes quando o filesystem precisa ser acessado. Os subagentes só devem ser usados quando a tarefa é totalmente contida em si mesma (ex: research externo, chamadas de API).

**Verificação antes de usar subagentes:**
```python
import os
assert os.path.exists("/tmp/openclaw-backup"), "Subagentes não vão funcionar"
```

## Auditoria Multi-Subagente — Padrão de Trabalho

Quando Felipe pedir "auditoria completa com subagentes":

1. **Primeiro:** fazer verificação direta com `execute_code` ANTES de delegar —，确认 `/tmp/openclaw-backup` e os paths críticos existem
2. **Segundo:** se subagentes falharem por filesystem, fazer o trabalho direto
3. **Terceiro:** os subagentes podem ser usados para tarefas PARALELAS que não dependem de ler código do projeto (ex: research externo, benchmark de libraries, validação de API keys)

## Erros Estruturais Comuns Encontrados (Padrões)

### Padrão 1: REDIS_DB default diferente do .env
```python
# bot.py — default = 0
REDIS_DB = int(os.getenv("REDIS_DB", "0"))

# config/.env — configurado = 1
REDIS_DB=1
```
**Resultado:** se `load_dotenv()` falhar ou rodar depois de um import que usa `REDIS_DB`, o default `0` é usado. Com Redis autenticado, DB=0 sem password → `NOAUTH Authentication required` → `is_user_allowed()` retorna `False` para todos.

**Fix:** padronizar default para `1` (matching o .env) e adicionar `socket_connect_timeout=5` na conexão.

### Padrão 2: Collection name ChromaDB hardcoded
```python
# codigo
coll = client.get_collection("mentorbot_kb")

# .env
CHROMA_COLLECTION=cyberbot_kb
```
**Resultado:** bot busca em collection vazia, RAG sempre retorna vazio.

**Fix:** declarar `CHROMA_COLLECTION = os.getenv("CHROMA_COLLECTION", "cyberbot_kb")` e usar a variável.

### Padrão 3: Duas funções Redis idênticas com propsitos diferentes
```python
def _get_allowed_redis():  # sem timeout
def get_redis():           # com timeout
```
**Resultado:** bugs sutis onde uma função funciona e outra não, dependendo do código que as chama.

**Fix:** unificar em `get_redis()` com `socket_connect_timeout` e fazer `_get_allowed_redis()` ser alias.

### Padrão 4: .env duplicado com propsitos mistos
```
.env (raiz)     → carregado por load_dotenv() padrão, não usado
config/.env      → carregado por load_dotenv("config/.env"), é o real
```
**Resultado:** confusão sobre qual arquivo editar, risco de valores contraditórios.

**Fix:** manter apenas `config/.env` como fonte da verdade, remover ou documentar o `.env` raiz.

### Padrão 5: redis_session.py código morto
```python
_client = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, decode_responses=True)
# SEM password, SEM db config
```
**Resultado:** se for importado por engano, quebra o bot silenciosamente.

**Fix:** remover ou corrigir com `password=REDIS_PASSWORD or None, db=REDIS_DB`.

### Padrão 6: Valores mascarados no .env pelo dotenv
O dotenv exibe `REDIS_PASSWORD=***` quando a variável contém palavras como `PASSWORD`, `KEY`, `SECRET`. O valor REAL no arquivo pode estar diferente.

**Diagnóstico definitivo:** `docker inspect <container> | grep -A2 requirepass` — buscar a senha real no container Docker, não no .env.

## Output Padrao

```
## Laudo Tecnico — <nome-do-servico>

### O que e bom
[Lista de decisoes arquiteturais solidas — baseadas no que foi lido]

### Problemas reais (producao)
| # | Problema | Severidade | Fix |

### Issues menores (nao bloqueiam deploy)
| # | Problema | Severidade | Acao |

### Para o primeiro deploy em outra maquina
[Passo a passo minimal, baseado no que existe]

### Veredicto
[Positivo ou negativo + razao]
```

## Telegram Bot — Checklist de Auditoria Específica

### Allowlist Security (CRÍTICO)
```python
# Verificar se existe fallback hardcoded para TELEGRAM_ALLOWED_USERS
grep -n "5556338087" src/bot.py
```
**Problema:** Se `os.getenv("TELEGRAM_ALLOWED_USERS", "5556338087")` existe → security risk. Deve dar fatal error se não configurado.

**Fix aplicado (2026-05-02):** `_load_allowed_users()` agora levanta `RuntimeError` se `TELEGRAM_ALLOWED_USERS` não está configurado. `_load_admin_users()` usa allowlist como fallback mínimo com warning.

### Fluxo de Acesso — /solicitar + Admin Approval
```bash
# Novos comandos implementados:
grep -n "/solicitar\|/aprovar\|/negar\|/pedidos" src/bot.py
```
**Problema:** Bot anterior simplesmente ignorava usuários não-autorizados (silêncio). Não havia mecanismo de onboarding.

**Fluxo implementado:**
- Usuário não-autorizado → mensagem clara "Acesso negado, digite /solicitar"
- `/solicitar` → registra pedido no Redis (TTL 24h) + notifica admins
- `/pedidos` (admin) → lista todos os pedidos pendentes
- `/aprovar <chat_id>` (admin) → adiciona à allowlist e remove do queue
- `/negar <chat_id>` (admin) → remove do queue

**Redis keys envolvidas:**
- `{BOT_INSTANCE_ID}:access_requests:{chat_id}` — pedido pendente (TTL 24h)

### Placeholder Values em .env
```bash
# Verificar valores *** que causam int() parse errors
grep "=***" config/.env.example
grep "=your_" config/.env.example
```
**Problema:** `LLM_MAX_TOKENS=***` → `ValueError` na linha `int(os.getenv("LLM_MAX_TOKENS"))`

**Fix aplicado:** setup.py agora valida e rejeita placeholders (`***`, `your_*`, `changeme`) para campos required. .env.example usa inteiros reais (2048, 1024).

### Redis Connection Function Duplication (CRÍTICO — causa bugs silenciosos)
```bash
# Duas funções Redis com propsitos diferentes — verifica se sobrasaram ou devem ser unificadas
grep -n "def _get_allowed_redis\|def get_redis\|def _get_rate_limiter_redis" src/bot.py
```
**Problema:** `_get_allowed_redis()` (allowlist) e `get_redis()` (sessions) so IDENTICAS em propsito mas uma pode ter `socket_connect_timeout` e a outra no. Diferencas sutis entre funcoes gera bugs que passam despercebidos.

**Checklist de consistencia:**
- Ambas usam `REDIS_DB` do mesmo env var?
- Ambas usam `REDIS_PASSWORD`?
- Ambas tem `socket_connect_timeout`?
- Ambas tem `decode_responses=True`?

**Fix:** Unificar em uma s funcao `get_redis()` que recebe parametro `db=None` (default REDIS_DB) e usar para tudo.

### ChromaDB Collection Name — Hardcoded vs Env Var
```bash
# Collection name frequentemente hardcoded no codigo mas diferente no .env
grep -n 'get_collection("\|get_or_create_collection("' src/
grep "CHROMA_COLLECTION" config/.env
```
**Problema classico:** Codigo faz `client.get_collection("mentorbot_kb")` mas .env define `CHROMA_COLLECTION=cyberbot_kb`. Chroma pode ter 3 collections (mentorbot_kb, cyberbot_kb, cyberbot_test) — bot busca na vazia.

**Fix:** Usar `os.getenv("CHROMA_COLLECTION", "mentorbot_kb")` no lugar de string hardcoded.

### redis_session.py — Dead Code ou Configuracao Paralela?
```bash
# Verificar se redis_session.py e importado em algum lugar
grep -r "redis_session" src/ --include="*.py"
grep -r "from src.shared" src/ --include="*.py"
```
**Problema:** Se `src/shared/redis_session.py` conecta Redis SEM password (apenas `host=REDIS_HOST, port=REDIS_PORT, decode_responses=True`), ele vai falhar com Redis autenticado. Se nao e importado em lugar nenhum, e dead code que causaria problema se alguém o ativasse.

### Redis Authentication — Masked Password Recovery
**CRÍTICO:** Sintoma: `AuthenticationError: invalid username-password pair` nos logs, mas o `.env` mostra `REDIS_PASSWORD=***` (parece já configurado).

O dotenv **mascara valores** que contêm palavras como `PASSWORD`, `KEY`, `SECRET` na exibição (não no arquivo). O valor real pode ser diferente do mostrado.

**Diagnóstico:**
```bash
# 1. Ver o valor REAL no arquivo (não o mascarado)
grep "REDIS_PASSWORD" config/.env

# 2. Testar conexão Redis com o valor do .env
python -c "
import os; from dotenv import load_dotenv; load_dotenv('config/.env')
import redis
pw = os.environ.get('REDIS_PASSWORD', '')
r = redis.Redis(host='localhost', port=6379, db=1, decode_responses=True, password=pw or None, socket_connect_timeout=3)
r.ping()
"

# 3. Se falhar → recuperar senha REAL do Docker
docker inspect <redis_container_name> | grep -A2 requirepass
# Ex: docker inspect mentoria-redis | grep -A2 requirepass

# 4. Atualizar .env com a senha real (copiar do --requirepass do inspect)
```

**Causa raiz:** O valor `***` no .env não é placeholder — é a exibição mascarada do dotenv. O valor real no arquivo pode ter sido overwrite ou corrompido. Sempre usar `docker inspect` como fonte da verdade para senhas Docker.

### Redis vs REDIS_URL
```bash
# bot.py usa REDIS_HOST/PORT/DB individuais ou REDIS_URL?
grep "REDIS_URL\|REDIS_HOST\|REDIS_DB" src/bot.py
```
**Problema:** Se usa `REDIS_HOST/PORT/DB` mas .env só define `REDIS_URL`, DB separation não funciona.

### Redis Key Prefix (Multi-instância) — BOT_INSTANCE_ID
```bash
grep -n '"mentorbot:' src/bot.py
```
**Problema:** 25+ ocorrências de `"mentorbot:"` hardcoded — duas instâncias colidem no Redis/ChromaDB.

**Fix aplicado (2026-05-02):** Todas as Redis keys agora usam `{BOT_INSTANCE_ID}` como prefixo:
```
{BOT_INSTANCE_ID}:allowed_users
{BOT_INSTANCE_ID}:admin_users
{BOT_INSTANCE_ID}:personas
{BOT_INSTANCE_ID}:config
{BOT_INSTANCE_ID}:in_progress
{BOT_INSTANCE_ID}:ratelimit:{chat_id}
{BOT_INSTANCE_ID}:access_requests:{chat_id}
```

**Variáveis de ambiente para multi-instância:**
```bash
BOT_INSTANCE_ID=mentorbot    # prefixo único para Redis keys
BOT_NAME=MentorBot           # nome exibido ao usuário
CHROMA_COLLECTION=mentorbot_kb  # coleção ChromaDB (pode ser compartilhada ou não)
LOG_FILE=/tmp/${BOT_INSTANCE_ID}.log  # log separado por instância
```

**Setup.py** agora inclui seção "Instancia" com esses 3 campos.

### Bot Name Hardcoded
```bash
grep -c "MentorBot" src/bot.py
```
**Problema:** `"MentorBot"` em 25 lugares — sem `BOT_NAME` configurável.

**Fix aplicado:** Logger agora usa `logging.getLogger(BOT_NAME)`. As personas ainda usam defaults no dict, mas o welcome message usa config do Redis (setado em runtime).

### Campo Migration pares→tema
```bash
# Verificar se _send_profile_menu tem fallback para 'pares'
grep -A3 "def _send_profile_menu" src/bot.py | grep "pares"
```
**Problema:** `_build_user_summary` tem fallback, `_send_profile_menu` não.

**Fix aplicado:** `_send_profile_menu` agora usa `user.get("tema") or user.get("pares") or "não definido"`.

### LOG_FILE Hardcoded
```bash
grep "mentorbot.log" src/bot.py
```
**Problema:** Duas instâncias writing no mesmo `/tmp/mentorbot.log`.

**Fix aplicado:** `LOG_FILE = os.getenv("LOG_FILE", f"/tmp/{BOT_INSTANCE_ID}.log")`

### .env Duplicado
```bash
ls .env.example config/.env.example
```
**Problema:** Dois templates com valores inconsistentes.

**Fix aplicado (2026-05-02):** `config/.env.example` removido. Apenas `.env.example` na raiz existe.

### SESSION_TTL Default Mismatch
```bash
grep "SESSION_TTL" config/.env.example src/bot.py
```
**Problema:** .env.example=86400, bot.py default=1800.

**Fix aplicado:** bot.py agora usa `"86400"` como default (alinhou com .env.example).

### Scripts de Lifecycle — start.sh, stop.sh, restart.sh, health.sh
```bash
ls scripts/
```
**Problema:** Não existiam scripts de start/stop/restart/health.

**Fix aplicado (2026-05-02):** Scripts em `scripts/`:
- `start.sh [INSTANCE_ID]` — valida .env, venv, vars obrigatórias, inicia com nohup
- `stop.sh [INSTANCE_ID]` — para via PID file `/tmp/{INSTANCE_ID}.pid`
- `restart.sh [INSTANCE_ID]` — stop + start
- `health.sh [INSTANCE_ID]` — verifica processo, Redis, ChromaDB, log por errors

**Uso:**
```bash
./scripts/start.sh cyberbot   # Instance cyberbot
./scripts/stop.sh cyberbot    # Para só esse
./scripts/health.sh           # Health completo
```

### Bot Name Hardcoded
```bash
grep -c "MentorBot" src/bot.py
```
**Problema:** `"MentorBot"` em 25 lugares — sem `BOT_NAME` configurável.

**Fix aplicado:** Logger usa `BOT_NAME` dinamicamente. Personas continuam com defaults hardcoded no dict (para retrocompatibilidade), mas o config运行时 é dinâmico.

### Logger Name
```bash
grep 'logging.getLogger' src/bot.py
```
**Fix aplicado:** `log = logging.getLogger(BOT_NAME)` — nome do logger mengikuti `BOT_NAME` env var.

## Armadilhas Conhecidas

1. **Nao propor refatoracao se o fluxo atual funciona** — o usuario pediu validacao, nao rewrites
2. **Nao inventar problemas teoricos** — basear tudo no que esta no codigo
3. **Nao propor melhorias desnecessarias** — se nao bloqueava deploy, e observacao
4. **Sempre ler os arquivos antes de patch** — verificar estado atual antes de modificar
5. **Verificar git status antes de commitar** — confirmar o que realmente mudou
6. **Subagent filesystem dependency** — antes de delegar tarefas a subagentes via `delegate_task`, verificar se `/tmp/openclaw-backup` existe. Se nao existir, fazer o trabalho diretamente com `execute_code` em vez de delegar. Subagentes nesta VM dependem desse directorio para operaes de arquivo.
7. **Redis DB mismatch** — SE o codigo usa `os.getenv("REDIS_DB", "0")` como default, pode conectar no DB errado se o .env nao for lido a tempo (ex: load_dotenv depois de imports). Sempre verificar o valor atual no runtime.

## Pos-Deploy Checklist (para lembrar)

Apos o deploy:

```bash
# 1. Verificar health
kill -USR1 <pid> && cat /tmp/mentorbot_health.json

# 2. Verificar ChromaDB collection
curl -s http://localhost:8000/api/v2/collections | python -m json.tool | grep name

# 3. Teste de onboarding (usuario novo)
# Enviar /start no Telegram e verificar fluxo

# 4. Verificar logs
tail -f /tmp/mentorbot.log | python -m json.tool
```
