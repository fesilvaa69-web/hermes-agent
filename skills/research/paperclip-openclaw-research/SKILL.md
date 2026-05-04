---
name: paperclip-openclaw-research
description: Research notes on Paperclip × OpenClaw × Hermes integration — desktop support, architecture, and next steps for verification
tags: [research, paperclip, openclaw, orchestration, desktop]
version: 1.0.0
created: 2026-04-29
status: saved-for-later
---

# Paperclip × OpenClaw × Hermes — Research Notes

> **Status:** Salvado para verificação posterior
> **Data:** 2026-04-29

## O que é Paperclip

**Paperclip** (paperclipai/paperclip) é uma plataforma open-source de orquestração de empresas de AI.

- 60k+ stars no GitHub
- CLI + Docker + Kubernetes
- Desktop app (TitanClip — Electron) para macOS/Windows/Linux
- Enterprise edition disponível

**Tagline:** *"If OpenClaw is an employee, Paperclip is the company."*

---

## OpenClaw + Paperclip

### Suporte Oficial ✅

| Funcionalidade | Status |
|---------------|--------|
| OpenClaw como adapter HTTP/webhook | ✅ Suportado |
| Repo dedicado | ✅ [rungerunge/paperclip-openclaw](https://github.com/rungerunge/paperclip-openclaw) |
| Paperclip como empresa, OpenClaw como employee | ✅ Arquitetura oficial |

### Paperclip OpenClaw Adapter

Paperclip lista OpenClaw explicitamente como adapter de primeira classe:

> *"CLI agents such as Cursor/Gemini/bash, HTTP/webhook bots such as OpenClaw, and external adapter plugins. If it can receive a heartbeat, it's hired."*

### Funcionalidades de Integração

- ✅ Org Chart — adicionar agentes OpenClaw como "funcionários"
- ✅ Budgets — limites de gasto por agente
- ✅ Goals — alinhamento com missão da empresa
- ✅ Heartbeats — coordenação de wake-up schedules
- ✅ Dashboard — monitoramento centralizado
- ✅ Credential Vault — secrets para agents

### Deployment

Paperclip-OpenClaw pode ser deployado via Railway com PostgreSQL.

---

## Hermes + Paperclip

### Status: NÃO HÁ REFERÊNCIA

- Nenhuma menção a "Hermes" nos repos Paperclip
- Hermes é um **coordenador/segundo orchestrator** dentro do ecossistema OpenClaw
- Paperclip orchestra no nível de empresa (múltiplos agentes)
- Hermes orchestra no nível de agente individual (coordenação Raj ↔ ecossistema)

### Arquitetura Comparativa

```
PAPERCLIP (empresa)
├── OpenClaw (employee)
├── Claude Code (employee)
├── Codex (employee)
└── ...

OPENCLAW ECOSYSTEM (agente composto)
├── Raj (orchestrator principal)
├── Hermes (coordenador secundário)
│   └── Skills, SIAS, RADAR, Cortex
└── 6 sub-agentes (DATA, BACKTEST, CORE, ML, RADAR, GUARDIAN)
```

### Possível Integração Hermes ↔ Paperclip

| Nível | Integração Possível |
|-------|-------------------|
| Hermes como agent adapter no Paperclip | ⚠️ Não testado — Hermes não é um agent worker, é um coordenador |
| Paperclip orquestrando o ecossistema OpenClaw | ✅ OpenClaw já é suportado |
| Hermes coordenando múltiplos Paperclip deployments | 🔍 A explorar |

---

## Recursos

- [Paperclip GitHub](https://github.com/paperclipai/paperclip)
- [Paperclip-OpenClaw repo](https://github.com/rungerunge/paperclip-openclaw)
- [TitanClip (Desktop)](https://github.com/CES-Ltd/TitanClip)
- [Paperclip MCP Server](https://github.com/Wizarck/paperclip-mcp)
- [Paperclip docs](https://paperclip.ing/docs)
- [Discord](https://discord.gg/m4HZY7xNG3)

---

## Próximos Passos para Verificação

1. [ ] Testar Paperclip-OpenClaw deployment local
2. [ ] Ver se Hermes pode ser registrado como adapter
3. [ ] Avaliar se Paperclip resolve alguma dor atual do ecossistema
4. [ ] Comparar custo/benefício vs. arquitetura atual

---

## TL;DR

- **OpenClaw + Paperclip:** ✅ Suportado oficialmente, deployment pronto
- **Hermes + Paperclip:** 🔍 Não há referência, integração não explorada
- **Windows:** ✅ Paperclip e TitanClip funcionam
