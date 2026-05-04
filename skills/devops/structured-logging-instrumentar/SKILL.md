---
name: structured-logging-instrumentar
description: Add structured logging decorator to Python functions for observability
triggers:
  - "instrumentar decorator"
  - "structured logging python"
  - "add logging to scheduler"
  - "observability data agent"
  - "@instrumentar"
---

# Skill: Structured Logging with @instrumentar Decorator

## When to use
When you need to add observability to Python functions in the OpenClaw ecosystem — particularly for schedulers, agents, and long-running tasks where you need to track:
- When a function starts and ends
- How long it takes (duration)
- Whether it succeeded or failed
- Unique invocation ID for correlation

## Prerequisites
- Target file: `cortex/maintenance/instrumentar.py`
- Target functions: schedulers, cron jobs, agent tasks
- Log output: `~/.openclaw/logs/data_agent_instrumentar.log`

## Approach

### Step 1 — Create the instrumentar module
Create `cortex/maintenance/instrumentar.py`:
```python
#!/usr/bin/env python3
"""
instrumentar.py — Decorator de instrumentação para observabilidade.
Baseado em: HERMES_TESTING_PROTOCOL.md §instrumentar

Uso:
    from instrumentar import instrumentar

    @instrumentar("T-DA-001", "data_agent")
    def my_task():
        pass
"""

import functools
import uuid
import logging
from pathlib import Path
from datetime import datetime

# Setup logger
LOG_FILE = Path.home() / ".openclaw" / "logs" / "data_agent_instrumentar.log"
LOG_FILE.parent.mkdir(parents=True, exist_ok=True)
logger = logging.getLogger("instrumentar")
logger.setLevel(logging.INFO)
if not logger.handlers:
    handler = logging.FileHandler(LOG_FILE)
    handler.setFormatter(logging.Formatter("%(message)s"))
    logger.addHandler(handler)

def instrumentar(tool_id: str, agent_id: str):
    """
    Decorator que adiciona logging estruturado a funções.
    
    Args:
        tool_id: Identificador único do instrumento (ex: "T-DA-001")
        agent_id: Nome do agente (ex: "data_agent", "core", "ml")
    
    Emite eventos:
        - START: quando a função inicia
        - SUCCESS: quando a função completa sem erro
        - ERROR: quando a função lança exceção
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            invocation_id = f"{func.__name__}_{uuid.uuid4().hex[:12]}"
            start = datetime.now()
            
            logger.info(
                f"▶ START | {func.__name__} | invocation_id={invocation_id} | "
                f"tool={tool_id} | agent={agent_id}"
            )
            
            try:
                result = func(*args, **kwargs)
                duration_ms = (datetime.now() - start).total_seconds() * 1000
                logger.info(
                    f"✔ SUCCESS | {func.__name__} | {duration_ms:.1f}ms | "
                    f"invocation_id={invocation_id}"
                )
                return result
            except Exception as e:
                duration_ms = (datetime.now() - start).total_seconds() * 1000
                logger.error(
                    f"✘ ERROR | {func.__name__} | {duration_ms:.1f}ms | "
                    f"{type(e).__name__}: {e} | invocation_id={invocation_id}"
                )
                raise
        return wrapper
    return decorator

# Test mode
if __name__ == "__main__":
    @instrumentar("test", "test_agent")
    def teste():
        pass
    teste()
    print("Resultado: ok")
```

### Step 2 — Import and apply to target functions
In the target file (e.g., `data_monitor_scheduler.py`):
```python
import sys
sys.path.insert(0, str(Path(__file__).parent.parent))
from cortex.maintenance.instrumentar import instrumentar

@instrumentar("T-DA-001", "data_agent")
def task_monitor():
    """Task de monitoramento - a cada hora."""
    # ... function body
```

### Step 3 — Assign Tool IDs
Tool ID format: `T-{AGENT}-{NUMBER}` (3 digits)

**DATA_AGENT** (T-DA):
- T-DA-001: task_monitor
- T-DA-002: task_heartbeat
- T-DA-003: task_daily_report
- T-DA-004: task_memory_update
- T-DA-005: task_index_update
- T-DA-006: check_new_files
- T-DA-007: send_telegram_message
- T-DA-008: load_state
- T-DA-009: save_state

**CORE_AGENT** (T-CO):
- T-CO-001: task_xxx
- etc.

## Key Design Decisions

1. **Invocation ID**: UUID-based, unique per call, allows correlating START→SUCCESS/ERROR
2. **Duration tracking**: In milliseconds, more useful than seconds for fast functions
3. **Separation of concerns**: Logger outputs structured text, not JSON (easier to read and grep)
4. **Exception re-raise**: The decorator re-raises exceptions so the caller still gets the error
5. **Path insertion**: `sys.path.insert(0, ...)` to allow importing from `cortex/maintenance/`

## Verification
```bash
# Test the module directly
python3 cortex/maintenance/instrumentar.py
# Expected output:
# ▶ START | teste | invocation_id=teste_xxx
# ✔ SUCCESS | teste | 0.xms | invocation_id=teste_xxx

# Check syntax of target file
python3 -c "compile(open('cortex/maintenance/data_monitor_scheduler.py').read(), 'scheduler', 'exec'); print('OK')"

# Check the log file
tail ~/.openclaw/logs/data_agent_instrumentar.log
```

## Common Pitfalls
- **Module not found**: Always use `sys.path.insert(0, str(Path(__file__).parent.parent))` before importing instrumentar
- **Syntax errors**: The decorator modifies functions but preserves their signatures — if you see functions "merged", you likely pasted the decorator inside the function body instead of above it
- **Git tracking**: The log file (`~/.openclaw/logs/data_agent_instrumentar.log`) should be in .gitignore

## Files
- `cortex/maintenance/instrumentar.py` — the decorator module (6.6KB)
- Applied in: `cortex/maintenance/data_monitor_scheduler.py` (9 functions)
- Log output: `~/.openclaw/logs/data_agent_instrumentar.log`

## SIAS Integration
This implements SIAS-TA-002 (ICE Score 45):
- Criterion: 100% of DATA_AGENT functions with @instrumentar
- Status: ✅ 9/9 functions instrumented
