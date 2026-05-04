---
name: port-security-check
description: [Imported from Raj OpenClaw ecosystem]
version: 1.0.0
author: Raj (imported via skill_sync)
tags: [devops, imported, raj-ecosystem]
category: devops
created: 2026-04-20
metadata:
  hermes:
    imported_from: raj-openclaw
    original_category: devops
---


# Port Security Check

## Procedimento

### 1. Listar portas em listen
```bash
ss -tlnp
```

### 2. Verificar bind address
| Bind | Exposto? | Seguro? |
|------|----------|---------|
| `127.0.0.1` | Não | ✅ |
| `::1` | Não | ✅ |
| `0.0.0.0` | Sim | ⚠️ Verificar |
| IP público | Sim | ❌ Alterar |

### 3. Verificar firewall
```bash
sudo iptables -L -n | head -30
```

### 4. Testar exposição externa
```bash
# IP público
curl https://api.ipify.org

# Testar porta específica
nc -zv <IP_PUBLICO> <PORTA>
```

## Portas Comuns

| Porta | Serviço | Esperado | No VM |
|-------|---------|----------|-------|
| 8000 | ChromaDB | 127.0.0.1 | ✅ (localhost-only, seguro) |
| 18789 | OpenClaw Gateway | 127.0.0.1 | ✅ |
| 22 | SSH | 0.0.0.0 (normal) | — |
| 3000 | Flask | 127.0.0.1 | — |
| 5432 | PostgreSQL | 127.0.0.1 | — |

## Ação

Se porta crítica exposta em 0.0.0.0:
1. Identificar serviço: `ps aux | grep <portaOuServico>`
2. Encontrar startup script ou systemd unit
3. Alterar bind para 127.0.0.1
4. Reiniciar serviço
5. Verificar novamente com `ss -tlnp | grep <porta>`

## Fix Rápido — ChromaDB Exposto

```bash
# 1. Identificar PID do processo
ps aux | grep chroma | grep -v grep

# 2. Matar e reiniciar em localhost
pkill -f "chroma.*8000"
uvicorn chromadb.app:app --host 127.0.0.1 --port 8000
# (ou via systemd/systemctl — checar qual startup method foi usado)

# 3. Verificar
ss -tlnp | grep 8000
```

**Trigger:** Alerta de segurança sobre portas expostas, ChromaDB offline handshake, diagnóstico de vulnerabilidade de vetor RAG.
