---
name: rag-populate-from-github
description: "Popula KB RAG com conteúdo de repositórios GitHub públicos — pesquisa massiva, extração, deduplicação e ingestão. Trigger: popule o RAG, expanda a KB com repos GitHub."
tags: [rag, chromadb, github, kb, knowledge-base, populate]
---

# rag-populate-from-github

> Popula uma knowledge base RAG (ChromaDB) com conteúdo de repositórios GitHub públicos.

**⚠️ Overlap note:** The `scripts/ingest_rag.py` WAVES pipeline (in cyberbot repo) is a newer automated approach that pulls from GitHub directly with markdown-aware chunking. This skill covers the older manual KB building approach. See `rag-kb-audit` for WAVES details and system Python fallback.

---

## Quando Usar

- Construir ou expandir KB vetorial de um bot consultivo
- Quando o usuário diz "popule o RAG", "busque repos GitHub sobre X"
- Quando a KB está vazia ou precisa de conteúdo massivo de fontes externas
- Antes de colocar um bot em produção com contexto de domínio específico

---

## Workflow (6 fases)

### Fase 1 — Planejamento da Estrutura KB

Antes de buscar, definir:
- **Diretório KB local** (ex: `kb/ict/`)
- **Collection ChromaDB** (ex: `mentorbot_kb`)
- **Tópicos/conceitos a cobrir** (lista prévia)
- **Constrangimentos:** não usar conteúdo da VM, só fontes externas

### Fase 2 — Pesquisa GitHub com Subagentes

Usar `delegate_task` com 2-3 subagentes em paralelo:
- Subagente A: busca por termo 1
- Subagente B: busca por termo 2
- Subagente C: busca por termo 3

**Comando para buscar repos (neste ambiente `gh search` não existe — usar `gh api`):**
```bash
gh api search/repositories -f q="TERMO" -f sort=stars -f order=desc --jq '.items[] | "\(.stargazers_count)⭐ | \(.full_name) | \(.description)"'
```

**Clonar repos prometedores:**
```bash
git clone --depth 1 https://github.com/OWNER/REPO.git /tmp/REPO-NAME
```

### Fase 3 — Extração e Write de Arquivos .md

Para cada repo clonado:
1. Ler README.md completo
2. Identificar conceitos únicos vs duplicados
3. Criar arquivo `.md` por conceito no diretório KB local
4. Usar `write_file` tool (não terminal echo)

**Nomenclatura:** `NN_CONCEITO_PRINCIPAL.md`

**Tamanho mínimo:** 1KB por arquivo

### Fase 4 — Deduplicação via Subagente

Usar subagente para comparar conceitos:
- Ler todos os arquivos KB
- Identificar conceitos duplicados (3+ arquivos)
- Identificar conceitos únicos (1 arquivo só — alto valor)
- Identificar gaps (conceitos importantes ausentes)

Output:
```
| Concept | Files | Duplicate? |
```

Arquivos < 3KB são candidatos a expansão.

### Fase 5 — Ingestão ChromaDB

```bash
cd /repo && python3 -m src.rag.ingest
```

**Verificar:**
```python
import chromadb
client = chromadb.HttpClient(host='localhost', port=8000)
coll = client.get_collection('NOME_COLLECTION')
print('Total docs:', coll.count())
```

**Se count = 0:** Verificar se ChromaDB server está rodando (`ps aux | grep chroma`).

### Fase 6 — Commit e Push

```bash
git add kb/ && git commit -m "docs: expand KB with X new concepts from GitHub"
git push origin master
```

---

## Armadilhas Comuns

