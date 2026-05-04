---
name: mentor-bot-coordinator
description: "Coordena projeto Bot Telegram com contexto vetorial (RAG + LLM). Trigger: construir bot consultivo Python com ChromaDB + Long Polling — sem webhooks, sem ngrok."
---

# MentorBot Coordinator — Hermes

> Coordena o projeto Bot Mentoria do conception à validação e oltre.
> **Stack validada:** Python 3.11 · Long Polling (getUpdates) · ChromaDB 0.6.3 REST · Redis Session · MiniMax (OpenAI-compatible)

## Project Root
`/home/felipe_silva_sorocaba/teste-bot-mentoria/`

## ⚠️ PAIN LESSONS — ERRORS RESOLVIDOS

| Erro | Causa | Fix |
|------|-------|-----|
| `Unsupported Chroma API implementation rest` | `chromadb.Client(Settings(chroma_api_impl="rest", ...))` — API antiga | Usar `chromadb.HttpClient(host, port)` |
| Think block `<think>` aparecendo na resposta | MiniMax/Claude inclui reasoning blocks no output | Função `_strip_think_tags()` em `bot.py` — 5-pass regex. **PITFALL: não use `<\/think>` (barra escapada) — o modelo envia `</think>` sem barra.** Passes: non-greedy `<think>[^<]*`, greedy, leading whitespace, orphan tags, full strip final. ```python\ndef _strip_think_tags(text):\n    for _ in range(5):\n        new_text = re.sub(r'<think>[^<]*', '', text, flags=re.IGNORECASE)\n        new_text = re.sub(r'</?think[^>]*>', '', new_text, flags=re.IGNORECASE)\n        new_text = re.sub(r'<think\\s', '', new_text, flags=re.IGNORECASE)\n        if new_text == text:\n            break\n        text = new_text\n    return re.sub(r'<[^>]+>', '', text)\n``` Aplicar SEMPRE antes de sendMessage |
| Tables markdown aparecendo cruas | Telegram não suporta markdown tables | Função `_markdown_table_to_text()` em `bot.py` — converte `\| col \|` em `📋 Header\n• Row` |
| "Cortex Bot" como nome do assistente | System prompt não personalizado | Renomear para "MentorBot — assistente pessoal de mentoria" em `bot.py` e `consumer.py` |
| System prompt diz "Cortex Bot" | System prompt genérico do template não foi personalizado | Renomear para "MentorBot — assistente pessoal de mentoria" em `bot.py` e `consumer.py` |
| System prompt restritivo | Prompt antigo dizia "Não execute código, não tome ações" | Remover restrições — bot de mentoria deve ser consultivo |
| Hallucination: bot dice que consegue criar arquivos | Bot sem tools, modelo inventando | Injetar `_ANTI_HALLUCINATION` no system prompt: "Você é UM ASSISTENTE DE TEXTO. Não gere código, não execute comandos..." |
| `Bad Request: chat not found` | Chat ID errado ou bot sem acesso ao privado | Usar `getUpdates` para descobrir chat_id real |
| `list_collections()` retorna só nomes | API mudou no ChromaDB 0.6 | Usar `client.list_collections()` retorna `list[str]`, não objetos |
| `collection 'mentoria' does not exist` | Collection criada como `cortex_kb`, não `mentoria` | Verificar collection real com `list_collections()` |
| `HTTP 400 Bad Request` no LLM | Modelo errado (`minimax-01`) ou base_url errada | Modelo correto: `minimax-m2.7` (não `minimax-01`). Base URL: `https://api.minimax.io/v1` (não `.chat`) |
| API key expira / 401 | Credenciais revogadas | Usar `~/.hermes/auth.json` → `credential_pool.minimax[0].access_token` como fallback — mesmo token do Hermes |
| ngrok install fails / disco 99% | URLs quebradas + disco cheio | **Long polling** — não precisa de ngrok nem URL pública |
| `409 Conflict: terminated by other getUpdates request` | Outro processo usando mesmo token (mesmo sem ser o bot) | 1) `pgrep -a -f "src.bot\|mentorbot\|8645482714"` 2) `kill -9 PID` 3) `sleep 1` 4) Reiniciar bot |
**Pipeline de resposta (bot.py `send_message`):**
```
LLM response
  ↓
1. re.sub(<think>[\s\S]*?
</think>, ...)   — remove think blocks
2. Code blocks → <pre>                   — Telegram HTML
3. Inline code → <code>                  — Telegram HTML
4. **bold** → <b>, *italic* → <i>, etc.
5. Horizontal rules (---) → removed
6. Blockquotes (> ) → removed
7. Headers (###) → <b>
8. _markdown_table_to_text()             — tables → 📋 Header / • Row
9. Truncate >4096 chars
10. sendMessage(parse_mode="HTML")
```

