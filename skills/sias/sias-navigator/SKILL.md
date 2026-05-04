---
name: sias-navigator
description: >
  LEIA ANTES DE QUALQUER TAREFA relacionada ao SIAS (autônomo ou debate).
  Orienta qual documento ler em qual ordem, quando usar cada ciclo,
  e executa verificação de guard rails antes de começar.
  Trigger: qualquer menção a SIAS, debate, melhoria, evolution council.
  NÃO usar para tasks de inventário puro ou bugfix simples.
version: 1.0
created: 2026-04-22
author: Hermes
category: sias
tags: [sias, navigator, boot, context]
related_skills: [sias-debate-conductor, hallucination-detector, loop-breaker]
---

# SIAS NAVIGATOR — Boot de Contexto Obrigatório

> **Baseado em:** SIAS MASTER ARCHITECTURE v2.0 — P9
> **Objetivo:** Garantir contexto correto antes de qualquer task SIAS

## Passo 1: Verificar Estado do Sistema (< 2 min)

```bash
# Executar ANTES de qualquer task SIAS
python3 ~/.openclaw/workspace/cortex/maintenance/guardrail_state_check.py

# SE bloqueios → resolver primeiro
# SE alertas → anotar e continuar
```

## Passo 2: Ler Documentos na Ordem Correta

### SITUAÇÃO: "Vou iniciar ciclo SIAS de melhoria"
1. `SIAS_MASTER_ARCHITECTURE.md` → Seção P2 (arquitetura autônoma)
2. `SIAS_PROTOCOL.md` → Etapa relevante
3. `SIAS_APPLIED_EXAMPLES.md` → Arquétipo da ferramenta alvo
4. `OPENCLAW_PLANEJADOR.md` → Templates de delegação (se OpenClaw)
5. `HERMES_EXECUTOR.md` → Protocolo de execução (se Hermes)

### SITUAÇÃO: "Vou iniciar debate técnico"
1. `SIAS_MASTER_ARCHITECTURE.md` → Seção P3 (arquitetura debate)
2. `SIAS_DEBATE_PROTOCOL.md` → Tipo A|B|C|D + templates
3. `SIAS_MASTER_ARCHITECTURE.md` → Seção P6 (anti-loop)
4. `OPENCLAW_PLANEJADOR.md` → Seção de delegação para Hermes (se propondo)

### SITUAÇÃO: "Recebi uma task para executar"
1. `HERMES_EXECUTOR.md` → Hierarquia de execução (Seção 1)
2. `SIAS_MASTER_ARCHITECTURE.md` → P8 (guard rails operacionais)
3. Documento específico da task (SIAS_PROTOCOL.md etapa correspondente)

### SITUAÇÃO: "Vou escrever [RESULT]"
1. `SIAS_MASTER_ARCHITECTURE.md` → P7.2 (checklist anti-alucinação)
2. `HERMES_EXECUTOR.md` → Seção 11 (template de resultado)

## Passo 3: Selecionar Ciclo Correto

```python
# Aplicar lógica de selecionar_ciclo() da Seção P1.1
# Em caso de dúvida: SIAS DEBATE (mais conservador que autônomo)

def selecionar_ciclo(trigger: dict) -> str:
    risco = calcular_risco(trigger)
    impacto = calcular_impacto_sistemico(trigger)
    
    if risco == "BAIXO" and impacto == "ISOLADO":
        return "SIAS_AUTONOMO"  # Rápido ou Médio
    
    elif risco == "MEDIO" or impacto == "MULTI_AGENTE":
        return "SIAS_DEBATE"  # 3 rodadas → SIAS
    
    elif risco == "ALTO" or impacto == "ARQUITETURAL":
        return "SIAS_DEBATE_HUMANO"  # Debate + aprovação Felipe
    
    else:
        return "HUMAN_NEEDED"  # Incerteza → escalar
```

### Matriz de Seleção de Ciclo

| CARACTERÍSTICA | SIAS AUTÔNOMO | SIAS DEBATE |
|----------------|---------------|-------------|
| Impacto | 1 ferramenta | Múltiplas ferramentas |
| Risco de reversão | < 5 min | > 5 min ou irreversível |
| Interface pública | Preservada | Pode ser afetada |
| Novo paradigma | Não | Sim |
| Tipo de mudança | Aditiva | Modificativa ou substitutiva |
| Aprovação humana | NÃO (SIAS R/M) | SIM se Tipo C/D |

**REGRA DE OURO:** Em dúvida sobre qual ciclo → DEBATE (mais conservador)

## Passo 4: Verificação Final Antes de Começar

- [ ] Guard rails de nível 0 verificados?
- [ ] Tipo de ciclo selecionado (autônomo ou debate)?
- [ ] Documento correto aberto para a etapa?
- [ ] Nenhum ciclo SIAS em conflito aberto?

## 6 Etapas do SIAS Autônomo (P2)

| Etapa | Responsável | Output | Checkpoint |
|-------|-------------|--------|------------|
| 1. DETECTAR | OpenClaw | detection_{N}.json | D1: hipótese com critério de falha? |
| 2. PESQUISAR | Hermes | research_{N}.md | P1: toda afirmação com URL? |
| 3. AVALIAR | OpenClaw | evaluation_{N}.md | A1: proposta rejeitada <14 dias? |
| 4. PROPOR | OpenClaw | proposal_{N}.md | P2: sucesso mensurável? |
| 5. TESTAR | Hermes | test_results_{N}.md | T1: output real executado? |
| 6. INTEGRAR | OpenClaw+Hermes | integration_log.md | I1: ciclo fechado no registry? |

## Invariantes do SIAS Autônomo

```
INVARIANTE 1 — UM CICLO POR VEZ:
  Nunca abrir SIAS-{N+1} enquanto SIAS-{N} está aberto.

INVARIANTE 2 — EVIDÊNCIA ANTES DE QUALQUER ETAPA:
  detection_{N}.json deve existir antes de [TASK:SIAS-P-{N}].
  research_{N}.md deve existir antes de evaluation_{N}.md.

INVARIANTE 3 — AMBIENTE ISOLADO OBRIGATÓRIO:
  Toda implementação acontece em sias_test/ — NUNCA em produção.

INVARIANTE 4 — APRENDIZADO REGISTRADO:
  Todo ciclo registra em ecosystem_patterns.md.
```

## Armadilhas

⚠️ **Pular o boot e ir direto para execução:** resultado com guard rails ignorados e contexto errado.

⚠️ **Misturar documentos:** usar template do SIAS_PROTOCOL em debate ou vice-versa causa confusão de etapas.

⚠️ **Assumir que o estado do sistema é o mesmo da última sessão:** sempre verificar `guardrail_state_check.py` primeiro.