| Situação | Solução |
|----------|---------|
| `gh search` not found | Usar `gh api search/repositories` |
| ChromaDB count = 0 após ingest | Verificar server rodando |
| Arquivo < 1KB | Expandir ou descartar |
| Conceito em 3+ arquivos | Manter só no mais completo |
| Só 1 batch no ingest log | Contar docs no ChromaDB diretamente |
| Arquivo em `tools/` não é ingerido | ingest.py lê só de `KB_DIR` (ex: `kb/ict/`) — mover para lá antes de ingesting |
| Arquivo existe na KB_DIR mas count não aumenta | ingest.py pode ter filtro `len(content) < 50` — verificar tamanho do arquivo |
| **Arg list too long ao enviar pro Chroma** | Chunk texts grandes exceedem argparse limit. Usar `--data @/tmp/payload.json` em vez de `-d '...'`. Escrever payload JSON num arquivo temp e passar como `f"@/tmp/payload.json"` |
| **Shallow git clone ainda ocupa muito espaço** | Repos com `--depth=1` ainda têm .git grande com pack files. Para recuperar espaço: `find .git/objects/pack -name "*.pack" -exec git index-pack --deflate --fix-thin {} \;` + `rm -f .git/objects/pack/*.pack; git gc --prune=now`. Alternativamente: `git fetch --depth=1` (já substitui histórico por 1 commit). |

## Ingestion Incremental (Resumable)

Para repos grandes (10k+ chunks), a ingestão deve ser incremental. NUNCA reclonar — o repo local já existe.

### Passo 1 — Descobrir quais chunks já existem

```python
import subprocess, json

COLLECTION_ID = "YOUR-COLLECTION-ID"
CHROMA_URL = "http://localhost:8000/api/v1"
PREFIX = "hacktricks"  # prefixo dos IDs neste repo

# Buscar IDs existentes com get + pagination
existing_ids = []
for offset in range(0, 30000, 100):
    r = subprocess.run([
        "curl", "-s", "-X", "POST",
        f"{CHROMA_URL}/collections/{COLLECTION_ID}/get",
        "-H", "Content-Type: application/json",
        "-d", json.dumps({"limit": 100, "offset": offset, "where": {}})
    ], capture_output=True, text=True)
    data = json.loads(r.stdout)
    if not data.get("ids"): break
    existing_ids.extend([i for i in data["ids"] if i.startswith(PREFIX)])

# Extrair último índice-ingido
ingested_indices = sorted([
    int(i.replace(f"{PREFIX}_", ""))
    for i in existing_ids
    if i.startswith(f"{PREFIX}_") and i.replace(f"{PREFIX}_", "").isdigit()
])
START_IDX = max(ingested_indices) + 1 if ingested_indices else 0
print(f"Already ingested: {len(ingested_indices)} chunks")
print(f"Next chunk index: {START_IDX}")
```

### Passo 2 — Ingestão Background com Log e Notify

```python
import subprocess, json, os, time
from pathlib import Path

# Escrever script em arquivo (execute_code sandbox tem módulos diferentes)
SCRIPT = '''
import sys, json, subprocess, os, time
from pathlib import Path

REPOS_DIR = os.path.expanduser("~/.rag_repos/cyberbot")
CHROMA_URL = "http://localhost:8000/api/v1"
COLLECTION_ID = "YOUR-COLLECTION-ID"
MAX_BATCH = 20
TMP_FILE = "/tmp/ingest_payload.json"
LOG_FILE = os.path.expanduser("~/cyberbot-sec/logs/ingest.log")

def log(msg):
    print(msg, flush=True)
    with open(LOG_FILE, "a") as f: f.write(msg + "\\n")

log("=== Starting ingestion ===")
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("all-MiniLM-L6-v2")
log("Model loaded!")

# ... (chunking logic)
# START_IDX = last_known_index + 1

for batch_start in range(0, len(chunks_to_ingest), MAX_BATCH):
    batch = chunks_to_ingest[batch_start:batch_start + MAX_BATCH]
    embs = model.encode([c["text"] for c in batch], convert_to_numpy=True).tolist()
    payload = {"ids": ids, "documents": texts, "metadatas": metas, "embeddings": embs}
    
    with open(TMP_FILE, "w") as f:
        json.dump(payload, f)
    
    r = subprocess.run([
        "curl", "-s", "-X", "POST",
        f"{CHROMA_URL}/collections/{COLLECTION_ID}/add",
        "-H", "Content-Type: application/json",
        "--data", f"@{TMP_FILE}"
    ], capture_output=True, text=True, timeout=60)
    
    if r.returncode == 0:
        total_added += len(texts)
        log(f"  [{START_IDX + batch_start + len(batch)}] +{len(batch)} (total: {total_added})")
    time.sleep(0.3)

os.remove(TMP_FILE)
log(f"DONE: {total_added} new chunks")
'''

script_path = "/home/PROJECT/scripts/ingest_incremental.py"
with open(script_path, "w") as f: f.write(SCRIPT)

# Rodar em background com tee pro log
# terminal tool com background=True + notify_on_complete=True
```

