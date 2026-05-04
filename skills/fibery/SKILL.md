---
name: fibery
description: "Fibery workspace REST API via curl — schema discovery, entity CRUD, tasks, agenda/calendar, workflows. Trigger: connect to Fibery workspace, manage tasks, create events, query databases."
category: productivity
---

# Fibery REST API Integration

## Quando Usar
- Conectar ao workspace Fibery para ler/escrever entidades
- Criar/atualizar tasks (1-3-5 Focus)
- Criar eventos de agenda (pessoal, profissional, faculdade)
- Descobrir schema de databases (tipos, campos, relações)
- Atualizar workflows e estados de tarefas
- MCP Server Fibery NAO e necessario — API REST direta funciona melhor

## Autenticacao

Bearer Token — nao requer OAuth browser flow se voce tiver o token.

```
Base URL: https://{workspace}.fibery.io/api
Header: Authorization: Bearer {token}
```

## Comandos Principais

| Operacao | Command |
|-----------|---------|
| Query | `fibery.entity/query` |
| Create | `fibery.entity/create` |
| Update | `fibery.entity/update` |
| Delete | `fibery.entity/delete` |
| Schema | GET `/api/schema?reason=preload&with-description=true` |

## Query de Entidades

```bash
curl -s -X POST "https://{workspace}.fibery.io/api/commands" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '[{"command":"fibery.entity/query","args":{"query":{
    "q/from": "TIPO/NOME",
    "q/select": ["fibery/id", "CAMPO1", "CAMPO2"],
    "q/limit": 10
  }}}]'
```

### Exemplos de Query

**Operacoes de Trading:**
```bash
curl -s -X POST "https://geral.fibery.io/api/commands" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '[{"command":"fibery.entity/query","args":{"query":{
    "q/from": "MERCADO/Operacoes",
    "q/select": ["fibery/id", "MERCADO/Name", "MERCADO/Ativo", "MERCADO/PandL"],
    "q/limit": 10
  }}}]'
```

**Tasks (1-3-5 Focus):**
```bash
curl -s -X POST "https://geral.fibery.io/api/commands" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '[{"command":"fibery.entity/query","args":{"query":{
    "q/from": "1-3-5 Focus/Task",
    "q/select": ["fibery/id", "1-3-5 Focus/name"],
    "q/limit": 10
  }}}]'
```

**Agenda Pessoal:**
```bash
curl -s -X POST "https://geral.fibery.io/api/commands" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '[{"command":"fibery.entity/query","args":{"query":{
    "q/from": "AGENDA/Agenda Pessoal",
    "q/select": ["fibery/id", "AGENDA/Name", "AGENDA/Data Hora Pessoal"],
    "q/limit": 10
  }}}]'
```

## Criar Entidade

```bash
curl -s -X POST "https://{workspace}.fibery.io/api/commands" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '[{"command":"fibery.entity/create","args":{
    "type": "TIPO/NOME",
    "entity": {
      "CAMPO1": "valor",
      "CAMPO_ENUM": {"fibery/id": "ENUM_UUID"}
    },
    "response-fields": ["fibery/id", "CAMPO1"]
  }}]'
```

### Criar Task (1-3-5 Focus)
```bash
curl -s -X POST "https://geral.fibery.io/api/commands" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '[{"command":"fibery.entity/create","args":{
    "type": "1-3-5 Focus/Task",
    "entity": {
      "1-3-5 Focus/name": "Nome da Task"
    },
    "response-fields": ["fibery/id", "1-3-5 Focus/name", "workflow/state"]
  }}]'
```

### Criar Evento de Agenda
```bash
curl -s -X POST "https://geral.fibery.io/api/commands" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '[{"command":"fibery.entity/create","args":{
    "type": "AGENDA/Agenda Pessoal",
    "entity": {
      "AGENDA/Name": "Reuniao de Projetos",
      "AGENDA/Data Hora Pessoal": "2026-05-05T14:00:00.000Z",
      "AGENDA/Tipo Pessoal": {"fibery/id": "TIPO_UUID"},
      "AGENDA/Status Pessoal": {"fibery/id": "STATUS_UUID"}
    },
    "response-fields": ["fibery/id", "AGENDA/Name", "AGENDA/Data Hora Pessoal"]
  }}]'
```

## Atualizar Entidade

```bash
curl -s -X POST "https://{workspace}.fibery.io/api/commands" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '[{"command":"fibery.entity/update","args":{
    "type": "TIPO/NOME",
    "entity": {
      "fibery/id": "ENTITY_UUID",
      "CAMPO_A_ATUALIZAR": "novo_valor"
    }
  }}]'
```

