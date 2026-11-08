# Changelog

All notable changes to this project are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] — 2026-11-08

Behaviour-changing release: three defects found by running the tool against a
real DSH 0.2.0-rc.2 installation are fixed, and two findings that were silently
invisible now affect the verdict.

### Fixed

- **A multi-line plain scalar no longer drops the whole layer.** YAML folds
  continuation lines of a plain scalar onto one value, which is how DSH's own
  presets write a long `description:`. The reader rejected such a line as
  "unexpected indentation", so the entire `cordis.patch.yml` failed to parse and
  every row it declared vanished from the composed tree silently. The user layer
  of a real desktop profile hit this at line 76.
- **A flow scalar may contain a non-separating colon.** `tarball: file:F:/…` in
  a flow mapping was tokenised as `file` plus a stray `:`, producing "malformed
  flow mapping". A colon now separates only when a space or flow indicator
  follows it.
- **The launcher no longer reports a working Node as missing.** In a confined
  sandbox a child spawn with piped stdio fails with `EPERM`; `findNode`,
  `whichDsh` and the three `self-test` CLI checks treated that as "not found".
  They now retry once with file-descriptor redirection. `self-test` went from
  4 failed checks to 0 on this machine.
- **`--probe` no longer accuses a plugin whose peer the harness supplies.** DSH
  resolves a plugin's `@deepseek-ai/*` peers through the installation; an
  isolated probe resolves from the plugin's directory and cannot see those
  roots. The probe now receives the profile's module roots and classifies such
  an import failure as "not observed" (`UNKNOWN`, deferred) instead of
  `FAILED`.
- **The differential YAML test no longer reports precision loss as a
  divergence.** An integer beyond 2^53 is kept as an exact digit string by this
  reader and rounded to a float by `js-yaml`; the comparison now treats the two
  as the same scalar. `tests/yaml-diff.mjs` reports 307/307 agreeing files.

### Changed

- **A composition problem is now a report row.** A bundle or patch layer that
  cannot be read used to appear only in the Markdown report, while the verdict
  and the exit code said "no findings" — the exact silent failure this tool
  exists to catch. Each problem is now emitted as `type: composition`, `FAILED`,
  so it reaches the verdict, the exit code, the text report, the JSON and the
  JUnit export by construction. The duplicate Markdown "Composition problems"
  section is gone; the rows are the single source.
- **Discovery notes reach the text report.** A preset directory whose name is
  not a valid id is printed as a `NOTE` line by the text renderer as well (still
  not a finding, still not in the exit code).
- **An empty sweep says so.** When every composed row is a platform package, the
  default scope verifies nothing; that is now a `NOTE` ("nothing outside the
  platform was verified") instead of a bare `ALL PASS`, and the Markdown report
  prints the same discovery-gap rows under a `Notes` heading rather than in the
  findings table.
- `TOOL.version` and `package.json` are now `0.2.0`.

### Added

- `LICENSE` (MIT), `.gitignore`, `CHANGELOG.md`, and a GitHub Actions workflow
  that runs the sanity checks and the behavioural suite on Node 18/20/22.
- Tests for the folded scalar (including blank-line and comment behaviour), the
  flow colon rule, a dropped layer becoming a `FAILED` row with exit code 1, and
  the harness-supplied-peer classification. The suite is now 44 tests.

## [0.1.0] — 2026-11-07

First complete implementation of the plan.

- Zero-dependency ESM; no DSH import, no Cordis surface, no `--dump-config`.
- L0–L4 ladder for plugins and a separate ladder for agent-preset modes, with
  `PASS`, `PASS(mounted)`, `DEGRADED`, `FAILED` and first-class `UNKNOWN`.
- Seven artifact classes; capability-binding cross-check against the composed
  tree; duplicate-tool and dangling-provider detection.
- Budget with a probe-call ledger and a wall-clock ledger (20 / 180s).
- Optional isolated mount probe (`--probe`); baseline diff; Markdown, JSON,
  JUnit and text reports; `doctor`, `self-test`, `prompt`, `menu` subcommands.
- Built-in YAML reader with a differential test against `js-yaml`.