**Telegram suporta:** `<b>`, `<i>`, `<u>`, `<s>`, `<code>`, `<pre>`, `<a>`
**Telegram NÃO suporta:** tables, `<blockquote>`, horizontal rules, listas markdown nativas

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MentorBot Architecture                             │
│                     (single-process long polling)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Telegram Long Polling                                                      │
│  getUpdates(offset) ──→ process_update()                                   │
│                              │                                              │
│         ┌───────────────────┼──────────────────────┐                     │
│         │                    │                      │                       │
│    /configurar ──→ _handle_command()     /usar persona ──→ _set_bot_config()│
│    /reset ──────→ clear session         nova_sessao ──→ welcome message     │
│         │                                               │                    │
│         ▼                                               ▼                    │
│  ┌─────────────────┐              ┌──────────────────────────────┐        │
│  │ Redis           │              │ Redis: mentorbot:config       │        │
│  │ mentorbot:      │              │  nome, tagline, tom,         │        │
│  │  in_progress    │              │  persona, saudacao          │        │
│  │  (dedup)        │              │  mentorbot:personas (JSON)  │        │
│  └─────────────────┘              │  sess:initialized:{chat}    │        │
│                                    └──────────────────────────────┘        │
│                              │                                              │
│         ┌────────────────────┼──────────────────────┐                      │
│         ▼                    ▼                      ▼                       │
│  RAG query            build_messages()        save_message()               │
│  (ChromaDB)          + _build_system_prompt   (Redis sess:)                │
│         │             (dinâmico por config)                                   │
│         ▼                                                                    │
│  MiniMax /v1/chat/completions                                               │
│         │                                                                    │
│         ▼                                                                    │
│  sendMessage(parse_mode="HTML")                                              │
│  + _clean_text() → 「code」guillemets, headers→emoji, tables→text         │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Novos componentes (2026-04-30):**
- `_PERSONAS` dict: 4 personas (mentor, analista, programador, conversacional)
- `_ANTI_HALLUCINATION`: regra anti-invenção no system prompt
- `mentorbot:config` (Redis hash): nome, tagline, tom, persona, saudacao
- `mentorbot:personas` (Redis hash): JSON dos 4 personas predefinidos
- `sess:initialized:{chat_id}` (Redis key c/ TTL): flag de nova sessão
- `_is_new_session()`: detecta primeira msg de cada chat
- `_get_welcome_message()`: gera saudação com {nome} e {tagline}
- `_handle_command()`: processa /configurar, /usar persona, /reset
- `threading.Event` (não itertools.count): typing pokes que param corretamente
- Redis Set `mentorbot:in_progress`: deduplicação por chat_id

**Inline Keyboards — Menus Interativos (2026-04-30):**
- `send_reply_markup()`: envia mensagem com `reply_markup` (inline_keyboard JSON)
- `answer_callback()`: responde `callback_query` (tira "loading..." do Telegram)
- `process_update()`: detecta `callback_query` em `allowed_updates: ["message", "callback_query"]`
- `_handle_callback()`: dispatcher de callbacks — navegação + ações
- Menus interativos: `_send_main_menu()`, `_send_persona_menu()`, `_send_tom_menu()`, `_send_level_menu()`, `_send_pares_menu()`, `_send_profile_menu()`
- Pares: toggle (clica adiciona, clica remove — `cfg_pares_0` a `cfg_pares_5`)
- Helper `_rm(rows)` + `_btn(text, callback)` + `_row(*btns)`: builders de markup
- Skill dedicada: `mentorbot-inline-keyboards` (telegram/)

