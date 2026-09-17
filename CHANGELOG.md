# Changelog

Reusable workflows, so a release here is a tag other repositories point at.
Callers pin a tag rather than a branch; see *The moving tag, and how it bites*
in the [README](README.md) for which tag to pin and when it moves.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the versions follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
— where "breaking" means a caller has to change its `with:` block.

## [1.1.0] — 2026-09-17

### Added

- `dart-package.yml`: the gate a published package needs, rather than the app
  checks in `flutter-android.yml`, which build an APK a package does not have.
  Formatting, `--fatal-infos` analysis, tests over both ends of the declared
  constraints, the `example/` that pana never looks at, a CHANGELOG entry for
  the version in `pubspec.yaml`, and the pub.dev score at a zero-point
  threshold. Failed goldens upload as an artifact, because a red golden run
  with nothing to look at tells you nothing you can act on.

### Changed

- The Python and Flutter workflows carry no project-specific defaults any more,
  so the repository could be made public without leaking one project's layout
  into everybody else's pipeline.

## [1.0.0] — 2026-09-08

### Added

- `python-backend.yml`: ruff, mypy, pytest with optional Postgres and Redis
  services, and `pip-audit`.
- `security.yml`: gitleaks over full history, CodeQL, Trivy, and
  dependency-review.
- `flutter-android.yml`: analyze, test, and an APK only when a caller asks for
  one — verifying a branch does not need a build artifact nobody installs.
- An advisory-ignore input, for the case where an advisory has no published fix
  and the alternative is a permanently red pipeline.
- A lint-report mode, so a project can see lint without gating on it while it
  works through the backlog.

### Fixed

- The security gate runs on a private repository without GHAS instead of
  failing on the CodeQL step.
- `permissions:` is spelled literally; GitHub does not evaluate an input there.
- pytest gets the packages the suite actually imports, and no longer has
  `DJANGO_DEBUG` forced on it.
- `trivy-action` is pinned to a tag that exists.
- A green run no longer uploads the failure artifacts that made it look red.
- Dropped the `pub audit` step: pub has no advisory command that exits
  non-zero, so the step could only ever pass.

### Changed

- Moved off the actions still running on Node 20 — `setup-java` v5 and a
  gitleaks release on Node 24.

[1.1.0]: https://github.com/CtrlAltDevelop/ci-workflows/releases/tag/v1.1.0
[1.0.0]: https://github.com/CtrlAltDevelop/ci-workflows/releases/tag/v1.0.0
