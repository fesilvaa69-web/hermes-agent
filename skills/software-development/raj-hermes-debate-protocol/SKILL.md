---
name: raj-hermes-debate-protocol
description: Protocolo estruturado de debate técnico entre Raj e Hermes para tool improvement usando SIAS. Debate formal 4-fase, consensus building, e implementação coordenada.
version: 1.0
created: 2026-04-20
category: software-development
tags: [sias, debate, tool-improvement, multi-agent, coordination]
tools: [delegate_task, terminal, search_files, read_file, patch, write_file]
---

# Raj-Hermes Debate Protocol — Tool Improvement

> **Versão:** 1.0 | **Data:** 2026-04-20  
> **Objetivo:** Debate técnico estruturado entre Raj e Hermes para tool improvement  
> **Base:** SIAS 6-fases + consensus building

---

## CONCEITO

O **Raj-Hermes Debate Protocol** é um framework formal onde dois agentes (Raj e Hermes) debatem melhorias de ferramentas de forma técnica e estruturada, seguindo princípios SIAS, até chegarem a um consenso para implementação.

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
│  Definir: o que, como, quando, ownership                │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  FASE 4 — COMPROMISSO                                   │
│  Documentar acordo em JSON                              │
│  Criar ticket de implementação SIAS                    │
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
    "concordancia": [
      "Item 1 com que o analista concorda"
    ],
    "objeçoes": [
      {
        "objeçao": "Descrição da objeção",
        "tipo": "technical|deps|scope|resourcing",
        "bloqueante": true|false,
        "resoluçao_sugerida": "Como resolver"
      }
    ],
    "condicionantes": [
      "Condição que deve ser satisfeita para apoio"
    ]
  },
  "data_analise": "2026-04-20"
}
```

### Marcadores de Posição

| Marcador | Significado | Ação |
|----------|-------------|------|
| `✔ CONCORDÂNCIA` | Concorda com a proposta | Adoptar na síntese |
| `✘ OBJEÇÃO` | Discordância técnica | Explicitar divergência e propor resolução |
| `⚠️ CONDICIONANTE` | Apoio condicional | Incluir como requisito na síntese |
| `❓ QUESTÃO` | Precisa clarification | Solicitar informação adicional |

### Processo de Análise

```
1. Analista recebe proposta (Fase 1)
2. Analista identifica:
   ├── Desafios técnicos (complexidade, deps, integração)
   ├── Riscos (breaking changes, performance, security)
   ├── Alternativas (outras abordagens possíveis)
   ├── Concordâncias (o que funciona bem)
   ├── Objeções (o que não funciona e por quê)
   └── Condicionantes (o que precisa mudar para apoiar)
3. Analista responde com análise completa (Fase 2)
```

---

## FASE 3 — SÍNTESE

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
    "itens_consensados": [
      "Item 1 acordado por ambos"
    ],
    "itens_divergentes_resolvidos": [
      {
        "item": "Item que havia divergência",
        "resolucao": "Como foi resolvido",
        "posicao_raj": "Posição original do Raj",
        "posicao_hermes": "Posição original do Hermes"
      }
    ],
    "itens_em_aberto": [
      "Item que ainda precisa de definição"
    ],
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

### Processo de Síntese

```
1. Ambos agentes recebem análises (Fase 2)
2. Identificam pontos de convergência
3. Resolvem divergências via:
   ├── Evidência (benchmark, docs, testes)
   ├── Compromisso (cada lado cede parcialmente)
   └── Escolha técnica (melhor solução wins)
