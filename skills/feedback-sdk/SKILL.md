---
name: feedback-sdk
description: Integrate the feedback system into any React + Cloudflare Worker project. Use when the user wants to add feedback collection, integrate feedback widget, or mentions "add feedback", "feedback SDK", "feedback widget".
---

# Feedback SDK Integration

Integrate `@groo.dev/feedback-sdk` into a React + Cloudflare Worker project. The SDK provides a feedback widget (React components) and a proxy middleware (Hono) that connects to the feedback system at `feedback.miranet.work`.

## Prerequisites

- `@groo.dev/feedback-sdk` published on npm
- A Feedback API key (from the feedback admin dashboard at `feedback.miranet.work`)
- Host project with a React-based web UI + at least one Cloudflare Worker with Hono

## Phase 1 — Detect Project Structure

Scan the project automatically. Do NOT ask questions you can answer by reading files.

### 1. Find the API worker

Scan for directories containing `wrangler.jsonc` or `wrangler.toml`. For each, check `package.json` for `hono` in dependencies.

- **One Hono worker found** → auto-select, confirm: "I found `api/` as your Hono API worker. Correct?"
- **Multiple Hono workers** → ask: "I found multiple Hono workers: `api/`, `gateway/`. Which is the primary API?"
- **No Hono workers** → stop: "This project needs a Hono-based API worker to integrate the feedback SDK."

Record: worker directory name, dev port (from `wrangler.jsonc` → `dev.port`).

### 2. Find the web project

Scan for directories with `vite.config.*`, `next.config.*`, or `react`/`react-dom` in `package.json` dependencies.

Detect the React framework by checking the web project's `package.json`:
- `@tanstack/react-router` → **TanStack Router** (root layout: `src/routes/__root.tsx`)
- `react-router` or `react-router-dom` → **React Router** (root: `src/App.tsx` or root route component)
- `next` → **Next.js** (root layout: `app/layout.tsx` or `pages/_app.tsx`)
- None → **Plain React** (root: `src/App.tsx` or `src/main.tsx`)

Record: web directory name, framework, root layout path, dev port (from `vite.config.*` → `server.port`).

### 3. Detect auth system

Check the API worker's `package.json` dependencies:
- `@clerk/backend` or `@clerk/clerk-sdk-node` → **Clerk**
- `better-auth` → **Better Auth**
- Neither → **Unknown** (will generate placeholder)

If auth detected, scan the worker's source files to find:
- Existing auth middleware import paths
- How the auth instance is created (e.g., `createAuth(c.env)` for Better Auth, `getAuth(c)` for Clerk)
- The user object shape available after auth

Record: auth system name, import path, user extraction pattern.

### 4. Summarize and confirm

Present findings to the user:

> "Here's what I found:
> - **Web:** `dashboard/` (TanStack Router, port 25885)
> - **API:** `api/` (Hono, port 56545)
> - **Auth:** Clerk
>
> Correct?"

Wait for confirmation before proceeding.

## Phase 2 — Gather Config

Ask these one at a time:

### 1. Feedback API key

> "What's your Feedback API key? (Generate one at feedback.miranet.work → Apps → your app → Generate Key)"

### 2. Feedback API URL

> "Feedback API URL? (default: `https://feedback.miranet.work/v1/sdk`)"

Accept default or custom URL.

### 3. Component variant

> "Which feedback component variant should I add?
> - **a) FeedbackFloating** (recommended) — floating button, bottom-right corner, visible on all pages
> - **b) FeedbackButton** — inline button you place anywhere
> - **c) FeedbackContextual** — triggered programmatically (for error handling)
> - **d) Multiple** — specify which ones
>
> All variants support `prefill` for pre-populating the form."

Default: FeedbackFloating.

### 4. Confirm

> "Ready to integrate:
> - Install `@groo.dev/feedback-sdk` in both `{web}` and `{api}`
> - Add proxy at `/api/@groo.dev/feedback` in `{api}`
> - Add `{variant}` to root layout in `{web}`
> - Add Vite dev proxy
>
> Proceed?"

## Phase 3 — Install & Wire

Execute these steps. Read each file before modifying it.

### 1. Install SDK

```bash
cd {web-dir} && npm install @groo.dev/feedback-sdk
cd {api-dir} && npm install @groo.dev/feedback-sdk
```

### 2. API worker — add secret

Add to `{api-dir}/.dev.vars`:
```
FEEDBACK_API_KEY={user's api key}
```

Add to `{api-dir}/.dev.vars.example`:
```
FEEDBACK_API_KEY=your-feedback-api-key
```

Run type generation:
```bash
cd {api-dir} && npm run cf-typegen
```

### 3. API worker — add auth middleware + proxy route

Create or modify the appropriate file to add the feedback proxy. The auth middleware must run BEFORE the proxy to set `X-Feedback-User`.

**For Clerk:**
```typescript
import { feedbackProxy } from '@groo.dev/feedback-sdk/hono'

// Auth middleware — extract user, set X-Feedback-User header
app.use('/api/@groo.dev/feedback/*', async (c, next) => {
  const auth = getAuth(c); // use the host's existing Clerk pattern
  if (!auth.userId) return c.json({ error: 'Unauthorized' }, 401);
  const user = await clerkClient(c.env).users.getUser(auth.userId);
  c.req.raw.headers.set('X-Feedback-User', JSON.stringify({
    id: auth.userId,
    name: [user.firstName, user.lastName].filter(Boolean).join(' '),
    email: user.emailAddresses[0]?.emailAddress ?? '',
  }));
  await next();
});

// Proxy
app.route('/api/@groo.dev/feedback', feedbackProxy({
  apiKey: c.env.FEEDBACK_API_KEY,
  apiBase: '{feedback-api-url}',
}));
```

