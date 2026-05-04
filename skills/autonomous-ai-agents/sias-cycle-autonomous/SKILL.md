---
name: sias-cycle-autonomous
description: Automate RAJ Learning Loop → SIAS bridge with insight_trigger.json
triggers:
  - sias autonomous cycle
  - learning loop sias integration
  - self-improving system
---

# SIAS Autonomous Cycle — RAJ Learning Loop → SIAS Bridge

## Architecture

```
RAJ Learning Loop (raj_learning_loop.py)
    │
    ├── 15min check → insights_pending.json
    ├── daily audit → create skill
    └── bridge_to_sias() → insight_trigger.json
                               │
                               ▼
                    sias/sias_state/insight_trigger.json
                    {
                      "triggered_by": "raj_learning_loop",
                      "insight": {...},
                      "suggested_agent": "core",
                      "suggested_focus": "reasoning",
                      "status": "pending"
                    }
                               │
                               ▼
                    SIAS Scheduler (sias/scheduler.py)
                    job_cycle() → improvement_db
```

## Implementation Steps

### 1. Create bridge file
```bash
mkdir -p ~/.openclaw/workspace/sias/sias_state/
cat > sias/sias_state/insight_trigger.json << 'EOF'
{
  "triggered_by": null,
  "triggered_at": null,
  "insight": {"name": null, "category": null, "priority": null},
  "suggested_agent": null,
  "suggested_focus": null,
  "status": "pending"
}
EOF
```

### 2. Add bridge_to_sias() to raj_learning_loop.py
```python
def bridge_to_sias(insight: dict) -> bool:
    """Write insight to trigger file for SIAS to pick up."""
    trigger_file = Path(__file__).parent / "sias" / "sias_state" / "insight_trigger.json"
    
    if insight.get("priority") != "high" and insight.get("complexity", 0) < 7:
        return False  # Only bridge significant insights
    
    trigger = {
        "triggered_by": "raj_learning_loop",
        "triggered_at": datetime.now().isoformat(),
        "insight": insight,
        "suggested_agent": "core",
        "suggested_focus": "reasoning",
        "status": "pending"
    }
    
    with open(trigger_file, "w") as f:
        json.dump(trigger, f, indent=2)
    
    return True
```

Integrate into `create_skill_from_insight()` and `run_insight_nudge()`.

### 3. Modify SIAS scheduler job_cycle()
Add reading of trigger file before running:
```python
def job_cycle(agent_id: str = None, focus_area: str = None):
    trigger_file = TRIGGER_PATH  # sias/sias_state/insight_trigger.json
    
    if Path(trigger_file).exists():
        with open(trigger_file) as f:
            trigger = json.load(f)
        
        if trigger.get("status") == "pending":
            agent_id = trigger.get("suggested_agent", agent_id)
            focus_area = trigger.get("suggested_focus", focus_area)
            trigger["status"] = "processing"
            # ... save trigger ...
    
    # Execute cycle with potentially redirected agent/focus
    # ...
    
    if Path(trigger_file).exists():
        trigger["status"] = "processed"
        # Save with processed timestamp
```

### 4. Start SIAS Scheduler
```bash
python3 sias/scheduler.py --start &
# Verify: pgrep -f "sias/scheduler.py"
```

## Critical Discovery: Process Lifecycle

**PROBLEM:** `sias/scheduler.py` uses `BlockingScheduler` which blocks the terminal. Starting with `&` or `nohup` works but **processes die when terminal closes or after ~1-2 hours**.

**VERIFICATION:**
```bash
pgrep -f "sias/scheduler.py"  # Check if alive
ps aux | grep scheduler      # Full process info
```

**SOLUTION:** Use a guardian/watcher process or systemd service. For manual restarts:
```bash
# Restart SIAS
python3 ~/.openclaw/workspace/sias/scheduler.py --start &
# Check
sleep 2 && pgrep -f "sias/scheduler.py"
```

## E2E Test
```python
# Create test trigger
trigger = {
    "triggered_by": "test",
    "triggered_at": datetime.now().isoformat(),
    "insight": {"name": "test", "category": "test", "priority": "high"},
    "suggested_agent": "core",
    "suggested_focus": "routing",
    "status": "pending"
}
with open(trigger_file, "w") as f:
    json.dump(trigger, f)

# Trigger manually
python3 sias/scheduler.py --trigger core

# Verify: status should be "processed" after cycle
```

## Pitfalls

1. **BlockingScheduler blocks terminal** — always run with `&` / background
2. **Processes die silently** — verify with `pgrep -f` after starting
3. **focus_area validation** — Guard Rail rejects invalid focus areas. Valid ones: `routing`, `reasoning`, `output`, `tool_use`, `self_reflection`, `tool_analysis`
4. **improvement_db.sqlite coverage() bug** — has NameError. Query with raw sqlite3 if needed

## Files

| File | Purpose |
|------|---------|
| `sias/sias_state/insight_trigger.json` | Bridge file |
| `sias/scheduler.py` | SIAS daemon |
| `sias/improvement_db.py` | SQLite interface |
| `raj_learning_loop.py` | RAJ Learning Loop daemon |