### Passo 3 — Monitorar Progresso

```python
import subprocess, time

time.sleep(60)  # esperar modelo carregar + primeiros batches
r = subprocess.run(["tail", "-20", "/path/to/log"], capture_output=True, text=True)
print(r.stdout)

# Verificar se ainda rodando
r2 = subprocess.run(["ps", "aux"], capture_output=True, text=True)
procs = [l for l in r2.stdout.split('\n') if 'ingest' in l.lower()]
print("\\nActive:", procs[:3] if procs else "NONE")
```

### Passo 4 — Re-ingestão após falha/interrupção

Sempre que re-rodar, recalcular `START_IDX` (Passo 1) — não assumir que o último batch completou. Se o log mostra que parou no índice 2000 mas Chroma tem 2200 chunks, o próximo run deve começar do 2200.

---

## Limpeza de Espaço em Disco Pré-Ingestão

Antes de ingestion massiva, analisar e liberar espaço:

```python
import subprocess, os

# Mapear onde está o espaço
r = subprocess.run(
    "du -sh /home/*/ 2>/dev/null | sort -rh | head -10",
    shell=True, capture_output=True, text=True
)
print(r.stdout)

# Targets seguros pra limpar (sempre verificar antes):
# - /tmp/openclaw-*/        → backups temp, pode reconstruir
# - backups_auto/trading_*  → backups antigos, verificar data
# NÃO limpar: backups_auto/openclaw_*.tar.gz (único backup do sistema)
```

```bash
# Converter repos shallow pra full-shallow (recupera ~30-50% do .git)
cd /path/to/repo/.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive
# Resultado: só o working tree + 1 commit = espaço mínimo
```

---

## Verificação de Chunk Count (Quando API Retorna Errors)

Se `count()` ou `get()` retorna erro, verificar pelo log:

```python
# Ler log de ingestão
r = subprocess.run(["grep", "-c", "DONE", "/path/to/ingest.log"],
    capture_output=True, text=True)
# Ou parsear todas as linhas com "total new:"
```

Ou usar query com embedding vazio + `n_results` alto:

```python
r = subprocess.run([
    "curl", "-s", "-X", "POST",
    f"{CHROMA_URL}/collections/{COLLECTION_ID}/query",
    "-H", "Content-Type: application/json",
    "-d", json.dumps({
        "query_embeddings": [[0.0]*384],
        "n_results": 1,
        "limit": 1,
        "include": ["uris", "metadatas"]
    })
], capture_output=True, text=True)
data = json.loads(r.stdout)
if data.get("documents") and "test_doc_1" in data["documents"][0]:
    print("Collection exists and is accessible")
```

---

## Chroma 0.6.3 REST API Workaround

O wrapper Python do ChromaDB tem bugs conhecidos na versão 0.6.3. Sempre preferir a **REST API via `curl`** em vez do client Python:

```bash
CHROMA_URL="http://localhost:8000/api/v1"
COLLECTION_NAME="my_kb"

# ✅ Criar collection (funciona)
curl -s -X POST "$CHROMA_URL/collections" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"$COLLECTION_NAME\",\"get_or_create\":true}"

# ⚠️ list_collections NUNCA funciona neste Chroma — retorna coroutine error
# ✅ Em vez disso, buscar por nome diretamente
curl -s "$CHROMA_URL/collections/$COLLECTION_NAME"

# ✅ Add documents
curl -s -X POST "$CHROMA_URL/collections/$COLLECTION_ID/add" \
  -H "Content-Type: application/json" \
  -d '{"ids":["doc1"],"documents":["texto"],"embeddings":[[0.1,0.2]]}'

# ✅ Query (usar embedding real de 384 dims para all-MiniLM-L6-v2)
curl -s -X POST "$CHROMA_URL/collections/$COLLECTION_ID/query" \
  -H "Content-Type: application/json" \
  -d '{"query_embeddings":[[0.0]*384],"n_results":3,"limit":3}'
```