**Sistema de Memória de Usuário (2026-04-30):**
- `user:{chat_id}` (Redis hash): nome, level, pares, objetivo, ultima_sessao, criado_em
- `load_user(chat_id)`: carrega perfil do usuário
- `save_user_field(chat_id, field, value)`: salva campo individual
- `is_new_user(chat_id)`: detecta se é primeiro contato
- `_build_user_context(chat_id)`: injeta contexto do usuário no system prompt
- `_send_user_profile(chat_id, msg_id)`: mostra painel de perfil
- Comandos novos: `/perfil`, `/perfil nome:`, `/perfil level:`, `/perfil pares:`, `/perfil objetivo:`
- Auto-detecção de nome via Telegram `first_name` na primeira mensagem
- Tom ICT disponível: `"ict"` — "direto, disciplinado, educacional. Foco em risk management, edge e execução."

## ⚠️ PITFALLS NOVOS (2026-05-02)

### 1. Bot rodando no path errado
**Sintoma:** Erro `ModuleNotFoundError: No module named 'redis'` mesmo com venv.
**Causa:** Processo do bot em `/teste-bot-mentoria/` (path de outro deploy) rodando junto com o correto em `/cyberbot/`.
**Diagnóstico:**
```bash
ps aux | grep src.bot
# Se ver TWO processos com paths diferentes → KILL o errado
```
**Fix:**
```bash
# Identificar PIDs
ps aux | grep src.bot
# Matar o errado
kill -9 <PID_errado>
# Reiniciar com venv correto
/home/felipe_silva_sorocaba/cyberbot/venv/bin/python -m src.bot
```

### 2. Persona CyberSec mas bot responde como MentorBot
**Sintoma:** Bot usa greeting "MentorBot" mesmo após `HSET mentorbot:config persona cyberbot`.
**Causa:** `mentorbot:config` é um **Redis HASH**, não string. Campo `persona` na hash aponta pra key em `mentorbot:personas`.
**Diagnóstico:**
```bash
# Ver tipo da key
docker exec mentoria-redis redis-cli -a <PASS> -n 1 TYPE mentorbot:config
# hash → é hash, não string

# Ver conteúdo da hash
docker exec mentoria-redis redis-cli -a <PASS> -n 1 HGETALL mentorbot:config

# Ver todas as personas
docker exec mentoria-redis redis-cli -a <PASS> -n 1 HGETALL mentorbot:personas
```
**Fix — mudar persona da hash:**
```python
import redis
r = redis.Redis(host='localhost', port=6379, db=1, decode_responses=True,
                password='eCdaqMIf-QicgF1RnGjX1FXjW76bZgReMQgpOwN3jfA')
r.hset('mentorbot:config', 'persona', 'cyberbot')  # HSET não SET
```
**NOTA:** `mentorbot:personas` também é hash — cada persona é JSON-encoded.

### 3. ingest.py escreve em collection errada
**Sintoma:** RAG query funciona (retorna chunks) mas bot responde sem conteúdo de Cybersecurity.
**Causa:** `ingest.py` usa `COLLECTION_NAME = "mentorbot_kb"` mas o bot faz query em `cyberbot_kb`. Os chunks foram ingeridos na collection errada.
**Verificar:**
```python
import chromadb
client = chromadb.HttpClient(host='localhost', port=8000)
for coll in client.list_collections():
    c = client.get_collection(coll['name'])
    print(f"{coll['name']}: {c.count()} chunks")
```
**Fix — corrigir COLLECTION_NAME no ingest.py:**
```python
# Em src/rag/ingest.py
COLLECTION_NAME = "cyberbot_kb"  # deve ser = bot.py CHROMA_COLLECTION
```

---

### 6. Truncation ≠ Chunking (BUG CRÍTICO — silencioso)
**Sintoma:** Arquivo .md de 20.000 chars tem só os primeiros 8.000 ingeridos. ChromaDB indexa 1 vetor por arquivo — o oposto do que RAG deve fazer.
**Causa:** `content[:8000]` em ingest.py é **TRUNCAÇÃO**, não chunking. Chunking real divide por headers Markdown e respeita fronteiras naturais.
**Fix:** Substituir `content[:MAX_CHARS]` por `chunk_document()` que usa `re.split(r'\n(?=#{1,3} )', content)` para dividir por headers MD.
**Verificação após fix:**
```python
# Antes: 1 chunk por arquivo
# Depois: N chunks por arquivo (1 por seção ou por max_tokens)
#KB re-ingerida deve ter count >> número de arquivos
coll.count()  # >> número de .md files
```

