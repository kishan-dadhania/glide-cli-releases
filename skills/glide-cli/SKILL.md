---
name: glide-cli
description: Terminal interface for the Glide Deployment Platform — deploy services, review PRs, run release pipelines, check release trains, and view history. Use this skill whenever the user mentions deploying to Glide, Glide CLI, glide deploy, glide release, PR reviews for Glide, checking deployment status, release trains, or any Glide platform operations even if they don't explicitly name the CLI tool.
---

# Glide CLI

Terminal tool for the **Glide AI-powered Deployment Platform**. Deploy to dev/QA, run AI code reviews, execute release pipelines, check release trains, and browse deployment history — all from the command line.

## Install

```bash
bash <(curl -fsSL https://gist.githubusercontent.com/kishan-dadhania/9f1726e99891456239b6c589ab0d7dc0/raw/install.sh)
```

Then configure:

```bash
glide login   # Enter API key + server URL (ask your platform admin)
glide health  # Verify connectivity
```

## Quick Reference

| Command | What it does |
|---------|-------------|
| `glide deploy <app> to <env> from <branch>` | Deploy an app |
| `glide deploy promote <app> --branch <b>` | Promote dev → QA |
| `glide deploy rerun <request-id>` | Rerun failed deployment |
| `glide deploy cancel <request-id>` | Cancel pending/executing deploy |
| `glide deploy apps` / `glide deploy branches` | List apps / branches |
| `glide status <request-id>` | Check deployment status |
| `glide pr-review <github-url>` | AI code review |
| `glide release <github-url>` | Full release pipeline |
| `glide release --skip-review` | Skip AI review in release |
| `glide release --skip-blockers` | Ignore review blockers |
| `glide history deploy` | Deployment history table |
| `glide history prs` | PR review history table |
| `glide history releases` | Release pipeline history |
| `glide release-train` | View open PRs on current release branch |
| `glide release-train --repo owner/repo` | Filter to one repo |
| `glide ui` | Open Web dashboard in browser |
| `glide config` / `glide login` | View/update settings |
| `glide --version` | Show version |

## Deploy Patterns

### Natural language (recommended)
```bash
glide deploy search to dev
glide deploy client to qa from release/26.05.24
glide deploy backend to dev from feature/my-branch
```

### Interactive (from repo dir)
```bash
cd ~/code/my-service
glide deploy               # auto-detects app + branch, prompts for env
```

### Watch / fire-and-forget
```bash
glide deploy backend to qa from master --no-watch   # returns request ID immediately
glide status <request-id> --watch                     # watch progress later
```

## PR Review Output Format

The AI review returns a structured result:
- **Verdict**: REQUEST_CHANGES / APPROVED / COMMENT
- **Risk level**: low / medium / high
- **Blockers**: must-fix issues before merge
- **Major issues**: should-fix before merge
- **Actions**: suggested fixes

## Release Pipeline

The release command runs: AI review → gate on blockers → create release branch → merge PRs → open release PR against main.

Use `--skip-review` to skip AI review and go straight to release.
Use `--skip-blockers` to proceed even if review finds blockers (admin only).

## Configuration

Config lives at `~/.glide/config.json`:
```json
{"api_url": "https://api-int-dev.bivotech.co/glide-agents", "api_key": "glide_..."}
```

Env var overrides: `GLIDE_API_URL`, `GLIDE_API_KEY` (useful for CI/CD).

## API Key Roles

| Role | Scopes |
|------|--------|
| `admin` | `admin:all` — full access |
| `developer` | `deploy`, `pr_review`, `release` |
| `deployer` | `deploy` only |

Keys are created by admins on the server via `make key-create NAME=<name> ROLE=<role>`.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `glide: command not found` | `pipx ensurepath` then restart shell |
| `401 Unauthorized` | `glide login` to re-enter API key |
| `500` on status | Use full UUID or `glide ui` |
| Wrong version | `pipx reinstall glide-cli` |
| `localhost:8000` in config | `glide config --url https://your-server.com` |

## Upgrading

```bash
pipx upgrade glide-cli
# or re-run the one-line installer
bash <(curl -fsSL https://gist.githubusercontent.com/kishan-dadhania/9f1726e99891456239b6c589ab0d7dc0/raw/install.sh)
```
