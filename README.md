# ci-workflows

One pipeline, four projects. Each project keeps two short files; everything that
is actually complicated lives here once and is versioned once.

Push this directory to `CtrlAltDevelop/ci-workflows` (**private**), then tag it:

```bash
git tag -a v1 -m "v1" && git push origin v1
```

Callers reference `@v1`. Moving that tag rolls every project forward at once —
which is the point of a shared repo, and also its main risk. See *Releasing*.

## The shape

```
feature branch ──PR──► develop     CI only. Cloud runners, no secret, no
                                    access to the office network.

develop ────────PR──► stage        the same checks, then build and start the
                                    stack on the project's own server, over a
                                    runner installed on that machine.

main                                production, when there is one.
```

No registry. Images are built on the machine that runs them and tagged with the
commit they came from — `business-os-backend:a1b2c3d` — which keeps the two
properties a registry would have given: a deploy names an exact build, and
rolling back is deploying an older tag rather than rebuilding and hoping. It
also stays free: private packages on GHCR have a 500 MB quota that one Django
image erases.

## The four workflows

| File | Does | Sees secrets |
|---|---|---|
| `python-backend.yml` | ruff, mypy, pytest (Postgres/Redis optional), `pip-audit` | no |
| `security.yml` | gitleaks over full history, CodeQL, Trivy, dependency-review | no |
| `flutter-android.yml` | analyze, test, APK/AAB — signed only behind an environment | keystore, gated |
| `deploy-stack.yml` | build on the server, compose up, migrate, health, trim, roll back | server-side only |

## Onboarding a project

1. Copy `examples/ci.yml`, `examples/stage-deploy.yml` and
   `examples/dependabot.yml` into the project's `.github/`.
2. Adjust `working-directory`, the `images` map, `project-name`, `project-dir`,
   the ports in `health-checks`, and the runner labels.
3. Create the `stage` environment in the project's repository settings.
4. Install a runner on that project's server with matching labels.

Roughly twenty lines of project-specific YAML. Nothing else is copied, so a fix
here fixes all four.

Several projects can share one machine: give each its own runner labels, its own
`docker compose -p` project name, and its own port range.

---

## Security, and why each piece is there

**No inbound port.** The servers are on `172.20.x.x`. Instead of exposing SSH
and storing a private key in Secrets, each server runs a runner that dials out
to GitHub. Nothing to port-forward, nothing to brute-force, no key to rotate.

**Least privilege by default.** Every workflow starts `permissions: {}` and each
job asks for exactly what it needs.

**Fork code never reaches a self-hosted runner.** Set *Fork pull request
workflows → require approval for all outside collaborators*, and never use
`pull_request_target`. `deploy-stack.yml` re-checks it as a backstop.

**Secrets stay on the server.** The environment file lives in `/opt/<project>/`,
outside the checkout, copied in at deploy and deleted afterwards. A deploy
cannot overwrite a secret and a wiped workspace cannot lose one.

**The keystore is gated.** Android release signing requires an environment. A
job only reaches environment secrets by declaring the environment, and a caller
job that `uses:` a reusable workflow cannot declare one — so the declaration
sits inside `flutter-android.yml`, and `sign: true` without an environment fails
on purpose. Pull-request builds are debug-signed and see nothing.

**The branch is the real gate.** Anyone who can push to the deploy branch can
run code on the server, because that is what a deploy is. Protect `develop` and
`stage` with a ruleset requiring the CI checks, and put a passkey on the GitHub
account. For a solo project this is the control that matters; a required
reviewer you approve yourself is a confirmation dialog, not a check.

**Findings go where they are read.** Trivy and CodeQL upload SARIF to the
security tab rather than failing quietly in a log.

### Two things still open

- **Android is debug-signed.** Before `sign: true` does anything, generate an
  upload keystore, wire `key.properties` into the Gradle signing config, and
  enable Play App Signing so the distribution key is Google's problem rather
  than a base64 blob in a secret.
- **Actions are pinned to tags, not commit SHAs.** A tag is mutable, which is
  the supply-chain hole this file otherwise argues against. Convert them once
  with `pinact run` (or `ratchet pin`) and let Dependabot maintain them — the
  SHAs are not something to write by hand.

## One-time account settings

- Actions → **Allow select actions**, and *require actions pinned to a
  full-length commit SHA* once the pinning pass above is done.
- Default `GITHUB_TOKEN` permissions → **read only**.
- Fork PR workflows → **require approval from all outside collaborators**.
- Secret scanning with **push protection**, on every repository.

## Releasing

`@v1` is a moving tag, so a bad commit here breaks every pipeline at once. Work
on `main`, and move `v1` only after the change has run green on one project. For
a change that breaks callers, cut `v2` and migrate the projects one at a time.
