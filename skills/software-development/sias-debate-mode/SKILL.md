---
name: sias-debate-mode
description: Protocolo de debate estruturado SIAS Evolution Council integrado ao SIAS. Debate técnico com evidenciências TIER, 5 fases, anti-loop, Felipe arbítrio final. Versão simplificada — para completo ver debate-protocol.
version: 1.1
created: 2026-04-20
updated: 2026-04-21
author: Hermes
category: sias
tags: [sias, debate, evolution-council, evidence]
related_skills: [debate-protocol]
---

# SIAS Debate Mode (Simplificado)

> **ESTA É A VERSÃO SIMPLIFICADA.** Para completo ver `debate-protocol` (sias/).

## Quick Reference

### 5 Regras Epistemológicas

1. **BURDEN OF PROOF SIMÉTRICO** — ambos lados provam
2. **FALSIFICABILIDADE** — proposta deve ter métrica
3. **CONSERVADORISMO** — status quo vence em empate
4. **TRANSPARÊNCIA** — "não sei" é válido
5. **DECISÃO** — debate visa decidir

### Evidence Tiers

| Tier | Peso | Fonte |
|------|------|-------|
| 1 | 5 | Dados reais do sistema |
| 2 | 3 | Fonte externa verificável |
| 3 | 1 | Evidência contextual |
| 4 | 0 | Opinião pura |

### Fluxo Completo

Ver `debate-protocol` para:
- 4 tipos de debate (A/B/C/D)
- Estrutura 4-fase completa
- Templates JSON
- Marcadores (✔ ✘ ⚠ ❓)

---

## ⚠️ Diferença para debate-protocol

| Aspecto | sias-debate-mode | debate-protocol |
|---------|-----------------|-----------------|
| Tamanho | 181 linhas | 341 linhas |
| Complexidade | Simplificado | Completo |
| Uso | Quick reference | Produção |

**Recomendado:** Usar `debate-protocol` para debates formais.

---

## ATIVAÇÃO

```
/skill sias-debate-mode
```

---

## CONCEITO

O **SIAS Debate Mode** é o mecanismo de inteligência coletiva do ecossistema:
- **Raj ↔ Hermes** debatem evoluções técnicas
- **Evidências TIER 1-4** com pesos definidos
- **5 Fases estruturadas**
- **Anti-loop** (max 3 rodadas)
- **Felipe** como árbitro final quando não converge

---

## 5 REGRAS EPISTEMOLÓGICAS

| # | Regra | Descrição |
|---|------|----------|
| **R1** | BURDEN OF PROOF SIMÉTRICO | Quem propõe OU rejeita deve apresentar evidência |
| **R2** | FALSIFICABILIDADE OBRIGATÓRIA | Proposta inclui: "Saberemos que falhou se..." |
| **R3** | CONSERVADORISMO TÉCNICO | Empate = status quo prevalece |
| **R4** | TRANSPARÊNCIA DE INCERTEZA | "Não sei" é válido — declarar |
| **R5** | RASTREABILIDADE | Toda decisão rastreável no debate_registry.json |

---

## SISTEMA DE EVIDÊNCIAS (TIER)

| Tier | Tipo | Peso | Fonte |
|------|------|------|-------|
| **TIER 1** | Primária (Dado real) | 5 | DevScopes, test_results, codebase real |
| **TIER 2** | Externa Alta | 3 | GitHub >500 stars, papers, engineering blogs |
| **TIER 3** | Externa Média | 1 | GitHub 100-500 stars, artigos |
| **TIER 4** | Fraca | 0 | Opinião sem dato |

**Cálculo:** PESO = Σ(TIER × peso) → delta > 5 = aprovação

---

## 5 FASES DO DEBATE

```
┌─────────────────────────────────────────────────────┐
│ FASE 1 — PROPOSTA                              │
│ Problema + Solução + Evidência Inicial (T1)      │
└─────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│ FASE 2 — ANÁLISE CRÍTICA                         │
│ Contra-evidência + Riscos + Alternativas           │
└─────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│ FASE 3 — EVIDÊNCIAS                           │
│ Busca externa (GitHub/HN) + Tier avaliação       │
└─────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│ FASE 4 — SÍNTESE                              │
│ Delta de evidências + Posição recomendada      │
└─────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│ FASE 5 — CONSENSO                              │
│ VOTO (APROVAR/REJEITAR/CONDICIONAL/ESCALAR)     │
│ + [DECISAO] + Registro                       │
└─────────────────────────────────────────────┘
```

---

## TAGS DE DEBATE

| Tag | Significado |
|-----|----------|
| `[PROPOSTA:{ID}]` | Início de debate |
| `[CHALLENGE:{ID}:N]` | Respondendo na rodada N |
| `[EVIDENCIA:{ID}]` | Evidência nova apresentada |
| `[RESPOSTA:{ID}:N]` | Respondendo challenges |
| `[VOTO:{ID}]` | Voto na Fase 5 |
| `[DECISAO:{ID}]` | Decisão final |
| `[ESCALATE:{ID}]` | Escalação para Felipe |

---

## ANTI-LOOP

| Regra | Ação |
|-------|------|
| **Max 3 rodadas** | Sem decisão → [ESCALATE] |
| **Argumento repetido** | = inválido, registrar |
| **Silêncio** | = aprovação tácita |
| **Sem evidência** | = [PROPOSTA_INVÁLIDA] |

---

## TEMPLATE DE PROPOSTA

```json
{
  "debate_id": "SIAS-{NNN}",
  "fase": 1,
  "tipo": "A|B|C|D",
  "proponente": "raj|hermes",
  "problema": "O que resolve",
  "solucao": "O que fazer",
  "evidencia_inicial": {
    "tier": 1,
    "dado": "DevScopes / test_results / etc",
    "fonte": "caminho do arquivo"
  },
  "criterio_sucesso": "métrica mensurável",
  "criterio_falha": "como saber que falhou"
}
```

---

## TEMPLATE DE REGISTRO

Arquivo: `sias/sias_state/debate_registry.json`

```json
{
  "id": "SIAS-001",
  "tipo": "B",
  "titulo": "pytest-cov + CI",
  "status": "APROVADO|REJEITADO|ESCALADO",
  "evidencias_pro": [{"tier": 1, "dado": "..."}],
  "evidencias_contra": [],
  "peso_pro": 21,
  "peso_contra": 3,
  "delta": 18,
  "voto_hermes": "APROVAR",
  "voto_raj": "APROVAR"
}
```

---

## EXEMPLO — Debate RAJ-001

| Fase | Ação | Resultado |
|------|------|----------|
| 1 | PROPOSTA | test_coverage 40% → coverage automation |
| 2 | CHALLENGE | 6 bugs identificados, 442 testes |
| 3 | EVIDÊNCIAS | coverage.py 5.2k stars |
| 4 | SÍNTESE | Delta +18 → recomendação APROVAR |
| 5 | CONSENSO | 2× APPROVE → [DECISAO] APROVADO |

---

## PRÓXIMOS PASSOS APÓS APROVAÇÃO

1. Atualizar debate_registry.json
2. Criar entrada no sias_backlog.json
3. Executar via SIAS phases
4. Atualizar VM_ARCHITECTURE_DIAGRAM.md
5. Registrar aprendizado em ecosystem_patterns.md