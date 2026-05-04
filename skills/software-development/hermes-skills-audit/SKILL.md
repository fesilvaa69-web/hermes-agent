---
name: hermes-skills-audit
description: Auditoria sistemática de skills Hermes + migração para repo GitHub específico de projeto. Inclui mapeamento, filtro por relevância, auditoria de conteúdo (ICT/domain-leakage, frontmatter, credenciais), patches, geração de documentação operacional (ARCHITECTURE, MAINTENANCE, OPERATIONS), criação de scripts utilitários, e push final. Executar quando Felipe pedir para auditar, migrar ou organizar skills.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [audit, skills, maintenance]
    related_skills: [soul-memory-updater, vm-diagram-maintenance]
---

# Hermes Skills Audit

Auditoria sistemática de skills — metodologia em 5 fases.

## Quando Usar

- Felipe pede "audite minhas skills"
- Após muitas sessões de criação de skills
- Periodicamente (recomendado: mensal)

## Fluxo de Auditoria (7 Fases)

### Fase 1: Mapeamento

```bash
# Listar todas skills
skills_list()

# Contar por categoria
ls -la ~/.hermes/skills/[categoria]/

# Mapear todas com data, tamanho e existência de SKILL.md
find ~/.hermes/skills -name "SKILL.md" -exec ls -la {} \; 2>/dev/null
```

### Fase 2: Filtrar por Relevância

Classificar cada skill em:
- **RELEVANTE**: aplicável ao projeto alvo (ex: MentorBot, Docker, Telegram)
- **GENERICA_ECOSSISTEMA**: útil no ecossistema mas não específica do projeto
- **DUPLICADA**: mesma skill em múltiplos diretórios
- **NAO_EXISTE**: referenciada mas diretório não existe

**Prioridade de auditoria:**
1. Core do projeto (ex: bot, docker, telegram para MentorBot)
2. DevOps/infra relacionada
3. Genéricas do ecossistema (verificar sewhite-label)

### Fase 3: Verificar GitHub Repo Existente

```bash
# Verificar se repo já existe
gh repo list fesilvaa69-web

# Criar repo se necessário
gh repo create hermes-mentorbot-skills --private
```

### Fase 4: Auditoria de Conteúdo (usar sub-agentes em paralelo)

Para cada skill, verificar:
- **ICT/Domínio específico**: grep por termos do domínio (ex: "ICT", "trading", "Supply Zone")
- **Frontmatter**: name + description obrigatórios; version, author, license, metadata.hermes opcionais
- **Desatualização**: comparar com código fonte real do projeto
- **Segurança**: verificar credenciais hardcoded (tokens, senhas)

```bash
# Verificar ICT content
grep -r "ICT\|trading\|Supply Zone\|Silver Bullet\|liquidity" ~/.hermes/skills/[skill]/

# Verificar frontmatter
head -20 ~/.hermes/skills/[skill]/SKILL.md

# Verificar credenciais
grep -r "token\|password\|secret\|key" ~/.hermes/skills/[skill]/SKILL.md
```

**Usar sub-agentes em paralelo** para acelerar (3-4 sub-agentes concurrently, cada um auditando 8-10 skills).

### Fase 5: Correções

Aplicar patches nas skills problemáticas:
- Remover conteúdo ICT/domain-specific
- Completar frontmatter
- Corrigir paths desatualizados
- Remover credenciais hardcoded

```python
# Padrão de frontmatter completo
---
name: skill-name
description: Descrição curta
version: 1.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [tag1, tag2]
    maintainer: Hermes Agent
---
```

### Fase 6: Migrar para GitHub Repo

1. Clonar repo localmente
2. Criar estrutura de diretórios (categoria/skill-name/)
3. Copiar skills validadas
4. Criar INDEX.md com catálogo
5. Commit + push

```bash
git clone git@github.com:fesilvaa69-web/hermes-mentorbot-skills.git ~/hermes-mentorbot-skills
```

**Estrutura recomendada:**
```
repo/
├── INDEX.md
├── mentorbot/          # skills específicas do bot
├── devops/            # skills Docker, infra
├── meta/              # authoring, index
└── misc/              # genéricas mas úteis
```

### Fase 7: Limpar e Validar

- Remover diretórios legados com conteúdo ICT (ex: docker/, sias/, tradicao/)
- Validar SKILL.md de cada skill copiada
- Verificar push final

```bash
# Commit final
git add -A && git commit -m "chore: clean migration of validated skills" && git push
```

### Fase 8: Gerar Documentação Operacional e Scripts

