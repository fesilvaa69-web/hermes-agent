---
name: rag-kb-audit
description: RAG Knowledge Base health audit — verifica ChromaDB, diagnostica gaps entre arquivos-fonte e chunks-ingidos, valida isolamento multi-usuário e query retrieval.
tags: [rag, chromadb, knowledge-base, audit, health-check]
related_skills: [rag-populate-from-github]
---

# rag-kb-audit

> Audita health de uma RAG Knowledge Base — verifica completude, identifica gaps, valida queries e isolamento multi-usuário.

**Trigger:** "audite a KB RAG", "verifique o RAG", "diagnostique a knowledge base", "RAG está populado?", "verifique se o conteúdo foi ingerido", "validar multi-usuário do bot".

---

## WAVES Naming Convention (for ingest_rag.py)

The `scripts/ingest_rag.py` pipeline categorizes repos by **directory name**. Repos in `~/.rag_repos/cyberbot/` MUST follow the format:

```
CATEGORY__Owner-RepoName
```

Examples:
- `Cybersecurity__web-security-top10` → category=Cybersecurity
- `Hack-with-Github__Awesome-Hacking` → category=Hack-with-Github
- `PayloadsAllTheThings__payloads` → category=PayloadsAllTheThings
- `OWASP__CheatSheetSeries` → category=OWASP

Current repos in `~/.rag_repos/cyberbot/`:
```
EbookFoundation__free-programming-books
Hack-with-Github__Awesome-Hacking
OWASP__CheatSheetSeries
carlospolop__hacktricks
codecrafters-io__build-your-own-x
swisskyrepo__PayloadsAllTheThings
trimstray__the-book-of-secret-knowledge
```

**Rename repos before ingesting** if they don't match the format:
```bash
mv old-name "Owner__RepoName"
```

## Quando Usar

- Após popular ou expandir uma KB RAG (ChromaDB)
- Antes de colocar bot em produção
- Quandobot dá respostas incompletas ("não tenho essa informação")
- Após mover/adicicionar arquivos na pasta KB
- Para validar que arquivo em `tools/` ou outro dir fora do ingest pipeline foi corretamente incluído

---

## Workflow (4 fases)

### Fase 1 — Inventário de Arquivos-Fonte

```bash
# Listar todos os arquivos .md que DEVEM estar na KB
find kb/ -name "*.md" -o -name "*.txt" | sort
```

**Comparar com o que o ingest.py realmente lê** — dado que `ingest.py` tem `KB_DIR` hardcoded (ex: `kb/ict/`), arquivos fora desse diretório **NUNCA serão ingeridos**.

```bash
# Verificar KB_DIR do ingest
grep "KB_DIR\|kb/" src/rag/ingest.py
```

### Fase 2 — Verificação ChromaDB (count + retrieval)

```python
from chromadb import HttpClient

CHROMA_HOST = "localhost"
CHROMA_PORT = 8000
COLLECTION  = "mentorbot_kb"   # ATENÇÃO: rag.py usa mentorbot_kb (NÃO cortex_kb)

client = HttpClient(host=CHROMA_HOST, port=CHROMA_PORT)
coll   = client.get_collection(COLLECTION)

print("Total chunks:", coll.count())
results = coll.get(include=['documents','metadatas'])
for i, (doc, meta) in enumerate(zip(results['documents'], results['metadatas'])):
    print(f"  [{i}] {meta['source']} | {doc[:80]}...")
```

**Verificar count:**
- Count = 0 → ChromaDB server não está rodando ou collection não existe
- Count < número de arquivos .md → arquivos não ingeridos

### Fase 3 — Teste de Query Retrieval

```python
tests = [
    "silver bullet",
    "order block",
    "kill zone",
    "risk management",
    "fibonacci",
]
for q in tests:
    r = coll.query(query_texts=[q], n_results=2)
    print(f"\nQuery: '{q}'")
    for i, doc_id in enumerate(r['ids'][0]):
        doc = r['documents'][0][i][:80].replace('\n', ' ')
        print(f"  [{i}] {doc_id} | {doc}...")
```

**Se query retorna 0 resultados** → embedding quebrado ou KB vazia.
**Se query retorna docs irrelevantes** → embedding model não funciona bem para o domínio.

