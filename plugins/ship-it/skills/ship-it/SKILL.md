---
name: ship-it
description: Deploy a web app or API to a cloud platform end to end. Inspects the repo, recommends a provider, asks only what it can't infer, writes infra-as-code, plans, applies after approval, and verifies. Ships with a Masonite profile and a Railway adapter.
---

# Ship It

Take an app that runs locally and get it running in production, reproducibly, without surprises on the bill or in the data.

The provider tools (CLIs, MCP servers) already handle provisioning. What this skill adds is judgment. It works out what the app actually needs to run, picks a platform for stated reasons, writes the setup down as code in the repo, and refuses to touch anything destructive without a human yes.

## Ground rules (read first, apply always)

1. **Plan before apply.** Never create, change or delete cloud resources until the user has seen the full plan: services, databases, env var *names*, domains, and an honest cost note. In unattended runs, prepare everything and stop at the plan.
2. **Destructive actions need an explicit yes in this conversation.** This covers deleting services, volumes, databases or variables, detaching volumes, and anything a provider marks destructive. A flag like `--yes` never substitutes for the user's answer. Never pass `--confirm-destructive` (or an equivalent) unless the user approved that specific change.
3. **Secrets never enter the chat, the repo, or logs.**
   - Generate secrets on the machine and set them straight into the provider's variable store.
   - Commit `.env.example` with names only. Never commit `.env`.
   - In plan output, show variable names, not values.
4. **Infra lives in the repo.** Every setting you apply should exist as a file in the project, so the deployment is reproducible and reviewable. Avoid one-off dashboard clicks and ad-hoc CLI mutations that aren't written down.
5. **Least privilege.** Prefer OAuth scoped to one project, or a project/environment token, over account-wide credentials. If you only have broad credentials, say so in the plan.
6. **Verify, don't assume.** Provider CLIs change. Before using a subcommand or flag you're not sure of, run `<cli> <command> --help` or check the provider's docs. Treat the commands in this file as a starting point, not gospel.
7. **Staging first** when the user wants it, or when the app has real users or data.

## Workflow

### Phase 0 — Preflight
- Confirm the repo is a git repo with a clean or committed tree, and note the remote (GitHub `owner/repo`) and default branch.
- Check which provider tooling is available: provider MCP tools in the session, CLI on PATH (`railway --version`, `flyctl version`, ...), and logged-in status (`railway whoami`).
- If nothing is installed or logged in, tell the user the exact one-line setup and wait. Don't attempt to log in on their behalf.

### Phase 1 — Inspect the app
Read the code. Don't guess. Produce a **process map**:

| Question | How to find out |
|---|---|
| Framework + version | dependency files (`requirements.txt`, `pyproject.toml`, `package.json`, ...) |
| Runtime version | `.python-version`, `pyproject` `requires-python`, `engines`, Dockerfile |
| HTTP entry point | framework profile below |
| Background jobs? | queue/job classes, queue config, `dispatch`/`push` calls |
| Scheduled tasks? | scheduler registrations, task classes |
| Database engine | DB config + driver packages (`psycopg2`, `mysqlclient`, ...) |
| Cache / broker | cache/queue config, `redis`, `pika`, ... |
| File uploads / local disk writes | storage config, `storage/` writes → needs a volume or bucket |
| Asset build | `package.json` build scripts, compiled assets dir |
| Health endpoint | existing route like `/health`; if none, plan to add one |
| Env vars read | grep the config dir for `env(` / `os.getenv` / `os.environ` → full list of names |

Output the map to the user in a short table: **processes** (web / worker / scheduler / ...), **backing services** (Postgres, Redis, bucket, volume), and **env vars** (names, and which ones are secrets).

### Phase 2 — Recommend a platform
Recommend one platform, with one runner-up, and give reasons tied to the process map. Heuristics:

- **Railway**: several processes plus managed Postgres/Redis in one project, infra-as-code with plan/apply, strong agent tooling. A good default for "web + worker + scheduler + DB".
- **Fly.io**: multi-region or latency-sensitive apps, Docker-first, comfortable with more knobs.
- **Render**: Heroku-like simplicity, Blueprint YAML, background workers and cron as first-class.
- **VPS + Kamal or Coolify**: lowest steady-state cost and full control, but the user owns OS patching and backups.
- **Vercel / Netlify**: frontends and serverless functions. Wrong fit for long-running Python workers.

For cost, never invent numbers. Fetch the provider's current pricing page and give a rough range with the assumptions stated (e.g. "3 small services + Postgres, low traffic"). If you can't fetch it, say the cost is usage-based and point to the pricing page.

### Phase 3 — Ask only what you can't infer
At most 4 questions, multiple choice where possible. Typical:
1. Platform: accept the recommendation or pick another.
2. Environments: production only, or staging + production.
3. Domain: the provider's generated domain for now, or a custom domain (which one).
4. Region: closest to the users (suggest one from what you know).

Don't ask about things the code already answers.

### Phase 4 — Prepare the repo (local changes only)
Make the app production-ready. Show a diff summary and commit only with the user's OK.
- Add the production server dependency (e.g. `gunicorn`).
- Pin the runtime version (e.g. `.python-version`).
- Add a health endpoint if missing (returns 200 + `{"status":"ok"}`, and touches nothing expensive).
- Write `.env.example` with every env var name and a comment, no values.
- Make sure `.env` is in `.gitignore`.
- Write the provider's infra-as-code file (see the adapter).
- Optional: add a CI workflow that plans infra on PRs and applies on merge, if the provider supports it.

### Phase 5 — Plan
Run the provider's dry-run/plan. Present:
- Resources to **create / change / destroy**, with every destroy line highlighted.
- Variable **names** being set, and which are secrets.
- Start command and pre-deploy command per service.
- The cost note from Phase 2.

Then ask: "Apply this plan?" Stop and wait.

### Phase 6 — Apply
- Set secrets first, directly from a local generator into the provider's variable store. Don't echo them.
- Apply the reviewed plan. If the provider says the plan is stale, re-plan and re-confirm. Never force.
- Trigger the first deploy if apply doesn't do it.

### Phase 7 — Verify
- Wait for the deploy to finish. Tail build and deploy logs for each service.
- `curl` the health endpoint on the public URL and expect a 200.
- Confirm migrations ran (pre-deploy logs).
- Confirm the worker started and is polling, and the scheduler loop is ticking (logs).
- If anything fails, go to the failure playbook. Fix forward in code, re-plan, re-apply. Don't patch live settings by hand.