---

### 7. Distance Threshold = 0 (sem filtro)
**Sintoma:** Queries genéricas ("qual seu nome?") ainda injetam chunks irrelevantes no prompt.
**Causa:** `RAG_MAX_DISTANCE` default era `1.35` ou inexistente — não filtrava ruído.
**Tabela de referência (cosseno, all-MiniLM-L6-v2):**
| Distance | Interpretação | Ação |
|----------|--------------|------|
| < 0.4 | Muito relevante | ✅ usar |
| 0.4–0.7 | Relevante | ✅ usar |
| 0.7–0.9 | Ruído provável | ⚠️ filtrar |
| > 0.9 | Descartar | ❌ rejeitar |
**Fix:** `RAG_DISTANCE_THRESHOLD=0.8` em `.env` + filtro em `worker/rag.py`:
```python
RAG_MAX_DISTANCE = float(os.getenv("RAG_MAX_DISTANCE", "0.8"))  # era 1.35
```
**Verificação:**
```python
coll.query(query_texts=["XSS filter bypass"], n_results=5)
# Todos os resultados devem ter distance < 0.8
```

---

### 8. Context truncado a 500-600 chars
**Sintoma:** LLM recebe contexto RAG incompleto — respostas superficiais.
**Causa:** `r['content'][:500]` ou `[:600]` em `llm_router.py` / `bot.py` trunca chunks grandes.
**Fix:** Aumentar para `[:1200]` ou sem limite se chunk size é controlado:
```python
# bot.py — injeção no system prompt
f"[Doc: {r['doc_id']}]\n{r['content'][:1500]}"
# llm_router.py — histórico
h["text"][:800]  # era [:500]
```

---

### 9. Git merge sobrescreve fixes locais
**Sintoma:** Após `git pull origin branch`, fix do `llm_router.py` ou `worker/rag.py` desapareceu.
**Causa:** Remote tinha trabalho avançado que sobrescreveu os arquivos locais.
**Diagnóstico:**
```bash
# Verificar se marcador do fix ainda existe
grep "chunk_document\|RAG_DISTANCE_THRESHOLD\|content\[:1200\]" src/rag/ingest.py src/worker/rag.py src/llm_router.py

# Ver diff vs remote
git fetch origin
git log --oneline HEAD..origin/HEAD  # commits remote à frente
```
**Fix:** Após merge/pull, **SEMPRE** verificar cada arquivo crítico:
- `src/rag/ingest.py` → `chunk_document` presente?
- `src/worker/rag.py` → `RAG_MAX_DISTANCE` ou `RAG_DISTANCE_THRESHOLD` presente?
- `src/llm_router.py` → `content[:1200]` ou similar?
Se faltou: re-aplicar o fix manualmente, depois commitar.
**Ordem segura:**
```bash
git stash               # salvar work-in-progress (se houver)
git pull --no-rebase origin branch
# verificar arquivos
git diff HEAD~1 --stat  # ver o que mudou no merge
git stash pop           # restaurar WIP
```

---

### 10. Metadata insuficiente para citação
**Sintoma:** LLM não consegue citar fonte com precisão — retorna "[Doc: chunk003]".
**Causa:** Metadata só tinha `source` (filename), sem `repo`, `category`, `heading`.
**Fix — schema de metadata:**
```python
batch_metadatas.append({
    "repo": repo,                    # ex: "swisskyrepo/PayloadsAllTheThings"
    "category": category,            # ex: "offensive"
    "heading": chunk["heading"][:200], # ex: "XSS Filter Evasion"
    "source": str(md_file.relative_to(repo_path)),  # ex: "XSS_Injection/README.md"
    "size": chunk["size"]
})
```
**No bot.py, injetar no context:**
```python
f"[{meta.get('category', 'KB')} — {meta.get('heading', meta.get('source', '?'))}]\n{chunk}"
```

