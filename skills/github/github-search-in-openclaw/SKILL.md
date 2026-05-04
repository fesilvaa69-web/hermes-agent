---
name: github-search-in-openclaw
description: How to search GitHub in this OpenClaw environment where `gh search` command is not available — use `gh api search/repositories` instead.
---

# GitHub Search in OpenClaw Environment

## Context
In this OpenClaw/Linux environment, `gh search` is NOT available:
```
$ gh search repos --help
unknown command "search" for "gh"
Available: actions, alias, api, auth, browse, code, codespace, ...
```

## Correct Approach: `gh api search/repositories`

Use the REST API directly:
```bash
gh api search/repositories?q=python+trading+stars:>100&sort=stars&per_page=10
```

Or with jq for formatted output:
```bash
gh api search/repositories \
  --jq '.items[] | "\(.full_name) \(.stargazers_count)⭐ \(.html_url)"' \
  -q "trading+python+stars:>100"
```

## Python Pattern
```python
import subprocess, json

token_result = subprocess.run(['gh', 'auth', 'token'], capture_output=True, text=True)
token = token_result.stdout.strip()

result = subprocess.run(
    ['gh', 'api', 'search/repositories',
     '-q', 'python+trading+stars:>100',
     '--jq', '.items[0:5] | .[] | {name, stars: .stargazers_count, url: .html_url}'],
    capture_output=True, text=True
)
data = json.loads(result.stdout)
for repo in data['items']:
    print(f"{repo['full_name']} {repo['stargazers_count']}⭐")
```

## Alternative: Python requests with curl
```python
import subprocess, json

token = subprocess.run(['gh', 'auth', 'token'], capture_output=True, text=True).stdout.strip()
r = subprocess.run([
    'curl', '-s', '-H', f'Authorization: Bearer {token}',
    '-H', 'Accept: application/vnd.github.v3+json',
    'https://api.github.com/search/repositories?q=python+trading+stars:>100&per_page=10'
], capture_output=True, text=True)
data = json.loads(r.stdout)
for repo in data['items']:
    print(f"{repo['full_name']} {repo['stargazers_count']}⭐")
```

## Available gh Commands in This Environment
| Command | Available |
|---------|-----------|
| `gh api <endpoint>` | ✅ |
| `gh auth status/token/login/logout` | ✅ |
| `gh browse <repo>` | ✅ |
| `gh repo clone/list/view` | ✅ |
| `gh search code` | ❌ |
| `gh search repos` | ❌ |

## Query Parameters for search/repositories
| Param | Description | Example |
|-------|-------------|---------|
| `q` | Search query | `python+trading+stars:>100` |
| `sort` | Sort field | `stars`, `updated`, `forks` |
| `per_page` | Results per page | `10` (max 100) |
| `order` | Sort order | `desc` (default) |

## Topics for Trading Agent Benchmark
1. Multi-agent trading: `multi+agent+trading+system+python`
2. ICT methodology: `ict+trading+strategy+python`
3. Architecture: `python+trading+bot+architecture`
4. ML for trading: `machine+learning+trading+signals+python`
5. Documentation: `trading+system+architecture+documentation`

## Verification
```bash
gh search repos --help
# Output: unknown command "search"

gh api --help
# Output: Available commands including api
```
