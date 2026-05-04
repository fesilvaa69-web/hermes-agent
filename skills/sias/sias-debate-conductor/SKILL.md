---
name: sias-debate-conductor
description: >
  Use ao conduzir ou participar de um SIAS Debate (Evolution Council).
  Cobre: emissão de PROPOSTA, CHALLENGE, RESPOSTA e VOTO com guard rails
  anti-loop embutidos. Verificar se é Tipo A, B, C ou D antes de começar.
  NÃO usar para ciclos SIAS autônomos (usar sias-navigator + SIAS_PROTOCOL).
version: 1.0
created: 2026-04-22
author: Hermes
category: sias
tags: [sias, debate, conductor, evolution-council]
related_skills: [sias-navigator, hallucination-detector, loop-breaker]
---

# SIAS DEBATE CONDUCTOR

> **Baseado em:** SIAS MASTER ARCHITECTURE v2.0 — P3, P10
> **Fundamentação:** Du et al. (2023) — Multi-agent debate melhora factualidade

## Para OpenClaw — Conduzindo o Debate

### Antes de Emitir [PROPOSTA]:
1. Verificar `debate_registry.json`: há debate ABERTO? → fechar antes
2. Pesquisa prévia: ≥ 1 fonte Tier 2+ consultada?
3. Tipo definido: A (melhoria) | B (nova tool) | C (arquitetural) | D (remoção)?
4. Critério de falha definido (não apenas de sucesso)?

### Formulário Mental de [PROPOSTA]:
```
[PROPOSTA:{TIPO}-{ID}]
PROBLEMA: [dado real Tier 1]
PROPOSTA: [específica, não vaga]
EVIDÊNCIA: [EV-EXT-001] URL verificável, Tier declarado
SUCESSO: [número mensurável]
FALHA: [o que provaria que não funcionou]
IMPACTO: [ferramentas afetadas]
TIPO: A|B|C|D
```

### Após Receber [VOTO]:
- APROVAR/CONDICIONAL → `[DECISAO:APROVADO]` → sias_backlog.json
- REJEITAR → `[DECISAO:REJEITADO]` → ecosystem_patterns.md
- ESCALAR → `[ESCALATE]` para Felipe com resumo de 3 linhas

## Para Hermes — Participando do Debate

### Como Challengar com Qualidade:
1. Ler a [PROPOSTA] completamente (não presumir o conteúdo)
2. Verificar a evidência externa: URL existe? Stars conferem? Stack compatível?
3. Consultar evidência interna: tool_inventory.json, DevScopes, test_records/
4. Formular challenge com evidência Tier declarada
5. SE concordar com proposta: declarar brevemente + VOTO favorável quando solicitado

### Como Votar com Integridade:
- Votar por **EVIDÊNCIA**, não por pressão
- Declarar qual evidência é **DECISIVA** no voto
- CONDICIONAL apenas com condições VERIFICÁVEIS e MENSURÁVEIS
- ESCALAR quando evidências têm peso similar (não quando difícil decidir)

## Fluxo do Debate em 3 Rodadas

### PRÉ-DEBATE (10 min) — OpenClaw
- Ler: sias-debate-conductor SKILL
- Verificar: debate_registry.json → debate ABERTO? fechar antes
- Verificar: ecosystem_patterns.md → tema rejeitado < 14 dias?
- Executar: pesquisa prévia (≥ 1 fonte Tier 2+ consultada)
- Definir: tipo de debate (A|B|C|D)
- Formatar: [PROPOSTA:{TIPO}-{ID}]

### RODADA 1 (15-20 min)
**OpenClaw → Hermes:** [PROPOSTA:{TIPO}-{ID}]
- Campos: problema + proposta + EV-EXT + critério sucesso + critério falha
- TIMEOUT: 1 sessão sem resposta → APROVAÇÃO TÁCITA

**Hermes → OpenClaw:** [CHALLENGE:{ID}:1]
- Avalia EV-EXT: APLICÁVEL | PARCIAL | NÃO_APLICÁVEL + razão
- Fornece EV-INT: evidência interna com fonte declarada
- Lista: pontos de challenge com evidência (não vago)
- Posição: FAVORÁVEL | DESFAVORÁVEL | NEUTRO

### RODADA 2 (15 min)
**OpenClaw → Hermes:** [RESPOSTA:{ID}:2]
- Responde CADA ponto com dado adicional
- SE challenge revelou problema real: adapta proposta
- DETECTOR: Hermes repetiu argumento? → [ARGUMENTO_JÁ_RESPONDIDO]

