---
name: debate-protocol
description: Protocolo de Debate Técnico Estruturado Hermes ↔ Raj — Fase D' do SIAS. Inclui 5 regras epistemológicas, 4 tipos de debate, estrutura 4-fase, e templates JSON.
version: 1.1
created: 2026-04-20
updated: 2026-04-21
author: Hermes
category: sias
tags: [sias, debate, hermes, raj, coordination, evidence]
related_skills: [sias-canary-scan, sias-cycle-execution, vm-diagram-maintenance]
---

# Debate Técnico Estruturado — Protocolo Hermes ↔ Raj

## Quando Usar Esta Skill

- SIAS ciclo identificado melhoria de arquitetura
- Nova ferramenta sendo proposta
- Evolução de ecossistema discutida
- Remoção/depreciação de ferramenta

## Filosofia

**Debate = ciência, não retórica.** Cada argumento precisa de evidência verificável.

### 5 Regras Epistemológicas

```
1. BURDEN OF PROOF SIMÉTRICO
   - Quem propõe E quem rejeita têm obrigação de evidência
   - silêncio = "não sei" é resposta válida

2. FALSIFICABILIDADE OBRIGATÓRIA
   - Proposta deve dizer como saberemos que falhou
   - Sem métrica mensurável = [REJEITADA]

3. CONSERVADORISMO TÉCNICO
   - Empate de evidências = status quo vence
   - Custo de reverter > Custo de não implementar

4. TRANSPARÊNCIA DE INCERTEZA
   - "Não sei" é resposta válida
   - Não é fraqueza — é integridade epistêmica

5. DECISÃO, NÃO DISCUSSÃO
   - Debate visa decidir, não persuadir
   - Opinião sem dado = Tier 0 (peso 0)
```

## Hierarquia de Evidências

| Tier | Peso | Fonte | Exemplo |
|------|------|-------|---------|
| 1 | 5 | Dados reais do sistema | DevScopes error_rate 8% |
| 2 | 3 | Fonte externa verificável | GitHub >500 stars, paper |
| 3 | 1 | Evidência contextual | "Funciona em outro projeto" |
| 4 | 0 | Opinião pura | "Acho que seria melhor" |

## 4 Tipos de Debate

### Tipo A — Melhoria Ferramenta ATIVA
- **Autonomia:** Total — Hermes implementa após debate
- **Trigger:** ICE Score > 7
- **Fluxo:** Proposta → Challenge (2 rodadas) → Hermes decide

### Tipo B — Nova Ferramenta
- **Autonomia:** Parcial — requer checklist
- **Checklist obrigatório:**
  ```
  □ tool_inventory.json: 0 sobreposições >70%
  □ Evidência necessidade: >3 tarefas manuais para suprir gap
  □ Evidência externa: >2 repositórios GitHub similares
  ```

### Tipo C — Evolução Ecossistema
- **Autonomia:** ZERO — Felipe approve obrigatório
- **Exemplos:** Mudança de formato state files, novo agente

### Tipo D — Remoção/Depreciação
- **Autonomia:** ZERO — Felipe approve obrigatório
- **Requires:** BM25 score = 0, última modificação >90 dias

### Guard Rails Anti-Loop (v2.0)

```
REGRA MÁXIMA: 3 RODADAS — NÃO HÁ RODADA 4

TURNO 1 → OpenClaw propõe + evidência
TURNO 2 → Hermes challenge + evidência  
TURNO 3 → OpenClaw resposta + [VOTO]

Se 3 rodadas sem decisão → [ESCALATE_TO_FELIPE]
```

```
ARGUMENTO REPETIDO = INVÁLIDO
- Marcar [ARGUMENTO_JÁ_RESPONDIDO]
- Não entra na contagem de rodadas
```

## Algoritmo de Decisão

```python
peso_pro = sum(ev.tier for ev in evidencias if ev.tipo == "PRO")
peso_contra = sum(ev.tier for ev in evidencias if ev.tipo == "CONTRA")
delta = peso_pro - peso_contra
forca = abs(delta) / max(peso_pro, peso_contra, 1)

if delta > 3 AND forca > 0.6:
    return "IMPLEMENTAR"
elif delta < -3 AND forca > 0.6:
    return "REJEITAR"
else:
    return "EMPATE"
```

## Fontes por Tema

| Tema | Fontes |
|------|--------|
| Trading | arxiv.org/q-fin, SSRN, Zipline blog |
| ML | chiphuyen.com, proceedings.mlr.press |
| SRE | Google SRE Book, honeycomb.io |
| Dados | DDIA (Kleppmann), Kafka docs, SQLite docs |
| HN | hn.algolia.com/api/v1/search (sem rate limit) |

## Uso do DebateEngine

```python
from sias.debate_engine import DebateEngine, DebateTipo, TipoEvidencia, Voto

engine = DebateEngine()

# Criar debate
debate = engine.criar_debate(
    tema="Adicionar retry logic ao radar_scheduler",
    tipo=DebateTipo.TIPO_A,
    ciclo_sias="SIAS-034"
)

# Adicionar evidências
engine.adicionar_evidencia(
    debate_id=debate.debate_id,
    agente="HERMES",
    tipo_evidencia=TipoEvidencia.PRO,
    conteudo="error_rate de 8% no DevScopes",
    tier=1,
    fonte="devscopes:radar_scheduler:2026-04-20"
)

# Adicionar rodada
engine.adicionar_rodada(
    debate_id=debate.debate_id,
    agente="HERMES",
    tipo="PROPOSTA",
    conteudo="Proponho adicionar tenacity retry"
)

# Calcular decisão
resultado = engine.calcular_decisao(debate.debate_id)
```

