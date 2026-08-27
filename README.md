# Skillbridge

SkillBridge is a verified talent pipeline for early-career professionals across Africa. Candidates are evaluated through structured assessments, practical tasks, and Interviews, then assigned a standardized score, and made discoverable to employers only when they are job-ready.

## Stack

- **Next.js 16** App Router (`proxy.ts`, `forbidden.tsx`, `unauthorized.tsx`)
- **React 19**, **TypeScript** (strict)
- **Tailwind v4** with shadcn `radix-maia` style
- **`@t3-oss/env-nextjs`** + **Zod 4** for build-time env validation

## Getting started

```bash
pnpm install
cp .env.example .env.local   # fill in values
pnpm dev
```

Open <http://localhost:3000>.

## Scripts

| Command          | What it does                     |
| ---------------- | -------------------------------- |
| `pnpm dev`       | Dev server                       |
| `pnpm build`     | Production build (validates env) |
| `pnpm start`     | Run the production build         |
| `pnpm lint`      | ESLint                           |
| `pnpm typecheck` | `tsc --noEmit`                   |

## CI/CD

GitHub Actions runs linting, TypeScript checks, a production dependency audit, a Trivy filesystem scan, and a production build for pull requests targeting `dev`, `staging`, or `main`.

Pushes to those branches run the same checks and deploy the built Next.js app to the matching VPS environment through SSH and PM2. The deploy job also checks `/api/health` after restarting the app.

### GitHub Environment setup

Create GitHub Environments named `dev`, `staging`, and `main`. Add these Variables to each environment:

| Variable                       | Purpose                        |
| ------------------------------ | ------------------------------ |
| `NEXT_PUBLIC_APP_URL`          | Public URL for the environment |
| `NEXT_PUBLIC_APP_NAME`         | Application name               |
| `NEXT_PUBLIC_API_URL`          | Backend API URL                |
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | Google OAuth client ID         |

Add the matching dotenv Secret (`DEV_ENV_FILE` in `dev`, `STAGING_ENV_FILE` in
`staging`, or `PRODUCTION_ENV_FILE` in `main`) and the following Secrets to
each environment:

| Secret                                                      | Purpose                                             |
| ----------------------------------------------------------- | --------------------------------------------------- |
| `DEV_ENV_FILE` / `STAGING_ENV_FILE` / `PRODUCTION_ENV_FILE` | Complete dotenv content for the matching deployment |
| `AUTH_SECRET`                                               | NextAuth secret                                     |
| `AUTH_GOOGLE_ID`                                            | Google OAuth client ID, if used server-side         |
| `AUTH_GOOGLE_SECRET`                                        | Google OAuth client secret, if used                 |
| `AUTH_URL`                                                  | Auth callback URL, if required by the deployment    |
| `SSH_KEY`                                                   | Private key for the deployment user                 |
| `SSH_HOST`                                                  | VPS hostname or IP address                          |
| `SSH_USER`                                                  | VPS deployment username                             |
| `SSH_KNOWN_HOSTS`                                           | Pinned host-key output for `SSH_HOST`               |

`SSH_KNOWN_HOSTS` should contain the output of `ssh-keyscan -H <host>` captured from a trusted machine. The deployment user’s server must have Node.js 20, pnpm 9, PM2, rsync, and curl installed. Deployments use these remote paths and PM2 processes:

| Branch    | Remote path                         | PM2 process              | Port   |
| --------- | ----------------------------------- | ------------------------ | ------ |
| `dev`     | `~/skillbridge/frontend/dev`        | `skillbridge-ui-dev`     | `3000` |
| `staging` | `~/skillbridge/frontend/staging`    | `skillbridge-ui-staging` | `4000` |
| `main`    | `~/skillbridge/frontend/production` | `skillbridge-ui-prod`    | `5000` |

## Environment variables

Schemas live in [`src/env/`](./src/env), split by side:

- [`src/env/server.ts`](./src/env/server.ts) — server-only vars. t3-env throws at runtime if a client component reads it.
- [`src/env/client.ts`](./src/env/client.ts) — `NEXT_PUBLIC_*` vars, safe everywhere.

Both are imported in [`next.config.ts`](./next.config.ts) so the build fails on any malformed value. Set `SKIP_ENV_VALIDATION=1` to bypass (Docker, lint-only CI).

| Var                    | Side   | Required | Notes                                 |
| ---------------------- | ------ | -------- | ------------------------------------- |
| `NODE_ENV`             | server | auto     | `development` / `test` / `production` |
| `API_BASE_URL`         | server | optional | Upstream API for server-side `fetch`  |
| `API_SECRET`           | server | optional | Bearer token forwarded server-side    |
| `NEXT_PUBLIC_APP_URL`  | client | optional | Defaults to `http://localhost:3000`   |
| `NEXT_PUBLIC_APP_NAME` | client | optional | Defaults to `Next Starter`            |

Use it like:

```ts
// Server code (route handlers, Server Components, Server Actions)
import { env } from "@/env/server";

await fetch(`${env.API_BASE_URL}/users`, {
  headers: { Authorization: `Bearer ${env.API_SECRET}` },
});

// Client code or shared metadata
import { env } from "@/env/client";

console.log(env.NEXT_PUBLIC_APP_URL);
```

## Proxy (`src/proxy.ts`)

Replaces the legacy `middleware.ts` (Next.js 16 renamed it). It runs before the cache and:

- Generates an `x-request-id` and forwards it to the request headers + response
- Sets baseline security headers (`X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`)
- Skips static assets via the matcher

Add auth gating, rewrites, or redirects there as needed. Note: `runtime` config is **not** allowed in `proxy.ts` — it always runs on Node.js.

## Route conventions wired up

| File                          | Purpose                                  |
| ----------------------------- | ---------------------------------------- |
| `src/app/loading.tsx`         | Root suspense fallback                   |
| `src/app/error.tsx`           | Client error boundary (`unstable_retry`) |
| `src/app/not-found.tsx`       | 404 page                                 |
| `src/app/forbidden.tsx`       | 403 page (calls `forbidden()`)           |
| `src/app/unauthorized.tsx`    | 401 page (calls `unauthorized()`)        |
| `src/app/robots.ts`           | `/robots.txt`                            |
| `src/app/sitemap.ts`          | `/sitemap.xml`                           |
| `src/app/api/health/route.ts` | Liveness probe at `GET /api/health`      |

`forbidden.tsx` and `unauthorized.tsx` require `experimental.authInterrupts: true`, already enabled in [`next.config.ts`](./next.config.ts).

## Project layout

```
src/
├── app/                # App Router routes & file conventions
│   └── api/health/     # Liveness probe
├── components/ui/      # shadcn components (added via `pnpm dlx shadcn@latest add ...`)
├── lib/utils.ts        # cn() helper
├── env/
│   ├── server.ts       # Server-only env schema
│   └── client.ts       # NEXT_PUBLIC_* env schema
└── proxy.ts            # Next.js 16 proxy (formerly middleware)
```
