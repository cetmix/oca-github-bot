---
name: sync-oca-upstream
description: >-
  Sync cetmix/oca-github-bot with upstream OCA/oca-github-bot while preserving
  Cetmix customizations. Use when updating from OCA, merging upstream, or
  refreshing this Cetmix fork of oca-github-bot.
---

# Sync OCA upstream into Cetmix oca-github-bot

## Facts (do not invent others)

- This repo: `https://github.com/cetmix/oca-github-bot` (`origin`).
- Upstream: `https://github.com/OCA/oca-github-bot` (remote name: `OCA`, branch: `master`).
- **Never commit or push to OCA.** Only `origin` (Cetmix) is a write remote. Keep `OCA` fetch-only (`git remote set-url --push OCA DISABLE_PUSH_TO_OCA`).
- Preferred Cetmix branch: `support_private_repo`.
- Companion: `../cetmix-maintainer-tools` / `cetmix/cetmix-maintainer-tools` — Dockerfile must pin that fork.

## Customizations to preserve

Authoritative inventory: [docs/CETMIX_CUSTOMIZATIONS.md](../../../docs/CETMIX_CUSTOMIZATIONS.md).

| Area | Files | Required end state |
|------|-------|--------------------|
| Private repos | `config.py`, `github.py`, `environment.sample` | `GITHUB_REPO_PRIVATE` + tokenized clone URL when set |
| Maintainer-tools | `Dockerfile` | `cetmix/cetmix-maintainer-tools@<SHA>` (not OCA) |
| Series list | `environment.sample` | `MAINTAINER_CHECK_ODOO_RELEASES` includes Cetmix series (e.g. `19.0`) |
| Export filter | `config.py`, `main_branch_bot.py`, `merge_bot.py`, `environment.sample` | `should_export_repo` gates wheel publish |
| README pointer | `README.rst` | Cetmix fork note + link to docs |

## Workflow

```
Sync progress:
- [ ] 1. Preconditions
- [ ] 2. Snapshot customizations
- [ ] 3. Fetch and merge OCA/master
- [ ] 4. Re-apply customizations
- [ ] 5. Verify inventory
- [ ] 6. Run tests
- [ ] 7. Stop for user review (commit/push only if asked)
```

### 1. Preconditions

```bash
git status   # must be clean (stash Cetmix WIP first)
git remote get-url OCA || git remote add OCA https://github.com/OCA/oca-github-bot.git
git remote set-url --push OCA DISABLE_PUSH_TO_OCA
git fetch OCA master
git merge-base HEAD OCA/master
git log --oneline HEAD..OCA/master
```

### 2. Snapshot

`PRESYNC=$(git rev-parse HEAD)`. Note Dockerfile maintainer-tools SHA.

### 3. Merge

```bash
git merge --no-ff OCA/master -m "$(cat <<'EOF'
Merge branch 'OCA-master'

EOF
)"
```

### 4. Re-apply

- Resolve `Dockerfile` to Cetmix maintainer-tools URL + current SHA (`git -C ../cetmix-maintainer-tools rev-parse HEAD` when that repo is the source of truth).
- Re-insert `GITHUB_REPO_PRIVATE` in `config.py` / `environment.sample` and private URL selection in `github.py` if lost; keep OCA `from e` / other edits.
- Re-apply export allow/deny (`should_export_repo`); match OCA’s current `build_and_publish_metapackage_wheel` signature.
- Keep README Cetmix fork pointer and `docs/CETMIX_CUSTOMIZATIONS.md`.

### 5. Verify

```bash
git diff OCA/master HEAD --stat
grep GITHUB_REPO_PRIVATE src/oca_github_bot/config.py environment.sample
grep cetmix-maintainer-tools Dockerfile
! grep 'github.com/OCA/maintainer-tools@' Dockerfile
```

### 6–7. Tests and finish

`tox -e py312` or venv + `pytest --vcr-record=none`. Do not push to OCA. Commit/push Cetmix only if the user asks.

## Anti-patterns

- Do not push, PR, or commit to `OCA/oca-github-bot`.
- Do not replace the whole of `config.py` / `github.py` from `$PRESYNC` (loses OCA additions).
- Do not pin `OCA/maintainer-tools` in the Dockerfile.
- Do not force-push Cetmix `support_private_repo` / `master`.