---

## Multi-Bot Propagation (clones do mesmo repo)

Quando um fix RAG é aplicado em `cyberbot/` e precisa ser propagado para `teste-bot-mentoria/` (ou outros clones):

### Checklist por arquivo

| Arquivo | Fix necessário | Verificação |
|---------|---------------|-------------|
| `src/rag/ingest.py` | `chunk_document()`, metadata enriquecida | `grep chunk_document` |
| `src/worker/rag.py` | `RAG_MAX_DISTANCE` via env, `CHROMA_COLLECTION` via env | `grep RAG_MAX_DISTANCE` |
| `src/llm_router.py` | `content[:1200]`, history `[:800]` | `grep "content\[:1"` |
| `src/bot.py` | `CHROMA_COLLECTION` via env, distance filter | `grep mentorbot_kb` (deve estar zero) |
| `.env` | `CHROMA_COLLECTION`, `RAG_DISTANCE_THRESHOLD=0.8`, `RAG_CHUNK_SIZE=1000`, `RAG_CHUNK_OVERLAP=100` | vars presentes |

### Passos de propagação

```bash
# 1. Aplicar no repo de origem (cyberbot/)
# 2. Git push
git add src/ llm_router.py worker/rag.py .env
git commit -m "feat(RAG): chunking real + threshold 0.8"
git push origin audit-generic-v2

# 3. No clone (teste-bot-mentoria/):
git fetch origin
git merge origin/audit-generic-v2
# Se rejeitado (ahead): git pull --no-rebase ou merge manual

# 4. Verificar que todos os fixes sobreviveram
for f in src/rag/ingest.py src/worker/rag.py src/llm_router.py; do
    echo "=== $f ==="; grep -c "chunk_document\|RAG_MAX_DISTANCE\|content\[:1200\]" $f || echo "MISSING"
done

# 5. Restart do bot
pkill -f "src.bot" && sleep 1
venv/bin/python -m src.bot &
```

### ⚠️ .env NÃO é versionado (gitignored)
Fixes em `.env` precisam ser aplicados **manualmente** em cada deploy.-use `grep` to verify the fix is present.
```
**Verificar:**
```python
import chromadb
client = chromadb.HttpClient(host='localhost', port=8000)
for coll in client.list_collections():
    c = client.get_collection(coll['name'])
    print(f"{coll['name']}: {c.count()} chunks")
```
**Fix — corrigir COLLECTION_NAME no ingest.py:**
```python
# Em src/rag/ingest.py
COLLECTION_NAME = "cyberbot_kb"  # deve ser = bot.py CHROMA_COLLECTION
```

### 4. ChromaDB collection com ID diferente após restart
**Sintoma:** Query funciona no código mas `curl` ao collection ID antigo retorna "InvalidCollection".
**Causa:** ChromaDB recrea collections ao restart se não houver persistência.
**Verificar ID atual:**
```python
client = chromadb.HttpClient(host='localhost', port=8000)
for coll in client.list_collections():
    print(coll['name'], coll['id'])
```
**Fix:** Sempre resolver collection por **nome**, não por ID hardcoded.

### 5. Checkpoint de ingestão não é persistente no código
**Sintoma:** Ingestão HackTricks (~15K+ chunks) foi interrompida. Ao re-rodar, qual `START_IDX` usar?
**Causa:** `ingest_hacktricks_incremental.py` tem `START_IDX` hardcoded. Se o processo morrer, o próximo START_IDX só está no log.
**Verificar checkpoint:**
```bash
# Log do último batch
tail -5 ~/cyberbot-sec/logs/hacktricks_incremental.log
# Deve mostrar: "Next run should start from index: XXXXX"

# Checkpoint formal (se existir)
cat ~/cyberbot-sec/docs/RAG/CHECKPOINT_INGESTION.md
```
**Fix — confirmar checkpoint e reiniciar:**
```bash
# Identificar START_IDX do log
START_IDX=$(grep "Next run should start from index:" ~/cyberbot-sec/logs/hacktricks_incremental.log | tail -1 | grep -oP '\d+$')
echo "Próximo START_IDX: $START_IDX"

