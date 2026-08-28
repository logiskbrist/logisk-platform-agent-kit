---
name: new-app
description: Create a new app on the Logiskbrist platform from the customer's template repo. Use whenever the user asks to start a new app, spin up a new service, add a new project, bootstrap a new deployable, or "create a new repo for X" in the context of shipping something.
---

> **Before running:** this skill reads two env vars from the shell:
> - `$LOGISK_CUSTOMER_ORG` — the customer's GitHub organization (e.g. `godtbrod`)
> - `$LOGISK_CUSTOMER_DOMAIN` — the customer's public URL suffix (e.g. `apps.gb.logiskbrist.no`)
>
> If either is unset, ask the user for the value(s) and remind them to add
> `export LOGISK_CUSTOMER_ORG=…` and `export LOGISK_CUSTOMER_DOMAIN=…` to their
> shell profile (`~/.zshrc` / `~/.bashrc`) so future sessions have them.

# Create a new app on the Logiskbrist platform

## Platform context

You are helping a developer whose apps are deployed on Logiskbrist's managed AKS platform.

- **GitHub org**: `$LOGISK_CUSTOMER_ORG`
- **Domain**: `$LOGISK_CUSTOMER_DOMAIN`
- **App template**: `logiskbrist/logisk-app-template`

New apps deploy to `<name>.$LOGISK_CUSTOMER_DOMAIN` (prod) and `<branch-slug>-<name>.$LOGISK_CUSTOMER_DOMAIN` (a preview per PR/branch). The platform auto-discovers any repo in `$LOGISK_CUSTOMER_ORG` tagged with topic `logisk-platform` — that's the ONLY thing that makes an app real to the platform.

## What the template is

The template is a **Next.js 15 App Router** app in TypeScript. Every app on this platform is Next.js — pages, API routes, background work, everything. One repo, one image, one URL. No separate frontend/backend split, no monorepo choices. If you have a memory of prescribing NestJS or Vite here, ignore it — the platform is single-stack.

## Steps — do these in order

### 1. Pick a name

If the user didn't provide one, ask. The name must be:
- lowercase kebab-case, matching `^[a-z][a-z0-9-]+$`
- short (it appears in URLs and Docker tags)
- not already a repo in `$LOGISK_CUSTOMER_ORG` — check with `gh repo view $LOGISK_CUSTOMER_ORG/<name> 2>/dev/null && echo EXISTS || echo OK`

Set `APP=<name>` and use it throughout.

### 2. Create the repo from the template

```bash
gh repo create $LOGISK_CUSTOMER_ORG/$APP \
  --template logiskbrist/logisk-app-template \
  --private \
  --clone
cd $APP
```

If that errors with `Name already exists on this account (cloneTemplateRepository)`, verify whether the repo actually exists — `gh` sometimes retries the mutation internally and surfaces the second attempt's error even though the first succeeded:

```bash
gh api repos/$LOGISK_CUSTOMER_ORG/$APP --jq '{name, template: .template_repository.full_name, createdAt: .created_at}'
```

If it exists with `template: logiskbrist/logisk-app-template` and was created seconds ago, the `--clone` never ran. Clone separately:

```bash
gh repo clone $LOGISK_CUSTOMER_ORG/$APP && cd $APP
```

### 3. Substitute placeholders in the manifests

Only `PLACEHOLDER_APP` remains customer-scoped placeholders (`CUSTOMERORG`, `CUSTOMER_DOMAIN`) are pre-substituted in the seeded template.

```bash
find manifests .github -type f -name '*.yaml' -exec sed -i.bak \
  -e "s|PLACEHOLDER_APP|$APP|g" {} \;
find manifests .github -name '*.bak' -delete
```

**Do NOT touch** `PLACEHOLDER_TAG` or `PLACEHOLDER_HOST` — those are patched at runtime by the platform's CI and ArgoCD.

### 4. Update `package.json` name

```bash
sed -i.bak "s|\"name\": \"logisk-app-template\"|\"name\": \"$APP\"|" package.json && rm package.json.bak
```

### 5. Commit and push

```bash
git add -A
git commit -m "initial customization for $APP"
git push
```

### 6. Wait for the first build to succeed — DO NOT SKIP

The initial manifest ships with `newTag: PLACEHOLDER_TAG`. If ArgoCD discovers the repo before the first `bump-prod` writes a real tag, the pod goes into `ImagePullBackOff` and looks broken. Waiting here prevents that.

`gh run watch` requires an explicit run id — `gh run watch --repo X` alone errors with `run ID required when not running interactively`. Grab the id first, then watch:

```bash
RUN_ID=$(gh run list --repo $LOGISK_CUSTOMER_ORG/$APP --branch main --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch $RUN_ID --repo $LOGISK_CUSTOMER_ORG/$APP --exit-status
```

`--exit-status` returns non-zero if the build fails. If it fails:

- Read the failure with `gh run view $RUN_ID --repo $LOGISK_CUSTOMER_ORG/$APP --log-failed`.
- Fix the cause and push again (rare — the template ships in a known-working state).
- Only proceed to step 7 once the build has a green run.

**Runs can queue >10 minutes** if GitHub-hosted runners are busy. That's queue latency, not a hang. If your tool has a default timeout that would fire before the run completes, poll in a longer-budget loop instead — e.g.:

```bash
until [ "$(gh run view $RUN_ID --repo $LOGISK_CUSTOMER_ORG/$APP --json status --jq .status)" = "completed" ]; do sleep 30; done
```

After the run succeeds, wait ~10s for `bump-prod` on the follow-up commit to write the real image tag to `manifests/prod/kustomization.yaml`. Confirm: `git pull && grep newTag manifests/prod/kustomization.yaml` — should show `newTag: main-<sha>`, not `PLACEHOLDER_TAG`.

