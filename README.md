# groo-dev/skills

A collection of Claude Code skills by [groo.dev](https://groo.dev).

## Install

### Claude Code plugin

```bash
claude plugin add groo-dev/skills
```

### vercel-labs/skills

```bash
npx skills add groo-dev/skills
```

## Available Skills

### launchpad

Scaffold flat multi-project repos with Cloudflare infrastructure.

```
/launchpad
```

Creates projects with:
- **Web** — Vite + React + TanStack Router/Query + axios + date-fns + Tailwind
- **API** — Hono + Drizzle on Cloudflare Worker with D1
- **Workers** — Minimal CF Workers for email, queues, cron
- **iOS** — SwiftUI
- **Android** — Kotlin

Each project is independent — no monorepo tooling, no shared packages. Cloudflare resources auto-named with project prefix.
