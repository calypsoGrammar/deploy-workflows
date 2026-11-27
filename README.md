# deploy-workflows

Shared GitHub Actions workflows for deploying calypsoGrammar apps, so every app deploys, rolls back and reports
the same way. Archon serves only projects that meet the contract below (CAL-2250), and Chronology reads what
these workflows record to show what is waiting to deploy (CAL-2251).

## The contract

An app meets it when `.github/workflows/deploy.yml` on main has a `workflow_dispatch` whose `target` offers
`production` (and `staging` when the app has one) and that takes a `ref`. In practice: call one of the shared
workflows here from the template.

- **Deploys are manual.** Only a run started by hand reaches production or staging; nothing deploys on a merge.
  (Automatic staging deploys were dropped in November 2026 to save Actions minutes.)
- **Environment names.** A target's variables and secrets live in the GitHub environment of the same name, or
  in the one the caller names with `environment` (for apps whose environment predates the contract).
- **Only green commits on main.** The commit must be on main and the caller's `required_checks` must have passed.
- **Rollback is the same workflow with an earlier `ref`.** Ticking `rollback` with no ref deploys the `stable` tag.
  A production rollback then sets the `deploy-hold` tag: production deploys refuse until it is cleared
  (Actions → stable → Run workflow → `clear-hold`), so the next merge can't re-ship the fault.
- **Migrations aren't rolled back.** Going to an older commit is refused when the migrations differ between the
  live commit and the target, unless `force_migrations` is ticked.
- **What readers count.** Every real run records a GitHub deployment with task `release` (payload: `previous`,
  `rollback`) and finishes it as `success` or `failure`. GitHub also records a deployment with task `deploy` for
  every job that names an environment, dry runs included; readers ignore those. That per-job record finishes a few seconds after ours and marks ours `inactive`,
  so a release's outcome is its newest status other than `inactive`.

## The stable tag

`stable.yml` runs daily in each app. When production's latest `release` succeeded at least 48 hours ago and
nothing newer has been released to production since, `stable` moves to it and a dated `stable-YYYY-MM-DD` tag is
added. Run it by hand to `pin` a chosen commit, or to `clear-hold`. Tags, not branches: a tag names one commit.

## Workflows

| Workflow | For |
| --- | --- |
| `.github/workflows/cloudways-laravel.yml` | Laravel apps on Cloudways: build on GitHub's runners, rsync over SSH, release tasks, live checks. Grown from picks_2's deploy. |
| `.github/workflows/github-pages.yml` | Static sites on GitHub Pages, from mazemaker's: production only (Pages has no staging), built, checked and published by hand. Caller: `templates/deploy-pages.yml`. The Pages source must be "GitHub Actions". |
| `.github/workflows/coolify.yml` | Apps on the VPS run by Coolify (coolify.calypsogrammar.com): pins the commit, starts a Coolify deployment through its API and waits for it, then checks the site. Caller: `templates/deploy-coolify.yml`. |
| `.github/workflows/cloudflare-pages.yml` | Static sites on Cloudflare Pages (`<name>.calypsogrammar.com`): production only, built here, checked and uploaded with Wrangler by hand. Caller: `templates/deploy-cloudflare-pages.yml`. |
| `.github/workflows/stable.yml` | The `stable` tag and the production hold. |

Droplet apps (Archon, Chronology, Convergence, droplet-backup) don't deploy from Actions: they install from Archon
config's Releases page (CAL-2136), which records the same `release` deployments.

## Adding an app on Cloudways

```bash
scripts/setup-cloudways calypsoGrammar/<repo> --server <name> --app <folder> --url https://<site> --checks "PHP tests"
```

Add `--staging-app <folder> --staging-url <url>` (and `--staging-server` if it's elsewhere) for a staging site.
Servers are listed in `~/.config/deploy-workflows/cloudways-servers` as `<name> <host> <user> [port]`.

The script makes a new SSH key for the repository and prints its public half; add it in Cloudways (Server →
Master Credentials → SSH Public Keys) and press Enter. It checks the app folder over SSH, then sets the GitHub
environment variables (`CLOUDWAYS_HOST`, `CLOUDWAYS_USER`, `CLOUDWAYS_PORT`, `CLOUDWAYS_APP`, `APP_URL`) and
secrets (`CLOUDWAYS_SSH_KEY`, `CLOUDWAYS_KNOWN_HOSTS`), and opens a pull request adding `deploy.yml`,
`stable.yml` and `.github/deploy-exclude.txt` from `templates/`. After merging, run a dry run.

The server needs an `.env` in the app folder before the first real deploy, and the Cloudways webroot set to
`public_html/public`; the deploy checks both. Optional environment settings: `CLOUDWAYS_PHP` (the PHP command for
artisan, detected when empty) and the `APP_BASIC_AUTH` secret (`user:password`) for a password-protected staging site.

## Adding a site on Cloudflare Pages

1. Create the Pages project (Workers & Pages → Pages → Connect to Git, or the API) with production branch `main`,
   attach `<name>.calypsogrammar.com`, and add the CNAME `<name>` → `<project>.pages.dev` at the DNS host.
2. Turn the project's automatic Git builds off (Settings → Builds & deployments, or the API's
   `source.config.deployments_enabled: false`), so only this workflow publishes.
3. In the repository: the secret `CLOUDFLARE_API_TOKEN` (a token with Cloudflare Pages: Edit) and the variable
   `CLOUDFLARE_ACCOUNT_ID`.
4. Add `deploy.yml` from `templates/deploy-cloudflare-pages.yml` (set `project_name` and `site_url`) and
   `stable.yml` from `templates/stable.yml`, and a CI job whose name is in `required_checks`. After merging, run a
   dry run, then a real one.

The build runs on GitHub's runners as on GitHub Pages; Wrangler uploads the built folder as the project's
production deployment for that commit.

## Adding an app on the VPS (Coolify)

1. Create the application in Coolify (a Dockerfile in the repository, its domain, storage and any scheduled task) and
   copy its UUID from the dashboard URL (`…/application/<uuid>`). Leave Coolify's automatic deploys off.
2. In the repository: the secrets `COOLIFY_TOKEN` (Coolify → Keys & Tokens → API tokens, with deploy and write),
   `CF_ACCESS_CLIENT_ID` and `CF_ACCESS_CLIENT_SECRET` (a Cloudflare Access service token that the Coolify Access
   application allows with a Service Auth policy).
3. Add `deploy.yml` from `templates/deploy-coolify.yml` (set `app_uuid`, `site_url` and `required_checks`) and
   `stable.yml` from `templates/stable.yml`. After merging, run a dry run: it checks the commit and the migrations and
   that the tokens reach Coolify, then a real one.

Coolify builds the image itself. The workflow pins the application's `git_commit_sha` to the released commit before
each deployment, which is what makes `ref` and rollbacks exact. The app's own start-up runs its migrations.

## Access

This repository is private, with Actions access set to repositories owned by calypsoGrammar, so other repositories
can call its workflows. Callers pin a tag (`@v1`); move the tag deliberately when changing a workflow.