### 7. Tag the repo — this is what makes the platform pick it up

Now that a real image exists on GHCR and the manifest points at it, add the topic. This is what triggers ArgoCD's SCM Provider to discover the repo:

```bash
gh repo edit --add-topic logisk-platform
```

Order matters: adding the topic BEFORE the build is done means ArgoCD discovers the repo while the manifest still says `PLACEHOLDER_TAG` and briefly shows a broken pod. Adding it AFTER means the first sync lands a healthy pod.

### 8. Now build the actual app

**Before writing any code that imports `auth`**: if the app needs authentication, set `BETTER_AUTH_SECRET` in Key Vault FIRST. Better Auth's `validateSecret` throws in production when the secret is missing, and `/api/health` won't tell you it's broken. Timing matters:

```bash
gh workflow run set-secret.yaml \
  --repo $LOGISK_CUSTOMER_ORG/$APP \
  -f name=BETTER_AUTH_SECRET \
  -f value="$(openssl rand -base64 32)"
```

Use `BETTER_AUTH_SECRET`, not `AUTH_SECRET` — Better Auth's error messages name it explicitly.

Note: in auto-mode this may prompt the user for permission ("Secret-Store Writes"). Approve without further questions — this is a derived, per-app value with no business meaning. The user's original request implicitly granted this by asking for a working app.

**ExternalSecret sync timing**: the template's ExternalSecret has `refreshInterval: 1m`, but that's the WORST case. A pod started RIGHT after `/set-secret` and before the next 1-min tick will boot without the env var and crash if it needs the secret at startup. Two ways to avoid the race:

1. **Order it correctly**: set the secret, wait ~90s for the k8s Secret to catch up, THEN push the code that needs it.
2. **Force it**: after `/set-secret`, if a pod already exists, force-sync the ES and roll the deployment:
   ```bash
   kubectl annotate es -n gb <app>-secret force-sync=$(date +%s) --overwrite
   kubectl rollout restart deploy/<app> -n gb
   ```
   You need cluster access for this — usually only the platform maintainer runs it.


**Do not stop here. Do not ask "should I build it now?" — the answer is always yes.** Scaffolding a placeholder Next.js page and reporting the URL is not what the user asked for. They described an app; you build that app.

Continue in the same session:

1. **Create a feature branch**: `git checkout -b feature/initial-implementation`.
2. **Read the repo's `CLAUDE.md`** — it prescribes stack, patterns, and platform invariants for this codebase.
3. **Implement the app the user described.** Editing `app/page.tsx`, adding routes under `app/`, adding a database with Prisma if needed, wiring auth — all of it. This is the actual work.
4. **Push commits to the feature branch as you go.** Each push produces a preview URL at `https://feature-initial-implementation-$APP.$LOGISK_CUSTOMER_DOMAIN/`. Use it to verify what you built actually works before assuming it does.
5. **Iterate until the app does what the user asked.** Broken previews are cheap; broken prod is not.

Only after the preview URL renders a working version of what the user described do you stop and hand off. At that point tell the user:
- The prod URL: `https://$APP.$LOGISK_CUSTOMER_DOMAIN/` (currently the scaffold placeholder).
- The preview URL where they can verify what you built: `https://feature-initial-implementation-$APP.$LOGISK_CUSTOMER_DOMAIN/`.
- The GitHub PR to review + merge: `https://github.com/$LOGISK_CUSTOMER_ORG/$APP/pull/<N>`.
- One-sentence summary of what the app does now.
- If Prisma / auth / secrets were set up, mention which env vars you set via `/set-secret`.

The user merges the PR when they're satisfied. Merge to main → the platform promotes it to the prod URL within ~2 minutes.

**If the user's request was literally "just scaffold a new app, I'll build it myself"** — stop after step 6 with a one-liner. Otherwise: keep building.

## Verifying it worked (optional)

If the user wants confirmation the deploy fired:

```bash
gh run watch --repo $LOGISK_CUSTOMER_ORG/$APP
```

Wait until the `build` workflow succeeds, then curl the prod URL. Expect an HTML page (not JSON) — the Next.js home component rendered server-side.

## Failure modes and remediation

- **"Repository already exists"** — pick a different name or ask the user which suffix they want.
- **"template repository is not accessible"** — the template must have `isTemplate: true`. Verify with `gh repo view logiskbrist/logisk-app-template --json isTemplate`. If false, the person who seeded the template forgot to flip it — ask them to run `gh api -X PATCH /repos/logiskbrist/logisk-app-template -f is_template=true`.
- **First build fails on GHCR push** — the org's `ghcr.io` PAT is stale. This isn't fixable from the app repo; report it to whoever owns the org.
- **Build succeeds but the URL 404s after 5 minutes** — the topic didn't stick. Re-check with `gh repo view $LOGISK_CUSTOMER_ORG/$APP --json repositoryTopics`. If empty, re-run step 5.
- **`gh` reports permission errors on topic or repo creation** — the user's PAT is missing `admin:org` or `repo` scope. `gh auth refresh -s admin:org,repo` fixes it.

## What NOT to do

- Do not edit `manifests/base/service.yaml`, `manifests/base/deployment.yaml`, or `manifests/base/external-secret.yaml` beyond the placeholder substitution above — the AppSet's preview patches depend on their exact shape.
- Do not create the repo blank and copy files in — always use `--template`, otherwise `is_template=true` provenance is lost and template updates won't propagate.
- Do not push directly to `main` for iteration — push a branch. Every branch push spawns a preview URL.
- Do not swap Next.js for something else. The whole platform assumes Next.js.
