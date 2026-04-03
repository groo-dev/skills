# groo-dev/skills

Agent skills for scaffolding and automating development workflows.

Works with any agent that supports the [open agent skills](https://agentskills.io) standard — Claude Code, Cursor, GitHub Copilot, Gemini CLI, Codex, and [40+ others](https://skills.sh).

## Installation

```bash
npx skills add groo-dev/skills
```

## Skills

### `/launchpad`

Scaffold flat multi-project repos with Cloudflare infrastructure. No monorepo tooling, no shared packages — each project gets its own dependencies.

| Project | Stack |
|---------|-------|
| **Web** | Vite + React + TanStack Router/Query + axios + date-fns + Tailwind |
| **API** | Hono + Drizzle on Cloudflare Workers with D1 |
| **Workers** | Minimal CF Workers for email, queues, cron |
| **iOS** | SwiftUI |
| **Android** | Kotlin |

Cloudflare resources are auto-named with the project prefix.

## Contributing

Contributions are welcome! Feel free to open an issue or submit a pull request.

## License

MIT
