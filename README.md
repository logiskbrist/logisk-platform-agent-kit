# logisk-platform-agent-kit

Claude Code plugin. Slash commands for developers of apps deployed on the Logiskbrist platform.

## Install

```bash
claude plugin install github:logiskbrist/logisk-platform-agent-kit
```

## Configure (once per developer)

Add these to your shell profile (`~/.zshrc`, `~/.bashrc`, or equivalent):

```bash
export LOGISK_CUSTOMER_ORG=<your GitHub org, e.g. godtbrod>
export LOGISK_CUSTOMER_DOMAIN=<your public URL suffix, e.g. apps.gb.logiskbrist.no>
```

Reload your shell. The plugin's skills read these variables at runtime.

## What ships

| Skill | Trigger | What it does |
|---|---|---|
| `/new-app` | "start a new app called blog", "spin up a new service" | Creates a new repo in your org from the canonical [logisk-app-template](https://github.com/logiskbrist/logisk-app-template), substitutes placeholders, tags with `logisk-platform`, pushes, watches the first build. |
| `/list-apps` | "what apps do we have?", "list our platform apps" | Enumerates every repo in your org tagged `logisk-platform`, with prod URL, last-updated, and open-PR count. |
| `/onboard-app` | "adopt this existing repo onto the platform" | Adds a Dockerfile + k8s manifests + GitHub Actions to an existing repo so it deploys through the platform without rewriting the app. |

## Under the hood

The plugin's `/new-app` skill uses the canonical [logiskbrist/logisk-app-template](https://github.com/logiskbrist/logisk-app-template) — no per-customer template fork required.

Workflow calls resolve to the canonical [logiskbrist/logisk-platform-workflows](https://github.com/logiskbrist/logisk-platform-workflows) at `@v3`.

## Support

This plugin is private. Customer devs are invited to the `logiskbrist` GitHub org with read access to this repo only.
