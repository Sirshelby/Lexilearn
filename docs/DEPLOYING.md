# Deploying LexiLearn

There are two ways to run LexiLearn in production: on Vercel (what I use), or anywhere that runs Docker. The code is the same for both; it detects Vercel and adjusts itself.

## Vercel

### How it's laid out

The repo becomes two Vercel projects:

```
browser ──> lexilearn            frontend project: static files on Vercel's CDN
               │  /api/* is proxied (a Vercel rewrite) to
               v
            lexilearn-api        backend project: FastAPI as one Vercel Function
               ├──> Neon Postgres       (Vercel Marketplace)
               └──> Upstash Redis       (Vercel Marketplace, optional)
```

I split it this way because Vercel deploys each half with zero configuration when the project's root directory points at the right folder: `frontend` is detected as a Vite app, and `backend` as a FastAPI app (it finds `app/main.py`). The rewrite means the browser only ever talks to the frontend's domain. That keeps the login cookie first-party and means the API doesn't need CORS.

### What changes when it runs on Vercel

The backend was written to run as a long-lived server, and a Vercel Function is different: instances start and stop on demand, can run many copies at once, and have no permanent disk. The code handles this:

| Concern | How it's handled |
| --- | --- |
| Database connections | When `VERCEL=1` (Vercel sets it), connections aren't kept open between requests. Neon's pooler does the pooling. |
| Connection pooler | Neon's pooled URL goes through PgBouncer. The app detects `-pooler` in the host name and turns off asyncpg's prepared statement cache, which otherwise breaks behind PgBouncer. |
| Database URL format | Neon gives `postgres://...?sslmode=require`. The app rewrites it for its async driver automatically. |
| Migrations | Run during the build by `backend/scripts/vercel_build.py`, before the new version goes live, using the direct (unpooled) connection. |
| Speech cache | Stored in Postgres, not on disk. |
| Rate limits | Shared through Redis if it's connected; otherwise each instance keeps its own counts. |
| Time limits | The function may run for up to 60 seconds (`backend/vercel.json`). The AI client retries once at most, so a slow provider can't run past that. |

### 1. Push the code to GitHub

Push this repository to GitHub (a new repo, or replace the old prototype). The `.gitignore` keeps `.env` out of git.

### 2. Create the API project

1. In Vercel, choose **Add New > Project** and import the repository.
2. Set **Root Directory** to `backend`. Vercel should detect FastAPI.
3. Name the project `lexilearn-api`. The frontend's config expects this name; step 5 covers what to do if it's taken.
4. Deploy.

**The first build will fail, and that's expected.** The build step stops with "DATABASE_URL is not set", because a production deployment without a database is a mistake. The next step adds one.

### 3. Add the database

1. Open the `lexilearn-api` project and go to the **Storage** tab.
2. Create a **Neon** Postgres database. Connect it to this project for all environments.
3. Neon adds `DATABASE_URL` (pooled) and `DATABASE_URL_UNPOOLED` (direct) to the project's environment variables. Both are used.

**Choose the region carefully.** Pick the Neon region closest to your learners. Then set the API's function region to the same place, under **Settings > Functions > Function Region**. Every request opens a new database connection, so any distance between the function and the database is paid on every single request.

### 4. Set the environment variables

In the API project, go to **Settings > Environment Variables** and add:

| Name | Value |
| --- | --- |
| `ENVIRONMENT` | `production` |
| `SECRET_KEY` | A long random string: `python -c "import secrets; print(secrets.token_urlsafe(48))"` |
| `COOKIE_SECURE` | `true` |
| `LLM_API_KEY` | Your new OpenAI key (optional; without it Lexi uses the offline tutor) |
| `LLM_MODEL` | A model your account can use, for example `gpt-4o-mini` |
| `ELEVENLABS_API_KEY` | Your new ElevenLabs key (optional; without it the browser's voices are used) |

**Optional: shared rate limits.** In the **Storage** tab, add **Upstash Redis** and connect it to the project. It sets `KV_REST_API_URL` and `KV_REST_API_TOKEN`, which the app picks up automatically.

Then go to **Deployments** and redeploy the latest one. The build log should show `Applying database migrations...`. Check that `https://lexilearn-api.vercel.app/api/v1/health` returns `{"status": "ok", ...}`. The `/docs` page is switched off in production on purpose.

### 5. Create the frontend project

1. Open `frontend/vercel.json`. Check that the rewrite destination matches the API project's production domain, which is shown on its overview page. If the name `lexilearn-api` was taken, Vercel will have added a suffix. Commit any change.
2. **Add New > Project**, import the same repository, and set **Root Directory** to `frontend`. Vercel should detect Vite.
3. Name it `lexilearn` and deploy. It needs no environment variables.
4. Open the URL and create an account.

If you add a custom domain, add it to the **frontend** project. The API can stay on its `vercel.app` address because learners never see it.

### Day to day

Pushing to `main` redeploys both projects. The API's build applies any new migrations before the new code goes live.

Preview deployments behave a little differently:

- **Frontend previews** still talk to the production API, because the rewrite points at a fixed address.
- **API previews** skip migrations, so an unmerged branch can't change the production database. If you turn on Neon's preview branching (each preview gets its own copy of the database), set `MIGRATE_PREVIEWS=1` for the Preview environment and previews will migrate their own copy.

### Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Build fails with "DATABASE_URL is not set" | The database isn't connected for that environment (step 3). |
| Every request returns 500, and the logs say "SECRET_KEY must be set" | `SECRET_KEY` is missing (step 4). |
| You're signed out every time you refresh the page | You're using the API's address directly instead of the frontend's, or `COOKIE_SECURE` is wrong. |
| Lexi gives simple, offline-style answers | `LLM_API_KEY` is missing or `LLM_MODEL` is wrong. Search the function logs for "LLM request rejected". |
| The first request after a quiet spell is slow | A cold start. This is normal for serverless functions. |

## Docker

```bash
cp .env.example .env    # set SECRET_KEY and POSTGRES_PASSWORD
docker compose up --build
```

This runs Postgres, the API and nginx, and serves the app on port 8080. Migrations run when the API container starts. It works on any VPS or container host.

## How the Vercel setup was tested

Before writing this guide I ran the API in Vercel mode (`VERCEL=1`) against PostgreSQL 16 behind PgBouncer in transaction mode. I configured PgBouncer more strictly than Neon (prepared statements disabled, only 3 server connections) and had 25 simulated learners go through the whole app at once: register, three practice sessions, mastering a skill, the tutor, progress, a story, and a token refresh. All 25 passed, with no errors in the logs.

I also checked the failure case. With a kept-open connection pool and the PgBouncer workaround switched off, all 25 failed with "prepared statement does not exist", which is why the workaround is there. The build script was run the same way: it migrated a fresh database through the direct connection, and it skipped migrations for preview builds.
