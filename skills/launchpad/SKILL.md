---
name: launchpad
description: Use when starting a new project, scaffolding a multi-project repo, or setting up Cloudflare infrastructure for a web + API application. Triggers on "new project", "scaffold", "create project", "init project", "start a new app".
---

# Launchpad

Scaffold flat multi-project repos with Cloudflare infrastructure. No monorepo tooling, no shared packages — each project has its own dependencies.

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

### 7. Application domain (only if API worker selected)

Ask for the domain the application will be hosted on. This is needed to configure Cloudflare worker routes so the API worker serves `/v1/*` on the same domain as the web app. If the user doesn't have a domain yet, suggest using a placeholder like `myapp.example.com` — they can update it later in wrangler.jsonc. Skip this question if no API worker was selected in step 3.

### 8. Resource creation

Ask if user wants Cloudflare resources created now via wrangler CLI, or later manually.

### 9. Remote resources

Ask if user wants to test with remote resources during development. If yes, add `remote: true` to all resource bindings in wrangler.jsonc.

## Phase 2 — Scaffold

### Initialize projects using CLI

**CRITICAL:** Always use CLI tools to initialize projects. Never write initial project code directly.

**Web projects:**
```bash
npm create vite@latest {name} -- --template react-ts
```

**Cloudflare Workers:**
```bash
npm create cloudflare@latest {name}
```
Select "Hello World" worker template when prompted.

**iOS projects:**
```bash
# Ask user to create via Xcode: File → New → Project → App (SwiftUI)
```

**Android projects:**
```bash
# Ask user to create via Android Studio: New Project → Empty Activity (Kotlin)
```

If any CLI fails or requires interactive input that can't be automated, **ask the user to run it manually** and continue after they confirm.

### Install dependencies

After CLI initialization, install dependencies via `npm install`. This pins to the latest stable version at install time — never use the `latest` tag.

**Web projects:**
```bash
cd {project}
npm install @tanstack/react-router @tanstack/react-query axios date-fns clsx tailwind-merge class-variance-authority lucide-react
npm install -D tailwindcss @tailwindcss/vite typescript @vitejs/plugin-react eslint wrangler
```

Add auth SDK if selected:
- Clerk: `npm install @clerk/clerk-react @clerk/themes`
- Better Auth: `npm install better-auth`

**API workers:**
```bash
cd {project}
npm install hono drizzle-orm
npm install -D drizzle-kit wrangler @types/node
```

Add backend SDKs:
- Clerk: `npm install @clerk/backend`
- Resend: `npm install resend`

**Lightweight workers:** Only install what's specifically needed. Don't add Hono or Drizzle.

### Rename projects

After initialization, rename project directories to match the user's chosen names. Update `name` field in package.json accordingly.

## Phase 3 — Configure

### Dev server ports

Each sub-project must use a unique 5-digit random port (10000–65535) for its dev server to avoid conflicts with other projects. Generate a port for each project using `echo $((RANDOM % 55536 + 10000))` and hardcode it in the config files below. Do not ask the developer — just pick the ports.

### wrangler.jsonc

