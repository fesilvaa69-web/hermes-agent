---
name: hermes-token-debug
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


# Hermes Token Debug

## Procedimento

### 1. Identificar o erro
Verificar logs: `tail -20 ~/.hermes_test/logs/gateway.log`
Buscar: `InvalidToken`, `token rejected`, `Unauthorized`

### 2. Comparar tokens nos .env
```bash
# Token no .hermes (produção)
cat ~/.hermes/.env | grep TOKEN

# Token no .hermes_test (teste)
cat ~/.hermes_test/.env | grep TOKEN

# Token esperado (validar via API)
curl -s "https://api.telegram.org/bot<TOKEN>/getMe"
```

### 3. Verificar lock files
```bash
# Lock files podem bloquear o bot
ls -la ~/.local/state/hermes/gateway-locks/
rm -f ~/.local/state/hermes/gateway-locks/telegram-bot-token-*.lock
```

### 4. Verificar processos
```bash
# Hermes pode estar rodando em outro local
ps aux | grep hermes | grep -v grep
```

### 5. Corrigir token
```bash
# Editar o .env correto
nano ~/.hermes/.env
# OU
nano ~/.hermes_test/.env
```

### 6. Reiniciar Hermes
```bash
# Matar processo antigo
kill -9 <PID>

# Reiniciar
cd ~/hermes_agent && nohup .venv/bin/python -m hermes_cli.main gateway run > ~/.hermes_test/logs/gateway.log 2>&1 &
```

## Armadilhas
- Tokens iguais podem ter typos (ex: `Nmm` vs `Nnm`)
- Lock file pode persistir mesmo após matar processo
- Hermes pode estar lendo de `.hermes/` e não de `.hermes_test/`
