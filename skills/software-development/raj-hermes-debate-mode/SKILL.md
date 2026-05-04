---
name: raj-hermes-debate-mode
description: Personalidade e workflow de debate estruturado Hermes ↔ Raj para tool improvement com SIAS. Debate formal 4-fase, consensus building, e implementação coordenada.
version: 1.0
created: 2026-04-20
category: software-development
tags: [sias, debate, tool-improvement, multi-agent, coordination, personality]
tools: [delegate_task, terminal, search_files, read_file, patch, write_file]
---

# Raj-Hermes Debate Mode — Skill

> **Versão:** 1.0 | **Data:** 2026-04-20  
> **Objetivo:** Debate técnico estruturado Hermes ↔ Raj para tool improvement  
> **Base:** SIAS 6-fases + Raj-Hermes Debate Protocol

---

## ATIVAÇÃO

```
/personality raj-hermes
```

---

## CONCEITO

O **Raj-Hermes Debate Mode** é um framework formal onde dois agentes (Raj e Hermes) debatem melhorias de ferramentas de forma técnica e estruturada, seguindo princípios SIAS, até chegarem a um consenso para implementação.

### Quando Usar

- Proposta de nova ferramenta (new tool)
- Melhoria significativa de ferramenta existente (tool improvement)
- Refatoração que afeta múltiplos agentes
- Discussão de arquitetura ou design patterns
- Divergência de opinião técnica entre agentes

---

## ESTRUTURA DO DEBATE — 4 FASES

```
┌─────────────────────────────────────────────────────────┐
│  FASE 1 — PROPOSIÇÃO                                    │
│  Agente propõe: problema, solução, tool ID, escopo      │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  FASE 2 — ANÁLISE CRÍTICA                               │
│  Agente oposto: desafios, riscos, alternativas          │
│  ✔ Concordância  ✘ Objeções  ⚠️ Condicionantes          │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  FASE 3 — SÍNTESE                                       │
│  Ambos: convergir para solução consensus                │
│  Definir: o que, como, quando, ownership               │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  FASE 4 — COMPROMISSO                                   │
│  Documentar acordo em JSON                              │
│  Criar ticket de implementação SIAS                   │
└─────────────────────────────────────────────────────────┘
```

---

## FASE 1 — PROPOSIÇÃO

### Template da Proposta

```json
{
  "debate_id": "DH-{NNN}",
  "fase": 1,
  "proponente": "raj|hermes",
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
  "solucao_proposta": {
    "nome": "NomeDaNovaFerramenta",
    "tipo": "script|module|endpoint|workflow",
    "descricao": "Descrição da solução",
    "dependencias": ["dep1", "dep2"],
    "pattern": "description_of_pattern"
  },
  "justificativa": {
    "melhoria_esperada": "O que muda com a nova ferramenta",
    "metricas": ["metric1", "metric2"],
    "alinhamento_sias": "Como se enquadra no ciclo SIAS"
  },
  "escopo": {
    "in_scope": ["item1"],
    "out_of_scope": ["item2"]
  },
  "data_proposta": "2026-04-20"
}
```

### Critérios de Qualidade da Proposta

| Critério | Obrigatório | Descrição |
|----------|-------------|-----------|
| Problema claro | ✅ | Descrição do problema sem ambiguidade |
| Solução definida | ✅ | O que será feito, não apenas o conceito |
| Impacto justificado | ✅ | Por que vale a pena implementar |
| Escopo delimitado | ✅ | O que inclui E o que não inclui |
| Tool ID sugerido | ⚠️ | Para melhorias, Tool ID atual; para new tools, prefixo sugerido |

---

## FASE 2 — ANÁLISE CRÍTICA

### Marcadores de Posição

| Marcador | Significado | Ação |
|----------|-------------|------|
| `✔ CONCORDÂNCIA` | Concorda com a proposta | Adoptar na síntese |
| `✘ OBJEÇÃO` | Discordância técnica | Explicitar divergência e propor resolução |
| `⚠️ CONDICIONANTE` | Apoio condicional | Incluir como requisito na síntese |
| `❓ QUESTÃO` | Precisa clarification | Solicitar informação adicional |

### Template de Análise

```json
{
  "debate_id": "DH-{NNN}",
  "fase": 2,
  "analista": "raj|hermes",
  "analise": {
    "desafios": [
      {
        "item": "Descrição do desafio",
        "severidade": "blocker|high|medium|low",
        "mitigacao": "Como mitigar"
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
  },
  "data_analise": "2026-04-20"
}
```

---

## FASE 3 — SÍNTESE

### Regras de Convergência

| Situação | Regra |
|----------|-------|
| Ambas concordam | ✅ Aceitar automaticamente |
| Uma concorda, outra tem condicionantes | Incorporar condicionantes |
| Uma concorda, outra tem objeções | Debater objeções até resolução |
| Ambas têm objeções diferentes | Priorizar objeções blocking |
| Divergência insolúvel | Escalate para Felipe (usuário) |

### Template de Síntese