For each worker, configure wrangler.jsonc:

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "{prefix}-{worker-name}",
  "main": "src/index.ts",
  "compatibility_date": "{today}",
  "observability": { "enabled": true },
  "upload_source_maps": true,
  "compatibility_flags": ["nodejs_compat"],
  "dev": { "port": {port} }
  // Add resource bindings below
}
```

**For API workers only**, add a `routes` array to the same wrangler.jsonc so the worker handles `/v1/*` on the application domain:

```jsonc
  "routes": [
    {
      "pattern": "{domain}/v1/*",
      "zone_name": "{zone}"
    }
  ]
```

`{domain}` is the application domain from Phase 1. `{zone}` is the root domain — if domain is `app.example.com`, zone is `example.com`. If domain has no subdomain (e.g. `example.com`), zone and domain are the same value.

**Resource naming — always use the project prefix:**

| Resource | Name pattern |
|---|---|
| Worker | `{prefix}-{name}` |
| D1 | `{prefix}-d1` |
| R2 | `{prefix}-r2` |
| KV | `{prefix}-kv` |
| Queue | `{prefix}-{purpose}-queue` |
| DLQ | `{prefix}-{purpose}-dlq` |
| Pages | `{prefix}-web` |

Never use generic names like "api", "database", "storage" that conflict with other projects.

### vite.config.ts

For web projects, configure vite with the dev server port. If the project has an API worker, include the `/v1` proxy so API calls work during local development:

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
  server: {
    port: {port},
    // Include proxy only if project has an API worker
    proxy: {
      "/v1": {
        target: "http://localhost:{api-port}",
        changeOrigin: true,
      },
    },
  },
});
```

Where `{api-port}` is the API worker's dev server port from wrangler.jsonc.

### package.json scripts

**All workers:**
```json
{
  "dev": "wrangler dev",
  "deploy": "wrangler deploy --minify",
  "cf-typegen": "wrangler types --env-interface CloudflareBindings"
}
```

**Workers with D1 (add these):**
```json
{
  "db:generate": "drizzle-kit generate",
  "db:migrate:local": "wrangler d1 migrations apply {prefix}-d1 --local",
  "db:migrate:remote": "wrangler d1 migrations apply {prefix}-d1 --remote"
}
```

**Web projects:**
```json
{
  "dev": "vite",
  "build": "tsc -b && vite build",
  "preview": "vite preview",
  "lint": "eslint .",
  "deploy": "npm run build && wrangler pages deploy dist --project-name {prefix}-web"
}
```

### Generate types

Run `npm run cf-typegen` in each worker after configuring wrangler.jsonc. This generates the `CloudflareBindings` interface. **Never create these types manually.**

For secrets: add them to `.dev.vars` first, then run `cf-typegen` so they appear in the generated types.

### Create Cloudflare resources

If user opted in, create resources via wrangler CLI:

```bash
wrangler pages project create {prefix}-web
wrangler d1 create {prefix}-d1
wrangler r2 bucket create {prefix}-r2
wrangler kv namespace create {prefix}-kv
wrangler queues create {prefix}-{purpose}-queue
```

Save the IDs from command output and update wrangler.jsonc bindings.

If `remote: true` was requested, add it to every resource binding.

### Drizzle setup (workers with D1)

Create `drizzle.config.ts`:
```typescript
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./migrations",
  dialect: "sqlite",
});
```

Create `src/db/schema.ts` with a minimal starter schema. Then:
```bash
npm run db:generate      # generates migration — NEVER edit the generated file
npm run db:migrate:local # apply locally
```

If remote resources are enabled: `npm run db:migrate:remote`

### API worker entry point

Create `src/index.ts` for API workers with all routes under `/v1`:

```typescript
import { Hono } from "hono";

const app = new Hono<{ Bindings: CloudflareBindings }>().basePath("/v1");

app.get("/health", (c) => c.json({ status: "ok" }));

export default app;
```

All API routes must be registered on this app instance — they will automatically be prefixed with `/v1`.

### Axios API client

Create `src/lib/api.ts` in web projects to configure axios for the API:

```typescript
import axios from "axios";

export const api = axios.create({
  baseURL: "/v1",
});
```

All API calls in the web app must use this `api` instance instead of importing axios directly.

### Environment config

Create `src/config.ts` in each project. Every env var accessed through validated getters that throw on missing values.

**Worker config pattern:**
```typescript
export function getConfig(env: CloudflareBindings) {
  return {
    get dbExample(): string {
      if (!env.SOME_SECRET) throw new Error("SOME_SECRET is not set in .dev.vars");
      return env.SOME_SECRET;
    },
  };
}
```

**Web config pattern:**
```typescript
export const config = {
  get someKey(): string {
    const val = import.meta.env.VITE_SOME_KEY;
    if (!val) throw new Error("VITE_SOME_KEY is not set in .env");
    return val;
  },
};
```

**Rules:**
- Vars go in wrangler.jsonc (under `vars`)
- Secrets go in `.dev.vars`
- Types generated via `cf-typegen` — never written manually
- Never set default values for env vars in code
- Every getter validates and throws

### Run dev servers

After all configuration is complete, use groo CLI to start dev servers for all sub-projects:

```bash
groo dev
```

This discovers all services with `dev` scripts, presents an interactive selector, and launches selected services in parallel with color-coded output. It auto-detects ports from vite.config and wrangler.jsonc — no extra configuration needed.

Instruct the user to run `groo dev` to verify all services start correctly before moving on.

## Phase 4 — Project Files

### .claude/settings.local.json

Create with deny rules for every worker that has migrations, and attribution config:

```json
{
  "permissions": {
    "deny": [
      "Edit({worker}/migrations/**)",
      "Write({worker}/migrations/**)"
    ]
  },
  "attribution": {
    "commit": "",
    "pr": ""
  }
}
```

### CLAUDE.md

Generate with project name, description, coding practices, and structure. Include ALL of these coding practices:

```markdown
# {Project Name}

{Description}

## Coding Practices

### Error Handling
- Never write code in huge try/catch blocks. Catch errors at the specific
  operation that can fail. Each catch must log the error with full context.
- Never suppress errors or build fallbacks that hide problems. Log everything,
  let errors surface.
- If the correct approach fails, throw an error. Don't patch around it with
  fallback solutions — debug and fix the root cause.

### Environment Variables & Configuration
- Never use default values where a value is expected. If an env var, function
  parameter, or API param is required and missing, throw an error immediately.
- Never create types for env vars manually. Run `npm run cf-typegen` to generate
  types from wrangler.jsonc. Add secrets to `.dev.vars`, then run cf-typegen.
- Access all env vars through config.ts getters that validate and throw on missing.
- Vars go in wrangler.jsonc. Secrets go in .dev.vars. Never set defaults in code.

### Database Migrations
- Never manually create or edit migration files in `migrations/` directories.
- Generate migrations from Drizzle schema: `npm run db:generate`
- Apply with wrangler: `npm run db:migrate:local` / `npm run db:migrate:remote`

### UI Design
- Always use the frontend-design skill to design user interfaces.

### Planning
- For complex or multi-step tasks, use the brainstorming skill first.
```

If iOS/SwiftUI project exists, add:
```markdown
### SwiftUI
- After writing SwiftUI code, use swiftui-pro skill to review before committing.
  Show the review to the developer and let them choose what to fix.
```

Include project structure listing and deployment instructions.

### README.md

Generate with:
- Project name + description
- Project structure (each sub-project and what it does)
- Prerequisites: Node.js, wrangler, groo CLI (`brew install groo-dev/tap/groo`)
- Setup: clone, install deps per project, copy `.env.example`/`.dev.vars.example`
- Run locally: `groo dev`
- Deploy: `npm run deploy` per project
- Environment variables table

### Example files

**`.env.example`** for each web project:
```
VITE_CLERK_PUBLISHABLE_KEY=pk_test_xxxx
```

**`.dev.vars.example`** for each worker:
```
CLERK_SECRET_KEY=sk_test_xxxx
RESEND_API_KEY=re_xxxx
```

List only the vars that are actually needed based on selected integrations.

### .gitignore

```
node_modules/
dist/
.dev.vars
.env
.wrangler/
*.log
.DS_Store
.playwright-mcp/
```

### Git init

```bash
git init
git add -A
git commit -m "chore: initial project scaffold via launchpad"
```

## Important Rules

- **Never write initial project code directly** — always use CLI tools to scaffold
- **If a CLI fails, ask the user to run it manually** — don't silently skip or hack around it
- **Never edit migration files** — only generate via `npm run db:generate`
- **Never create env var types manually** — only via `npm run cf-typegen`
- **Never use default values for env vars** — throw errors on missing values
- **Always use project prefix for Cloudflare resources** — prevents naming conflicts
- **Install deps via `npm install`** — pins versions at install time, never use `latest` tag
- **For email integration, always use Resend**
- **Each project is independent** — no shared packages, no monorepo tooling, duplication preferred
- **All API routes must be under `/v1`** — use `app.basePath("/v1")` in Hono, web app calls `/v1/*` with relative URLs (same domain)