### Phase 8 — Report
Give a short summary covering:
- the URL
- the services and what each runs
- where the infra file lives
- how to deploy next time (push to branch, or the provider's deploy command)
- how to roll back (provider's redeploy-previous or `git revert` + push)
- the cost note
- open follow-ups (custom domain DNS records, backups, alerts)

---

## Framework profile: Masonite (Python, 5.x)

Source: docs.masonite.dev → Getting Started → Deployment, Queues, Task Scheduling.

- **Runtime:** Python 3.10–3.13. Pin it.
- **HTTP entry:** `wsgi.py` at the project root exposes the WSGI callable `application`.
  - Start command: `gunicorn --workers 2 wsgi:application`. Gunicorn binds to `0.0.0.0:$PORT` automatically when `PORT` is set, which PaaS platforms do.
  - Size workers roughly `(2 × cores) + 1`. Start small on PaaS.
  - **Never** use `python craft serve` in production. It's the dev server.
- **Required env:**
  - `APP_ENV=production` and `APP_DEBUG=False`. Debug mode leaks env vars and source code on error pages.
  - `APP_URL=https://<domain>`.
  - `APP_KEY`: generate fresh for production, never reuse the dev key. Use `python craft key --dont-store`, which prints the key without writing `.env`, and pipe it straight into the provider's secret store.
- **Database:** read `config/database.py` to see exactly which env names it reads (commonly `DB_CONNECTION`, `DB_HOST`, `DB_PORT`, `DB_USERNAME`, `DB_PASSWORD`, `DB_DATABASE`, or a single URL). Map the provider's DB variables onto *those* names. Postgres needs a driver package (`psycopg2-binary`) in requirements.
- **Migrations:** `python craft migrate`. Run it as the web service's pre-deploy step, so a failed migration blocks the deploy. If it prompts in production, check `python craft migrate --help` for a force/non-interactive flag.
- **Queues:** drivers are `async`, `database` and `amqp`.
  - `async` runs jobs in the web process. Jobs are lost on restart, so don't use it in production for anything that matters.
  - `database` is the recommended default. It needs the jobs tables: `python craft queue:table` and `python craft queue:failed`, then migrate. Run it as a separate long-lived service: `python craft queue:work --driver database`.
  - `amqp` needs RabbitMQ. Only use it if the app already does.
  - The `database` driver needs no Redis. Only add Redis if the cache/session config uses it.
- **Scheduler:** `python craft schedule:run` must run **every minute**, and the app decides which tasks are due. On a PaaS, the simplest reliable form is a tiny long-lived service:
  `sh -c 'while true; do python craft schedule:run; sleep 60; done'`
  Only use platform cron instead if it supports a 1-minute interval and you've checked that.
- **Static files:** served by the built-in WhiteNoise provider. No nginx needed on a PaaS.
- **Assets:** if `package.json` has a production build (`npm run prod`), add it to the build step. This may require a Dockerfile or a Node-capable builder.
- **Health route:** if missing, add one, e.g. in `routes/web.py`: `Route.get("/health", "HealthController@show")` with a controller returning `{"status": "ok"}`.
- **Maintenance mode** (optional, for risky migrations): `python craft down` / `python craft up`.

Process map for a typical Masonite API: **web** (gunicorn) + **worker** (queue:work, if jobs exist) + **scheduler** (loop, if tasks exist) + **Postgres**. All three app services use the same repo and the same env, and differ only in start command.

---

## Provider adapter: Railway

Source: docs.railway.com → AI/MCP server, Infrastructure as Code.

**Tooling**
- One-time setup: `railway setup agent`. This installs agent skills and MCP, and reuses `railway login`.
- MCP tools (remote): `whoami`, `list-projects`, `create-project`, `list-services`, `redeploy`, `accept-deploy` (destructive), and `railway-agent` (multi-step debugging and log analysis).
- Use the CLI for infra-as-code, and MCP or `railway-agent` for inspection and debugging.

**Infra as code (use this, not `railway.json`)**
- `railway.json` / `railway.toml` (Config as Code) is **deprecated**, and Railway stops reading those files on 2026-12-01. Use **Infrastructure as Code**: a single `.railway/railway.ts` (TypeScript, GA) describing the whole project.
- Install it in the repo with `npm install railway` (dev dependency is fine). This needs Node even for a Python app.
- Commands:
  - `railway link`: choose the project/environment.
  - `railway config init`: scaffold the file.
  - `railway config pull`: import an existing project.
  - `railway config plan`: read-only diff. Values are redacted by default. **Show this to the user.**
  - `railway config apply`: interactive confirm.
- Non-interactive apply requires `--yes`, and destructive changes additionally require `--confirm-destructive`. Pass that second flag only after explicit user approval of the destroy lines.
- One file per repo/project. Omitting a resource from the file means deleting it on the next apply, so warn the user about this.
- Secrets: in the file, declare them as `preserve()` (keep the value already in Railway) or reference a shared environment variable via `ctx.shared.NAME`. Set the actual value with the CLI's variable command, so the value never lands in the repo. Check `railway variable --help` for the current syntax.
- CI: the `railwayapp/config` GitHub Action plans on PRs and applies the reviewed plan on merge. It needs an environment-scoped project token as the `RAILWAY_TOKEN` secret.

**Template for a Masonite app** (adapt the env names to what `config/` actually reads):

```ts
// .railway/railway.ts
import { defineRailway, github, postgres, project, service } from "railway/iac";

export default defineRailway((ctx) => {
  const prod = ctx.environment === "production";
  const db = postgres("postgres");
  const source = github("OWNER/REPO", { branch: "main" });

  // Same env for every process; secrets come from shared vars, not this file.
  const env = {
    APP_ENV: prod ? "production" : "staging",
    APP_DEBUG: "False",
    APP_KEY: ctx.shared.APP_KEY,
    // Map to whatever config/database.py reads:
    DB_CONNECTION: "postgres",
    DB_HOST: db.env.PGHOST,
    DB_PORT: db.env.PGPORT,
    DB_USERNAME: db.env.PGUSER,
    DB_PASSWORD: db.env.PGPASSWORD,
    DB_DATABASE: db.env.PGDATABASE,
  };

  const web = service("web", {
    source,
    start: "gunicorn --workers 2 wsgi:application",
    preDeploy: "python craft migrate",
    healthcheck: "/health",
    healthcheckTimeout: 60,
    env: { ...env, APP_URL: "https://${{RAILWAY_PUBLIC_DOMAIN}}" },
    // domains: prod ? ["api.example.com"] : [],
  });

  // Include only if the app dispatches jobs:
  const worker = service("worker", {
    source,
    start: "python craft queue:work --driver database",
    env,
  });

  // Include only if the app registers scheduled tasks:
  const scheduler = service("scheduler", {
    source,
    start: "sh -c 'while true; do python craft schedule:run; sleep 60; done'",
    env,
  });

  return project("APP_NAME", { resources: [db, web, worker, scheduler] });
});
```

**Notes**
- Railway's default builder (Railpack) detects Python from requirements and `.python-version`. Switch to a Dockerfile only if you need system packages or a Node asset build.
- After the first apply, generate a public domain for `web` if one wasn't created: `railway domain` (check `--help`).
- The worker and scheduler don't need public domains.
- Before applying, confirm the template's variable reference syntax and helper names against `railway config plan` output. If plan errors on a field, check the IaC reference page rather than guessing.

---

## Failure playbook

| Symptom | Likely cause | Fix |
|---|---|---|
| Healthcheck fails, logs show "connection refused" | Server bound to 127.0.0.1 or wrong port | Bind `0.0.0.0:$PORT` (gunicorn does this when `PORT` is set). Remove any hardcoded `--bind`. |
| `ModuleNotFoundError: gunicorn` | Not in requirements | Add it, commit, redeploy |
| Pre-deploy fails on migrate | DB env names don't match config, or driver missing | Re-read `config/database.py`, fix the mapping, add `psycopg2-binary` |
| 500 errors, debug page visible | `APP_DEBUG` not False | Fix the env, redeploy, and treat any exposed secrets as leaked (rotate them) |
| Jobs never run | No worker service, or the web app uses the `async` driver | Add the worker and set the default queue driver to `database` |
| Scheduled tasks never fire | No scheduler process | Add the scheduler loop service |
| Plan shows unexpected destroys | A resource was omitted from the file, or the wrong env is linked | Stop. Run `railway status`, fix the file, re-plan. |
| Apply rejected as stale | Someone changed the env since the plan | Re-plan and re-confirm. Never force. |

For anything unclear, hand the logs to `railway-agent` (MCP) with a specific question, e.g. "why is service web failing its healthcheck?"

---

## Extending this skill

- **New framework:** add a profile section with entry point + start command, required env, migrations, background processes, scheduler, static/assets, and health route. Keep it factual and link to the framework's deploy docs.
- **New provider:** add an adapter section with tooling setup, an infra-as-code format with plan/apply commands, how secrets are set, how destructive changes are gated, a template for the reference app, and provider-specific failure rows.
- Framework knowledge and provider knowledge stay separate, so any profile can combine with any adapter.
