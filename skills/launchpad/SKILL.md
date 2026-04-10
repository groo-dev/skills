---
name: launchpad
description: Use when starting a new project, scaffolding a multi-project repo, or setting up Cloudflare infrastructure for a web + API application. Triggers on "new project", "scaffold", "create project", "init project", "start a new app".
---

# Launchpad

Scaffold flat multi-project repos with Cloudflare infrastructure. No monorepo tooling, no shared packages — each project has its own dependencies.

The LLM handles the conversation (Phase 1), builds a config JSON, and hands off to `groo launchpad` which does all the scaffolding, configuration, and file generation deterministically.

## Phase 1 — Gather Requirements

Ask these questions one at a time. Be conversational. When presenting multiple choice options, give each a short answer key (a/b/c or y/n) so the user can respond with a single character.

### 1. Project name

Ask the user for a project name. This becomes:
- The root directory name
- The Cloudflare resource prefix (e.g. `{name}-d1`, `{name}-api`)

### 2. One-line description

Ask what the project does in one sentence. Used for README and CLAUDE.md.

### 3. Sub-projects and features

Based on the description, **proactively suggest** which sub-projects are needed and what features each should have. Present your suggestion as a complete architecture and let the user accept, adjust, or remove.

Available project types:

| Type | Base scaffold | When to suggest |
|---|---|---|
| **web** | Vite + React + TypeScript | Almost always — most projects need a frontend |
| **worker** | Minimal Cloudflare Worker | Backend, API, queue consumers, cron, email handling |
| **ios** | SwiftUI | Only if user mentions mobile/iOS |
| **android** | Kotlin | Only if user mentions mobile/Android |

Each project gets features that drive what the CLI installs and configures:

**Web features:**

| Feature | What it sets up |
|---|---|
| `tailwind` | Tailwind CSS + Vite plugin |
| `shadcn` | shadcn/ui init, component library, cn() util |
| `tanstack-router` | @tanstack/react-router, route structure |
| `tanstack-query` | @tanstack/react-query, QueryClient provider |
| `axios` | axios HTTP client. If project has a worker with `hono`, configures `/v1` baseURL automatically. |
| `auth` | Frontend auth SDK (provider: `clerk`, `better-auth`, `simple`) |

**Worker features:**

| Feature | What it sets up |
|---|---|
| `hono` | Hono framework, entry point with basePath `/v1`, routes in wrangler.jsonc |
| `drizzle` | Drizzle ORM + Kit, schema, db scripts in package.json |
| `auth` | Backend auth SDK (provider: `clerk`, `better-auth`, `simple`) |
| `email` | Email SDK (provider: `resend`) |

A bare `worker` with no features gives a minimal CF Worker entry point. A bare `web` gives Vite + React + TypeScript.

Think about what each project actually needs and suggest based on the project description, not a fixed template. When a project has both web and worker with auth, add the `auth` feature to both — the CLI installs the frontend SDK (e.g. `@clerk/clerk-react`) on web and the backend SDK (e.g. `@clerk/backend`) on worker.

### 4. Cloudflare resources

Analyse the project requirements and suggest which Cloudflare resources are needed. For each resource, reason about whether it should be a single shared instance or separate instances — consider data isolation needs, access patterns, and operational concerns. Explain your reasoning so the user can make an informed decision.

For example: a project with an API worker and a queue worker might share one D1 (same data, different access patterns) but need separate queues (different event types). Present your suggestion with reasoning and let the user accept or adjust.

| Resource | When to suggest |
|---|---|
| D1 | Project stores data |
| R2 | File uploads, attachments, backups |
| KV | Session cache, config storage, rate limiting |
| Queues | Async processing, event-driven workflows |
| AI Gateway | LLM features |

Skip this question if the project has no workers.

### 5. Application domain

Ask for the preferred domain the application will be hosted on. If the user doesn't have one, suggest `{name}.groo.bot` as a subdomain. This domain is used to configure Cloudflare worker routes (for workers with `hono` feature serving `/v1/*`) and Pages custom domains. Domain is required if any worker has the `hono` feature — don't let the user skip in that case. If no worker has `hono`, the user can skip.

### 6. Resource creation (only if resources were selected)

Ask if user wants Cloudflare resources created now via wrangler CLI, or later manually. Skip if no resources were selected in step 4 — default `create_resources` to `false`.

