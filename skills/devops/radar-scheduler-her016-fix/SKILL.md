---
name: radar-scheduler-her016-fix
description: Fix HER-016 para radar_telegram_scheduler.py — flock PID lock + APScheduler orphan job cleanup. Aplica em ~/.openclaw/workspace/trading_agent/scripts/radar_telegram_scheduler.py.
version: 1.0
created: 2026-04-22
tags: [her-016, radar, apscheduler, flock, pid-lock, openclaw]
---

# HER-016 Fix — RADAR Scheduler PID Lock + Orphan Cleanup

## Contexto

O RADAR scheduler morre durante incidentes e ao restart adiciona jobs duplicados ao store porque:
1. Não verifica se já existe processo vivo via PID file
2. Não usa lock advisory (flock)
3. Não limpa jobs órfãos do APScheduler job store

## Arquivo Alvo

```
~/.openclaw/workspace/trading_agent/scripts/radar_telegram_scheduler.py
```

**ATENÇÃO:** O scheduler é Python (APScheduler), NÃO JavaScript. O workspace `workspace-radar/` contém código JS do bot, mas o scheduler de briefings é Python em `trading_agent/scripts/`.

## Fix Aplicado (HER-016 Bundled)

### 1. Import fcntl

```python
import fcntl  # Added alongside existing imports
```

### 2. No main(), antes de criar o scheduler:

```python
# === HER-016 FIX: flock + duplicate job cleanup ===
PID_FILE = "/tmp/radar_scheduler.pid"
LOCK_FILE = "/tmp/radar_scheduler.lock"

# 1. Check if PID file has a living process
if os.path.exists(PID_FILE):
    with open(PID_FILE, 'r') as f:
        old_pid = f.read().strip()
    try:
        os.kill(int(old_pid), 0)  # Signal 0 = check if process exists
        logger.info(f"⚠️ Scheduler ja esta rodando (PID {old_pid}). Saindo.")
        sys.exit(0)
    except (OSError, ValueError):
        logger.warning(f"PID {old_pid} invalido/morto — limpando job store e continuando.")

# 2. Acquire exclusive flock to prevent concurrent starts
try:
    lock_fd = open(LOCK_FILE, 'w')
    fcntl.flock(lock_fd.fileno(), fcntl.LOCK_EX | fcntl.LOCK_NB)
    logger.info(f"✅ Lock adquirido (fd={lock_fd.fileno()})")
except IOError:
    logger.warning("⚠️ Nao foi possivel adquirir lock — scheduler pode estar rodando. Saindo.")
    sys.exit(0)

scheduler = BlockingScheduler(timezone=brt)

# 3. Clean up orphaned jobs from previous crashed instance
existing_jobs = scheduler.get_jobs()
if existing_jobs:
    logger.info(f"🧹 Limpando {len(existing_jobs)} job(s) orfao(s) do store anterior:")
    for job in existing_jobs:
        logger.info(f"  - Removendo: {job.id} ({job.name})")
        scheduler.remove_job(job.id)
    logger.info("✅ Jobs orfaos removidos.")
```

## Como Verificar

```bash
# Syntax check
python3 -m py_compile ~/.openclaw/workspace/trading_agent/scripts/radar_telegram_scheduler.py

# Testar restart
python3 ~/.openclaw/workspace/trading_agent/scripts/radar_telegram_scheduler.py
# Deve mostrar "✅ Lock adquirido" e "🧹 Limpando N job(s) orfao(s)"
```

## Restoration (Rollback)

Se precisar reverter:
```bash
git checkout HEAD~1 -- ~/.openclaw/workspace/trading_agent/scripts/radar_telegram_scheduler.py
```

## Related Files

| File | Purpose |
|------|---------|
| `/tmp/radar_scheduler.pid` | PID file (criado pelo scheduler) |
| `/tmp/radar_scheduler.lock` | Lock file (flock advisory) |
| `~/.openclaw/logs/radar_scheduler.log` | Log output |
