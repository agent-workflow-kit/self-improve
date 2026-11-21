# self-improve: implementation plan

This plan implements [`interface.md`](interface.md) (the single normative source) and [`design.md`](design.md). Each stage is expanded into bite-sized test-first steps when its issues are picked up; this file holds the goals, success criteria and tests per stage, not code.

## Conventions for every stage

- **Gateway boundary.** Every subprocess, clock, home-directory and terminal access goes through `src/self_improve/gateway.py`, so tests substitute fakes. Nothing else imports `subprocess`, `os.getlogin`, `pwd` or opens `/dev/tty`.
- **Fakes and pinned sibling schemas.** `tests/fakebin/` holds fake `claude-session` and `session-audit` executables that print documents validated against the pinned copies in `schemas/pinned/` (never against the live tools in CI). A fake can be scripted to fail at each step (launch, adopt, status, resume, forget).
- **Process tests and live tests.** Tests marked `process` run real file-system races and crashes (kill points, symlink swaps, a second process) with no model. Tests marked `live` use the real `claude-session`, `session-audit` and a cheap model, are deselected by default and never run in CI.
- **Golden files.** Every `--json` output, the review view and every hook payload has a golden file and a schema validation of the real output; the generator in the design workflow is not a runtime dependency.
- **Spike regressions.** Each behavior verified in the design spikes becomes a test: pairs apply exactly, a session sees only staged inputs, an injected candidate changes nothing outside the artifact, a headless resume keeps the artifact contract, the exchange rename closes the verify-to-replace race for rename-style writers, symlink, hard-link and mode cases, the escape round trip.
- **`make verify`** (ruff, format, pyright, pytest, markdown lint) passes before every commit. Interface and schemas change together with the code; the registry row in `awkit-docs` is updated when the interface changes.
- **Safety rules in code review.** Text from retros, models and files never reaches argv or env; the tool never runs `git apply`, `patch` or a shell on model output; one YAML loader (`safe_load`) and one sanitizer exist; nothing writes a user file except `apply.py`.

## Module map

| Module | Stage | Responsibility |
|---|---|---|
| `gateway.py` | 0 | The only place that runs subprocesses, reads the home directory, the clock and the terminal. |
| `cli.py` | 0 | argparse commands, global flags, exit-code mapping, JSON envelopes. |
| `config.py` | 0 | Load and validate config (ownership, drop-ins, safe YAML, safety-key rules). |
| `store.py` | 0 | SQLite proposals, runs, applies rows, scan state, leases, additive migrations. |
| `audit.py`, `log.py` | 0 | `audit.jsonl` (attempted and outcome pairs) and `log.jsonl`. |
| `render.py` | 0, 2 | The injective escape, the review view and the diff. |
| `targets.py` | 1 | Safe open walk, target checks, tiers and the glob dialect, `targets` resolution, plugin-cache refusal. |
| `secrets.py` | 1 | Secret shapes and masked environment values. |
| `parse.py` | 2 | Bounded JSON, YAML and TOML parsing. |
| `edit.py` | 2 | Pair rules, character and line-ending rules, markdown line forms, the bounded changed-line count, applying pairs to bytes. |
| `retro.py` | 3 | Read retros through `session-audit` (show and list), supersession, candidate snapshot. |
| `draft.py` | 4, 5 | Skill preflight, launch plan check, staging, prompt, start and adopt, outcome mapping, recovery, `edit` resume. |
| `artifact.py` | 4 | Strict artifact reading and validation. |
| `apply.py` | 6 | The write sequence, pre-images, apply recovery, `undo`. |
| `hooks.py` | 6, 7 | Hook runner and payload builders. |
| `scan.py` | 7 | Scan loop, budgets, lease, wedge report. |
| `cron.py` | 7 | Managed crontab block. |
| `plugin/` | 4 | The bundled skill, marker file and profile. |

## Stage 0: Foundations

**Goal**: A package that installs, prints `--version` from metadata, loads a safe config, owns the state directory and database, writes audit and log records, and maps every error to the documented exit code and envelope.
**Success Criteria**: `validate` runs offline and prints the documented document; the config safety checks refuse a symlink, a foreign owner and group- or other-writable files; the store migrates additively and a newer database exits 20 `store_newer`; the audit pair writer refuses to proceed when the first append fails; fakes for both siblings exist.
**Tests**: golden `--json` for `validate` and the error envelope; config safety cases (including drop-ins and a symlinked ancestor); migration test with a future schema; audit-pair failure test; fake claude-session and session-audit scripted from pinned schemas.
**Status**: Not Started

## Stage 1: Targets, tiers and secrets

**Goal**: Turn a path into a checked target and a tier, and refuse files that must never reach a model.
**Success Criteria**: The safe open walk refuses symlinked ancestors, FIFOs, hard links, foreign owners and special mode bits; tiers follow the documented table and glob dialect (case rules included); plugin-cache and marketplace targets are refused with the source named; `targets` entries resolve candidates exactly as documented and can never lower a tier; secret shapes and masked values refuse a target or context file before staging; `resolve` works.
**Tests**: path corpus for the glob dialect and tier decisions; the symlink, FIFO, hard-link and ancestor cases as process tests; secret corpus including the whole-word `KEY`, `KEYS`, `AUTH` and `AUTHORIZATION` rule and the full-path-match exception; golden outputs for refusals.
**Status**: Not Started

## Stage 2: Edit engine and review view

