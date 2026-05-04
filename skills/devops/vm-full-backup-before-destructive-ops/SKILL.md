---
name: vm-full-backup-before-destructive-ops
description: Backup de ecossistema completo (OpenClaw + Hermes) para GitHub privado antes de operações destrutivas na VM — exclui .db, credentials, logs; remove embedded git repos; cria repo novo se remote tem arquivos grandes.
triggers:
  - "backup completo antes de desinstalar"
  - "backup da VM antes de coisa drástica"
  - "mirror ecosystem to github"
  - "clonar ecossistema para outro ambiente"
---

# VM Full Backup Before Destructive Ops

> **Versão:** 1.0 | **Data:** 2026-05-02
> **Classe:** Backup de ecossistema completo para GitHub privado antes de operação destrutiva na VM

---

## TRIGGER

Usar quando Felipe pedir para fazer backup completo do ecossistema (OpenClaw, Hermes, ou ambos) antes de:
- Desinstalar / reinstalar software
- Operação "drástica" na VM
- Migrar ou refazer infraestrutura
- Antes de qualquer limpeza agressiva de disco

**Também serve para:** clonagem de ecossistema para outro ambiente.

---

## WORKFLOW

### Fase 1 — Verificar espaço em disco

```bash
df -h /
```

Se `Avail` < 2GB, fazer limpeza primeiro:
```bash
# Cache apt/pip (maior ganho — ~3GB)
find ~/.cache -type f -delete
find ~/.cache -type d -delete

# Backups velhos
rm -rf ~/.openclaw/backups/*
rm -rf ~/.openclaw/workspace/cortex/backups/*

# Session audits antigos (>30 dias)
find ~/.openclaw/workspace/cortex/synapse/session_audit -name '*.md' -mtime +30 -delete

# Briefings antigos (antes de data X — já no GitHub)
find ~/.openclaw/workspace/cortex/synapse/radar/briefings -name '*.md' ! -newer ref_file -delete
```

### Fase 2 — Copiar para dir temporário com rsync

```bash
BACKUP=/tmp/ecosystem-backup
rm -rf $BACKUP && mkdir -p $BACKUP

# RSYNC com exclusões
rsync -av \
  --exclude=logs/ \
  --exclude=__pycache__/ \
  --exclude='*.pyc' \
  --exclude='*.db' \
  --exclude='*.sqlite' \
  --exclude='*.sqlite3' \
  --exclude='chroma.sqlite3' \
  --exclude='local_search.db' \
  --exclude='.chroma_db/' \
  --exclude='credentials/' \
  --exclude='.openclaw-npm-cache/' \
  --exclude='plugin-runtime-deps/' \
  --exclude='state-snapshots/' \
  ~/.openclaw/ $BACKUP/openclaw/

rsync -av \
  --exclude=logs/ \
  --exclude=__pycache__/ \
  --exclude='*.pyc' \
  --exclude='state.db' \
  --exclude='state-snapshots/' \
  ~/.hermes/ $BACKUP/hermes/
```

### Fase 3 — Remover repos git embutidos

Workspaces de agentes OpenClaw contêm `.git` directories embutidos que causam erro "embedded git repository" no push:

```bash
# Encontrar todos .git embutidos
find $BACKUP -name '.git' -type d

# Remover todos
find $BACKUP -name '.git' -type d -exec rm -rf {} +
```

### Fase 4 — Criar repo GitHub limpo

**⚠️ Se o remote repo já tem commits com arquivos grandes, NÃO usar --force** — o GitHub guarda o histórico e vai rejeitar. Criar repo novo:

```bash
gh repo create NOME_DO_REPO --private --description "Backup completo — TODO: excluir após restore"
```

### Fase 5 — Commit e Push

```bash
cd $BACKUP
git init
git add -A
git commit -m "Full backup: [data] — [descrição do conteúdo]"
git remote add origin git@github.com:USER/REPO.git
git push -u origin main
```

### Fase 6 — Verificar

```bash
gh repo view USER/REPO --json url,name,description,isPrivate
du -sh $BACKUP
```

---

## PITFALLS

| Problema | Solução |
|----------|---------|
| `rsync: write failed: No space left on device` | Limpar cache primeiro (3-5GB livres necessários) |
| `error: adding embedded git repository` | Remover `.git` directories embutidos com find |
| `GH001: Large files detected` (>100MB) | Remover state.db, *.sqlite, chroma.sqlite3 antes do commit |
| `! [rejected] main -> main (fetch first)` | O remote tem arquivos grandes no histórico → criar repo novo |
| `git push --force` não funciona com arquivos grandes no remote | Criar novo repo, push limpo |
| `.git/objects/pack/` grande no backup local | Fazer `git clone --depth 1` em vez de rsync para backup final |

---

## O QUE EXCLUIR

| Tipo | Razão |
|------|-------|
| `*.db`, `*.sqlite`, `*.sqlite3` | Bases de dados — rebuild é possível |
| `state.db`, `state-snapshots/` | Hermes state — dinâmico |
| `logs/` | Arquivos de log — não são fonte |
| `__pycache__/` | Bytecode — não é fonte |
| `credentials/` | Secrets — nunca commitar |
| `.chroma_db/`, `chroma.sqlite3` | Vector DB — muito grande, rebuild |
| `.openclaw-npm-cache/`, `plugin-runtime-deps/` | Cache npm — não é fonte |
| `node_modules/` | Dependências — reconstruir com npm install |

---

## RESTORE

```bash
# Num novo ambiente ou após reinstall
git clone git@github.com:USER/REPO.git /tmp/restore
rsync -av /tmp/restore/openclaw/ ~/.openclaw/
rsync -av /tmp/restore/hermes/ ~/.hermes/
```