## Output: debate_registry.json

```json
{
  "debate_id": "DEBATE-HER-001",
  "ciclo_sias": "SIAS-034",
  "tipo": "A",
  "tema": "Adicionar retry logic",
  "status": "CONCLUIDO",
  "peso_pro": 8.0,
  "peso_contra": 5.0,
  "delta": 3.0,
  "forca": 0.62,
  "voto_hermes": "APROVADO",
  "voto_raj": "APROVADO",
  "decisao": "IMPLEMENTAR"
}
```

## Critério de Done

```
□ Debate com 3 rodadas OU decisão tomada
□ Todos argumentos marcados com Tier
□ Peso calculado (peso_pro, peso_contra, delta, forca)
□ Voto de cada agente registrado
□ Decisão final: IMPLEMENTAR | REJEITAR | EMPATE→FELIPE
□ debate_registry.json atualizado
□ Se Tipo C ou D: Felipe notificado
```

## Armadilhas a Evitar

1. **Debate sobre ferramenta não utilizada** — só debater ferramentas ativas
2. **Feature creep** — mínimo necessário, não máxima perfeição
3. **Ignorar patterns aprendidos** — verificar registry antes de propor
4. **Evidência sem fonte** — toda evidência precisa de origem

---

# ANEXO — Templates JSON (Fases 1-4)

## Fase 1 — Template Proposta

```json
{
  "debate_id": "DH-{NNN}",
  "fase": 1,
  "proponente": "hermes|raj",
  "tipo": "new_tool|tool_improvement|refactor",
  "ferramenta_alvo": {
    "nome": "nome_da_ferramenta.py",
    "caminho": "~/.openclaw/workspace/{agent}/",
    "tool_id_atual": "T-XX-NNN"
  },
  "problema": {
    "descricao": "Problema que a ferramenta resolve",
    "impacto": "high|medium|low",
    "casos_uso": ["caso1", "caso2"]
  },
  "solucao": {
    "descricao": "Solução proposta",
    "escopo": "mínimo|completo",
    "ownership": "hermes|raj"
  },
  "evidencia": [
    {
      "tipo": "tier_1|tier_2|tier_3|tier_4",
      "fonte": "descrição da fonte",
      "relevancia": "high|medium|low"
    }
  ]
}
```

## Fase 2 — Template Análise Crítica

```json
{
  "debate_id": "DH-{NNN}",
  "fase": 2,
  "analista": "raj|hermes",
  "desafios": [
    {
      "desafio": "Descrição do desafio",
      "tipo": "technical|deps|scope|resourcing",
      "bloqueante": true|false
    }
  ],
  "riscos": [
    {
      "risco": "Descrição do risco",
      "probabilidade": "high|medium|low",
      "impacto": "high|medium|low",
      "mitigacao": "Plano de mitigação"
    }
  ],
  "alternativas": [
    {
      "alternativa": "Nome da alternativa",
      "descricao": "Como funcionária",
      "prons": ["pro1"],
      "contras": ["contra1"],
      "viabilidade": "high|medium|low"
    }
  ],
  "concordancia": ["Item 1 com que o analista concorda"],
  "objeçoes": [
    {
      "objeçao": "Descrição da objeção",
      "tipo": "technical|deps|scope|resourcing",
      "bloqueante": true|false,
      "resoluçao_sugerida": "Como resolver"
    }
  ],
  "condicionantes": ["Condição que deve ser satisfeita para apoio"]
}
```

## Marcadores de Posição

| Marcador | Significado | Ação |
|----------|-------------|------|
| ✔ CONCORDÂNCIA | Concorda com a proposta | Adoptar na síntese |
| ✘ OBJEÇÃO | Discordância técnica | Explicitar divergência e propor resolução |
| ⚠️ CONDICIONANTE | Apoio condicional | Incluir como requisito na síntese |
| ❓ QUESTÃO | Precisa clarification | Solicitar informação adicional |

## Fase 3 — Síntese

```json
{
  "debate_id": "DH-{NNN}",
  "fase": 3,
  "sintese": {
    "problema_concordado": "Descricao",
    "solucao_consenso": "Descricao",
    "escopo": "mínimo|completo",
    "riscos_mitigados": ["risco1"],
    "condicionantes_resolvidas": ["cond1"]
  },
  "ownership": "hermes|raj",
  "timeline": {
    "fase_1": "data",
    "fase_2": "data"
  }
}
```

## Fase 4 — Compromisso

```json
{
  "debate_id": "DH-{NNN}",
  "fase": 4,
  "decisao": "IMPLEMENTAR|REJEITAR|EMPATE",
  "voto_hermes": "APROVADO|REJEITADO|ABSTENCAO",
  "voto_raj": "APROVADO|REJEITADO|ABSTENCAO",
  "delta": 3.0,
  "forca": 0.62,
  "ticket_sias": "SIAS-XXX",
  "action_items": [
    {
      "item": "Descrição",
      "owner": "hermes|raj",
      "prazo": "YYYY-MM-DD"
    }
  ]
}
```

---

## Skills Relacionadas

- `sias-canary-scan` — Phase A do SIAS
- `sias-cycle-execution` — 6-fase testing workflow
- `vm-diagram-maintenance` — Manter VM diagram atualizado
5. **3 rodadas sem decisão e não escalar** — sempre escalar para Felipe