**Collection ID:** Após criar via REST, o response contém o ID (ex: `"id":"98802466-..."`). Hardcodar este ID — não há `list_collections` confiável.

---

## Embedding: Modelo Persistente via Subprocess

Cada batch que carrega o modelo do zero custa ~15s. Para ingestion massiva, usar subprocess persistente com comunicação via stdin/stdout:

```python
def load_embedding_model():
    """Start persistent subprocess — modelo carrega UMA vez."""
    code = """import sys, json
from sentence_transformers import SentenceTransformer
model = SentenceTransformer('all-MiniLM-L6-v2')  # ~15s cold start
while True:
    line = sys.stdin.readline()
    if not line: break
    texts = json.loads(line)
    embs = model.encode(texts, convert_to_numpy=True)
    for e in embs: print(json.dumps(e.tolist()))
    sys.stdout.flush()"""
    return subprocess.Popen(
        ["python3.10", "-c", code],
        stdin=subprocess.PIPE, stdout=subprocess.PIPE,
        stderr=subprocess.PIPE, text=True
    )

def get_embeddings_batch(proc, texts, batch_size=10):
    """Get embeddings via persistent subprocess."""
    if not texts: return []
    proc.stdin.write(json.dumps(texts[:batch_size]) + "\n"); proc.stdin.flush()
    return [json.loads(proc.stdout.readline().strip()) for _ in range(len(texts[:batch_size]))]
```

**Benchmark:** Modelo carregado uma vez → ~0.25s/embedding. Modelo recarregado a cada call → ~15s/embedding. **10.000 chunks**: 42min vs 2.5h.

---

## Verificação de Repos GitHub Antes de Clonar

Sempre verificar se o repo existe E qual a branch correta antes de ingerir:

```bash
# Verificar repo e branch principal
git ls-remote --heads https://github.com/OWNER/REPO.git

# Se vazio → repo não existe, privado, ou renomeado
# Testar variações de casing e owner:

# Para buscar repos no GitHub (sem auth para code search):
curl -s "https://api.github.com/search/repositories?q=TERMO+in:name&sort=stars&order=desc" \
  | python3 -c "import sys,json; [print(i['full_name'], i['stargazers_count']) for i in json.load(sys.stdin)['items'][:5]]"
```

**Repos verificados na prática:**
- ✅ `carlospolop/hacktricks` — o correto (não `hacktricks/Awesome-Hacking`)
- ❌ `christophetd/OWASP-Goat` — não existe mais
- ✅ `OWASP/CheatSheetSeries`, `swisskyrepo/PayloadsAllTheThings` — OK

---

## Ingestion Script Template (Chroma REST + Embedding Subprocess)

