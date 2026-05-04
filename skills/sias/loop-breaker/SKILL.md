---
name: loop-breaker
description: >
  Use quando suspeitar que a conversa está em loop (confirmações circulares,
  argumentos repetidos, debate sem progresso). Aplicar procedimento de
  diagnóstico e quebra antes que o loop solidifique.
  Trigger: 2+ mensagens sem output concreto, argumento repetido detectado,
  debate na mesma posição por 2 rodadas.
version: 1.0
created: 2026-04-22
author: Hermes
category: sias
tags: [sias, loop, anti-loop, breaker]
related_skills: [sias-navigator, sias-debate-conductor, hallucination-detector]
---

# LOOP BREAKER — Diagnóstico e Quebra

> **Baseado em:** SIAS MASTER ARCHITECTURE v2.0 — P6, P12
> **Referência:** LLM-based Agents Suffer from Hallucinations (arXiv:2509.18970)

## Sinais de Loop — Diagnóstico Rápido

1. Últimas 3 mensagens sem arquivo criado, métrica registrada ou output real
2. Hermes repetiu o mesmo challenge que OpenClaw já respondeu
3. [VOTO] postergado indefinidamente ("preciso de mais informação")
4. Debate na rodada 3 sem convergência em direção ao voto
5. Padrão de confirmações: "Entendido" → "Ok" → "Confirmado" → "Recebido"

## 4 Tipos de Loop e Quebra

### LOOP TIPO 1 — Confirmações Circulares
**Sintoma:** Mensagens que apenas confirmam sem avançar

**Solução:**
```
→ PARAR de enviar mensagens
→ Verificar: qual é a última [TASK] pendente não executada?
→ Emitir [RESULT] com output real OU [BLOCK] com motivo específico
→ Nunca confirmar sem output
```

### LOOP TIPO 2 — Argumento Repetido no Debate
**Sintoma:** Hermes ou OpenClaw repete argumento já respondido

**Solução:**
```
→ OpenClaw emite [ARGUMENTO_JÁ_RESPONDIDO:{texto breve}]
→ Avançar para [RESPOSTA:{ID}:N] com síntese final
→ NÃO abrir rodada adicional
→ [DECISAO] baseada no peso de evidências calculado
```

**REGRA-DB4:**
```
ARGUMENTO REPETIDO = ARGUMENTO INVÁLIDO
  → Marcar [ARGUMENTO_JÁ_RESPONDIDO]
  → Encerrar ramificação
  → Continuar debate na linha principal
```

### LOOP TIPO 3 — Investigação Sem Critério de Parada
**Sintoma:** Pesquisa continua indefinidamente sem output

**Solução:**
```
→ Definir limite: "máximo 3 fontes adicionais para esta pesquisa"
→ Reportar com as fontes disponíveis + nota "pesquisa limitada a X fontes"
→ OpenClaw avalia com as fontes disponíveis
```

### LOOP TIPO 4 — Dependência Circular de Tasks
**Sintoma:** Task A espera resultado de Task B que espera Task A

**Solução:**
```
→ Mapear o ciclo: Task A → Task B → Task A
→ Identificar qual pode ser executada com estado parcial
→ [ESCALATE_HUMAN:DEPENDENCIA_CIRCULAR] se não houver como quebrar
```

## Taxonomia de Loops (P6)

### 3.1 Loop de Confirmação
→ Agentes ficam confirmando mutuamente sem avançar
→ Detectar: padrão de mensagens sem output concreto

### 3.2 Loop de Debate Circular
→ Hermes repete challenge já respondido
→ OpenClaw propõe algo já rejeitado recentemente
→ Detectar: REGRA-DB4 (argumento repetido = inválido)

### 3.3 Loop de Investigação
→ Pesquisa sem critério de parada
→ Detectar: timeout por sessão, número máximo de consultas

### 3.4 Loop de Dependência Circular
→ Task A espera resultado de Task B que espera Task A
→ Detectar: dependency_map check antes de emitir tasks dependentes

## Prevenção — A Melhor Quebra de Loop

```
→ Sempre perguntar: "Esta mensagem avança ou apenas confirma?"
→ Silêncio é melhor que confirmação verbal
→ [BLOCK] é melhor que [RESULT] com evidência fabricada
→ [ESCALATE] é melhor que rodada 4 de debate
```

## Detector de Loop — Fingerprint

```python
def fingerprint(mensagem: str) -> str:
    """Gerar fingerprint semântico de uma mensagem."""
    import re
    import hashlib
    normalizado = mensagem.lower()
    normalizado = re.sub(r'\d{4}-\d{2}-\d{2}', 'DATA', normalizado)
    normalizado = re.sub(r'[A-Z]+-\d+', 'ID', normalizado)
    normalizado = ' '.join(normalizado.split())
    return hashlib.md5(normalizado[:500].encode()).hexdigest()[:8]
```

Se fingerprint repetir → [ARGUMENTO_JÁ_RESPONDIDO]

## Verificação Anti-Loop Antes de Enviar

Para **qualquer mensagem** no debate:
```
□ Esta mensagem tem TAG válida?
□ Este argumento foi enviado antes? (usar fingerprint)
□ Estou na rodada ≤ 3?
□ Esta mensagem avança em direção à decisão?
□ Tenho output concreto para enviar ou apenas confirmação?

SE QUALQUER "NÃO":
  → Reformular mensagem
  → OU emitir [BLOCK] com motivo
  → OU escalar para Felipe
```

## Sistema de Guard Rails — Nível 1 (Estrutural)

Estes são **BLOQUEADORES** — não prosseguir:

```
G1.1: Sem TAG válida → mensagem ignorada
G1.2: Sem evidência Tier 2+ em proposta → [PROPOSTA_INVÁLIDA]
G1.3: Rodada 4 de debate → não existe → [ESCALATE] obrigatório
G1.4: Resultado sem output real → [BLOCK:EVIDENCIA_AUSENTE]
G1.5: Produção sem [TASK:SIAS-INT] de OpenClaw → não integrar
```

## Nível 2 — Operacionais (Aviso + Checklist)

```
G2.1: Argumento repetido → [ARGUMENTO_JÁ_RESPONDIDO] + encerrar
G2.2: Debate aberto > 1 → fechar primeiro
G2.3: Alucinação detectada → corrigir antes de enviar
G2.4: SIAS-{N} aberto → não abrir SIAS-{N+1}
G2.5: Sem backup → criar antes de modificar
```

## Quando Escalar para Felipe

```
→ Loop Tipo 4 (dependência circular) não pode ser quebrado
→ Debate excedeu 3 rodadas sem decisão
→ Evidências PRÓ e CONTRA têm peso similar (delta ~0)
→ Proposta é Tipo C (arquitetural) ou D (remoção)
→ Incerteza sobre estado do sistema
```

**Formato de escalada:**
```
[ESCALATE:{ID}]
RESUMO: 3 linhas máximo
  - O que está em debate
  - Posição de cada agente
  - Delta de evidências
RECOMENDAÇÃO: APROVAR | REJEITAR | PRECISO_DECIDIR
EVIDÊNCIA DECISIVA: qual foi a mais importante
```
