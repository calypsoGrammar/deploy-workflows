# deploy-workflows

Shared GitHub Actions workflows for deploying calypsoGrammar apps, so every app deploys, rolls back and reports
the same way. Archon serves only projects that meet the contract below (CAL-2250), and Chronology reads what
these workflows record to show what is waiting to deploy (CAL-2251).

## The contract

An app meets it when `.github/workflows/deploy.yml` on main has a `workflow_dispatch` whose `target` offers
`production` (and `staging` when the app has one) and that takes a `ref`. In practice: call one of the shared
workflows here from the template.

- **Production is manual.** Only a run started by hand reaches production; nothing deploys to production on a
  merge. Staging may deploy on every green CI run on main.
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

## Access

This repository is private, with Actions access set to repositories owned by calypsoGrammar, so other repositories
can call its workflows. Callers pin a tag (`@v1`); move the tag deliberately when changing a workflow.
