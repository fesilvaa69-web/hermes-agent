---
name: sias-master-v2-joint-implementation
description: Workflow conjunto Hermes+Raj para implementar SIAS MASTER ARCHITECTURE v2.0
category: sias
tags: [sias, joint-workflow, raj-coordination, implementation]
created: 2026-04-28
status: awaiting-felipe-document
---

# SIAS MASTER ARCHITECTURE v2.0 — Implementação Conjunta Hermes ↔ Raj

> **Criado:** 2026-04-28
> **Status:** Aguardando documento de Felipe
> **Participantes:** Hermes (@Jarvis_Hermess_Bot) + Raj (@Raj_AIgentBot)

---

## CONTEXTO

Implementação anterior FALHOU porque Hermes e Raj implementaram EM PARALELO sem coordenação.
Felipe rejeitou e exigiu workflow conjunto.

**Erro:** Duplicações, referências antigas não auditadas, trabalho não verificado em conjunto.

---

## WORKFLOW OBRIGATÓRIO

```
FELIPE (documento fases)
    ↓
[PLANEJAMENTO CONJUNTO]
    ├── Analisar documento juntos
    ├── Dividir tarefas (Hermes vs Raj)
    ├── Definir ordem de execução
    └── Estabelecer checkpoints de verificação
    ↓
[EXECUÇÃO PARALELA]
    ├── Hermes executa sua parte
    └── Raj executa sua parte
    ↓
[AUDITORIA CONJUNTA]
    ├── Verificar duplicidades
    ├── Verificar referências antigas
    ├── Arquivar ou substituir antigos
    └── Validar completude
    ↓
[INTEGRAÇÃO]
    ├── Integrar no Cortex como fonte de verdade
    └── Atualizar bootstrap files
```

---

## FASES DO DOCUMENTO (Aguardando Felipe enviar)

1. **Fase 1:** Analisar e arquivar documentos SIAS antigos
2. **Fase 2:** Implementar SIAS MASTER ARCHITECTURE v2.0
3. **Fase 3:** Verificar duplicados e referências
4. **Fase 4:** Atualizar Cortex e bootstrap files

---

## ARTEFATOS A AUDITAR

### Skills SIAS
- `sias-navigator`
- `sias-debate-conductor`
- `hallucination-detector`
- `loop-breaker`
- `sias-canary-scan`
- `sias-cycle-execution`
- `debate-protocol`

### Scripts SIAS
- `sias/loop_detector.py`
- `sias/hallucination_guardian.py`
- `cortex/maintenance/guardrail_state_check.py`
- `sias/sias_health_dashboard.py`
- `sias/debate_decision_engine.py`
- `sias/debate_engine.py`
- `sias/hn_search.py`

### Arquivos de Estado
- `sias_state/sias_backlog.json`
- `sias_state/ecosystem_patterns.md`
- `sias_state/integration_log.md`
- `sias_state/debate_registry.json`

---

## REGRAS CRÍTICAS

1. **NUNCA implementar em paralelo sem planejamento conjunto primeiro**
2. **SEMPRE auditar duplicados antes de criar novos artefatos**
3. **Arquivar** antigos — não apenas substituir silenciosamente
4. **Documentar** decisões no Cortex/synapse
5. **Atualizar MEMORY.md e SOUL.md** apenas no FINAL da tarefa

---

## PRÓXIMO PASSO

Aguardar Felipe enviar documento com fases de implementação.
Ao receber: fazer planejamento conjunto com Raj antes de qualquer execução.
