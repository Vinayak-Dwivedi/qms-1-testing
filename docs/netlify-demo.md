# Netlify Demo Deployment — iLeads QMS (UI only)

Deploys the **web UI with pre-seeded demo data**. The background worker
(transcription/audit pipeline) is **not** part of this deploy — Netlify is
serverless and can't run a persistent worker. The UI renders the demo dataset
from `prisma/seed.ts` (a "Beetel" workspace: 3 campaigns, 5 agents, the
24-parameter rubric, and 6 fully-audited demo calls).

What the repo already provides: `netlify.toml` (build + Next runtime plugin),
Prisma `binaryTargets` for the Netlify Lambda runtime, and an idempotent seed.

---

## Step 1 — Create a free serverless Postgres (Neon)

The app needs a Postgres it can reach from the cloud (local/SQLite won't work on
serverless).

1. Sign up at <https://neon.tech> (free tier is enough).
2. Create a project → copy the **pooled** connection string. It looks like:
   `postgresql://USER:PASSWORD@ep-xxx-pooler.region.aws.neon.tech/neondb?sslmode=require`

> Alternative: Netlify's built-in **Netlify DB** (also Neon). If you use it,
> it exposes `NETLIFY_DATABASE_URL` — copy that value into a `DATABASE_URL`
> env var so the app and Prisma pick it up.

## Step 2 — Push this branch to GitHub

The config files (`netlify.toml`, schema change) must be on the branch Netlify
builds. From the repo:

```powershell
git add netlify.toml .env.netlify.example prisma/schema.prisma docs/netlify-demo.md
git commit -m "chore: Netlify demo deploy config"
git push origin vinayak
```

## Step 3 — Create the Netlify site

1. <https://app.netlify.com> → **Add new site → Import an existing project**.
2. Connect GitHub, pick this repo, and choose the `vinayak` branch.
3. Netlify auto-detects Next.js and reads `netlify.toml` — leave build settings
   as detected.

## Step 4 — Set environment variables

Site settings → **Environment variables** → add (see `.env.netlify.example`):

| Key | Value |
|---|---|
| `DATABASE_URL` | the Neon pooled string from Step 1 |
| `APP_SECRET` | 64-byte random — `node -e "console.log(require('crypto').randomBytes(64).toString('base64url'))"` |
| `APP_PASSWORD` | demo login password, e.g. `demo-password` |
| `APP_BASE_URL` | your site URL, e.g. `https://your-site.netlify.app` |
| `AUDIO_STORAGE_PROVIDER` | `local` |
| `MOCK_STT` | `true` |
| `SHOW_MOCK_ACTIONS` | `true` |

> `DATABASE_URL` and `APP_SECRET` must exist at **build** time too (the app
> validates env on import). Netlify exposes site env vars to the build by
> default, so just adding them is enough.

## Step 5 — Deploy

Trigger a deploy (it runs automatically on the first import, or **Deploys →
Trigger deploy**). The build command:

```
prisma generate → prisma migrate deploy → seed demo data → next build
```

When it's green, open the site, log in with `APP_PASSWORD`, and you'll see the
seeded dashboard, calls, transcripts, and audit scores.

---

## CLI alternative (instead of Steps 3–5)

```powershell
npm i -g netlify-cli
netlify login                 # opens a browser
netlify init                  # link/create the site
netlify env:import .env.netlify   # after filling real values into a .env.netlify
netlify deploy --build --prod
```

---

## Known risks / notes

- **Next.js 16 is very new.** Netlify's Next runtime (`@netlify/plugin-nextjs`
  v5) may lag the newest Next release. If the build fails inside the plugin,
  options are: pin to the latest plugin, or host the UI on a platform with
  first-class Next 16 support (e.g. Vercel) — same env vars apply.
- **No uploads/processing in the demo.** Upload + transcription + audit need the
  worker, which isn't deployed here. The seeded calls already show the full UI.
- **Re-seeding on every deploy is intentional** and safe — the seed is
  idempotent (upserts), so the demo always returns to a known-good state.
