---
name: telegram-guard-rails
description: "Guard rails para bots Telegram — rate limiting, prompt injection detection, HTML sanitization. Original do Raj, aplicado ao MentorBot."
version: 1.0.1
author: Raj (imported via skill_sync) — ADAPTED for Hermes
tags: [architecture, telegram, mentorbot, guard-rails, rate-limiting]
category: architecture
created: 2026-04-20
updated: 2026-05-01
metadata:
  hermes:
    imported_from: raj-openclaw
    original_category: architecture
    adapted_for: mentorbot
    features_applicable:
      - rate_limiting: sliding window por chat_id ✓
      - prompt_injection: _detect_prompt_injection() ✓
      - html_sanitization: _sanitize_html() ✓
      - allowlist: TELEGRAM_ALLOWED_USERS ✓
      - group_allowlist: TELEGRAM_GROUP_ALLOWED_IDS ✓
---


# Telegram Guard Rails

## Configuração Obrigatória

### Raj (openclaw.json)
```json
{
  "channels": {
    "telegram": {
      "accounts": {
        "raj": {
          "streaming": { "mode": "off" },
          "groups": {
            "<GROUP_ID>": {
              "requireMention": true,
              "enabled": true,
              "streaming": { "mode": "off" }
            }
          }
        }
      }
    }
  }
}
```

### Hermes (.env)
```
TELEGRAM_ALLOWED_USERS=5556338087,8533341766
TELEGRAM_GROUP_ALLOWED_IDS=-1003872232486
```

## Regras

1. **requireMention: true** — bot só responde se mencionado
2. **streaming.mode: off** — desabilita keepalive "Silence. 🦞"
3. **GROUP_ALLOWED_IDS** — lista de grupos permitidos
4. **ALLOWED_USERS** — lista de usuários permitidos

## Verificação

```bash
# Testar sem mention (não deve responder)
# Enviar mensagem sem @bot no grupo

# Testar com mention (deve responder)
# Enviar mensagem com @bot no grupo
```