Após migrar as skills, se o repo for de OPS (ex: `hermes-mentorbot-ops`), gerar docs complementares:

**docs/SETUP.md** — Guia de deploy rápido (pré-requisitos, .env, docker compose up)
**docs/ARCHITECTURE.md** — Diagrama de componentes, fluxos, integrações, env vars, bugs conhecidos
**docs/MAINTENANCE.md** — Monitoramento (health checks, logs), backup (Redis, ChromaDB), upgrades, troubleshooting, limpeza
**docs/OPERATIONS.md** — Deploy, restart, rollback, acesso a containers, secrets management

**scripts/** — Scripts Bash funcionais:
- `health_check.sh` — checa todos os containers e serviços
- `backup_redis.sh` — Redis BGSAVE + cópia de segurança
- `cleanup_sessions.sh` — limpa sessions Redis com TTL > 7 dias
- `restart_bot.sh` — docker compose restart
- `logs_bot.sh` — streaming de logs
- `restore_backup.sh` — restore de backup

```bash
# Exemplo health_check.sh
#!/bin/bash
CONTAINERS="mentorbot-redis mentorbot-chroma mentorbot-app hermes-mentorbot"
for c in $CONTAINERS; do
  docker ps --filter "name=$c" --format "{{.Names}}: {{.Status}}" | grep -v "Up" && echo "🔴 $c DOWN"
done
```

```bash
# Commit da documentação
git add docs/ scripts/
git commit -m "feat: add complete ops documentation and scripts"
git push
```

### Checklist de Auditoria Completa

## Checklist de Auditoria Completa

```
□ Listar todas as skills (skills_list)
□ Mapear cada skill com data, tamanho, SKILL.md
□ Filtrar por relevância ao projeto alvo
□ Verificar se repo GitHub existe ou criar
□ Auditar conteúdo (ICT, frontmatter, segurança) — usar sub-agentes em paralelo
□ Corrigir skills problemáticas (patch)
□ Migrar skills validadas para repo local
□ Criar INDEX.md com catálogo
□ Limpar diretórios legados (remover ICT/OpenClaw)
□ Commitar e push final
□ Validar push no GitHub
```

---

## Armadilhas Comuns

| Problema | Solução |
|----------|---------|
| Dois repos com nomes similares | Identificar ANTES de começar: `hermes-mentorbot-skills` (genérico) vs `hermes-mentorbot-ops` (projeto específico). Usar sempre o repo de OPS |
| Plano não salvo | Usar path absoluto `~/.hermes/plans/` |
| Gateway cai durante auditoria | Verificar processos com `ps aux` |
| Skills duplicadas não identificadas | Comparar com `wc -l` + diff |
| Conteúdo ICT em skillswhite-label | grep por termos domínio específico antes de subir |
| Skills inexistentes no repo | Verificar git ls-tree, não confiar só no INDEX |
| Token hardcoded em skill | Remover credenciais antes de subir ao GitHub |
| Diretórios legados ICT no repo | Remover antes do commit final (rm -rf docker/ sias/ tradicao/) |
| Doc vazia (0 linhas) após sub-agente | Verificar `wc -l` de cada doc após gerar; se 0, gerar manualmente |
| Arquitetura divergence (collection RAG) | Documentar como BUG KNOWN no ARCHITECTURE.md para o time não perder tempo debugando |

## Insights de Auditorias Anteriores

- **Skills ICT/OpenClaw** em `hermes-mentorbot-skills`: docker/, sias/, tradicao/, cortex-radar/, hermes-maintenance/ devem ser removidos
- **Sub-agentes em paralelo**: 3-4 concurrentes para 100+ skills acelera 3-4x
- **Frontmatter mínimo**: name + description; versão completa inclui version + author + license + metadata.hermes
- **GitHub repo verificado primeiro**: `gh repo list` antes de criar
- **INDEX.md limpo**: gerar após limpar para não listar skills removidas
- **Docs ops**: SETUP.md, ARCHITECTURE.md, MAINTENANCE.md, OPERATIONS.md devem ser gerados na Fase 8 para repos de OPS

---

## Estrutura de Output

Ao completar auditoria, gerar:

```markdown
# Skills Audit Report — YYYY-MM-DD

## Resumo
- Total skills: XX
- Categorias: XX
- Novas: X | Atualizadas: X | Removidas: X

## Gaps Identificados
...

## Consolidações
...

## Próximos Passos
...
```

---

## Skills Relacionadas

- `soul-memory-updater` — auto-update de identidade
- `vm-diagram-maintenance` — manter VM diagram
- `cortex-reference` — docs de schedulers