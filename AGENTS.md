# brightbean-studio (DROIX fork) — instructions

DROIX's self-hosted fork of [BrightBean Studio](https://github.com/brightbeanxyz/brightbean-studio), an
AGPL-3.0 social media scheduling, approvals and analytics app: Django 5, HTMX + Alpine.js templates,
Tailwind 4, PostgreSQL (which is also the job queue through `django-background-tasks`). It runs live at
**https://social.droix.ai**. These rules apply to every agent and tool working in this repository.

**This repository is public** (an AGPL fork). Never commit credentials, `.env` files, server IP
addresses, customer data or DROIX business figures here. Those live in the private DROIX wiki (below).

## Runtime-specific instructions

| Runtime | File | How it is loaded |
|---|---|---|
| Claude Code | `CLAUDE.md` | Automatically; it imports this file |
| Any other | — | This file only |

A rule for every agent goes in this file. A project rule never lives only in a runtime's private
memory: other agents never see it.

## DROIX wiki

DROIX's business knowledge lives in the private **droix-wiki** repository
(`github.com/droixuk/droix-wiki`, an LLM-maintained Obsidian vault). It is the reference for who DROIX
is and how it works: brands, departments, processes, policies, sales and social channels, the
software DROIX buys and builds, and the servers and domains it all runs on.

- **Read before you guess.** Before work that touches DROIX's brands, social channels, approval flows,
  marketing processes or infrastructure, read the matching wiki pages. Start at `wiki/index.md`.
  Pages most relevant to this project:
  - `wiki/software/postiz.md` — the other self-hosted social tool DROIX runs (postiz.droix.store).
  - `wiki/processes/marketing-and-reviews.md`, `wiki/processes/marketplace-and-social-support.md`
  - `wiki/policies/marketing-consent-and-tracking.md`
  - `wiki/servers/coolify-primary.md` — the Coolify host this app is deployed on.
  - `wiki/domains/domain-droix-ai.md` — the `droix.ai` domain and its DNS.
- **Reading it.** Use a local clone if one exists; otherwise read through the GitHub API, for example
  `gh api repos/droixuk/droix-wiki/contents/wiki/index.md --jq .content | base64 -d`.
- **The wiki is the source of business truth; the code decides how this app behaves.** When they
  disagree, report it. Do not "fix" either side silently.
- **Do not edit droix-wiki from this project.** It has its own rules (its `AGENTS.md`: provenance
  marks, contradiction log, sensitive-data scan) and only changes when the user asks for a wiki task.
  The wiki learns about projects by ingesting their `BUSINESS.md`. A business fact about how DROIX
  uses this app (which brands and accounts, who approves, which platforms) goes into this repo's
  `BUSINESS.md`, never into code comments or agent memory. That file and its droix-wiki source entry
  do not exist yet: create them only when the user asks.

## Upstream and fork

| Remote | Repository | Use |
|---|---|---|
| `origin` | `droixuk/brightbean-studio` | Our fork. Push here. Coolify builds from it. |
| `upstream` | `brightbeanxyz/brightbean-studio` | The original project. Fetch and merge their fixes; never push. |

- Keep DROIX changes small and separable so upstream merges stay easy. Prefer configuration and
  environment variables over editing upstream code.
- A general-purpose change is offered upstream: branch from `upstream/main`, commit only that
  change, push the branch to `origin` and open the PR against `brightbeanxyz/brightbean-studio`.
  Then merge the same branch into our `main` to deploy it. DROIX-only files (`AGENTS.md`,
  `CLAUDE.md`) live on our `main` only and never go into an upstream branch.
- Bring in upstream fixes with `git fetch upstream && git merge upstream/main` on a branch, then run
  the checks below. Read any new migration before it ships.

## Deployment

- Hosted on the DROIX primary Coolify instance as the Docker Compose service "brightbean"
  (containers `app-…`, `worker-…`, `postgres-…`). Traefik terminates TLS and routes
  `social.droix.ai` straight to the app container on port 8000; the repository's `Caddyfile` and
  `docker-compose.prod.yml` are **not** used in production.
- The Coolify compose file **pins a commit**: the build context URL is
  `https://github.com/droixuk/brightbean-studio.git#<commit>` in three places (migrate, app, worker).
  To ship: push to `origin`, change that hash to the new commit in all three, redeploy.
- Production config is the env block in that compose file (`DJANGO_SETTINGS_MODULE=config.settings.production`,
  `APP_URL`, `ALLOWED_HOSTS`, …), not this repo. A copy of the compose file is taken before every
  edit and kept on the server under `~/recovery/`.
- Platform credentials (`PLATFORM_*`, webhook tokens) are referenced in the compose file as
  `'${NAME}'` and their values stored as encrypted Coolify environment variables, set through the
  Coolify API. Never write a secret value into the compose file, this repo or a commit message.
- Configured so far: Facebook, Instagram, Threads (one Meta app), YouTube and Google Business
  Profile (one Google OAuth client, External / In production). LinkedIn awaits its API approval;
  outgoing email is not configured yet.
- Redeploying, editing the Coolify compose, and anything that writes to the production database need
  the user's go-ahead in the current conversation. Reading logs and running read-only checks do not.

## Working here

| Task | Command |
|---|---|
| Setup | `make setup` |
| Run locally | `make dev` (server, worker, Tailwind watcher) or `make docker-up` |
| Tests | `make test` |
| Lint / format | `make lint` / `make format` |
| Types | `make typecheck` |
| New migration | `make migrations`, then read the generated file |

- Code layout: Django apps in `apps/`, one module per social platform in `providers/`, settings in
  `config/settings/{base,development,production,test}.py`, templates in `templates/`.
- Specs: `development_specs/feature-spec-social-media-management-v2.md` (what the product does) and
  `development_specs/architecture.md` (how it is built). Read the relevant section before a feature
  change.
- A change that adds a migration is not "just a rebuild" on deploy: say so when proposing to ship it.
- Commit only when asked. Branch first if on `main`.