```python
#!/usr/bin/env python3.10
"""RAG Ingestion — Chroma REST API + sentence-transformers persistent worker"""

import os, sys, json, time, subprocess
from pathlib import Path

CHROMA_URL = "http://localhost:8000/api/v1"
COLLECTION_ID = "YOUR-COLLECTION-ID-HERE"  # hardcoded após create
PYTHON_EMBED = "python3.10"
SUPPORTED_EXTS = {".md", ".py", ".sh", ".txt", ".yml", ".yaml", ".json", ".rst", ".adoc"}

def load_model():
    code = """import sys, json
from sentence_transformers import SentenceTransformer
model = SentenceTransformer('all-MiniLM-L6-v2')
while True:
    line = sys.stdin.readline()
    if not line: break
    texts = json.loads(line)
    embs = model.encode(texts, convert_to_numpy=True)
    for e in embs: print(json.dumps(e.tolist()))
    sys.stdout.flush()"""
    return subprocess.Popen(["python3.10", "-c", code],
        stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)

def get_emb(proc, texts):
    if not texts: return []
    proc.stdin.write(json.dumps(texts) + "\n"); proc.stdin.flush()
    return [json.loads(proc.stdout.readline().strip()) for _ in range(len(texts))]

def chunk_file(filepath, max_chars=800):
    try:
        with open(filepath, "r", errors="ignore") as f: content = f.read()
    except: return []
    ext = os.path.splitext(filepath)[1]
    if ext not in SUPPORTED_EXTS or len(content) < 30: return []
    if ext in {".py",".sh",".js",".ts",".go",".rs",".yml",".yaml",".json",".toml"}:
        lines = content.split("\n"); chunks, cur, clen = [], [], 0
        for ln in lines:
            if clen+len(ln)>max_chars and cur: chunks.append("\n".join(cur)); cur, clen = [], 0
            cur.append(ln); clen += len(ln)
        if cur: chunks.append("\n".join(cur))
        return [c for c in chunks if len(c)>50]
    paras = content.split("\n\n"); chunks, current = [], ""
    for p in paras:
        p = p.strip()
        if not p: continue
        if len(current)+len(p)>max_chars:
            if current: chunks.append(current)
            current = p[:max_chars]
        else: current = (current+"\n\n"+p) if current else p
    if current: chunks.append(current)
    return [c for c in chunks if len(c)>50]

def ingest_repo(repo_url, branch, category, cap=500):
    """Clone, chunk, embed, and ingest a single repo."""
    repo_name = repo_url.split("/")[1]
    repo_path = f"/tmp/rag_repos/{repo_name}"

    if not os.path.exists(repo_path):
        subprocess.run(["git", "clone", "--depth","1", "-b", branch,
                        f"https://github.com/{repo_url}.git", repo_path],
                       capture_output=True, timeout=120)

    files = []
    for root, dirs, filenames in os.walk(repo_path):
        dirs[:] = [d for d in dirs if d not in [".git","node_modules","__pycache__"]]
        for fn in filenames:
            if any(fn.endswith(e) for e in SUPPORTED_EXTS):
                files.append(os.path.join(root, fn))

    all_chunks = []
    for fi in files: all_chunks.extend(chunk_file(fi))
    all_chunks = all_chunks[:cap]

    proc = load_model(); total = 0
    for i in range(0, len(all_chunks), 10):
        batch = all_chunks[i:i+10]
        doc_ids = [f"{repo_name}_{i+j}" for j in range(len(batch))]
        embs = get_emb(proc, batch)
        if len(embs) == len(batch):
            payload = {"ids": doc_ids, "documents": batch,
                "metadatas": [{"source": repo_url, "category": category}] * len(batch),
                "embeddings": embs}
            r = subprocess.run(["curl", "-s", "-X", "POST",
                f"{CHROMA_URL}/collections/{COLLECTION_ID}/add",
                "-H", "Content-Type: application/json", "-d", json.dumps(payload)],
                capture_output=True, text=True, timeout=60)
            if r.returncode == 0: total += len(batch)
    proc.stdin.close(); proc.wait(timeout=5)
    return total
```

**Cap por repo:** 500-2000 chunks para evitar timeout. 500 chunks ≈ 2min com subprocess persistente.

---

## Checklist Pré-Deploy

- [ ] ChromaDB count > 0
- [ ] Query retrieval retorna resultados relevantes
- [ ] Todos arquivos .md da KB_DIR estão ingeridos
- [ ] Não há arquivos fora da KB_DIR que deveriam estar dentro
- [ ] Redis keys `user:*` e `sess:*` isoladas por chat_id
- [ ] Onboarding novo usuário (/start) funciona
- [ ] Git push realizado após mudanças na KB
- [ ] Todos repos do planning verificados com `git ls-remote --heads` antes de clone
- [ ] Collection ID hardcoded (list_collections bugado)
- [ ] Embedding model usa subprocess persistente (não recarrega por batch)
- [ ] Chunking: 800 char max, filtro <50 chars, extensões válidas
- [ ] Cap por repo: 500-2000 chunks (evitar timeout)
