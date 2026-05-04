---
name: sias-canary-scan
description: SIAS 6-phase cycle Phase A — Canary scan de ferramentas. Testa cada tool in-place, identifica gaps, classifica por ICE score. Usado para auditing de manutenção, diagnóstico de здоровья sistema, e validação de integridade.
category: software-development
---

# SIAS — Phase A: Canary Scan
> **Versão:** 2.0 | **Atualizado:** 2026-04-22
> **Baseado em:** SIAS MASTER ARCHITECTURE v2.0 — P1

Valida saúde de um conjunto de ferramentas/scripts executando cada um e classificando resultados.

## Quando usar

- CYCLE_xxx auditing de ferramentas de manutenção
- Diagnóstico de saúde do ecossistema (Cortex, OpenClaw, etc.)
- Validação pós-migração ou pós-deploy
- Primeira fase do ciclo SIAS (A→B→C→D→E→F)

## Execução

### 1. Listar ferramentas

```bash
cd ~/.hermes/workspace
find <dir> -maxdepth 1 -type f \( -name "*.py" -o -name "*.sh" \) | sort
```

### 2. Testar cada ferramenta

```bash
# Scripts Python (usar python3, não python)
python3 <tool>.py 2>&1; echo "EXIT:$?"

# Scripts Shell
bash <tool>.sh 2>&1; echo "EXIT:$?"

# Para long-running daemons (data_monitor_scheduler, etc.)
# usar timeout 30s
timeout 30 python3 <tool>.py 2>&1; echo "EXIT:$?"
```

### 3. Classificar resultado

| Código Exit | Classificação |
|-------------|---------------|
| 0 | ✅ PASS |
| 1 | ⚠️ WARN (funcional mas com alertas) |
| 2 | ❌ ERROR (arquivo não existe / args faltantes) |
| 124 | ⏱️ TIMEOUT (daemon que não termina) |

## Armadilhas descobertas

### Nomes de arquivo divergem da documentação

O index/map pode listar `reindex_cortex.py` mas o arquivo real é `reindex_cortex.sh`. **Sempre verificar com `ls` antes de executar.**

### Long-running daemons

`data_monitor_scheduler.py` inicia como daemon e não retorna. Usar `timeout 30` para testar apenas a inicialização.

### Scripts shell executados com `python`

`check_schedulers.sh` é shell, não Python. Se executar com `python3` dá SyntaxError. Sempre verificar shebang com `head -1 <file>`.

### Ferramentas que requerem argumentos

Algumas falham com exit 1 se não receberem args:
- `inspect_scheduler.py <scheduler_file.py>` — requer path
- `cortex_quarantine.sh {file} '{reason}' [--dry-run]` — requer 2 args
- `decide_keeper.py` — não suporta `--dry-run` (erro: `FileNotFoundError: '--dry-run'`)

### Dry-run nem sempre existe

Se uma tool não suporta `--dry-run`, a flag passa como argumento e causa erro. Testar primeiro sem flags.

### Bugs comuns encontrados em tools de manutenção

**1. Variáveis indefinidas (NameError)**
Constantes usadas mas nunca definidas causam `NameError` na execução. Exemplo: `SYNAPSE` usado mas não definido. Fix: adicionar definições no topo do arquivo.

**2. Caminhos de diretório errados**
`os.path.expanduser("~/backups/")` expande para `/home/user/backups/`, não para o caminho esperado em `~/.hermes/workspace/workspace/cortex/backups/`. Sempre usar caminhos completos.

**3. Padrões de arquivo errados no health check**
O health check pode buscar `*.bak_*` mas o backup_rotate cria `cortex_backup_*.tar.gz`. Incluir múltiplos padrões de arquivo.

**4. `import os` faltando**
Código usa `os.path` mas `import os` foi esquecido → `NameError`. Verificar imports ao analisar erros.

**5. Caminhos relativos em chamadas subprocess**
`subprocess.run(["python3", script, "data_scheduler.py"])` falha se o subprocess espera caminho completo. Montar caminho absoluto antes de chamar.

**6. Subdiretórios ARQUIVOS_CORE duplicados**
Padrão: `BACKUP_AGENTES_CORE/*/ARQUIVOS_CORE/` contém cópias exatas dos arquivos do diretório pai. Identificar com `detect_duplicates.sh`, remover apenas após confirmação.

## Template — Tabela de Resultados

| Ferramenta | Status | Notas |
|------------|--------|-------|
| `<tool>.py` | ✅/⚠️/❌ | <observação> |

## Template — ICE Score

| Gap | Impacto (1-3) | Confiança (1-3) | Esforço (1-3) | Score | Prioridade |
|-----|---------------|-----------------|---------------|-------|------------|
| `<descrição>` | 🔴/🟡/🟢 | 🔴/🟡/🟢 | 🔴/🟡/🟢 | I×C×E | P1/P2/P3 |

Score ≥ 6 = P1 (corrigir primeiro)
Score 3-5 = P2
Score 1-2 = P3

## Ciclo SIAS completo

| Fase | Nome | Descrição |
|------|------|-----------|
| A | Canary Scan | Testar cada ferramenta, coletar resultados |
| B | ICE Score | Priorizar gaps encontrados |
| C | Pesquisa | Investigar causas raiz dos gaps |
| D | Testar | Executar testes unitários/integração |
| E | Integrar | Aplicar correções ao sistema |
| F | Inovações | Propor melhorias para próximo ciclo |