# Editar ingest_hacktricks_incremental.py → START_IDX = $START_IDX
sed -i "s/^START_IDX = .*/START_IDX = $START_IDX/" ~/cyberbot-sec/scripts/ingest_hacktricks_incremental.py

# Re-rodar
python3 ~/cyberbot-sec/scripts/ingest_hacktricks_incremental.py
```

---

**Pipeline de resposta (bot.py `send_message`):**
```
teste-bot-mentoria/
├── docker-compose.yml      # Redis 7 + ChromaDB 0.6.3 (127.0.0.1 only)
├── requirements.txt
├── config/.env             # TELEGRAM_BOT_TOKEN, REDIS_*, CHROMA_HOST
├── test_pipeline.py        # Testa Redis + ChromaDB + MiniMax (sem Telegram)
├── kb/ict/                 # KB ICT: 11 arquivos ~57KB (SMC, Kill Zones, Silver Bullet, etc.)
├── tools/                  # Manuais de ferramentas (ex: TradingView indicator)
└── src/
    ├── bot.py              # ✅ ARQUITETURA ATIVA — long polling single-process
    ├── rag/
    │   └── ingest.py       # Popula ChromaDB com kb/ict/
    ├── worker/
    │   ├── claude.py       # MiniMax OpenAI-compatible
    └── shared/
```

## Como Rodar

### 1. Subir serviços
```bash
cd ~/teste-bot-mentoria
docker compose up -d
```

### 2. Setup
```bash
cd ~/teste-bot-mentoria
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
```

### 3. Criar bot no Telegram
```
DM @BotFather → /newbot → escolher nome → copiar TOKEN
```

### 4. Configurar token
```bash
# Editar config/.env
TELEGRAM_BOT_TOKEN=123456789:ABCdef...   # token real do BotFather
```

### 5. Popular ChromaDB (opcional — já tem 3 docs)
```bash
source venv/bin/activate
python -m src.rag.ingest
```

### 6. Testar pipeline (sem Telegram)
```bash
source venv/bin/activate
python test_pipeline.py
```

### 7. Rodar bot
```bash
source venv/bin/activate
python -m src.bot  # USA venv, não python3 do sistema!
```

**⚠️ Erro `ModuleNotFoundError: No module named 'redis'`:** Significa que o bot foi iniciado com `python3` do sistema em vez do venv. Sempre usar `./venv/bin/python -m src.bot` ou `source venv/bin/activate && python -m src.bot`.

**Logs:** `background` output ou `tail -f` do processo background.

## Allowlist / Restrições de Acesso

Para permitir só users específicos, ver skill **`telegram-guard-rails`**:

```
TELEGRAM_ALLOWED_USERS=5556338087,8533341766
TELEGRAM_GROUP_ALLOWED_IDS=-1003872232486
```

Configurar no `.env` e validar no `process_update()` do bot.py.

## RAG Tuning (hotfix 2026-05-01)
- `RAG_MAX_RESULTS=6` — busca o dobro e filtra por distância
- `RAG_MAX_DISTANCE=1.35` — descarta chunks com distância > 1.35
- Em `config/.env` (não commited — secrets no .gitignore)
- Implementado em `src/worker/rag.py` via `RAG_MAX_DISTANCE` env var

## LLM Router (`src/llm_router.py`)
- 3 rotas: `NO_RAG` (saudação), `SIMPLE` (KB direta), `COMPLEX` (raciocínio)
- Modelo: `minimax-m2.7`, base_url: `api.minimax.io/v1`
- Variáveis: `ROUTER_ENABLED`, `LLM_MODEL_FAST`, `LLM_MODEL_STRONG`
- `config/.env` não versionado — aplicar changes manualmente no deploy
```bash
TELEGRAM_BOT_TOKEN=           # BotFather token (obrigatório)
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_DB_JOB=0
CHROMA_HOST=127.0.0.1:8000
SESSION_TTL=86400             # 24h — reseta a cada mensagem
MAX_CONTEXT_MESSAGES=30       # msgs mantidas em contexto
POLL_INTERVAL=1.0            # segundos entre polls (sem mensagem)
LOG_LEVEL=INFO