4. Documentam consenso
5. Criam plano de ação
```

### Regras de Convergência

| Situação | Regra |
|----------|-------|
| Ambas concordam | ✅ Aceitar automaticamente |
| Uma concorda, outra tem condicionantes | Incorporar condicionantes |
| Uma concorda, outra tem objeções | Debater objeções até resolução |
| Ambas têm objeções diferentes | Priorizar objeções blocking |
| Divergência insolúvel | Escalate para Felipe (usuário) |

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

## TEMPLATE DE DEBATE COMPLETO

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
│  Analista: {hermes|raj}                                    │
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
│  Status: {committed|implemented|closed}                      │
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

## EXEMPLO PRÁTICO — Debate DH-001

### FASE 1 — Proposta (Hermes → Raj)

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
  O usuário solicita briefings com frequência (diário, pré-market,
  pós-market). Cada vez, o RADAR recalcula tudo do zero.
  Isso causa:
  - Tempo de espera longo para o usuário
  - Uso ineficiente de recursos (CPU/GPU)
  - Rate limiting da API de briefing

SOLUÇÃO:
  Criar `briefing_cache.py` — uma ferramenta de cache inteligente
  para briefings RADAR:
  
  - Cache em arquivo JSON (~/.openclaw/workspace-radar/cache/)
  - TTL configurável (default: 1h para pré-market, 4h para diário)
  - Key: hash do ticker + timeframe + parâmetros
  - Invalidação: manual ou automática (novo candle detectado)
  - Prefetch: gera próximo briefing em background

JUSTIFICATIVA:
  - Reduzir latency de briefing de 60s para <1s (cache hit)
  - Decrease API calls por ~70%
  - Melhorar UX significativamente
  - Alinhado com SIAS: ferramenta reutilizável entre ciclos

ESCOPO:
  IN: 
    - Cache de briefings em JSON
    - TTL management
    - Cache invalidation manual
    - HIT/MISS logging
  OUT:
    - Cache distribuído (Redis, etc.)
    - Cache de indicadores individuais
    - Prefetch automático

─────────────────────────────────────────────────────────────
```

### FASE 2 — Análise (Raj → Hermes)

```
┌─────────────────────────────────────────────────────────────┐
│  FASE 2 — ANÁLISE CRÍTICA                                   │
│  Analista: Raj                                              │
│  Data: 2026-04-20                                           │
└─────────────────────────────────────────────────────────────┘

✔ CONCORDÂNCIA:
  - Problema real: 60s é muito tempo para briefing
  - Abordagem de cache em arquivo é simples e efetiva
  - TTL management é necessário

✘ OBJEÇÕES:
  - [HIGH] Cache invalidation por "novo candle" requer integração
    com data monitor — complexidade adicional não escopada
    Resolução: Remover invalidation automática; deixar apenas
    manual + TTL

  - [MEDIUM] Hash de parâmetros precisa ser estável entre
    execuções (ordem de keys no dict pode variar em Python <3.7)
    Resolução: Usar JSON canônico (sorted keys) ou fingerprint
    explícito

  - [LOW] Log de HIT/MISS deveria usar @instrumentar para
    rastreabilidade SIAS
    Resolução: Adicionar decorators T-RAD-0XX às funções de cache

⚠️ CONDICIONANTES:
  - Dependência: Depende de briefing_inteligente.py estar
    instrumentado e testado (CYCLE 045-049)
  - Performance: Cache hit em <50ms; Cache miss não pode ser
    mais lento que gerar briefing original

ALTERNATIVAS:
  - Redis como backend: Mais robusto para produção
    Prós: Persistência, TTL nativo, cluster-ready
    Contras: Requer Redis installed; adiciona dependência externa
    Viabilidade: LOW (ambiente atual não tem Redis)
  
  - SQLite como backend: Mais portável que Redis
    Prós: Zero deps, queries eficientes
    Contras: Overhead de SQLite para arquivos pequenos
    Viabilidade: MEDIUM

─────────────────────────────────────────────────────────────
```

### FASE 3 — Síntese (Ambos)

```
┌─────────────────────────────────────────────────────────────┐
│  FASE 3 — SÍNTESE                                           │
│  Data: 2026-04-20                                           │
└─────────────────────────────────────────────────────────────┘

DECISÃO: approved_with_conditions

SOLUÇÃO CONSENSADA:
  Criar `briefing_cache.py` com:
  
  1. Cache em arquivo JSON (~/.openclaw/workspace-radar/cache/)
  2. Key = hash SHA256 de (ticker + timeframe + sorted params)
  3. TTL em segundos (parametrizável, default 3600)
  4. Invalidation: apenas manual via função expurgar()
  5. HIT/MISS logging com @instrumentar (T-RAD-012 a T-RAD-015)
  6. Thread-safe via file locking (fcntl)

ITENS RESOLVIDOS:
  - Invalidacao automática: REMOVIDA do escopo
    Raj: Concordou após ver que complexidade era alta
    Hermes: Aceitou remover para simplificar

  - Hash instável: RESOLVIDO com json.dumps(params, sort_keys=True)
    Hermes: Vai usar JSON canônico
    Raj: Confirmou que funciona em Python 3.7+

  - @instrumentar: CONFIRMADO
    Ambos: Adicionar decorators T-RAD-012 a T-RAD-015

ITENS EM ABERTO:
  - Nome final do arquivo (briefing_cache vs briefing_cached)
  - Localização (workspace-radar/ vs cortex/maintenance/)

PLANO:
  [CYCLE_050] Hermes: Implementar briefing_cache.py
               - Criar arquivo com template
               - Adicionar @instrumentar
               - Implementar funções: get, set, invalidate
               
  [CYCLE_051] Raj: Testar + Instrumentar RADAR handlers
               - Criar testes pytest
               - Integrar com radar_live_commands.py
               - Atualizar vm_diagram.md

─────────────────────────────────────────────────────────────
```

