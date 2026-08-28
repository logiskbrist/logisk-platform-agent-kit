---
name: onboard-app
description: Adopt an EXISTING repo onto the Logiskbrist platform without rewriting its code. Wraps the app in a Dockerfile + manifests + CI so it deploys on push. Use when the user says "make this repo run on our platform", "adopt this existing app", "deploy this on the cluster", "onboard this to Logisk", "integrate this repo" — anything that means "I already have a working app, give me push-to-deploy". NOT for creating new apps (that's /new-app).
---

> **Before running:** this skill reads two env vars from the shell:
> - `$LOGISK_CUSTOMER_ORG` — the customer's GitHub organization (e.g. `godtbrod`)
> - `$LOGISK_CUSTOMER_DOMAIN` — the customer's public URL suffix (e.g. `apps.gb.logiskbrist.no`)
>
> If either is unset, ask the user for the value(s) and remind them to add
> `export LOGISK_CUSTOMER_ORG=…` and `export LOGISK_CUSTOMER_DOMAIN=…` to their
> shell profile (`~/.zshrc` / `~/.bashrc`) so future sessions have them.

# Onboard an existing app to the Logiskbrist platform

## What this skill does — and doesn't

**Does**: adds the platform's Docker + Kubernetes + CI glue so the app deploys on every push to `main` and gets a preview URL per PR. Detects the app's stack (Node package manager, port, health endpoint, build/start commands) and wraps them.

**Doesn't**: rewrite the app's code. Doesn't switch from Express to Next.js, doesn't switch from Drizzle to Prisma, doesn't move from npm to pnpm. The app's business logic is untouched. If the app is missing a `/health` endpoint entirely, that ONE two-line route is the only code change.

## Platform context

- **GitHub org**: `$LOGISK_CUSTOMER_ORG`
- **Domain**: `$LOGISK_CUSTOMER_DOMAIN`
- **Workflows source**: `logiskbrist/logisk-platform-workflows`

The platform's actual contract is small: your app ships as a Docker image, listens on HTTP on a known port, and answers 200 on a health endpoint. That's it. Prisma / Better Auth / Next.js are how `/new-app` builds greenfield apps; this skill deals with what's already there.

## Prerequisites (halt if any fails)

- Inside a git repo (`git rev-parse --show-toplevel` succeeds).
- Repo lives at `$LOGISK_CUSTOMER_ORG/<something>` on GitHub, `<something>` matching `^[a-z][a-z0-9-]+$` (lowercase kebab; existing repos with hyphens are fine, but uppercase or underscores will break k8s naming).
- Has a `package.json` (Node app) — for other runtimes, don't run this skill yet; ask the developer.
- Repo does NOT already have the `logisk-platform` topic (already onboarded — stop and say so).

If the repo name doesn't match `^[a-z][a-z0-9-]+$`, tell the developer and stop. Renaming a repo is their call.

## Steps — do these in order

### 1. Take inventory

Read the repo. Record:

| Item | How to detect | Fallback |
|---|---|---|
| Package manager | `pnpm-lock.yaml` → pnpm 10; `package-lock.json` → npm; `yarn.lock` → yarn | Ask the developer |
| Build script | `package.json` `scripts.build` | Skip build step in Dockerfile |
| Start script | `package.json` `scripts.start` | Fail; ask developer for the start command |
| Port | grep source for `process.env.PORT` and `.listen(` + look at `.env.example` | Assume 3000 (Node default) |
| Health endpoint | grep for `/health`, `/api/health`, `/healthz` in source | See step 3 — add one |
| DB usage | dependency includes `pg`, `postgres`, `drizzle-orm`, `prisma`, `mysql2`, etc. | No DB wiring needed |
| Framework hint (for adding health if missing) | dependency includes `express`, `fastify`, `koa`, `hono`, `next`, `nest`, etc. | Ask the developer to add one |

Print a one-paragraph "Onboarding report" to the developer: **"I see a `<pkg-mgr>` `<framework-hint>` app that builds with `<build cmd>`, starts with `<start cmd>`, expects port `<port>`, has `<health-path or NONE>` for health, uses `<db>`. I'll add a Dockerfile, k8s manifests, GitHub Actions, and (if needed) a two-line health route. No app code changes otherwise. Reply 'go' or tell me to stop."**

Wait for the developer's response. If they say stop, exit cleanly.

### 2. Set variables

```bash
APP=$(basename "$PWD")                              # repo name, must already match kebab-case
PORT=<detected port>                                # from step 1
HEALTH_PATH=<detected or /health>                   # step 3 will add /health if none was found
PKGMGR=<pnpm|npm|yarn>
```

### 3. Add a health endpoint if the app doesn't have one

If step 1 found an existing health path, skip.

If the framework is **Express**, add to the routes file (usually `server/index.ts`, `server/routes.ts`, or `app.js`) as close to the top-level routes as possible:

```ts
app.get('/health', (_req, res) => res.type('text').send('ok'))
```

Then set `HEALTH_PATH=/health`.

If the framework is **Fastify**: `fastify.get('/health', async () => ({ status: 'ok' }))`.

If it's **Next.js**: create `app/api/health/route.ts` — see the template's version for the exact 4 lines.

If the framework is anything else: tell the developer *"add a route at `/health` that returns 200 OK, then re-run me"* and stop.

### 4. Write the Dockerfile

Only write if there is NO existing `Dockerfile`. If one exists, tell the developer *"you already have a Dockerfile. Reconcile it against this reference — it must build the app, run on port `$PORT`, and have `/health` responding."* Include the reference below in your report. Then continue with the rest of the steps.

Reference Dockerfile (adjust `<lockfile>` and `<pkgmgr>` to match):

```dockerfile
# syntax=docker/dockerfile:1
FROM node:24-slim
WORKDIR /usr/src/app

COPY package.json <lockfile> ./
RUN <corepack setup if pnpm> \
    && <pkgmgr> install <frozen-lockfile flag>

COPY . .

RUN <pkgmgr> run build

# If the app reads DATABASE_URL from env and uses the shared platform Postgres,
# construct it from the PG* env vars (populated automatically by the base
# ExternalSecret). If your app doesn't need Postgres, delete this ENTRYPOINT.
ENTRYPOINT ["sh", "-c", "\
  if [ -n \"${PGHOST:-}\" ] && [ -z \"${DATABASE_URL:-}\" ]; then \
    export DATABASE_URL=\"postgresql://${PGUSER}:${PGPASSWORD}@${PGHOST}:${PGPORT}/${PGDATABASE:-<repo_underscore>}?sslmode=require\"; \
  fi; \
  exec \"$@\"", "--"]

ENV NODE_ENV=production
ENV PORT=<detected port>
EXPOSE <detected port>

CMD ["<pkgmgr>", "start"]
```

Substitute:
- `<lockfile>` = `pnpm-lock.yaml*` / `package-lock.json` / `yarn.lock`
- `<corepack setup if pnpm>` = `corepack enable && corepack prepare pnpm@10.15.0 --activate && ` (or empty for npm/yarn)
- `<pkgmgr>` = `pnpm` / `npm` / `yarn`
- `<frozen-lockfile flag>` = `--frozen-lockfile` (pnpm), `ci` in place of `install` (npm), `--immutable` (yarn 4+) or `--frozen-lockfile` (yarn 1)
- `<repo_underscore>` = the repo name with hyphens replaced by underscores (Postgres identifier)

The `ENTRYPOINT` wrapper is optional but harmless — it constructs `DATABASE_URL` from `PG*` if the app expects it, without touching your app code. Preserves any explicit `DATABASE_URL` the developer sets via `/set-secret`.

### 5. Write `.dockerignore`

If missing, write the template's default:

```
node_modules
dist
build
.next
generated
coverage
.git
.gitignore
.gitattributes
.github
manifests
.claude
.cursor
.vscode
.idea
.env
.env.*
.DS_Store
*.log
Dockerfile
.dockerignore
```

If a `.dockerignore` exists, LEAVE IT — the developer may have tuned it.

### 6. Merge `.gitignore` additions

Ensure these lines exist in `.gitignore` (append if missing, don't rewrite):

```
node_modules/
dist/
.next/
generated/
*.tsbuildinfo
.env
.env.*
!.env.example
```

### 7. Write manifests

Copy from `$LOGISK_CUSTOMER_ORG/logisk-app-template/manifests/` (fetch via `gh api ... | base64 -d`), substituting `PLACEHOLDER_APP=$APP` and adjusting these ports:

- `manifests/base/deployment.yaml`:
  - `containerPort: 3000` → `containerPort: $PORT`
  - Every probe's `httpGet.port: 3000` → `$PORT`
  - Every probe's `httpGet.path: /api/health` → `$HEALTH_PATH`
- `manifests/base/service.yaml`:
  - `targetPort: 3000` → `$PORT`

Everything else in the manifests (kustomization, ingress with `PLACEHOLDER_HOST`, external-secret with `dataFrom` patterns, prod/preview overlays) is unchanged.

### 8. Write GitHub Actions workflows

Copy `.github/workflows/build.yaml`, `set-secret.yaml`, `delete-secret.yaml`, `list-secrets.yaml`, `preview-cleanup.yaml` from the template. No modifications — the `PLACEHOLDER_CUSTOMERORG` in these files is already substituted in the seeded template.

### 9. Substitute PLACEHOLDER_APP

```bash
find manifests .github -type f -name '*.yaml' -exec sed -i.bak \
  -e "s|PLACEHOLDER_APP|$APP|g" {} \;
find manifests .github -name '*.bak' -delete
```

Do NOT touch `PLACEHOLDER_TAG` or `PLACEHOLDER_HOST` — CI and ArgoCD substitute those at runtime.

### 10. Commit + push

```bash
git checkout -b onboard-to-platform
git add -A
git commit -m "onboard: add Docker + k8s manifests + GitHub Actions for platform deploy"
git push -u origin onboard-to-platform
```

Open a PR (see step 12). Do NOT push directly to main.

### 11. Wait for the first build to succeed

Same pattern as `/new-app`:

```bash
RUN_ID=$(gh run list --repo $LOGISK_CUSTOMER_ORG/$APP --branch onboard-to-platform --limit 1 --json databaseId --jq '.[0].databaseId')
until [ "$(gh run view $RUN_ID --repo $LOGISK_CUSTOMER_ORG/$APP --json status --jq .status)" = "completed" ]; do sleep 30; done
gh run view $RUN_ID --repo $LOGISK_CUSTOMER_ORG/$APP --json conclusion --jq .conclusion
```

If it fails, read the failure with `gh run view $RUN_ID --repo $LOGISK_CUSTOMER_ORG/$APP --log-failed`. Common failures:

- `pnpm install --frozen-lockfile` fails because npm/yarn was actually used → fix the Dockerfile's install command.
- Build fails because a script assumes a working directory / env → the reference Dockerfile might not cover the app's build oddity. Adjust.
- Docker build fails on missing files → check the app's `.dockerignore` isn't excluding something the build needs.

Iterate on the branch until green.

### 12. Open a PR

```bash
gh pr create --title "Onboard to Logiskbrist platform" \
  --body "Adds Docker + k8s manifests + GitHub Actions so this repo deploys automatically. Preview will come up at feature-onboard-to-platform-$APP.$LOGISK_CUSTOMER_DOMAIN once merged and re-pushed to main." \
  --draft
```

Report the preview URL: `https://onboard-to-platform-$APP.$LOGISK_CUSTOMER_DOMAIN/` — should render the same as the app renders locally, and `$HEALTH_PATH` should return 200.

### 13. Add the `logisk-platform` topic AFTER the developer merges

Wait for the developer to merge the PR. Then tell them to run (or they'll ask their next agent to run):

```bash
gh repo edit --add-topic logisk-platform
```

Without this topic, ArgoCD doesn't discover the repo. Doing it AFTER merge means the very first ArgoCD sync sees a real image tag (written by `bump-prod` on the merge commit), never a `PLACEHOLDER_TAG` window.

## Failure modes and remediation

- **Prerequisite fails**: report which one and stop.
- **App has no `build` script**: skip the build step in the Dockerfile.
- **App has no `start` script**: ask the developer for the exact command, use it as the Dockerfile CMD.
- **Framework not recognized for health**: ask the developer to add `/health` themselves and re-run.
- **Existing Dockerfile**: preserve it, output the reference for reconciliation, still add manifests + workflows.
- **DB detected but app expects DATABASE_URL and doesn't have it**: the ENTRYPOINT wrapper constructs it from `PG*`. If the app expects something exotic (Mongo URI, individual `DB_HOST`/`DB_PORT`), the developer will need to `/set-secret` those directly.

## What NOT to do

- Do NOT rewrite the app's code beyond a single missing health route.
- Do NOT change the package manager (npm→pnpm or similar) — that's a separate opinionated migration, not the point of this skill.
- Do NOT modify `package.json` (except if the developer explicitly asks). The app's dependency choices are the app's.
- Do NOT push to main. Open a PR — the developer reviews.
- Do NOT add the `logisk-platform` topic before the first successful build on the target branch (creates a bad-tag ArgoCD window).
