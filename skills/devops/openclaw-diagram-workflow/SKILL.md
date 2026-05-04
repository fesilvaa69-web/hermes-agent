---
name: openclaw-diagram-workflow
description: Workflow DIAGRAM para análise sistemática de ferramentas OpenClaw — ciclos de lint, teste, cobertura e inovação
triggers:
  - Análise de ferramenta do ecossistema OpenClaw
  - Cycle 004+ do workflow de mapeamento
---

# DIAGRAM Workflow — OpenClaw Code Analysis

## Workflow Completo

### Cycle Structure
Cada cycle analisa uma ferramenta seguindo 5 fases:

| Fase | Descrição |
|------|-----------|
| **A — Análise Estática** | Linting (ruff), contagem LOC, identificação de imports/propósito |
| **B — Problemas** | Levantamento de issues (CRÍTICA/MÉDIA/BAIXA) |
| **C — Testes** | Criar/rodar pytest via SIAS 4 camadas |
| **D — Cobertura** | Medir coverage com pytest-cov |
| **E — Inovação** | Propor melhorias + score final |

## Ferramentas já analisadas (Cicles 001-004)

| Cycle | Ferramenta | Testes | Score |
|-------|------------|--------|-------|
| 001 | bias_diario.py | 17 | 77.0% |
| 002 | data_monitor_scheduler.py | 21 | 78.5% |
| 003 | callback_handler.py | 16 | 80.0% |
| 004 | diario_scheduler.py | 23 | 78.0% |
| **TOTAL** | | **77** | |

## Fila de Ferramentas Identificadas
- [x] diario_scheduler.py (cycle 004)
- [ ] monitor_engine.py
- [ ] diario_core.py
- [ ] bias_automacao/briefing_generator.py
- [ ] proativo/pattern_detector.py
- [ ] interactive_handler.py
- [ ] media_storage.py
- [ ] photo_handler.py

## Innovation Types
1. **RetryDecorator** — exponential backoff para APIs externas
2. **MessageBus** — abstração Pub/Sub
3. **HealthDashboard** — métricas Prometheus
4. **Config via Environment** — variáveis de ambiente

## Commands
```bash
ruff check ~/.openclaw/workspace-core/skills/diario-trade/diario_scheduler.py
find ~/.openclaw -name "*.py" | head -20 | xargs wc -l
cd ~/.openclaw/workspace-core/skills/diario-trade && pytest --tb=short -v
pytest --cov=. --cov-report=term-missing
```

## Output Format (DIAGRAM v1.15)
```
# 📊 DIAGRAM v1.15 — CYCLE 004
## 🔍 FERRAMENTA: `diario_scheduler.py`
...
## 🎯 FASE E — INOVAÇÃO
...
## 📋 RESUMO CICLOS ATUALIZADO
...
```
