---
name: docker-compose-production-hardening
description: Docker Compose production hardening para serviços Python — healthchecks, resource limits, auto-heal, Dockerfile multi-stage, e avaliação k8s.
tags: [docker, compose, production, healthcheck, kubernetes, hardening]
---

# Docker Compose Production Hardening

> Author: Hermes Agent (@Jarvis_Hermess_Bot)

## Quando Usar

- Serviço Python existente precisa de deploy em produção via Docker
- Precisa de healthchecks, resource limits, restart policies
- Avaliar se Kubernetes faz sentido para o caso
- Adicionar auto-heal scripts
- Multi-container architecture (app + redis + chroma + monitoring)

## Padrão de Entrega (sempre fazer todos)

### 1. docker-compose.yml Production

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.prod
    env_file:
      - config/.env
    volumes:
      - ./kb:/app/kb:ro
      - ./logs:/app/logs
    restart: unless-stopped
    mem_limit: 512m
    cpus: 1.0
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - mentorbot_backend

  redis:
    image: redis:7-alpine
    container_name: mentorbot-redis
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD:-mentoriabot}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    mem_limit: 256m
    cpus: 0.5

  chroma:
    image: chromadb/chroma:0.6.3
    container_name: mentorbot-chroma
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/v2/heartbeat"]
      interval: 10s
      timeout: 5s
      retries: 3
    mem_limit: 512m
    cpus: 1.0

networks:
  mentorbot_backend:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
```

### 2. Dockerfile.prod (multi-stage)

```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /app
RUN python -m venv /opt/venv
COPY requirements.txt
RUN /opt/venv/bin/pip install -r requirements.txt

FROM python:3.11-slim AS production
RUN groupadd -r appuser && useradd -r -g appuser appuser
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY --chown=appuser:appuser src/ ./src/
RUN chmod +x src/main.py
USER appuser
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8001/health || exit 1
ENTRYPOINT ["python", "-m", "src.main"]
```

### 3. healthcheck.sh com auto-heal

```bash
#!/bin/bash
# Health check + auto-heal para Docker Compose
# Uso: ./scripts/healthcheck.sh [--json]

check_service() {
    docker compose -f "$(dirname "$0")/../docker-compose.yml" ps "$1" \
        --format json 2>/dev/null | grep -o '"State":"[^"]*"' | cut -d'"' -f4
}

# Redis e ChromaDB: restart automático se down
if [[ $(check_service redis) != "running" ]]; then
    docker compose restart redis
fi
```

### 4. K8s evaluation doc

Sempre criar `docs/KUBERNETES_ALTERNATIVE.md` avaliando:
- **Não faz sentido** para menos de 5 containers, 1 nó, times sem experiência k8s
- **Faz sentido** para mais de 10 containers, multi-nó, zero-downtime obrigatório
- **Alternativa**: k3s (kubernetes leve, compatível com manifests)

## Checklist de Produção

- [ ] Healthchecks em TODOS os serviços
- [ ] Resource limits (CPU/mem) definidos
- [ ] Restart policy (unless-stopped)
- [ ] Rede isolada (bridge custom)
- [ ] Secrets via env_file (nunca hardcoded)
- [ ] Usuário não-root no container
- [ ] Multi-stage build (imagem menor)
- [ ] Healthcheck script com auto-heal
- [ ] Docs: KUBERNETES_ALTERNATIVE.md
- [ ] Git commit com tudo

## Armadilhas Conhecidas

1. **Portas em 127.0.0.1** — ChromaDB e Redis binds localhost-only por segurança
2. **depends_on sem condition** — container sobe antes do dependente estar ready
3. **UID do volume** — volumes Docker podem ter permissões root; usar chown -R
4. **docker.sock** — só montar se o container precisar restartar outros containers
5. **Healthcheck no Dockerfile** — conflita com healthcheck no compose; usar só no compose

## Commits Típicos

```
feat: docker-compose production setup (4 containers + healthchecks + auto-heal)
docs: adiciona kubernetes alternative guide
fix: Dockerfile.prod — usuário não-root + multi-stage build
```
