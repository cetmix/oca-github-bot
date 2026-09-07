# Cetmix customizations

This repository is a Cetmix fork of [OCA/oca-github-bot](https://github.com/OCA/oca-github-bot).

**Write only to Cetmix** (`cetmix/oca-github-bot`). Never commit or push to the OCA upstream repo; sync is fetch + merge into this fork only.

Only the changes below are intentional Cetmix customizations. Everything else should track OCA `master`.

Companion tools: [cetmix/cetmix-maintainer-tools](https://github.com/cetmix/cetmix-maintainer-tools) (local clone: `../cetmix-maintainer-tools`). The bot Dockerfile pins that fork instead of `OCA/maintainer-tools`.

## Remotes

| Remote | URL | Role |
|--------|-----|------|
| `origin` | `https://github.com/cetmix/oca-github-bot.git` | Only remote allowed for push |
| `OCA` | `https://github.com/OCA/oca-github-bot.git` | Fetch-only upstream (`push` URL must be `DISABLE_PUSH_TO_OCA`) |

Working branch for Cetmix deployments: `support_private_repo` (tracks `origin/support_private_repo`). Cetmix `master` may lag; prefer syncing and deploying from `support_private_repo`.

## 1. Private repository clone support

**Commit:** `2db6316` (`Support private repo`)

When `GITHUB_REPO_PRIVATE` is set (truthy string), `temporary_clone` fetches and clones with a tokenized HTTPS URL so private org repos work.

| File | Required end state |
|------|--------------------|
| `src/oca_github_bot/config.py` | `GITHUB_REPO_PRIVATE = os.environ.get("GITHUB_REPO_PRIVATE")` |
| `src/oca_github_bot/github.py` | `repo_url = repo_url_with_token if config.GITHUB_REPO_PRIVATE else repo_url_no_token` |
| `environment.sample` | Documented `GITHUB_REPO_PRIVATE=True` |

Do not drop OCA improvements in the same files (e.g. `raise Retry(...) from e`).

## 2. Cetmix maintainer-tools pin

**Commits:** Dockerfile bumps on `support_private_repo` (e.g. `64d8598`, merge restore)

`Dockerfile` must install from `cetmix/cetmix-maintainer-tools`, not `OCA/maintainer-tools`.

Current pin (refresh when syncing maintainer-tools):

```text
git+https://github.com/cetmix/cetmix-maintainer-tools@d9fd54890e4bbdcda5e93d05cd0d33b17010bfab#egg=oca-maintainers-tools
```

After updating the pin, confirm the SHA exists on `cetmix/cetmix-maintainer-tools` and that Cetmix customizations there are intact (see that repo’s `docs/CETMIX_CUSTOMIZATIONS.md`).

## 3. Maintainer-check Odoo series list

**File:** `environment.sample`

`MAINTAINER_CHECK_ODOO_RELEASES` includes series Cetmix still maintains (currently through `19.0`). Take new series from OCA when they add them; keep any newer Cetmix-only series that deployment still uses.

## 4. README fork pointer

**File:** `README.rst`

Short note at the top identifying this as the Cetmix fork and linking `docs/CETMIX_CUSTOMIZATIONS.md` plus `cetmix-maintainer-tools`. Keep the rest of the OCA README (features, running, development).

## 5. Wheel export allow/deny lists

| File | Change |
|------|--------|
| `src/oca_github_bot/config.py` | `OCABOT_EXPORT_REPOS_ALLOW`, `OCABOT_EXPORT_REPOS_DENY`, `should_export_repo()` |
| `src/oca_github_bot/tasks/main_branch_bot.py` | Gate `build_and_publish_*` with `should_export_repo` |
| `src/oca_github_bot/tasks/merge_bot.py` | Same gate before pre-merge wheel publish |
| `environment.sample` | Document the two env vars |

Deny list wins over allow list. Empty allow list means “all repos except deny”.

When merging OCA, preserve OCA’s current `build_and_publish_metapackage_wheel(...)` signature (no separate series argument as of OCA `eb48c96`).

## 6. Ignore Runboat commit status on merge

**Why:** Cetmix Runboat posts `runboat/build` on merge-bot branches. Kept in
the ignore list so any leftover status-webhook path does not treat Runboat as
required.

**Files:**

| File | Required end state |
|------|--------------------|
| `src/oca_github_bot/config.py` | Default `GITHUB_STATUS_IGNORED` includes `runboat/build` |
| `environment.sample` | Documented default includes `runboat/build` |

OCA default stays `ci/runbot,codecov/...` only — keep `runboat/build` when
merging upstream into this fork.

## 7. Merge without waiting for CI

**Why:** OCA waits for green checks on the temporary `*-ocabot-merge-pr-*`
branch. Cetmix private/addon repos often never complete any check there
(missing `{series}-ocabot-*` workflow triggers, `norunboat`, queued empty
CodeRabbit/Cursor suites), so PRs stayed on `bot is merging ⏳` forever.

**File:** `src/oca_github_bot/tasks/merge_bot.py` — `merge_bot_start`

After preparing and pushing the merge-bot branch, call `_merge_bot_merge_pr`
immediately. Do not wait for `status` / `check_suite` webhooks.
`merge_bot_status` remains for compatibility with delayed webhooks but is no
longer required for a successful `/ocabot merge`.

On OCA sync: keep this immediate-finalize behavior; do not restore
“awaiting test results” as the only path.

## Not customizations

| Observation | Action on sync |
|-------------|----------------|
| New OCA tasks/webhooks (e.g. `label_modified_addons`) | Take OCA |
| Dependency bumps, ruff/pre-commit, README feature docs | Take OCA |
| `MODULE_LABEL_COLOR` and related env sample lines | Take OCA |
| `MAIN_BRANCH_BOT_MIN_VERSION` default / wheel build API changes | Take OCA |

## Syncing from OCA

```bash
git status   # start clean; stash Cetmix WIP first
git remote get-url OCA || git remote add OCA https://github.com/OCA/oca-github-bot.git
git remote set-url --push OCA DISABLE_PUSH_TO_OCA
git fetch OCA master
PRESYNC=$(git rev-parse HEAD)
git merge --no-ff OCA/master -m "Merge branch 'OCA-master'"
```

Resolve conflicts:

- `Dockerfile` → keep `cetmix/cetmix-maintainer-tools@<current Cetmix SHA>`
- `config.py` / `github.py` / `environment.sample` → keep private-repo bits; keep OCA additions (`MODULE_LABEL_COLOR`, task list comments, etc.)
- Prefer combining, not blind `checkout $PRESYNC` of whole files

Verify:

```bash
git rev-parse OCA/master
git diff OCA/master HEAD --stat
# Expect only Cetmix customization files

grep GITHUB_REPO_PRIVATE src/oca_github_bot/config.py environment.sample
grep -A2 'GITHUB_REPO_PRIVATE' src/oca_github_bot/github.py
grep runboat/build src/oca_github_bot/config.py environment.sample
grep cetmix-maintainer-tools Dockerfile
! grep 'github.com/OCA/maintainer-tools@' Dockerfile
```

Run tests (`tox -e py312` or project venv + `pytest`). Do not commit/push to OCA. Commit/push to Cetmix only when asked.

## Related local layout

```text
pipeline-tools/
  oca-github-bot/              # this fork (cetmix/oca-github-bot)
  cetmix-maintainer-tools/     # Cetmix fork of OCA/maintainer-tools
```