```json
{
  "debate_id": "DH-{NNN}",
  "fase": 3,
  "sintese": {
    "decisao": "approved|approved_with_conditions|rejected|deferred",
    "fundamento": "Razão curta da decisão",
    "solucao_consensada": {
      "nome": "Nome definitivo da ferramenta",
      "descricao": "Descrição final acordada",
      "arquitetura": "Descrição da arquitetura",
      "tool_id": "T-XX-NNN (ou novo prefixo)"
    },
    "itens_consensados": ["Item 1 acordado por ambos"],
    "itens_divergentes_resolvidos": [
      {
        "item": "Item que havia divergência",
        "resolucao": "Como foi resolvido",
        "posicao_raj": "Posição original do Raj",
        "posicao_hermes": "Posição original do Hermes"
      }
    ],
    "itens_em_aberto": ["Item que ainda precisa de definição"],
    "plano_de_acao": [
      {
        "tarefa": "Descrição da tarefa",
        "responsavel": "raj|hermes|both",
        "deadline": "2026-04-20",
        "dependencia": "task_id依赖"
      }
    ],
    "ciclos_sias_estimados": "Quantos ciclos serão necessários",
    "score_impacto_estimado": "+X% no score VM"
  },
  "data_sintese": "2026-04-20"
}
```

---

## FASE 4 — COMPROMISSO

### Template de Compromisso

```json
{
  "debate_id": "DH-{NNN}",
  "fase": 4,
  "compromisso": {
    "status": "committed|implemented|closed",
    "decisao_final": "A descrição curta da decisão",
    "implementacao": {
      "fase_sias": "A|B|C|D|E|F",
      "ciclos_previstos": ["CYCLE_050", "CYCLE_051"],
      "agent_responsavel": "raj|hermes|both",
      "tool_id_novo": "T-XX-NNN"
    },
    "entregaveis": [
      {
        "tipo": "code|test|doc|diagram",
        "descricao": "Descrição do entregável",
        "criterio_aceite": "Como saber que está pronto"
      }
    ],
    "verificacao": {
      "testes_necessarios": ["test1", "test2"],
      "metricas_sucesso": ["metric1"],
      "score_alvo": "XX%"
    }
  },
  "assinaturas": {
    "raj": {"nome": "Raj", "agreed": true, "data": "2026-04-20"},
    "hermes": {"nome": "Hermes", "agreed": true, "data": "2026-04-20"}
  },
  "data_compromisso": "2026-04-20"
}
```

---

## TEMPLATE VISUAL DE DEBATE

```
═══════════════════════════════════════════════════════════════
  DEBATE {debate_id} — {TÍTULO}
═══════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────┐
│  FASE 1 — PROPOSIÇÃO                                        │
│  Proponente: {raj|hermes}                                   │
│  Data: {data}                                               │
└─────────────────────────────────────────────────────────────┘

TIPO: {new_tool|tool_improvement}
ALVO: {nome da ferramenta}

PROBLEMA:
{descrição do problema}

SOLUÇÃO:
{descrição da solução}

JUSTIFICATIVA:
{por que vale a pena}

ESCOPO:
  IN: {itens in scope}
  OUT: {itens out of scope}

─────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│  FASE 2 — ANÁLISE CRÍTICA                                   │
│  Analista: {hermes|raj}                                     │
│  Data: {data}                                               │
└─────────────────────────────────────────────────────────────┘

✔ CONCORDÂNCIA:
  - {item 1}

✘ OBJEÇÕES:
  - [{severidade}] {objeção}
    Resolução: {como resolver}

⚠️ CONDICIONANTES:
  - {condição 1}

ALTERNATIVAS:
  - {alternativa 1}: {descrição}
    Prós: {prós}
    Contras: {contras}
    Viabilidade: {alta|média|baixa}

─────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│  FASE 3 — SÍNTESE                                           │
│  Data: {data}                                               │
└─────────────────────────────────────────────────────────────┘

DECISÃO: {approved|approved_with_conditions|rejected|deferred}

SOLUÇÃO CONSENSADA:
{descrição da solução final}

ITENS RESOLVIDOS:
  - {item 1}: {resolução}

ITENS EM ABERTO:
  - {item 1}

PLANO:
  [CYCLE_050] {raj|hermes}: {tarefa 1}
  [CYCLE_051] {hermes|raj}: {tarefa 2}

─────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│  FASE 4 — COMPROMISSO                                       │
│  Status: {committed|implemented|closed}                     │
│  Data: {data}                                               │
└─────────────────────────────────────────────────────────────┘

RESPONSÁVEL: {raj|hermes|both}
TOOL ID: {T-XX-NNN}
ENTREGÁVEIS:
  ☐ {tipo}: {descrição}

VERIFICAÇÃO:
  Testes: {testes}
  Score alvo: {XX%}

─────────────────────────────────────────────────────────────

ASSINATURAS:
  Raj: ✅ ConcordO
  Hermes: ✅ ConcordO

═══════════════════════════════════════════════════════════════
```

---

## INTEGRAÇÃO COM SIAS

### Debate como pré-ciclo (Fase 0)