# ── LLM Parameters (expostos em 2026-04-30) ──────────────────────
LLM_TEMPERATURE=0.3          # Criatividade: 0.1=preciso, 0.9=genial
LLM_TOP_P=1.0                # Nucleus sampling (default 1.0 = disabled)
LLM_FREQUENCY_PENALTY=0.0    # Penalidade por repetição de tokens
LLM_PRESENCE_PENALTY=0.0     # Penalidade por temas já abordados
LLM_MAX_TOKENS=1024           # Limite de tokens na resposta

# ── RAG ──────────────────────────────────────────────────────────
RAG_MAX_RESULTS=3             # Quantos docs do vectorDB buscar

# ── Personality ───────────────────────────────────────────────────
# Tones: formal, amigavel, tecnico, conciso
BOT_TONE=amigavel
```

## Diagnóstico: Bot responde com persona errada

Quando o bot responder com nome/nome errado (ex: "ChatBot" em vez de "MentorBot"):

**Passo 0: Verificar se é BUG DE HANDLER DE COMANDO (RETORNO ANTECIPADO)**
Antes de qualquer diagnóstico — se um comando como `/start` não dispara onboarding, verifique se ele tem `return` ANTES do fluxo compartilhado de `process_update()`:
```python
# ❌ BUG: handler com return que ignora onboarding
if cmd == "/start":
    welcome = _get_welcome_message()
    bot.send_message(chat_id, welcome)
    return ok  # ← early return! onboarding nunca roda aqui

# ✅ CORRETO: handler /start integrado ao fluxo de onboarding
if cmd == "/start":
    r.delete(f"sess:{chat_id}")
    r.delete(f"sess:initialized:{chat_id}")
    if is_new_user(chat_id):
        first_name = from_data.get("first_name", "Aluno")
        save_user_field(chat_id, "nome", first_name)
        save_user_field(chat_id, "onboarding_step", "objetivo")
        # ... mensagem de onboarding
        return ok
    else:
        welcome = _get_welcome_message()
        bot.send_message(chat_id, welcome)
        return ok
```
O fluxo de onboarding fica em `process_update()` (após handlers de comando). Se um handler retorna antes (com `return True`), o código abaixo nunca executa — incluindo `_is_new_session()` e `_get_onboarding_step()`.

**Passo 1: Verificar Redis config**

Quando o bot responder com nome/nome errado (ex: "ChatBot" em vez de "MentorBot"):

**Passo 1: Verificar Redis config**
```python
import redis
r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)
cfg = r.hgetall('mentorbot:config')
print(cfg)
# Deve mostrar: persona=mentor, nome=MentorBot, tagline=Assistente ICT...
```

**Passo 2: Verificar perfis dos alunos (onboarding)**
```python
for uid in ['1078269852', '8772339542']:
    u = r.hgetall(f'user:{uid}')
    print(f'user:{uid}:', u)
