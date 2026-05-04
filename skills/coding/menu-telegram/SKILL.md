---
name: menu-telegram
description: "[DESATUALIZADA] Importada do Raj ecosystem — não reflete menus/callbacks do MentorBot. Ver telegram/mentorbot-inline-keyboards para menus corretos."
version: 1.0.1
author: Raj (imported via skill_sync) — PATCHED by hermes
tags: [coding, imported, raj-ecosystem, DESATUALIZADA]
category: coding
created: 2026-04-20
updated: 2026-05-01
metadata:
  hermes:
    imported_from: raj-openclaw
    original_category: coding
    patched: true
    patch_reason: Callback data (menu_*, act_*, sys_*, sias_*) não corresponde ao MentorBot (cfg_perfil, cfg_persona, cfg_tom, etc.). Skill mantida para referência do Raj, mas NÃO usar para desenvolvimento MentorBot.
---


# Menu Telegram

## Estrutura

```python
from cortex.skills.raj_menu.handlers.menu_handler import MENUS, process_callback

# Definir menu
MENU = {
    "menu_principal": {
        "text": "Título",
        "buttons": [
            [{"text": "Opção 1", "callback_data": "act_1"}],
            [{"text": "Opção 2", "callback_data": "act_2"}]
        ]
    }
}

# Handler
def handle_action(action: str) -> str:
    if action == "act_1":
        return "Resultado"
    return "Unknown"

# Processar callback
result = process_callback(callback_data)
```

## Enviar Menu

```python
message(
    action="send",
    channel="telegram",
    target="5556338087",
    interactive={
        "blocks": [
            {"type": "text", "text": "Escolha:"},
            {"type": "buttons", "buttons": [{"label": "Opção", "value": "act_1"}]}
        ]
    }
)
```

## Padrões

| Pattern | Uso |
|---------|-----|
| `menu_*` | Menu principal |
| `act_*` | Ação executável |
| `sys_*` | Sistema |
| `sias_*` | SIAS commands |

## Debug

```python
# Testar process_callback isoladamente
from cortex.skills.raj_menu.handlers.menu_handler import process_callback
result = process_callback("menu_principal")
print(result)
```
