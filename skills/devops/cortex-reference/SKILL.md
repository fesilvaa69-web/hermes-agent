---
name: cortex-reference
description: Documentação de referência do Cortex — schedulers, state files, comandos de manutenção e coordenadas Hermes↔Raj.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [cortex, maintenance, reference, scheduler]
    related_skills: [vm-diagram-maintenance, soul-memory-updater]
---

# Cortex Reference

Documentação completa do Cortex para manutenção e coordenação.

## Estrutura do Cortex

```
~/.openclaw/workspace/cortex/
├── synapse/              # Protocolos e demandas
├── references/         # Referências externas
├── maintenance/        # Schedulers de manutenção
└── deep_store/        # Sessions de deep work
```

---

## Schedulers do Cortex

### Schedulers Ativos

| Scheduler | Script | PID Esperado | Frequência |
|-----------|--------|-------------|------------|
| SIAS | `sias/scheduler.py` | variável | 6 jobs |
| RADAR | `radar/scheduler.py` | 462542 | contínuo |
| DIARIO | `trading_agent/diario_scheduler.py` | 467795 | diário |
| DATA Monitor | `data_monitor_scheduler.py` | 467837 | hourly |

### Comandos de Restart

```bash
# SIAS
cd ~/.openclaw/workspace && python sias/scheduler.py &

# RADAR
cd ~/.openclaw/workspace-radar && python scheduler.py &

# DIARIO
cd ~/.openclaw/workspace && python trading_agent/diario_scheduler.py &

# DATA Monitor
cd ~/.openclaw/workspace && python cortex/maintenance/data_monitor_scheduler.py &
```

---

## State Files

### SIAS State

| Arquivo | Conteúdo | Atualização |
|---------|--------|------------|
| `sias_state/debate_counter.json` | Contador de ciclos | a cada debate |
| `sias_state/debate_registry.json` | Registro de debates | a cada debate |
| `sias_state/canary_scan_*.json` | Resultados de canary | manual |
| `sias_state/diagnosis_*.json` | Diagnósticos | manual |

### VM State

| Arquivo | Conteúdo |
|---------|---------|
| `vm_state/vm_diagram.md` | Diagrama vivo (50KB) |
| `vm_state/debate_registry.json` | Cópia local |

---

## Coordenação Hermes ↔ Raj

### Arquivo de Estado Compartilhado

```bash
# Ler estado atual
read_file(path="~/.hermes/cortex/notes/ultimo_estado.md")

# Atualizar estado
patch(path="~/.hermes/cortex/notes/ultimo_estado.md", old_string="### Estado: ANTIGO", new_string="### Estado: NOVO")
```

### Protocolo de Sync

1. **Antes de iniciar trabalho significativo**: atualizar `ultimo_estado.md`
2. **Após completar**: marcar como COMPLETO
3. **Se interrompido**: marcar INTERROMPIDO + ação pendente

### Formato de Estado

```markdown
## Estado: [INTERROMPIDO|COMPLETO|EM_PROGRESSO]

### Tarefa: [descrição]

### Ação Pendente: [se interrumpido]

### Contexto: [INFO para próximo]
```

---

## Comandos de Manutenção

### Verificar Saúde

```bash
# Ver schedulers
ps aux | grep -E 'scheduler|hermes|openclaw' | grep -v grep

# Ver logs de erro
tail -f ~/.hermes/logs/errors.log

# Ver memória
free -h

# Ver disk
df -h
```

### Troubleshooting

| Problema | Solução |
|---------|--------|
| Scheduler não inicia | verificar Python path |
| 401 API error | verificar API key |
| Gateway offline | `hermes gateway restart` |
| Alta memória | verificar processes |

---

## Directories Importantes

| Diretório | Conteúdo |
|-----------|---------|
| `~/.openclaw/workspace/` | Raj workspace |
| `~/.hermes/` | Hermes workspace |
| `~/.openclaw/workspace/cortex/` | Cortex central |
| `~/.openclaw/workspace/sias/` | SIAS engine |
| `~/.hermes/cortex/notes/` | Estado compartilhado |

---

## Links Rápidos

- **vm_diagram.md**: `~/.openclaw/workspace/vm_state/vm_diagram.md`
- **debate_engine.py**: `~/.openclaw/workspace/sias/debate_engine.py`
- **DELEGACAO_DEBATE**: `~/.openclaw/workspace/cortex/synapse/DELEGACAO_DEBATE_SIAS_DEMANDA_COMPLETA.md`
- **AGENTS.md**: `~/.openclaw/workspace/AGENTS.md`

---

## ⚠️ Regras de Manutenção

1. **NUNCA** restart sem verificar causa raiz
2. **SEMPRE** atualizar `ultimo_estado.md` antes de restart
3. **REPORTAR** gaps claramente
4. **SOLICITAR** approval antes de mudanças significativas