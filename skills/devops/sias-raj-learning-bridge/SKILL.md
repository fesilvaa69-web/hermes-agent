---
name: sias-raj-learning-bridge
description: Ponte entre Raj Learning Loop e SIAS — reporta ciclos para improvement_db.sqlite após cada execução
---

# SIAS ↔ Raj Learning Loop Bridge

## Quando Usar

Quando o Raj Learning Loop detecta um insight sistêmico (architecture, devops, system, scheduler), este ciclo deve ser **reportado ao SIAS DB** e um trigger deve ser criado para o próximo ciclo SIAS.

## Procedimento

### 1. Verificar estado da ponte

```bash
# Raj Learning Loop vivo?
cat /tmp/raj_learning_scheduler.pid
ps aux | grep $(cat /tmp/raj_learning_scheduler.pid) | grep -v grep

# SIAS Scheduler vivo?
ps aux | grep scheduler.py | grep -v grep

# insight_trigger.json pendente?
cat ~/.openclaw/workspace/sias/sias_state/insight_trigger.json
```

### 2. Iniciar Raj Learning Loop (se morto)

```bash
cd ~/.openclaw/workspace
nohup python3 raj_learning_loop.py --start > ~/.openclaw/logs/raj_learning.log 2>&1 &
echo $! > /tmp/raj_learning_scheduler.pid
```

**Jobs do Raj Learning Loop:**
- A cada 15 min: `run_insight_nudge()` — verifica `insights_pending.json`, cria skills, chama `bridge_to_sias()`
- Diário (20h): `daily_audit()` — audit completo
- Semanal (Sunday 21h): retrospective

### 3. Iniciar SIAS Scheduler (se morto)

```bash
nohup python3 ~/.openclaw/workspace/sias/scheduler.py --start > ~/.openclaw/logs/sias_scheduler.log 2>&1 &
# PID será diferente do /tmp/raj_learning_scheduler.pid
```

### 4. Fluxo da ponte (automático quando ambos os schedulers estão a funcionar)

```
Raj Learning Loop (raj_learning_loop.py)
    │
    ├─ insights_pending.json tem insights?
    │       │
    │      SIM├─ create_skill_from_insight()
    │       │       │
    │       │      +─ bridge_to_sias(insight)
    │       │              │
    │       │              ▼
    │       │     insight_trigger.json written
    │       │              │
    │       │              ▼
    │       │     SIAS Scheduler detecta trigger
    │       │              │
    │       │              ▼
    │       │     run_cycle(agent, focus)
    │       │              │
    │       │              ▼
    │       │     improvement_db.sqlite ← ciclo registado
    │
    └─ NÃO: ciclo termina silenciosamente
```

### 5. Registar ciclo manualmente no SIAS DB

```python
import sqlite3
from datetime import datetime
from pathlib import Path

db_path = Path.home() / ".openclaw/workspace/sias/improvement_db.sqlite"
conn = sqlite3.connect(db_path)
cur = conn.cursor()

cycle_id = "SEU-UUID-AQUI"
now = datetime.now().isoformat()

cur.execute("""
    INSERT INTO improvement_cycles 
    (id, agent_id, focus_area, cycle_number, status, started_at, completed_at, triggered_by, notes, created_at)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
""", (cycle_id, "hermes", "system_cleanup", 30, "completed", now, now, "manual", "CYCLE_030: LIMPEZA v1.37", now))

conn.commit()
conn.close()
```

## Verificação

```bash
# DB: ciclo registado?
python3 -c "
import sqlite3
from pathlib import Path
db = Path.home() / '.openclaw/workspace/sias/improvement_db.sqlite'
conn = sqlite3.connect(db)
cur = conn.cursor()
cur.execute('SELECT id, agent_id, focus_area, cycle_number, status, triggered_by FROM improvement_cycles ORDER BY cycle_number DESC LIMIT 5')
for r in cur.fetchall(): print(r)
conn.close()
"

# Logs
tail -20 ~/.openclaw/logs/raj_learning.log
tail -20 ~/.openclaw/logs/sias_scheduler.log
```

## Pitfalls

- Se `raj_learning_loop.py` morrer, `insight_trigger.json` fica "pending" para sempre — verificar PID regularmente
- O token Telegram no `raj_learning_loop.py` (RAJ_BOT_TOKEN) é diferente do Hermes — manter separados
- CYCLE_030 "LIMPEZA" não é um ciclo SIAS agent-specific — registar como `agent_id=hermes, focus_area=system_cleanup`

## Gap original

Este skill resolve o GAP 2 do CYCLE_031: "Learning Loop não conectado ao SIAS". A ponte `bridge_to_sias()` existe no código mas o scheduler morto impedia a execução. Este skill documenta o procedimento completo para manter a ponte viva.
