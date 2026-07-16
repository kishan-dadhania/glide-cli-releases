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

Always use `--no-watch` to get the request ID immediately, then poll status every 30s until terminal.

**Step 1 — Fire the deploy:**
```bash
cd /Users/kishand/GLIDE/OTHER_CODE/glide-agents/glide-cli
uv run glide deploy deploy <app> to <env> [from <branch>] --no-watch
```
Returns a request ID immediately. Example: `🚀 Deployment requested! ID: 414d7753-a4f1-45ae-90a2-b1c909a6e457`

**Step 2 — Poll every 30s until terminal state:**
```bash
uv run glide status <full-uuid>
```
Use the **full UUID** (not the 8-char prefix). Check the `Status` field:

| Status | What to do |
|--------|-----------|
| `pending`, `executing` | Wait 30s, poll again. Repeat until terminal. |
| `awaiting_approval` | **Dev deploys can be auto-approved.** Ask the user: "This deploy needs approval — approve it?" If yes, call the API directly (see Approving deploys below). For QA/prod deploys the user must approve via `glide ui`. |
| `succeeded` | Report success + Jenkins build URL if shown |
| `failed` | **Get the reason first.** If the status output doesn't show a failure detail, run `glide history deploy --limit 5` and `glide deploy apps` to cross-check. Common causes: unknown app name, invalid branch, Jenkins build failure. Report the specific reason to the user, then offer: `uv run glide deploy rerun <id>` |
| `cancelled` | Report it was cancelled |

**Variants:**
- `glide deploy promote <app> --branch <b>` — promote dev → QA (same polling pattern)
- `glide deploy rerun <request-id>` — retry a failed deploy (same polling pattern)
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
uv run glide history deploy --limit 20
uv run glide history prs --limit 20
uv run glide history releases --limit 20
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

## Approving deploys

When a deploy hits `awaiting_approval`, call the decision API directly. Read API URL + key from `~/.glide/config.json`:

```bash
API_URL=$(cat ~/.glide/config.json | python3 -c "import sys,json; print(json.load(sys.stdin)['api_url'])")
API_KEY=$(cat ~/.glide/config.json | python3 -c "import sys,json; print(json.load(sys.stdin)['api_key'])")
curl -s -X POST "$API_URL/api/v1/deploy/<full-uuid>/decision" \
  -H "X-Glide-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"decision":"approved"}'
```

Returns `{"status":"decision_submitted"}` on success. Then resume 30s polling — the deploy will move to `executing` → `succeeded`/`failed`.

**For dev deploys**: the deployer's own `deploy` scope is sufficient to approve — no admin needed.

## Reading output

- Deploy with `--no-watch`: returns a request ID immediately — poll with `glide status <id>`
- Status output: shows App, Env, Status, and for successful deploys a Jenkins build URL
- PR review: structured verdict with risk level and blockers
- History: tables with request ID, app, env, status, timestamp — use `--limit` (not `-n`)

## Error recovery

| Symptom | Fix |
|---------|-----|
| `glide: command not found` | Use `uv run glide` from the repo directory |
| `401 Unauthorized` | Re-run `glide login` |
| `500` on status check | Use the full UUID (not the 8-char prefix); or check `glide ui` |
| Config URL didn't update | Re-run `glide config --url <url>` — it may need a second attempt |
| Deploy fails with no reason | Run `glide history deploy --limit 5` and `glide deploy apps` to diagnose |
| History flag `-n` not recognized | Use `--limit` (e.g. `--limit 20`), not `-n` |

⚠️ **UUID truncation**: the CLI shows 8-char prefixes in UI but the API needs full UUIDs. When debugging with curl, always use full UUIDs.

⚠️ **Command structure**: deploy uses double `deploy`: `glide deploy deploy <app> to <env>`. Not `glide deploy <app> to <env>`.

## Command → endpoint quick reference

| CLI Command | API Endpoint | Auth Scope |
|-------------|-------------|------------|
| `glide deploy deploy ...` | `POST /deploy/` | `deploy` |
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