### Fase 4 — Diagnóstico de Gaps (Arquivos Não-Ingeridos)

```
Gaps comuns:
1. Arquivo existe em /tools/ mas KB_DIR do ingest.py é kb/ict/ → arquivo IGNORADO
2. Arquivo existe mas < 50 chars → ignorado pelo ingest (size filter)
3. ChromaDB count < número de arquivos → re-ingest necessário
```

**Fix para gap tipo 1 (arquivo fora da KB_DIR):**
```bash
# Mover arquivo para dentro da KB_DIR
mv tools/05_SOME_FILE.md kb/ict/NN_SOME_FILE.md

# Re-ingerir
python3 -m src.rag.ingest
```

**Verificar ChromaDB após fix:**
```python
coll.count()  # deve aumentar
```

---

## Validação Multi-Usuário (Redis)

Verificar que cada chat_id tem estado isolado:

```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

# Keys por usuário
keys = [k for k in r.keys() if k.startswith('user:') or k.startswith('sess:')]
for k in sorted(keys):
    print(k, r.type(k), r.hgetall(k) if ':' in k else r.get(k))
```

**Estado esperado por usuário:**
```
user:{chat_id}     → hash (nome, level, pares, objetivo)
sess:{chat_id}     → hash (messages)
sess:initialized:{chat_id} → string "1"
```

**Isolamento em processamento simultâneo:**
- `mentorbot:in_progress` é um SET — cada chat_id é item independente
- Dois usuários podem ser processados simultaneamente sem conflito

---

## Verificação de Onboarding Novo Usuário

Para simular novo usuário (testar welcome + perfil):
```
1. /reset ou /start → limpa sessão e dispara onboarding
2. is_new_user(chat_id) retorna True na primeira interação
3. welcome personalizado com base na persona configurada
4. /perfil → mostra configuração atual do usuário
5. /configurar → menus inline para alterar persona/tom/level
```

**Verificar flags Redis:**
```python
r.exists(f"sess:initialized:{novo_chat_id}")   # deve ser 0 antes do /start
r.exists(f"user:{novo_chat_id}")              # deve ser 0 antes do /start
```

---

## Output de Auditoria (template)

```
# 📊 RAG KB Audit — {data}

## Coleção: {COLLECTION}
| Métrica | Valor |
|---------|-------|
| Total chunks | {count} |
| Arquivos-fonte | {n_arquivos} |
| ChromaDB server | {online/offline} |

## Query Retrieval Tests
| Query | Resultados | Docs retornados |
|-------|-----------|-----------------|
| silver bullet | ✅/❌ | doc1, doc2 |
| order block | ✅/❌ | doc1 |

## Gaps Encontrados
| Tipo | Arquivo | Ação |
|------|---------|------|
| fora KB_DIR | tools/X.md | mover para kb/ict/ |

## Multi-Usuário
| Check | Status |
|-------|--------|
| Redis keys por chat_id | ✅ |
| Isolamento confirmado | ✅ |
```

---

## Armadilhas

| Situação | Solução |
|----------|---------|
| ChromaDB count = 0 | Verificar se server está rodando (`ps aux \| grep chroma`) |
| Arquivo existe mas não ingere | Verificar KB_DIR do ingest.py — pode ser restrito a subdir |
| Query retorna vazio | ChromaDB server pode estar com collection errada ou embedding quebrada |
| `tools/` arquivos ignorados | ingest.py só lê de `KB_DIR` — mover para dentro da KB |
| Resultados com distância > 1.3 | **Embedding model mismatch** — ver Seção 5 |
| Query retorna docs irrelevantes | **Embedding model mismatch** — ver Seção 5 |
| Bot não responde com conteúdo da KB mas RAG count > 0 | **Collection mismatch** — ingest.py usa collection diferente do bot.py. Verificar COLLECTION_NAME no ingest.py vs CHROMA_COLLECTION no bot.py. Ambos devem apontar para a mesma collection. |
| Redis retorna "WRONGTYPE Operation against key" | Key existe mas é tipo diferente (ex: hash vs string). Usar `TYPE key` para diagnosticar. Se config é hash, usar `HGET key campo` não `GET key`. |

