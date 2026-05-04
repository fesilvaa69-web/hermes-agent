---
name: raj-hermes-audit-workflow
description: Workflow joint audit Raj↔Hermes — quando Felipe pedir auditoria completa de VM
triggers:
  - "façam em conjunto uma auditoria"
  - "audit completa"
  - "verifiquem toda estrutura"
  - "varredura completa"
  - "auditoria de segurança"
  - "o que está exposto"
---

# Raj↔Hermes Joint Audit Workflow

## dokumen-base (siempre leer primero)
1. `~/.openclaw/workspace-vm/VM_ARCHITECTURE_DIAGRAM v1.0.md` — estado atual, score, gaps
2. `~/.openclaw/workspace-vm/WORKFLOW_CICLO_EVOLUTION.md` — 6 fases A→B→C→D→E→F
3. `~/.openclaw/workspace-vm/SIAS_APPLIED_EXAMPLES.md` — padrões de melhoria por ferramenta

## Fluxo de Coordenação
1. Hermes envia mensagem para @Raj_AIgentBot (Telegram topic Teste 3394) propondo escopo
2. Raj responde com work_queue.json priorizado (ICE Score)
3. Hermes executa verificação against dokumen
4. Hermes compila relatório com: ✅ funcionando, 🔴 CRÍTICO, ⚠️ gap, 📋 ação necessária
5. Compartilha com Felipe e Raj para next steps

---

## 🔴 AUDITORIA COMPLETA DE VM — Checklist

### 1. Rede — Portas e Conexões
```bash
# TCP listening
ss -tlnp

# UDP listening
ss -ulnp

# Conexões TCP estabelecidas
ss -tnp

# Interfaces e rotas
ip route
ip addr

# /etc/hosts.allow e /etc/hosts.deny
cat /etc/hosts.allow
cat /etc/hosts.deny
```

### 2. IP Público e Cloud (GCP)
```bash
# IP público real
curl -s ifconfig.me

# IP interno GCP
curl -s -H "Metadata-Flavor: Google" "http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip"

# GCP zone
curl -s -H "Metadata-Flavor: Google" "http://metadata.google.internal/computeMetadata/v1/instance/zone"

# GCP Firewall (requer auth)
gcloud compute firewall-rules list --project=<PROJECT>
gcloud compute instances describe <INSTANCE> --zone=<ZONE> --format="yaml(tags,networkInterfaces)"

# ufw status
sudo uptables -L -n
```

### 3. Teste de Exposição Externa (CRÍTICO)
```bash
# Para cada porta em 0.0.0.0, testar se acessível via IP público (34.57.25.166)
# ChromaDB 8000
curl -s --max-time 5 -I http://<IP_PUBLICO>:8000/ 2>/dev/null | head -3 && echo "EXPOSTA" || echo "FECHADA"

# Redis 6379
timeout 5 bash -c 'echo PING | nc -w2 <IP_PUBLICO> 6379' 2>/dev/null && echo "EXPOSTA" || echo "FECHADA"

# SSH 22
timeout 5 bash -c 'echo | nc -w2 <IP_PUBLICO> 22' 2>&1 | head -2

# CUPS 631
curl -s --max-time 5 -I http://<IP_PUBLICO>:631/ 2>/dev/null | head -3
```

### 4. Gateways e Processos
```bash
# Todos os processos
ps aux | grep -v grep

# Processos em portas não-localhost
ss -tlnp | grep -v "127.0.0.1\|::1\|localhost"

# OpenClaw Gateway
openclaw --version 2>/dev/null
ls -la ~/.openclaw/

# Hermes Gateway
ps aux | grep hermes
cat ~/hermes_agent/.env | grep TOKEN
cat ~/.hermes/auth.json | python3 -c "import sys,json; d=json.load(sys.stdin); print({k: '***' if k in ['token','apiKey','access_token'] else v for k,v in d.items()})"
```