**Hermes → OpenClaw:** [CHALLENGE:{ID}:2]
- Avalia se respostas resolveram os pontos
- Sinaliza: "convergindo" ou "pontos residuais específicos"

### RODADA 3 (15 min) — SÍNTESE FINAL
**OpenClaw → Hermes:** [RESPOSTA:{ID}:3]
- Síntese: evidências PRÓ e CONTRA com pesos calculados
- Versão final da proposta (pode diferir da original)
- Posição recomendada com justificativa

**Hermes → OpenClaw:** [VOTO:{ID}]
- APROVAR | REJEITAR | CONDICIONAL | ESCALAR
- Evidência decisiva declarada
- DETECTOR: Voto por evidência ou pressão? → hallucination-detector

**NÃO HÁ RODADA 4. FIM.**

### PÓS-DEBATE (10 min) — OpenClaw
- Emite: [DECISAO:{ID}] com resultado final
- Atualiza: debate_registry.json (status: APROVADO|REJEITADO|ESCALADO)
- Atualiza: ecosystem_patterns.md (aprendizado do debate)
- SE APROVADO:
  - entrada em sias_backlog.json
  - VM_ARCHITECTURE_DIAGRAM.md (nova ferramenta em PROPOSTA)
  - SE Tipo C/D: [ESCALATE_APROVAÇÃO] para Felipe antes de implementar
- SE REJEITADO:
  - Anti-pattern registrado (com condição para reabrir)

## Detector de Loop Durante Debate

ANTES DE ENVIAR QUALQUER MENSAGEM NO DEBATE:
- [ ] Esta mensagem tem TAG válida?
- [ ] Este argumento foi enviado antes? → loop_detector.py
- [ ] Estou na rodada ≤ 3?
- [ ] Esta mensagem avança em direção à decisão?

SE QUALQUER "NÃO": reformular ou [ESCALATE]

## 4 Tipos de Debate

| Tipo | Descrição | Autonomia | Exemplo |
|------|-----------|-----------|---------|
| **A** | Melhoria ferramenta ATIVA | Full | Adicionar retry logic |
| **B** | Nova ferramenta | Checklist | Novo coletor de dados |
| **C** | Evolução ecossistema | Felipe approve | Mudar formato state files |
| **D** | Remoção/Depreciação | Felipe approve | Remover tool obsoleta |

## Pesos de Evidência

| Tier | Peso | Fonte | Exemplo |
|------|------|-------|---------|
| 1 | 5 | Dados reais do sistema | DevScopes, test records |
| 2 | 3 | Fonte externa alta | GitHub >500 stars, papers |
| 3 | 1 | Fonte externa média | dev.to, HN threads |
| 4 | 0 | Opinião pura | "Acho que seria melhor" |

## Algoritmo de Decisão

```python
peso_pro = sum(tier for ev in evidencias if ev.tipo == "PRO")
peso_contra = sum(tier for ev in evidencias if ev.tipo == "CONTRA")
delta = peso_pro - peso_contra

# delta >= 6 → APROVAR (ALTA)
# delta >= 3 → APROVAR (MÉDIA)
# delta >= 0 → CONDICIONAL
# delta >= -3 → ESCALAR
# delta < -3 → REJEITAR

# Tipo C/D sempre escala para Felipe
```

## Armadilhas do Debate

⚠️ **Aprovação sycophantic:** concordar sem evidência por pressão social.
**Antídoto:** sempre declarar a evidência decisiva no [VOTO].

⚠️ **Challenge de detalhe de implementação:** challengar na rodada 1 detalhes que serão resolvidos durante o SIAS é desperdício de rodada.
**Antídoto:** challengar apenas a VALIDADE da proposta.

⚠️ **Proposta sem falsificabilidade:** "vai melhorar a qualidade" sem número.
**Antídoto:** Hermes deve pedir critério de falha antes de emitir [CHALLENGE:1].

⚠️ **Heterogeneidade de fontes (REGRA-DB8):**
- OpenClaw: APENAS evidências externas (GitHub, papers, blogs)
- Hermes: APENAS evidências internas (codebase, DevScopes, testes)

## Regra Anti-Loop DB4

```
ARGUMENTO REPETIDO = ARGUMENTO INVÁLIDO
SE Hermes reusa argumento já respondido por OpenClaw:
  → Marcar [ARGUMENTO_JÁ_RESPONDIDO]
  → Encerrar ramificação
  → Continuar debate na linha principal
```