---

## Seção 5 — Embedding Model Mismatch (Causa Silenciosa)

### Sintomas

- ChromaDB tem `count() > 0` (KB populada)
- Query retorna documentos mas com **distância > 1.0** (valores típicos: 0.99, 1.2, 1.48)
- Distância alta = sem similaridade real — contexto injetado no prompt é ruído
- Bot responde com informação genérica em vez de conteúdo da KB

### Causa Raiz

ChromaDB reutiliza silenciosamente o embedding model de collections existentes. Se o ChromaDB server tem uma collection antiga com modelo `X` e o novo deploy ingere com modelo `Y`, as queries usam `X` mas os documentos foram indexados com `Y` → vetores em espaços diferentes.

**Nunca é óbvio** — o ChromaDB não dá erro, retorna documentos "válidos" com distância alta.

### Verificação (no container ou host)

```python
# 1. Ver qual embedding model o ChromaDB usa na criação da collection
# (collection.metadata mostra o que foi gravado)
coll = client.get_collection("mentorbot_kb")
print(coll.metadata)  # deve ter {"embedding_model": "sentence-transformers/all-MiniLM-L6-v2"}

# 2. Se metadata vazia ou diferente → MISMATCH
# 3. Testar distância direta
results = coll.query(query_texts=["termo que deve existir na KB"], n_results=3)
print(results["distances"][0])  # > 1.0 = problema
```

### Fix — Código (2 arquivos)

**`src/rag/ingest.py`** — especificar embedding fixo na criação da collection:

```python
from chromadb.utils import embedding_functions

EMBEDDING_MODEL = "sentence-transformers/all-MiniLM-L6-v2"

ef = embedding_functions.DefaultEmbeddingFunction()
client.delete_collection(COLLECTION_NAME)  # wipe antes de recriar
collection = client.create_collection(
    name=COLLECTION_NAME,
    embedding_function=ef,
    metadata={"embedding_model": EMBEDDING_MODEL},
)
```

**`src/bot.py` e `src/worker/rag.py`** — filtrar por distância na query:

```python
MIN_DISTANCE = 1.3  # distância > 1.3 = baixa relevância

filtered = [
    {"doc_id": ids[i], "content": docs[i], "distance": dists[i]}
    for i in range(len(docs))
    if dists[i] < MIN_DISTANCE
]
if not filtered:
    log.warning("RAG: todos resultados com distância > 1.3 — retornando vazio")
```

### Checklist Anti-Mismatch

- [ ] `ingest.py` cria collection com `embedding_function=` explícito
- [ ] `collection.metadata` tem `embedding_model` preenchido
- [ ] `bot.py` e `worker/rag.py` têm filtro de distância (≥1.3)
- [ ] Logs mostram warning quando RAG retorna vazio por distância
- [ ] Após fix: reingest da KB com wipe (`delete_collection` antes de criar)

---

## System Python Fallback (for ingest when venv pip fails)

When disk space is critically low (≥97%) and `pip install` fails, `/usr/bin/python3` (system Python 3.10) already has:
- `sentence-transformers` installed
- `all-MiniLM-L6-v2` model cached (~88MB)

Use it directly:
```bash
cd /home/felipe_silva_sorocaba/cyberbot
REPO_BASE=~/.rag_repos/cyberbot CHROMA_COLLECTION=cyberbot_kb \
  /usr/bin/python3 scripts/ingest_rag.py --repo owner/repo
```

Or run all repos:
```bash
REPO_BASE=~/.rag_repos/cyberbot CHROMA_COLLECTION=cyberbot_kb \
  /usr/bin/python3 scripts/ingest_rag.py --all
```

This bypasses the need for venv pip install when disk is full.

---

## Checklist Pré-Deploy

- [ ] ChromaDB count > 0
- [ ] Query retrieval retorna resultados relevantes
- [ ] Todos arquivos .md da KB_DIR estão ingeridos
- [ ] Não há arquivos fora da KB_DIR que deveriam estar dentro
- [ ] Redis keys `user:*` e `sess:*` isoladas por chat_id
- [ ] Onboarding novo usuário (/start) funciona
- [ ] Git push realizado após mudanças na KB