**Goal**: Validate and apply find/replace pairs on bytes, and render exactly what a person will confirm.
**Success Criteria**: Every pair rule, character rule, line-ending rule and size cap behaves as documented; the markdown line-form rule refuses frontmatter and `!`/`@` constructs outside the closed forms; the changed-line count is bounded and linear, the cap is `max(6, ratio * lines)`; JSON, YAML and TOML targets still parse under the bounded parsers; the escape round-trips every string and the diff reconstructs the result exactly.
**Tests**: the spike corpus ported (find/replace cases) plus new cases for each rule; property tests for the escape and for diff reconstruction; timing tests at the line caps; a lint test that only the safe YAML loader is used.
**Status**: Not Started

## Stage 3: Retros, proposals and the read commands

**Goal**: Read retros, store proposals through the lifecycle, and offer the read and close commands.
**Success Criteria**: `superseded` is read from `retro list` rows; dedupe and `--again` follow the documented lookup with no unique-index conflicts; the candidate snapshot (finding text) is stored so later commands never read the retro; `list`, `show`, `reject` and `prune` behave per the lifecycle and retention tables; proposals from `propose` that stop before a model run are stored `unresolved` or `refused`.
**Tests**: lifecycle transition table test (every allowed and refused transition); dedupe and `--again` pseudocode as a table-driven test including a retro `--again` re-run; retention and prune; golden JSON for `list` and `show`.
**Status**: Not Started

## Stage 4: Drafting

**Goal**: Run one isolated headless draft and store a validated proposal.
**Success Criteria**: The skill preflight, launch plan check (first draft assertions, including the `file:<sha256 prefix>` profile label check), staging, prompt and start-and-adopt sequence work with the fakes and, in the live suite, with a real cheap model; the artifact validator enforces every documented check in order; outcome mapping, the sandbox check, cleanup and recovery by stored session id behave per the tables; the bundled skill, marker and profile ship in the wheel.
**Tests**: scripted fakes for each outcome row; artifact validator negative corpus; recovery after a kill at each point; packaging test for the hidden plugin directory; live: the real round trip, an injected candidate, a vague candidate giving `no_change` (this stage's live gate is the 0.x release gate).
**Status**: Not Started

## Stage 5: Revising drafts

**Goal**: `edit` resumes the drafting session with a person's instruction and replaces the draft only on success.
**Success Criteria**: The resume passes the full flag set again and the launch plan check accepts only the documented carried origins; a failed revision leaves the proposal unchanged with the failure recorded; revisions are capped; a forgotten session exits `session_gone`; the live target is re-staged and re-hashed.
**Tests**: fake resume with every failure; the origins fail-closed test; the profile label test (a path, a wrong prefix and a different profile's hash all fail); a live headless resume keeping the artifact contract.
**Status**: Not Started

## Stage 6: Accept, undo and audit

**Goal**: The only writes to user files, with confirmation, recovery and audit.
**Success Criteria**: The accept and undo orders hold (state re-check after the lease, human confirmation on `/dev/tty` with the foreground check, `--yes` only with a `human:` actor and the exact plan hash, `--allow-sensitive` terminal-only); the write sequence uses the lock, no-follow open, hash check, same-directory temporary file and the exchange rename with its post-check (re-verify plus replace where the exchange is unavailable, with the warning); pre-images are stored; apply recovery decides by hash comparison and is idempotent; every write has an `attempted` and an outcome record; `pre-accept` and `post-accept` hooks work.
**Tests**: the apply race stress test and all symlink, hard-link, mode and exchange cases (process); a crash at each of the documented kill points, then recovery; two terminals racing accept, reject and undo; audit pairing; golden review view; the live apply race.
**Status**: Not Started

## Stage 7: Scan, cron, hooks and the harness surface

**Goal**: Unattended drafting that can never accept, plus the reads a harness needs.
**Success Criteria**: `scan` drafts only ordinary-tier targets, exits 12 `sandbox_lowered` unless the sandbox is required, honors every budget, deadline and lease rule and reports a wedge; `cron install|uninstall` writes and removes the managed block (golden, with the end line, interval and env-file flags); `pre-draft`, `post-attention` and the other hooks fire with payloads that carry no text and validate against their schemas; `validate --probe`, `export` and the attention rows work.
**Tests**: scan loop table tests with budgets and a wedged lease; cron golden and malformed-block refusal; hook payload contract tests through the notice-route payload rules; a drop-in hook list that appends.
**Status**: Not Started

## Stage 8: Live suite and release readiness

**Goal**: Prove the verified behaviors against the real tools and close the open live checks.
**Success Criteria**: The live suite passes against current `claude-session` and `session-audit`; the frontmatter keys and `!`/`@` syntax that Claude Code acts on are checked and the line-form rule is adjusted if needed; the resume dry-run origin keys and the bytes claude-session hashes into `file:<sha256 prefix>` are recorded; the macOS fallback is documented as untested; the README and interface match the implementation.
**Tests**: the full `live` marker set; a manual checklist for macOS and for another-owner files.
**Status**: Not Started

## Version roadmap

- **0.x**: stages 0 to 8, interface 0.1.0-design becomes the 0.1.0 contract. No automatic apply, no TUI, headless drafting only.
- **1.0** requires: the live suite green against released siblings; the frontmatter and body-line live check done; a person has run propose, edit, accept and undo on real targets; the awkit adapter for self-improve exists or its notes are posted.
- **1.x** (additive): `--mode interactive` revising through agent-spawn; a `browse` TUI; proposals that create new files (a new skill); automatic apply of low-risk targets behind a gatectl policy in shadow first; retro candidate classes added by session-audit.
- **2.0**: nothing planned.