```

**Passo 3: Se config errada, resetar**
```python
import redis
r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)
cfg = {
    'tom': 'ict',
    'saudacao': (
        'Olá! Sou o {nome}, {tagline}. '
        'Posso ajudar com setups ICT, risk management, '
        'análise de kill zones e Premium/Discount zones.\n\n'
        'Me conte: o que quer aprender hoje?'
    ),
    'nome': 'MentorBot',
    'tagline': 'Assistente ICT — Smart Money, Order Blocks, Kill Zones',
    'persona': 'mentor'
}
r.hset('mentorbot:config', mapping=cfg)
```

**Passo 4: Reiniciar bot**
```bash
pkill -f "src.bot" && sleep 1 && python -m src.bot &
```

**Causa raiz mais comum:** Config migrada/mudada manualmente no Redis sem atualizar todos os campos (persona + nome + tagline devem estar consistentes).

---

**Tones disponíveis:**
| Tone | Descrição |
|------|-----------|
| `amigavel` | Direto, amigável, prestativo (default) |
| `formal` | Linguagem culta, profissional |
| `tecnico` | Preciso, terminologia técnica, detalhado |
| `conciso` | Extremo: vá direto ao ponto |
| `ict` | Mentor ICT — direto, disciplinado, educacional. Foco em risk management, edge e execução. Não dê sinais, dê conhecimento. |

## Comandos do Bot (via Telegram)

```
/configurar                          → menu interativo (inline keyboards)
/configurar nome: Meu Bot            → muda nome do bot
/configurar tagline: Minha tagline   → muda tagline
/configurar tom: amigavel           → muda tom (formal|amigavel|tecnico|conciso)
/usar persona mentor                 → troca persona completa (mentor|analista|programador|conversacional)
/reset                              → limpa sessão e volta welcome
/perfil                              → ver perfil do usuário
/perfil nome: Felipe                 → salvar nome
/perfil level: intermediário         → salvar nível
/perfil pares: EURUSD, GBPJPY        → salvar pares
/perfil objetivo: Aprender ICT        → salvar objetivo
```

**Menu interativo `/configurar`** — botões inline para:
| Botão | Ação |
|-------|------|
| 👤 Meu Perfil | Ver perfil completo + editar |
| 🎭 Persona | ICT Mentor / Analista / DevBot / ChatBot |
| 🗣️ Tom | Formal / Amigável / Técnico / Conciso / ICT |
| 📊 Nível | 🌱 Iniciante / 🌿 Intermediário / 🌳 Avançado |
| 💱 Pares | Toggle: EURUSD / GBPJPY / USDJPY / NAS100 / US30 / XAUUSD |

**Personas disponíveis:**
| Persona | Nome | Tagline | Tom |
|---------|------|---------|-----|
| `mentor` | MentorBot | Seu assistente pessoal de mentoria | amigavel |
| `analista` | AnalistaBot | Especialista em análise de mercado | técnico |
| `programador` | DevBot | Assistente de desenvolvimento | técnico |
| `conversacional` | ChatBot | Companheiro de conversas | amigavel |

**Boas-vindas (nova sessão):** Quando `_is_new_session()` detecta primeira mensagem de um chat, o bot envia `_get_welcome_message()` (template com {nome} e {tagline} da persona ativa) ANTES de processar a mensagem do usuário.

O tone muda o system prompt injetado no contexto — todos os arquivos que chamam LLM (`bot.py`, `consumer.py`) usam `_build_system_prompt()` dinâmico (não mais `SYSTEM_PROMPT` hardcoded do `.env`).

**Valores validados:** SESSION_TTL=86400 (24h) + MAX_CONTEXT=30 msgs — session fluida como WhatsApp.

## Token Budget (context window management)

Para evitar que o contexto estoure a janela do modelo, o bot usa budget de caracteres:

```env
# Session
MAX_SESSION_CHARS=50000   # ≈ 12.5K tokens (1 token ≈ 4 chars)
```

**Como funciona (`_truncate_by_tokens()` em bot.py):**
1. Carrega últimas 30 mensagens do Redis
2. Separa system messages (mantidas) vs. conversation history
3. Remove mensagens mais antigas até caber no limite de chars
4. Reserva ~2000 chars para a pergunta atual do usuário

**Conversão:** 50.000 chars ≈ 12.500 tokens — espaço suficiente para conversas longas + RAG + resposta.

**Conversor de tabelas:** `_markdown_table_to_text()` é aplicado NA RESPOSTA antes de `send_message()` — não no histórico. Isso garante que o contexto mantém a formatação original mas o Telegram recebe texto legível.

## Collection ChromaDB
- **Nome correto:** `mentorbot_kb` (NÃO `cortex_kb` — esse é bug conhecido)
- **Embedding:** server-side (ChromaDB nativo — sem sentence-transformers)
- **Test retrieval:**
```python
from chromadb import HttpClient
c = HttpClient(host='127.0.0.1', port=8000)
coll = c.get_collection('mentorbot_kb')
print('Total docs:', coll.count())
coll.query(query_texts=['sua query'], n_results=3)
```

## Referência: Hermes vs MentorBot Clone
- Hermes: 15+ plataformas, 80 skills, 6 agentes, ~2000+ linhas só Telegram
- MentorBot clone: 1 plataforma, ~350 linhas bot.py, 1 processo, ~1h para MVP
- Velocidade: E2E <10s com RAG + MiniMax