### 7. Remote resources (only if resource creation is yes)

Ask if user wants to test with remote resources during development. If yes, set `remote: true` in the config. Skip this question if the user chose not to create resources in step 6 — default `remote` to `false`.

## Phase 2 — Build Config & Run CLI

### 1. Build config JSON

Map the Phase 1 answers to the config schema below. Check if the current directory is empty — if so, set `root` to `"."`. Otherwise, set `root` to the project name (creates a new subdirectory).

### 2. Show config to user

Present the JSON and ask the user to confirm before proceeding. This is their last chance to adjust before scaffolding begins.

### 3. Write config file

Write the confirmed config to `launchpad.json` in the current working directory (not inside the target root — the CLI creates that directory).

### 4. Run the CLI

```bash
groo launchpad --config launchpad.json
```

The CLI handles everything: scaffolding, dependency installation, config file generation, boilerplate code, Cloudflare resource creation, project files (CLAUDE.md, README.md, TODO.md, GitHub Actions, .gitignore), and git init.

### 5. On success

Delete `launchpad.json` — it's no longer needed after a successful run. Then tell the user to run `groo dev` to start all dev servers and verify everything works. After that, ask if they'd like to start planning the development of the project. If yes, invoke the `brainstorming` skill.

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
  "resources": ["d1", "kv"],
  "projects": [
    {
      "name": "dashboard",
      "type": "web",
      "features": [
        { "type": "tailwind" },
        { "type": "shadcn" },
        { "type": "tanstack-router" },
        { "type": "tanstack-query" },
        { "type": "axios" },
        { "type": "auth", "provider": "clerk" }
      ]
    },
    {
      "name": "api",
      "type": "worker",
      "features": [
        { "type": "hono" },
        { "type": "drizzle" },
        { "type": "auth", "provider": "clerk" },
        { "type": "email", "provider": "resend" }
      ]
    },
    {
      "name": "queue-consumer",
      "type": "worker"
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
| `domain` | string | no | Application domain. Required if any worker has `hono` feature. |
| `resources` | array | no | `d1`, `r2`, `kv`, `queues`, `ai-gateway`. Shared across all workers. CLI creates each once and binds to every worker. |
| `projects` | array | yes | At least one project |
| `create_resources` | bool | yes | Whether to create Cloudflare resources via wrangler |
| `remote` | bool | yes | Whether to add `remote: true` to resource bindings |

### Project fields

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Project directory name. Must be unique across projects. |
| `type` | enum | yes | `web`, `worker`, `ios`, `android` |
| `features` | array | no | List of feature objects. Each has a `type` and optional config fields. |

### Feature objects

| Feature type | Available on | Optional fields | Description |
|---|---|---|---|
| `tailwind` | web | — | Tailwind CSS + Vite plugin |
| `shadcn` | web | — | shadcn/ui component library |
| `tanstack-router` | web | — | TanStack Router |
| `tanstack-query` | web | — | TanStack Query |
| `axios` | web | — | Axios HTTP client. Auto-configures `/v1` baseURL if a worker has `hono`. |
| `hono` | worker | — | Hono framework with `/v1` basePath |
| `drizzle` | worker | — | Drizzle ORM + Kit |
| `auth` | web, worker | `provider`: `clerk`, `better-auth`, `simple` | Auth SDK for the project type |
| `email` | worker | `provider`: `resend` | Email SDK |

Omit `features` when a project has none (bare web or bare worker).

### Feature dependencies

Some features and resources imply other features. The CLI auto-includes these — don't add them manually to the config. Only include the feature the user actually chose.

| When config has | Auto-includes | Reason |
|---|---|---|
| `shadcn` on web | `tailwind` on that project | shadcn requires Tailwind |
| `d1` in resources | `drizzle` on all workers | Workers need ORM for DB access |
| `drizzle` on any worker | `d1` in resources | Drizzle needs a database |
| `auth(better-auth)` or `auth(simple)` | `d1` in resources + `drizzle` on all workers | Self-hosted auth stores data in D1 |

### Validation rules

- Feature types must be valid for the project type (e.g. `hono` only on `worker`, `tailwind` only on `web`)
- `ios` and `android` projects must not have `features`
- `domain` is required if any worker has the `hono` feature
- Project names must be unique
- `auth` feature must include `provider`
- `email` feature must include `provider`