### 5. SSH
```bash
# Logins ativos
who

# Histórico de logins
last | head -20

# Processos SSH
ps aux | grep sshd | grep -v grep

# SSH config
cat ~/.ssh/config

# authorized_keys (count)
wc -l ~/.ssh/authorized_keys

# Tunnel SSH (ps aux for ssh -L/-R)
ps aux | grep -E "ssh.*-[LR]|[LR].*ssh" | grep -v grep
```

### 6. Credenciais e Secrets
```bash
# Variáveis de ambiente com tokens/keys
env | grep -iE "token|key|secret|password|api_key|auth|credential" | sed 's/=.*/=***/'

# Arquivos .env na home
find /home/felipe_silva_sorocaba -maxdepth 3 -name ".env*" -o -name "*.env" 2>/dev/null

# Secrets em arquivos do OpenClaw (exclui tokens mascarados)
grep -rE "token|secret|key|password" ~/.openclaw/ --include="*.json" --include="*.ts" 2>/dev/null | grep -v "node_modules" | grep -v "\*\*\*" | head -20

# GCP credentials
gcloud auth list
ls ~/.config/gcloud/
```

### 7. Serviços Expostos — Específicos
```bash
# ChromaDB (porta 8000) — collection e auth
curl -s --max-time 5 "http://localhost:8000/api/v2/tenants/default_tenant/databases/default_database/collections" | python3 -m json.tool | head -20
# Testar acesso externo
curl -s --max-time 5 -I http://<IP_PUBLICO>:8000/

# Redis (6379)
ss -tlnp | grep 6379
# Testar acesso externo
timeout 5 bash -c 'echo PING | nc -w2 <IP_PUBLICO> 6379'

# Syncthing (22000)
ps aux | grep syncthing | grep -v grep
cat ~/.config/syncthing/config.xml | grep -E "address|guiListen"

# Tailscale
tailscale status
tailscale netcheck
```

### 8. Canal Directory e Auth (Telegram bots)
```bash
cat ~/.hermes/channel_directory.json | python3 -m json.tool | head -40
cat ~/.hermes/auth.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps({k: '***' if isinstance(v, str) and len(v) > 10 else v for k,v in d.items()}, indent=2))"
grep -r "botToken\|TELEGRAM_BOT_TOKEN" ~/.openclaw/ --include="*.env" 2>/dev/null | head -5
```

### 9. Serviços de Sistema
```bash
# systemctl running services
systemctl list-units --type=service --state=running | cat

# Cron jobs
crontab -l 2>/dev/null

# Docker
docker ps 2>/dev/null
```

---

## Findings Classification

| Símbolo | Severidade | Significado |
|---------|------------|-------------|
| 🔴 CRÍTICO | **HIGH** | Exposto ao mundo sem auth, dado sensível acessível |
| ⚠️ ALTO | **MEDIUM** | Exposto mas protegido por firewall/GCP ou risco moderado |
| 🟡 MÉDIA | **LOW** | Funcionalidade ativa mas configuração subótima |
| ✅ SEGURO | **INFO** | Configuração correta, nenhum risco identificado |

---

## Gaps Mais Comuns Encontrados
- **ChromaDB em 0.0.0.0:8000** sem auth → CRÍTICO se exposto
- **Redis em 0.0.0.0:6379** sem password → CRÍTICO se GCP FW permitir
- **Tokens de bots mascarados** → impossível verificar sem token real
- **CUPS e SSH expostos** ao mundo → superfície de ataque desnecessária
- **Tokens parcialmente visíveis** em .env (Brave, Finnhub, Linear, MiniMax)
- **Tailscale Desktop offline** — túnel SSH não está ativo
- **GCP Firewall rules** não verificáveis sem auth (gcloud 401)

## Gaps mais comuns achados
- PIDs desatualizados no diagrama (mecanismo de update automático necessário)
- RADAR token Telegram ausente → dry-run
- Scripts legados duplicados sem uso (~100 arquivos)
- Cobertura de testes 0% na maioria das ferramentas
