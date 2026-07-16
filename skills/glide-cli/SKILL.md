---
name: glide-cli
description: Operate the Glide CLI — deploy, review PRs, run releases, view the release train, and manage config. Use when the user mentions `glide`, `glide deploy`, `glide pr-review`, `glide release`, `glide release-train`, `glide config`, `glide health`, `glide login`, the Glide Deployment Platform, or the glide-agents repo CLI.
---

# Glide CLI

Terminal client for the Glide Deployment Platform. Talks to `<api_url>/api/v1/` via `X-Glide-API-Key`.

## Before anything else — check setup

```bash
which glide
# If not found, run from repo:
cd /Users/kishand/GLIDE/OTHER_CODE/glide-agents/glide-cli
uv run glide <command>
```

## Auth & config

Config: `~/.glide/config.json`. Run `glide login` to set up. Env vars `GLIDE_API_URL`, `GLIDE_API_KEY` override the file.

⚠️ `glide config --url <url>` sometimes prints success but doesn't write. Verify: `cat ~/.glide/config.json`.

## All commands support --json

Every CLI command accepts `--json` / `-j` for machine output. **Always use `--json` as an agent.**

⚠️ `--json` must come BEFORE positional args: `glide status --json <id>`, NOT `glide status <id> --json`.

## Action recipes

### "deploy X to Y" — use --wait (one command, one result)

```bash
cd /Users/kishand/GLIDE/OTHER_CODE/glide-agents/glide-cli
uv run glide deploy deploy <app> to <env> [from <branch>] --no-watch --json --wait
```
Polls every 5s, returns final status JSON. Timeout 15 min. Before deploying, check:
```bash
uv run glide deploy check <app> <env> --json
```

Variants: `promote`, `rerun`, `cancel`.

### "approve / reject deploy X"

```bash
uv run glide approve <full-uuid> [--json]
uv run glide reject <full-uuid> [--json]
```
Dev deploys: deployer's `deploy` scope is enough. QA: needs admin.

### "check deploy status"

```bash
uv run glide status --json <full-uuid>
```

Status reference:
| Status | Action |
|--------|--------|
| `pending`, `executing` | Wait, re-poll |
| `awaiting_approval` | `glide approve <id>` for dev; tell user for QA |
| `succeeded` | Report success |
| `failed` | Diagnose via `glide history deploy --limit 5 --json` + `glide deploy apps --json`, then offer `glide deploy rerun <id>` |
| `rejected`, `cancelled`, `expired` | Report and offer redeploy |

### "review PR X"

```bash
uv run glide pr-review <github-pr-url> [<url2> ...]
```

### "release X"

```bash
uv run glide release <github-pr-url> [--skip-review] [--skip-blockers]
```

### "release train"

```bash
uv run glide release-train [--repo owner/repo]
```

### "history"

```bash
uv run glide history deploy --limit 20 [--status succeeded/failed]
uv run glide history prs --limit 20
uv run glide history releases --limit 20
```
NOTE: `--status` does client-side filtering. Use `--json` for raw results.

### "list apps/branches"

```bash
uv run glide deploy apps --json
uv run glide deploy branches --json
```

### "open dashboard"

```bash
uv run glide ui
```

## Error recovery

| Symptom | Fix |
|---------|-----|
| `glide: command not found` | `uv run glide` from repo |
| `401` | `glide login` |
| `500` on status | Use full UUID, not 8-char prefix |
| Config URL didn't update | Re-run the command — may need second attempt |
| Deploy fails, no reason | `glide history deploy --limit 5 --json` + `glide deploy apps --json` |
| Flag `-n` not recognized | Use `--limit`, not `-n` |
| Dev deploy `awaiting_approval` | `glide approve <id>` — dev uses self-confirm, not QA approval |

⚠️ **UUID truncation**: CLI shows 8-char prefix but API needs full UUID.
⚠️ **Deploy syntax**: `glide deploy deploy <app> to <env>`. Natural-language `glide deploy <app> to <env>` now auto-routes.

## Command → endpoint

| CLI Command | Endpoint | Scope |
|-------------|----------|-------|
| `glide deploy deploy ...` | `POST /deploy/` | `deploy` |
| `glide deploy cancel/rerun/promote` | varies | `deploy` |
| `glide deploy apps/branches` | `GET /deploy/{apps,branches}` | any |
| `glide deploy check` | `GET /deploy/concurrency-check` | `deploy` |
| `glide approve/reject <id>` | `POST /deploy/{id}/decision` | `deploy` |
| `glide status <id>` | `GET /deploy/{id}` | any |
| `glide pr-review <urls>` | `POST /prs/review` | `pr_review` |
| `glide release <urls>` | `POST /releases/pipeline` | `release` |
| `glide release-train` | `GET /releases/repos` + `/releases/train` | any |
| `glide history *` | `GET /{deploy,prs,releases}/history` | any |
| `glide ui` | opens browser | none |
| `glide health` | `GET /health` | none |

## Security

Keys: SHA-256 hashed, stored in `api_key` table. Roles: `admin`→all, `developer`→deploy+review+release, `deployer`→deploy only.
