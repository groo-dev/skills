---
name: launchpad
description: Use when starting a new project, scaffolding a multi-project repo, or setting up Cloudflare infrastructure for a web + API application. Triggers on "new project", "scaffold", "create project", "init project", "start a new app".
---

# Launchpad

Scaffold flat multi-project repos with Cloudflare infrastructure. No monorepo tooling, no shared packages — each project has its own dependencies.

The LLM handles the conversation (Phase 1), builds a config JSON, and hands off to `groo launchpad` which does all the scaffolding, configuration, and file generation deterministically.

## Phase 1 — Gather Requirements

Ask these questions one at a time. Be conversational.

### 1. Project name

Ask the user for a project name. This becomes:
- The root directory name
- The Cloudflare resource prefix (e.g. `{name}-d1`, `{name}-api`)

### 2. One-line description

Ask what the project does in one sentence. Used for README and CLAUDE.md.

### 3. Sub-projects

Based on the description, **proactively suggest** which sub-projects are needed. Present your suggestion and let the user accept, add, or remove.

Available project types:

| Type | Stack | When to suggest |
|---|---|---|
| **Web app** | Vite + React + TanStack Router + TanStack Query + axios + date-fns + Tailwind CSS | Almost always — most projects need a frontend |
| **API worker** | Hono + Drizzle on Cloudflare Worker | When the project needs a backend with database |
| **Lightweight worker** | Minimal CF Worker (no Hono/D1) | For event-driven tasks: email handling, queue consumers, cron jobs |
| **iOS app** | SwiftUI | Only if user mentions mobile/iOS |
| **Android app** | Kotlin | Only if user mentions mobile/Android |

### 4. Cloudflare resources

For each worker, **proactively suggest** which resources it needs. User accepts, adds, or removes.

| Resource | When to suggest |
|---|---|
| D1 | API workers that store data |
| R2 | Projects with file uploads, attachments, backups |
| KV | Session cache, config storage, rate limiting |
| Queues | Async processing, event-driven workflows |
| AI Gateway | Projects using LLM features |

### 5. Auth

Ask if the project needs authentication. Suggest options:
- **Clerk** — managed auth with prebuilt UI components, orgs support
- **Better Auth** — open-source, self-hosted, flexible
- **Simple email/password** — custom implementation
- **None** — no auth needed

### 6. Email

Ask if the project needs email integration. If yes, always use **Resend**.

### 7. Application domain

Ask for the preferred domain the application will be hosted on. If the user doesn't have one, suggest `{name}.groo.bot` as a subdomain. This domain is used to configure Cloudflare worker routes (for API workers serving `/v1/*`) and Pages custom domains. If no API worker was selected in step 3, this is still useful for the Pages deployment but not strictly required — let the user skip if they prefer.

### 8. Resource creation

Ask if user wants Cloudflare resources created now via wrangler CLI, or later manually.

### 9. Remote resources

Ask if user wants to test with remote resources during development. If yes, add `remote: true` to all resource bindings in wrangler.jsonc.

## Phase 2 — Build Config & Run CLI

### 1. Build config JSON

Map the Phase 1 answers to the config schema below. Use `"."` for `root` if the user wants to scaffold in the current directory, or the project name if creating a new directory.

### 2. Show config to user

Present the JSON and ask the user to confirm before proceeding. This is their last chance to adjust before scaffolding begins.

### 3. Write config file

Write the confirmed config to `launchpad.json` in the target directory.

### 4. Run the CLI

```bash
groo launchpad --config launchpad.json
```

The CLI handles everything: scaffolding, dependency installation, config file generation, boilerplate code, Cloudflare resource creation, project files (CLAUDE.md, README.md, TODO.md, GitHub Actions, .gitignore), and git init.

### 5. On success

Tell the user to run `groo dev` to start all dev servers and verify everything works.

### 6. On failure

Read the CLI error output. Common actions:
- **Config validation error** — fix the config JSON based on the error message and retry
- **Scaffold command failed** — ask the user to run the failing command manually, then retry with `groo launchpad --config launchpad.json` (it resumes from where it left off)
- **Cloudflare resource creation failed** — likely an auth issue; ask the user to run `wrangler login`, then retry
- **To start completely fresh** — run `groo launchpad --config launchpad.json --clean`

## Config Schema Reference

```json
{
  "name": "myapp",
  "root": ".",
  "description": "A task management app with team collaboration",
  "domain": "myapp.groo.bot",
  "projects": [
    {
      "name": "dashboard",
      "type": "web",
      "auth": "clerk"
    },
    {
      "name": "api",
      "type": "api-worker",
      "auth": "clerk",
      "email": "resend",
      "resources": ["d1", "kv"]
    },
    {
      "name": "email-handler",
      "type": "lightweight-worker",
      "resources": []
    }
  ],
  "create_resources": true,
  "remote": false
}
```

### Top-level fields

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Project name — used as Cloudflare resource prefix. Must be alphanumeric, hyphens, underscores. |
| `root` | string | yes | `"."` for current directory, or a directory name to create |
| `description` | string | yes | One-line project description |
| `domain` | string | no | Application domain. Required if any project has type `api-worker`. |
| `projects` | array | yes | At least one project |
| `create_resources` | bool | yes | Whether to create Cloudflare resources via wrangler |
| `remote` | bool | yes | Whether to add `remote: true` to resource bindings |

### Project fields

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Project directory name. Must be unique across projects. |
| `type` | enum | yes | `web`, `api-worker`, `lightweight-worker`, `ios`, `android` |
| `auth` | enum | no | `clerk`, `better-auth`, `simple`. Not allowed on `lightweight-worker`, `ios`, `android`. |
| `email` | enum | no | `resend`. Only allowed on `api-worker`. |
| `resources` | array | no | `d1`, `r2`, `kv`, `queues`, `ai-gateway`. Only allowed on `api-worker` and `lightweight-worker`. |

### Validation rules

- `web` projects must not have `resources` or `email`
- `lightweight-worker` projects must not have `auth` or `email`
- `ios` and `android` projects must not have `resources`, `auth`, or `email`
- `domain` is required if any project has type `api-worker`
- Project names must be unique
