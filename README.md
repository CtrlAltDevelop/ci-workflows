# ci-workflows

Reusable GitHub Actions checks for Python and Flutter projects. Each caller
project keeps one thin workflow file; everything complicated — lint, type
checks, tests, dependency and secret scanning — lives here once and is
versioned once.

**Checks only.** There is no deploy workflow here, and that is deliberate — see
*Why there is no deploy* below.

## The three workflows

| File | Does | Sees secrets |
|---|---|---|
| `python-backend.yml` | ruff, mypy, pytest (Postgres/Redis optional), `pip-audit` | no |
| `security.yml` | gitleaks over full history, CodeQL, Trivy, dependency-review | no |
| `flutter-android.yml` | analyze, test, and an APK when asked for one | keystore, only when signing |

All three run on GitHub's cloud runners. None of them touches a server, holds a
key, or has any route into a private network.

## Using it

Copy `examples/ci.yml` into a project's `.github/workflows/`, and
`examples/dependabot.yml` into its `.github/`. Delete the jobs and ecosystems
you don't need (a Python-only repo drops the `app` job and the `pub`/`gradle`
entries; a Flutter-only repo drops `backend` and `pip`/`docker`), then adjust
`working-directory`, the Flutter version and the codegen commands to match your
layout. That is the whole integration — about twenty lines per project, so a
fix here fixes every project at once.

Each workflow's inputs are documented as comments in its `on: workflow_call:`
block — see [`python-backend.yml`](.github/workflows/python-backend.yml),
[`flutter-android.yml`](.github/workflows/flutter-android.yml) and
[`security.yml`](.github/workflows/security.yml) for the full list.

## The moving tag, and how it bites

Callers reference `@v1`, which is resolved **when a run starts**. Change a
workflow here, forget to move the tag, and the next run silently uses the old
version — the failure then looks exactly like a bug you already fixed. This cost
two debugging sessions during bring-up.

Push and retag in one command, always:

```bash
git push origin main && git tag -f -a v1 -m "v1" && git push -f origin v1
```

For a change that breaks callers, cut `v2` and migrate projects one at a time
rather than moving `v1`.

## Access

This repository is public, so any repository can call these workflows with no
extra setup. If you fork it and keep the fork private, a caller in a different
private repository needs *Settings → Actions → General → Access* set to
**accessible from repositories owned by the user** on this repo — without it
every caller fails with "reusable workflow not found", which is not an obvious
error message.

## Why there is no deploy

There was a `deploy-stack.yml`: it built on the target server through a
self-hosted runner, tagged images by commit, ran migrations and health checks,
and rolled back by tag. It was removed in September 2026, and the reasons are
worth keeping so the idea is not reinvented without them:

- **Two repositories and one moving tag.** Every pipeline fix meant a commit
  here, a retag, and a re-trigger in the calling project. A meaningful share of
  the failures during bring-up were not real — they were runs that started
  before the tag moved.
- **A self-hosted runner is a standing shell inside the network.** It executes
  repository code as a user in the `docker` group, which on Linux is
  root-equivalent. That is a fair trade for real automation and a poor one for
  automation that is fighting you.
- **The gain was small at this size.** One developer per project, one test
  server. What a deploy pipeline really buys is safety when several people push
  and nobody is watching.

Deploying is now a command on the server — `git pull` and
`docker compose up -d --build`, wrapped in a `make` target. What that costs,
stated plainly: nothing prevents deploying untested code, there is no deploy
history, and rollback is a checkout rather than a click.

That is a size-appropriate trade, not a permanent verdict. When a project gets a
production server or a second person pushing, rebuilding the deploy half is the
right call — start from the git history of this repository, where the removed
workflow still lives.

## Security notes for the checks that remain

- Every workflow starts `permissions: {}`; each job asks for exactly what it
  needs.
- Android release signing requires a GitHub Environment. A job only reaches
  environment secrets by declaring the environment, and a caller job that
  `uses:` a reusable workflow cannot declare one — so the declaration sits
  inside `flutter-android.yml`, and `sign: true` without an environment fails on
  purpose. Pull-request builds are debug-signed and never see the key.
- Actions here are pinned to tags, not commit SHAs — a mutable reference, and
  the one real supply-chain gap left. Convert with `pinact run` and let
  Dependabot maintain them.

## License

[MIT](LICENSE).