**For Better Auth:**
```typescript
import { feedbackProxy } from '@groo.dev/feedback-sdk/hono'
import { createAuth } from '{host-auth-import}' // detected from existing code

app.use('/api/@groo.dev/feedback/*', async (c, next) => {
  const auth = createAuth(c.env);
  const session = await auth.api.getSession({ headers: c.req.raw.headers });
  if (!session) return c.json({ error: 'Unauthorized' }, 401);
  c.req.raw.headers.set('X-Feedback-User', JSON.stringify({
    id: session.user.id,
    name: session.user.name,
    email: session.user.email,
  }));
  await next();
});

app.route('/api/@groo.dev/feedback', feedbackProxy({
  apiKey: c.env.FEEDBACK_API_KEY,
  apiBase: '{feedback-api-url}',
}));
```

**For Unknown auth:**
```typescript
import { feedbackProxy } from '@groo.dev/feedback-sdk/hono'

// TODO: Replace with your auth system's user extraction
app.use('/api/@groo.dev/feedback/*', async (c, next) => {
  // const user = await getAuthenticatedUser(c);
  c.req.raw.headers.set('X-Feedback-User', JSON.stringify({
    id: 'replace-with-user-id',
    name: 'Replace with user name',
    email: 'replace@example.com',
  }));
  await next();
});

app.route('/api/@groo.dev/feedback', feedbackProxy({
  apiKey: c.env.FEEDBACK_API_KEY,
  apiBase: '{feedback-api-url}',
}));
```

IMPORTANT: The `feedbackProxy` needs `env.FEEDBACK_API_KEY` at runtime. Since Hono route mounting happens at module level, you need to handle this. Two patterns:

**Pattern A — if the host app uses `app.route()` at module level:**
Mount the proxy inside a route handler that has access to `c.env`:
```typescript
const feedbackRoutes = new Hono<{ Bindings: Env }>();
feedbackRoutes.use('*', authMiddleware); // from above
feedbackRoutes.route('/', feedbackProxy({ apiKey: '', apiBase: '' })); // won't work — needs env

// Instead, forward manually or use a factory:
feedbackRoutes.all('/*', async (c) => {
  const proxy = feedbackProxy({
    apiKey: c.env.FEEDBACK_API_KEY,
    apiBase: '{feedback-api-url}',
  });
  return proxy.fetch(c.req.raw, c.env, c.executionCtx);
});
app.route('/api/@groo.dev/feedback', feedbackRoutes);
```

**Pattern B — if the host app creates the Hono app per-request (less common):**
Mount normally since env is available at creation time.

Read the host's `src/index.ts` to determine which pattern to use.

### 4. Web project — add component

Find the root layout file (detected in Phase 1). Add the import and component.

**FeedbackFloating (most common):**
```tsx
import { FeedbackFloating } from '@groo.dev/feedback-sdk/react'

// Add inside the root layout's return, at the end (after other content):
<FeedbackFloating apiBase="/api/@groo.dev/feedback" />
```

**FeedbackButton:**
```tsx
import { FeedbackButton } from '@groo.dev/feedback-sdk/react'

// Tell the user to place this wherever they want:
<FeedbackButton apiBase="/api/@groo.dev/feedback">
  Send Feedback
</FeedbackButton>
```

**FeedbackContextual:**
```tsx
import { FeedbackContextual } from '@groo.dev/feedback-sdk/react'

// Add to root layout with state:
const [feedbackOpen, setFeedbackOpen] = useState(false);

<FeedbackContextual
  apiBase="/api/@groo.dev/feedback"
  open={feedbackOpen}
  onClose={() => setFeedbackOpen(false)}
/>
```

### 5. Vite proxy (dev only)

Read `{web-dir}/vite.config.ts`. Add to `server.proxy`:

```typescript
'/api/@groo.dev/feedback': 'http://localhost:{worker-port}',
```

If `server.proxy` doesn't exist, create it.

### 6. Verify

```bash
cd {api-dir} && npx tsc --noEmit
cd {web-dir} && npx tsc --noEmit
```

Report success:

> "Feedback SDK integrated:
> - API proxy at `/api/@groo.dev/feedback` in `{api}/`
> - `{Variant}` added to root layout in `{web}/`
> - Vite dev proxy configured
>
> Restart `groo dev` to test. The feedback button should appear in your app."

## Troubleshooting

- **401 from proxy:** The auth middleware isn't setting `X-Feedback-User`. Check the middleware runs before the proxy route.
- **CORS errors:** The Vite proxy should handle this in dev. In production, ensure the worker serves both the UI and API on the same domain.
- **"Invalid API key":** Verify `FEEDBACK_API_KEY` is set in `.dev.vars` and the worker was restarted.
- **Widget doesn't open:** Check browser console for import errors. Verify `@groo.dev/feedback-sdk` is installed in the web project.
