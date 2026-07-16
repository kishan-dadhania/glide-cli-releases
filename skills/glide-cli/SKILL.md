---
name: glide-cli
description: Operate the Glide CLI — deploy, review PRs, run releases, view the release train, and manage config. Use when the user mentions `glide`, `glide deploy`, `glide pr-review`, `glide release`, `glide release-train`, `glide config`, `glide health`, `glide login`, the Glide Deployment Platform, or the glide-agents repo CLI.
---

# Glide CLI

Terminal client for the Glide Deployment Platform. Talks to `<api_url>/api/v1/` via `X-Glide-API-Key`.

## Before anything else — check setup

```bash
# Is it installed?
which glide

# If not installed globally, run from the repo:
cd /Users/kishand/GLIDE/OTHER_CODE/glide-agents/glide-cli
uv run glide <command>
```

## Auth & config

Config lives at `~/.glide/config.json`. Run `glide login` to set it up interactively.

Env var overrides (take precedence over file): `GLIDE_API_URL`, `GLIDE_API_KEY`.

⚠️ `glide config --url <url>` sometimes prints success but doesn't write. Verify: `cat ~/.glide/config.json`.

## Action recipes

### User says "deploy X to Y"

```bash
uv run glide deploy <app> to <env> [from <branch>]   # watch live
uv run glide deploy <app> to <env> --no-watch         # fire-and-forget → returns request ID
```

Then monitor:
```bash
uv run glide status <request-id> --watch
```

**Variants:**
- `glide deploy promote <app> --branch <b>` — promote dev → QA
- `glide deploy rerun <request-id>` — retry a failed deploy
- `glide deploy cancel <request-id>` — cancel a pending/executing deploy

### User says "review PR X"

```bash
uv run glide pr-review <github-pr-url>
# or multiple: uv run glide pr-review <url1> <url2>
```

Returns: verdict (REQUEST_CHANGES/APPROVED/COMMENT), risk level, blockers, major issues, suggested fixes.

### User says "release X"

```bash
uv run glide release <github-pr-url>                  # full pipeline: review → gate → release
uv run glide release <url> --skip-review              # skip AI review
uv run glide release <url> --skip-blockers            # proceed despite blockers (admin)
```

### User says "what's on the release train?"

```bash
uv run glide release-train                            # all repos
uv run glide release-train --repo owner/repo          # filter to one repo
```

### User says "show deployment history"

```bash
uv run glide history deploy [-n 20]
uv run glide history prs [-n 20]
uv run glide history releases [-n 20]
```

### User says "list apps / branches"

```bash
uv run glide deploy apps
uv run glide deploy branches
```

### User says "open the Glide dashboard"

```bash
uv run glide ui
```

## Reading output

- Deploy/watch: shows progress bar, Jenkins build URL, final status
- `--no-watch`: returns a request ID like `a1b2c3d4` — use with `glide status <id>`
- PR review: structured verdict with risk level and blockers
- History: tables with request ID, app, env, status, timestamp

## Error recovery

| Symptom | Fix |
|---------|-----|
| `glide: command not found` | Use `uv run glide` from the repo directory |
| `401 Unauthorized` | Re-run `glide login` |
| `500` on status check | Use the full UUID (not the 8-char prefix); or check `glide ui` |
| Config URL didn't update | Re-run `glide config --url <url>` — it may need a second attempt |

⚠️ **UUID truncation**: the CLI shows 8-char prefixes in UI but the API needs full UUIDs. When debugging with curl, always use full UUIDs.

## Command → endpoint quick reference

| CLI Command | API Endpoint | Auth Scope |
|-------------|-------------|------------|
| `glide deploy ...` | `POST /deploy/` | `deploy` |
| `glide deploy cancel <id>` | `DELETE /deploy/{id}` | `deploy` |
| `glide deploy rerun <id>` | `POST /deploy/{id}/rerun` | `deploy` |
| `glide deploy promote <app> --branch <b>` | `POST /deploy/` | `deploy` |
| `glide deploy apps` | `GET /deploy/apps` | any auth |
| `glide deploy branches` | `GET /deploy/branches` | any auth |
| `glide status <id> [--watch]` | `GET /deploy/{id}` | any auth |
| `glide pr-review <urls>` | `POST /prs/review` | `pr_review` |
| `glide release <urls>` | `POST /releases/pipeline` | `release` |
| `glide release-train` | `GET /releases/repos` + `GET /releases/train` | any auth |
| `glide history deploy` | `GET /deploy/history` | any auth |
| `glide history prs` | `GET /prs/history` | any auth |
| `glide history releases` | `GET /releases/history` | any auth |
| `glide ui` | — (opens browser) | none |
| `glide health` | `GET /health` | none |

## Security model

Server hashes keys (SHA-256) → looks up in `api_key` table. Roles:
- `admin` → `admin:all`
- `developer` → `deploy`, `pr_review`, `release`
- `deployer` → `deploy` only