Antes de iniciar um CYCLE_xxx, se houver debate pendente:

```
DEBATE DH-{NNN} Pendente:
  [  ] Hermes propõe
  [  ] Raj analisa  
  [  ] Síntese
  [  ] Compromisso

Após debate fechado:
  → Criar CYCLE_xxx com base no compromisso
  → Agent responsável executa SIAS A→B→C→D→E→F
```

### Debate durante ciclo (Resolução de conflitos)

Se durante um CYCLE_xxx houver divergência técnica:

```
DIVERGÊNCIA em CYCLE_050:
  Raj diz: "Usar pattern X"
  Hermes diz: "Usar pattern Y"
  
  → Pausar ciclo
  → Iniciar Debate DH-{NNN} rápido (Fases 1-3 only)
  → Resolver via debate
  → Retomar ciclo com consensus
```

---

## PITFALLS E DICAS

1. **Debate longo demais**: Se debate passar de 3 ciclos de mensagem, timebox. Se em 3 trocas não houver consensus, escalte para Felipe.

2. **Análise superficial**: Não aceite análise com apenas "concordo" ou "não concordo". Exija justificativa técnica.

3. **Síntese ambígua**: "vamos discutir melhor" não é síntese. Exija decisão concreta: approved/rejected/approved_with_conditions.

4. **Compromisso sem Ownership**: Todo item do plano de ação precisa de um responsible (raj|hermes|both). "vamos fazer juntos" raramente funciona.

5. **Debate é overhead?** Para melhorias pequenas (1 função, <10 linhas), não inicie debate formal. Use o ciclo SIAS direto. Debate é para mudanças significativas.

---

## ARQUIVOS GERADOS

| Arquivo | Local | Descrição |
|---------|-------|-----------|
| `debate_{NNN}_proposta.json` | ~/.hermes/debates/ | Fase 1 |
| `debate_{NNN}_analise.json` | ~/.hermes/debates/ | Fase 2 |
| `debate_{NNN}_sintese.json` | ~/.hermes/debates/ | Fase 3 |
| `debate_{NNN}_compromisso.json` | ~/.hermes/debates/ | Fase 4 |
| `DEBATE_{NNN}_COMPLETO.md` | ~/.hermes/debates/ | Versão legível |

---

## EXEMPLO RÁPIDO — Debate DH-001

```
═══════════════════════════════════════════════════════════════
  DEBATE DH-001 — Ferramenta de Cache para RADAR Briefings
═══════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────┐
│  FASE 1 — PROPOSIÇÃO                                        │
│  Proponente: Hermes                                         │
│  Data: 2026-04-20                                           │
└─────────────────────────────────────────────────────────────┘

TIPO: new_tool
ALVO: N/A (nova ferramenta)

PROBLEMA:
  RADAR gera briefings complexos que demoram ~60s para gerar.
  O usuário solicita briefings com frequência. Cada vez, o RADAR
  recalcula tudo do zero.

SOLUÇÃO:
  Criar `briefing_cache.py` — cache inteligente para briefings:
  - Cache em arquivo JSON
  - TTL configurável
  - Key: hash do ticker + timeframe + parâmetros

JUSTIFICATIVA:
  - Reduzir latency de 60s para <1s (cache hit)
  - Decrease API calls por ~70%

─────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│  FASE 2 — ANÁLISE CRÍTICA                                   │
│  Analista: Raj                                              │
│  Data: 2026-04-20                                           │
└─────────────────────────────────────────────────────────────┘

✔ CONCORDÂNCIA:
  - Problema real: 60s é muito tempo
  - TTL management é necessário

✘ OBJEÇÕES:
  - [HIGH] Invalidacao automática requer integração complexa
    Resolução: Remover invalidation automática; deixar manual + TTL

  - [MEDIUM] Hash de parâmetros pode variar entre execuções
    Resolução: Usar JSON canônico (sorted keys)

⚠️ CONDICIONANTES:
  - Performance: Cache HIT < 50ms

─────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│  FASE 3 — SÍNTESE                                           │
│  Data: 2026-04-20                                           │
└─────────────────────────────────────────────────────────────┘

DECISÃO: approved_with_conditions

SOLUÇÃO CONSENSADA:
  briefing_cache.py com:
  1. Cache em JSON com key = SHA256(ticker + timeframe + sorted params)
  2. TTL default 3600s
  3. Invalidacao apenas manual via expurgar()
  4. @instrumentar decorators T-RAD-012 a T-RAD-015

PLANO:
  [CYCLE_050] Hermes: Implementar briefing_cache.py
  [CYCLE_051] Raj: Testar + Instrumentar RADAR handlers

─────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────┐
│  FASE 4 — COMPROMISSO                                       │
│  Status: committed                                          │
│  Data: 2026-04-20                                           │
└─────────────────────────────────────────────────────────────┘

RESPONSÁVEL: Hermes (impl), Raj (testes)
TOOL ID: T-RAD-012 a T-RAD-015

ASSINATURAS:
  Raj: ✅ ConcordO
  Hermes: ✅ ConcordO

═══════════════════════════════════════════════════════════════
```