### FASE 4 — Compromisso

```
┌─────────────────────────────────────────────────────────────┐
│  FASE 4 — COMPROMISSO                                       │
│  Status: committed                                          │
│  Data: 2026-04-20                                           │
└─────────────────────────────────────────────────────────────┘

RESPONSÁVEL: Hermes (implementação), Raj (testes)
TOOL ID: T-RAD-012 a T-RAD-015
ARQUIVO: ~/.openclaw/workspace-radar/briefing_cache.py

ENTREGÁVEIS:
  ☐ code: briefing_cache.py (5 funções)
  ☐ test: test_briefing_cache.py (8-10 testes)
  ☐ doc: Docstrings em todas funções

VERIFICAÇÃO:
  Testes: pytest test/test_briefing_cache.py -v
  Métricas: Cache HIT < 50ms, MISS overhead < 5%
  Score alvo: +0.5% VM (nova ferramenta RADAR)

─────────────────────────────────────────────────────────────

ASSINATURAS:
  Raj: ✅ ConcordO
  Hermes: ✅ ConcordO

═══════════════════════════════════════════════════════════════
```

---

## INTEGRAÇÃO COM CICLOS SIAS

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

## COMO EXECUTAR O DEBATE

### Pré-requisitos

1. Ambos agentes terem acesso ao workspace
2. Contexto do projeto (ferramentas existentes, scores)
3.Canal de comunicação (Telegram thread ou arquivo)

### Fluxo de Execução

```python
# Exemplo de como o debate funciona na prática:

# 1. Hermes identifica necessidade de tool improvement
#    → Lança proposta no canal (Fase 1)

# 2. Raj recebe e faz análise crítica (Fase 2)
#    → Responde com ✔✘⚠️ no canal

# 3. Ambos convergem (Fase 3)
#    → Resolvem objeções
#    → Chat/conversa técnica

# 4. Documentam compromisso (Fase 4)
#    → Arquivo JSON
#    → Próximo CYCLE alinhado

# 5. Implementação via SIAS
#    → CYCLE_xxx executa o plano
```

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

## PITFALLS E DICAS

1. **Debate longo demais**: Se debate passar de 3 ciclos de mensagem, há risco de nunca terminar. Defina timebox: se em 3 trocas não houver consensus, escalte para Felipe.

2. **Análise superficial**: Não aceite análise com apenas "concordo" ou "não concordo". Exija justificativa técnica.

3. **Síntese ambígua**: "vamos discutir melhor" não é síntese. Exija decisão concreta: approved/rejected/approved_with_conditions.

4. **Compromisso sem Ownership**: Todo item do plano de ação precisa de um responsible (raj|hermes|both). "vamos fazer juntos" raramente funciona.

5. **Debate é overhead?** Para melhorias pequenas (1 função, <10 linhas), não inicie debate formal. Use o ciclo SIAS direto. Debate é para mudanças significativas.

---

## MATURIDADE DO DEBATE

| Nível | Descrição | Quando usar |
|-------|-----------|-------------|
| 1 — Informativo | Um agente informa o outro | Changes pequenas, já acordadas |
| 2 — Consultivo | Um pergunta, outro responde | Precisa de input, não aprovação |
| 3 — Deliberativo | Debate estruturado com consensus | Mudanças significativas |
| 4 — Formal | Debate completo com arquivo + assinatura | New tools, refactors grandes |

**Recomendado:** Começar no nível 3 (Deliberativo) para tool improvements significativos.
