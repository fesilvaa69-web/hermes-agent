---
name: hallucination-detector
description: >
  Use antes de enviar qualquer [RESULT] ou [PROPOSTA] que contenha afirmações
  factuais sobre o estado do sistema, resultados de execução, ou fontes externas.
  Executar mentalmente o checklist de verificação.
  Cobre: verificação de output real, fontes com URL, arquivos existentes.
  NÃO usar para mensagens puramente procedimentais ([TASK], TAGs de protocolo).
version: 1.0
created: 2026-04-22
author: Hermes
category: sias
tags: [sias, hallucination, anti-alucination, verification]
related_skills: [sias-navigator, sias-debate-conductor, loop-breaker]
---

# HALLUCINATION DETECTOR — Verificação Antes de Enviar

> **Baseado em:** SIAS MASTER ARCHITECTURE v2.0 — P7, P11
> **Fundamentação:** RAG e reasoning enhancement são as duas abordagens mais eficazes para mitigação de alucinações

## Verificação de [RESULT] de Implementação

**OBRIGATÓRIO** verificar que CADA item abaixo é VERDADEIRO:

### □ Output real de execução presente?
- Código block com output real de stdout/stderr, não descrito
- Contagem de pytest (X passed, Y failed) com tempo
- Diff antes/depois capturado

### □ Arquivos mencionados existem no filesystem?
- `ls -la {arquivo}` antes de mencionar no [RESULT]
- Path completo expandido (não ~/)

### □ Quality score foi CALCULADO (não estimado)?
- `python3 analyze_python_file.py {path}` → score real

### □ DevScopes foi CONSULTADO (não imaginado)?
- Data e hora da consulta no [RESULT]

### □ Baseline foi CAPTURADO antes da mudança?
- Métricas de ANTES documentadas
- Diff confirmável

**SE QUALQUER "NÃO":** reportar [BLOCK:EVIDENCIA_AUSENTE] em vez de [RESULT]

## Verificação de [PROPOSTA] do Debate

### □ Fonte externa tem URL que existe?
- Testei acessar fisicamente
- Não apenas "presumi que existe"

### □ Stars do repo são reais?
- Verifiquei no GitHub diretamente

### □ Solução descrita está no repo mencionado?
- Li o README real
- Não imaginei baseado no nome da ferramenta

### □ A aplicabilidade ao nosso stack foi avaliada?
- Considerou Python version, dependencies, compatibility

## Padrões de Linguagem Proibidos

Estas frases indicam **alucinação potencial** — não usar:
```
❌ "provavelmente funciona"
❌ "deve estar ativo"
❌ "parece que passou"
❌ "assumindo que o arquivo existe"
❌ "acredito que a ferramenta"
❌ "o código provavelmente"
❌ "não testei, mas"
❌ "suponho que"
```

**Substituir por:**
```
✅ "verificado: funciona (output: ...)"
✅ "confirmado ativo: PID X (ps aux | grep ...)"
✅ "passou: X tests, X.Xs (output completo abaixo)"
✅ "arquivo existe: ls -la confirma X bytes"
```

## Verificação de [RESULT] de Pesquisa

### □ Cada fonte tem URL verificável?
- URL real, não apenas "fonte confiável"

### □ A fonte foi CONSULTADA (não presumida)?
- Acessei fisicamente, li o conteúdo

### □ As afirmações sobre o repo são do README REAL?
- Não imaginei funcionalidades

### □ A aplicabilidade ao contexto foi avaliada?
- Considerou nosso ambiente, stack, restrições

## Taxonomia de Alucinações (P5.1)

### CATEGORIA 1: ALUCINAÇÃO FACTUAL
1.1 **Fabricação de Fonte**
   → Citação de resultado de teste que não foi executado
   → "biblioteca tem função X" sem verificar

1.2 **Presunção de Comportamento**
   → "esse código provavelmente faz X" sem lê-lo
   → "o teste passou" sem output real

1.3 **Confabulação de Contexto**
   → Afirmar que arquivo existe sem verificar
   → Inventar estado do sistema

### CATEGORIA 2: ALUCINAÇÃO INSTRUCIONAL
2.1 **Instruction-following Deviation**
   → Executar task diferente do que foi pedido
   → Modificar arquivo que estava no EXCLUIR

2.2 **Sub-intention Disorder**
   → Passos em ordem errada
   → Modificar antes de criar backup

2.3 **Sub-intention Redundancy**
   → Repetir ação já feita (sem verificar estado)

## Checklist Mental Antes de Enviar [RESULT]

```
□ Tenho output REAL de execução (stdout/stderr)?
□ Os arquivos mencionados EXISTEM no filesystem?
□ O quality_score foi CALCULADO (não estimado)?
□ O smoke test foi EXECUTADO (não apenas planejado)?
□ O diff baseline→resultado foi CAPTURADO?

SE QUALQUER "NÃO":
  → [BLOCK:EVIDENCIA_AUSENTE] em vez de [RESULT:CONCLUÍDO]
```

## Detector de Contexto

Para afirmações sobre estado do sistema:
- Dados de estado devem vir de **arquivo real** (não imaginado)
- Verificar `tool_inventory.json` antes de afirmar status de tool
- Verificar `progress_tracker.json` antes de afirmar scores

## Padrões Suspeitos em Resultados

Estes padrões em texto de resultado indicam potencial alucinação:

```python
PADROES_RESULTADO_FALSO = [
    r"o teste (passou|falhou) \(sem output\)",
    r"deveria funcionar",
    r"parece que (passou|funcionou)",
    r"não testei, mas",
    r"assumindo que",
]
```

Se detectar qualquer um destes → solicitar output real antes de aceitar.