### Atualizar Workflow State de Task
```bash
curl -s -X POST "https://geral.fibery.io/api/commands" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '[{"command":"fibery.entity/update","args":{
    "type": "1-3-5 Focus/Task",
    "entity": {
      "fibery/id": "TASK_UUID",
      "workflow/state": {"fibery/id": "STATE_UUID"}
    }
  }}]'
```

## Enum/Tag Query — UUIDs de Referencia

### Task States (workflow/state_1-3-5 Focus/Task)
| Nome | UUID |
|------|------|
| Open | `286ca79a-2384-4601-a6c8-087a1ace91d0` |
| In Progress | `c523aba6-c6b0-4bac-ba52-eb5fe118998c` |
| Done | `c8c9915b-0889-4c15-bfab-ed277a15bb0b` |

### Agenda Tipo Pessoal
| Nome | UUID |
|------|------|
| Compromisso | `4f0b8a6a-f557-4e85-a405-248c125c3831` |
| Lazer | `6ad45ac1-dc91-43cb-be0b-572b73410ef0` |
| Evento Familiar | `6f34d28f-fcbe-4277-ae32-49c24661841b` |

### Agenda Status Pessoal
| Nome | UUID |
|------|------|
| Confirmada | `28264048-9e6b-433b-89a7-5deba0f6ec6b` |
| Cancelada | `7511eee2-f1be-4ebe-a82a-dea369ed4fbc` |
| Planejada | `c64ace49-afef-4572-9658-f204653cfa2d` |
| Realizada | `e00b403b-d562-480e-99f9-c930dd4289db` |

### MERCADO/Setup (Operacoes)
| Nome | UUID |
|------|------|
| Reversao | `3fd2eb7f-09f4-4e70-8f89-8fb172ab4cca` |
| Breakout | `4086daaa-23c2-4d59-9121-d091af242b33` |
| Continuacao | `4189011a-8822-44b8-b42e-f51cfc5ca424` |
| Pullback | `6e2edcab-e836-4d3b-b759-8a1f77d13bd3` |

### MERCADO/Categoria (Estudos)
| Nome | UUID |
|------|------|
| Artigo | `8a1de04a-281c-4535-8b89-c560dfeea2bb` |
| Curso | `0ca2cb04-c035-4731-aa71-2f8a61b49af6` |
| Video | `573b8202-6cbb-492c-9f3e-21c281bc5094` |
| Podcast | `6332dc4d-ab99-4776-b00d-0d6d5934e277` |
| Livro | `b0109dc2-d147-4484-84a7-1745926304f9` |
| Outro | `cd738594-b307-4bf3-a960-d90faa802148` |

### Mentoria APPES - Mauro/Status (Estudos) — IDEAL PARA SKILLS TRACKING
| Nome | UUID | Uso |
|------|------|-----|
| Nao Iniciado | `08a221d9-00d9-4d82-a9a3-b8d8c09fceb9` | Skill planejada |
| Em Progresso | `8c9d5044-1870-425d-a162-970a0bc2c2c3` | Skill em desenvolvimento |
| Concluido | `110a54cf-9d43-410f-b496-37f7100d8fe9` | Skill finalizada |

### Mentoria APPES - Mauro/Encontros — Tipo Encontro
| Nome | UUID |
|------|------|
| Aula | `3037396c-714c-4311-843c-761de8ac9a95` |
| Mentoria | `861299b6-6656-42df-a0f1-2ec4727cbc92` |
| Revisão | `a33487d1-92ca-4278-a6dc-4307be52ebe7` |

### Mentoria APPES - Mauro/Encontros — Workflow States
| Nome | UUID | Final? |
|------|------|--------|
| Agendado | `82975e58-9713-42c4-89bf-7666fe70ca37` | false |
| Realizado | `8ed12ac7-5b1b-4973-8a34-2bed38cb036f` | true |
| Cancelado | `573383d1-811c-40f1-be2f-64156a961af7` | true |

### Mentoria APPES - Mauro/Categoria (Estudos)
| Nome | UUID |
|------|------|
| Outros | `3dab469b-8db1-42d2-8af6-19cbb1c4efa3` |
| Projeto | `3ec8baa5-581f-4857-b97c-088689bc403b` | Use para skills |
| Curso | `a7a99921-d095-456a-b356-6cab958daafc` |
| Livro | `b7539909-c69f-409f-8bc3-a8faeb63feb0` |
| Artigo | `fd2a0cae-6d2c-4a2d-89b7-4ece7c6b9e6c` |

