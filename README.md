# ship-it

An agent skill that takes an app from "runs on my machine" to "running in production". It inspects your repo, recommends a cloud platform, asks only what it can't work out, writes infrastructure-as-code into your repo, shows you a plan, and applies it only after you approve.

Ships with:
- **Framework profile:** Masonite 5 (Python)
- **Provider adapter:** Railway (Infrastructure as Code, `.railway/railway.ts`)

## ⚠️ Read this first

This skill drives real cloud tooling. It can **create resources that cost money** and, if you approve it, **change or delete** them. It is built to be careful:

- **Plan before apply.** Nothing is created or changed until you've seen the full plan and said yes.
- **Destructive changes need an explicit yes.** Deleting services, volumes, databases or variables is never done on a blanket approval.
- **Secrets stay out of chat, the repo and logs.** They are generated locally and set straight into the provider's secret store.
- **Infra lives in your repo**, so every deployment is reproducible and reviewable.

Like any skill, it runs with your agent's permissions. Review `SKILL.md` before using it, and try it on a non-critical project first.

## Install

### Claude Code (plugin)

```
/plugin marketplace add agustind/shipit
/plugin install ship-it@ship-it
```

### Any Agent Skills–compatible agent

```
npx skills add agustind/shipit
```

### Manual

Copy `plugins/ship-it/skills/ship-it/` into your agent's skills folder, e.g. `~/.claude/skills/ship-it/`.

## Prerequisites (Railway)

- Railway CLI, logged in: `railway login`
- Agent tooling: `railway setup agent`
- Node.js (Railway IaC files are TypeScript, even for Python apps)
- Your app pushed to GitHub

## Usage

In your project, ask your agent:

> ship it

or call it directly with `/ship-it:ship-it` in Claude Code.

The agent then:
1. Maps your app's processes (web / worker / scheduler) and backing services.
2. Recommends a platform, with reasons.
3. Asks up to 4 questions.
4. Prepares the repo: production server, health route, `.env.example`, infra file.
5. Shows the plan and waits for your approval.
6. Applies the plan, verifies the health check and logs, and reports back.

## Contributing

Framework knowledge and provider knowledge are kept separate, so contributions are self-contained:

- **New framework profile:** entry point and start command, required env, migrations, background processes, scheduler, static assets, health route.
- **New provider adapter:** tooling setup, infra-as-code format with plan/apply, how secrets are set, how destructive changes are gated, a template, and failure rows.

Please test against a real deployment and include the docs links you used.

## License

MIT
