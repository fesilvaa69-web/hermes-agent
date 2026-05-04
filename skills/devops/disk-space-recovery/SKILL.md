---
name: disk-space-recovery
description: Recover from disk exhaustion on the Hermes VM — diagnose, triage, clean up, and restore tool functionality. Triggered when terminal/execute_code/patch tools fail with "No space left on device" or "/tmp/openclaw-backup not found".
category: devops
---

# Disk Space Recovery

> **Versão:** 1.0 | **Data:** 2026-05-02
> **Contexto:** VM Hermes, Ubuntu, 39GB root partition, user Felipe

---

## Trigger Conditions

Any of these signals indicate disk space problem:
- Tool error: `OSError: [Errno 28] No space left on device`
- Tool error: `FileNotFoundError: [Errno 2] No such file or directory: '/tmp/openclaw-backup'`
- `df -h /` shows Use% ≥ 97%

---

## Diagnostic Sequence

### Step 1: Check current state
```bash
df -h /
ls /tmp/openclaw-backup 2>/dev/null && echo "OK" || echo "MISSING"
```

### Step 2: Identify largest consumers
```bash
# Top-level home directories (fast)
du -sh /home/felipe_silva_sorocaba/*/ 2>/dev/null

# OpenClaw ecosystem (common culprit)
du -sh /home/felipe_silva_sorocaba/.openclaw/

# Plugin runtime deps (often 1.2GB+)
du -sh /home/felipe_silva_sorocaba/.openclaw/plugin-runtime-deps/

# Docker
docker system df
```

### Step 3: Verify /tmp/openclaw-backup
```bash
mkdir -p /tmp/openclaw-backup
```
This single command restores terminal/patch tool functionality.

---

## Known Cleanup Targets

| Target | Size | Safe to Delete? |
|--------|------|-----------------|
| `~/backups_auto/` | ~725MB | ✅ Yes — auto backups, obsolete |
| `~/BACKUP_PRE_AUDIT_*` | ~62MB | ✅ Yes — Feb 2026 audits, obsolete |
| `~/BACKUP_PRE_DBCORTEX_*` | ~12MB | ✅ Yes — Feb 2026, obsolete |
| `~/BACKUP_PRE_TRAINING_*` | ~236KB | ✅ Yes — Feb 2026, obsolete |
| `.openclaw/plugin-runtime-deps/` | ~1.2GB | ⚠️ Caution — OpenClaw runtime deps, can rebuild |
| `cyberbot-sec/` | ~500KB | ✅ Yes — old logs |
| Docker images (reclaimable) | ~741MB | ✅ Yes — `docker system prune -a` |

### Safe one-liner cleanup (run before any ingest)
```bash
rm -rf ~/backups_auto/ \
  ~/BACKUP_PRE_AUDIT_20260228_203343 \
  ~/BACKUP_PRE_DBCORTEX_20260228_222623 \
  ~/BACKUP_PRE_TRAINING_CONSOLIDATION_20260228_221627
```

---

## Recovery After Cleanup

1. Recreate `/tmp/openclaw-backup`: `mkdir -p /tmp/openclaw-backup`
2. Test: `echo "test" && df -h /`
3. If still tools fail → check `/tmp` itself: `du -sh /tmp/*`

---

## Disk Space Budget

- Total: 39GB
- Safe operating: keep ≥ 1GB free (97% = 101MB, critically low)
- Target: stay below 90% usage

---

## Key Insight: System Python Fallback

When disk space is critically low and pip install fails, `/usr/bin/python3` (system Python 3.10) already has:
- `sentence-transformers` installed
- `all-MiniLM-L6-v2` model cached (~88MB)

Use system Python for one-off scripts:
```bash
/usr/bin/python3 scripts/ingest_rag.py --repo owner/repo
```

This bypasses the need to pip install into a venv.

---

## Pitfalls

- **Don't delete `.openclaw/workspace/`** — contains Raj's active state
- **Don't delete `~/.rag_repos/`** — these are the knowledge base repos
- **Don't delete venvs** in bot directories — they are actively running
- **`/tmp/hermes_sandbox_*` files** — can accumulate, safe to delete via `rm -rf /tmp/hermes_sandbox_*`

---

## Verification

After recovery, confirm tools work:
```bash
echo "terminal OK"
python3 -c "from chromadb import HttpClient; print('chroma OK')"
```