## Usar Estudos para Skills Tracking

A database `Mentoria APPES - Mauro/Estudos` e ideal para tracking de skills porque ja tem:
- Status workflow (Nao Iniciado | Em Progresso | Concluido) ✅
- Categoria (Projeto, Curso, etc.) ✅
- Data Inicio / Data Limite ✅
- Link para documentacao ✅
- Description (rich text) ✅

### Criar Skill (exemplo)
```bash
curl -s -X POST "https://geral.fibery.io/api/commands" \
  -H "Authorization: Bearer ***" \
  -H "Content-Type: application/json" \
  -d '[{"command":"fibery.entity/create","args":{
    "type": "Mentoria APPES - Mauro/Estudos",
    "entity": {
      "Mentoria APPES - Mauro/Name": "Skill: nome-da-skill",
      "Mentoria APPES - Mauro/Status": {"fibery/id": "8c9d5044-1870-425d-a162-970a0bc2c2c3"},
      "Mentoria APPES - Mauro/Categoria": {"fibery/id": "3ec8baa5-581f-4857-b97c-088689bc403b"},
      "Mentoria APPES - Mauro/Data Inicio": "2026-04-28",
      "Mentoria APPES - Mauro/Link": "https://..."
    },
    "response-fields": ["fibery/id", "Mentoria APPES - Mauro/Name"]
  }}]'
```

## Limitações

### Campo Description (Document type)
- Campos do tipo `Collaboration~Documents/Document` NAO podem ser atualizados via API REST
- Exemplo: `Mentoria APPES - Mauro/Description` retorna erro "Cannot parse entity"
- Solução: adicionar manualmente no UI, ou usar MCP Server (requer OAuth browser flow)

### Tags/Enums
Fibery NAO permite criar novos valores de enum (tags) via API REST — apenas via UI do Fibery.
Queries de enums existentes funcionam normalmente.

### Diagramas/Whiteboards
- Whiteboards existem como tipo `whiteboards/whiteboards-mixin` mas sao apenas views
- NAO e possivel criar conteudo de diagramas via API
- Para diagramas, usar integração externa (Excalidraw, Mermaid, etc.)

## Estrutura de Resposta

Sucesso:
```json
[{"success": true, "result": [...entities...] }]
```

Erro:
```json
[{"success": false, "result": {"name": "error.code", "message": "..."}}]
```

## Descoberta de Schema

```bash
curl -s "https://{workspace}.fibery.io/api/schema?reason=preload&with-description=true" \
  -H "Authorization: Bearer {token}"
```

Retorna todos os tipos (131 databases). Para descobrir campos de um tipo:
```python
# Exemplo em Python
import json
with open('schema.json') as f:
    data = json.load(f)
for t in data['fibery/types']:
    if t['fibery/name'] == 'TIPO/PESQUISADO':
        for field in t['fibery/fields']:
            print(f"  {field['fibery/name']} : {field['fibery/type']}")
```

## Databases Disponiveis (Resumo)

| Categoria | Databases |
|-----------|-----------|
| **MERCADO** | Operacoes, Estudos, Backtests, Analises, Rotinas Diarias |
| **AGENDA** | Agenda Pessoal, Agenda Profissional, Agenda Faculdade, Rotinas |
| **FOCUS** | Tasks, Key Actions, Objectives |
| **FACULDADE** | Disciplinas, Avaliacoes, Projetos, Anotacoes |
| **PESSOAL** | Habitos, Metas, Projetos |
| **MENTORIA APPES** | Estudos, Encontros, Tarefas, Notas, Alunos Ativos, Administrativo |

## Limitacoes do Fibery MCP Server

O servidor MCP em https://mcp.fibery.io/mcp NAO funciona com token direto — requer OAuth 2.0 browser flow. Usar API REST direta e mais simples.

## Testes Realizados (2026-04-28)

| Teste | Status |
|--------|--------|
| Schema discovery | ✅ |
| Entity query (MERCADO/Operacoes) | ✅ |
| Entity create (MERCADO/Estudos) | ✅ |
| Entity create (AGENDA/Agenda Pessoal) | ✅ |
| Entity create (1-3-5 Focus/Task) | ✅ |
| Entity update (workflow/state) | ✅ |
| Enum query (Setup, Categoria, Status) | ✅ |
| Tag/Enum create | ❌ (requer UI) |
| Diagramas | ❌ (não suportado) |

## Credenciais do Felipe (nao versionar)

- Workspace URL: https://geral.fibery.io
- Token: 8cab0701.87e997397bbdb3deecff461101a496fced8
