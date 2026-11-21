# self-improve: design

Status: designed (interface 0.1.0-design). The normative rules are in [`interface.md`](interface.md); this document gives the reasons, the decisions and the internals. It links to interface headings and never restates their tables.

Section references (§N) point to `../awkit-docs/architecture.md` unless marked "design guide" (`../awkit-docs/design-guide.md`).

## Role and layer

Turn the candidates of a stored `session-audit` retro into proposals: exact edits to one local file (a skill, an instructions file, a config file, a permission profile), drafted by an isolated model session, reviewed as a tool-computed diff and written only by a human-confirmed `accept`. Layer 3 (workflow), post-v1, fully standalone at a terminal. Skill-wrapper tool (§5.5, design guide §11); default slot `improve`, bundled as `self-improve:improve` in `plugin/`.

## Uses

| Tool | Binding | What for |
|---|---|---|
| session-audit | A (hard) | `retro show --json` to read a stored retro and `retro list --json` for the `superseded` flag, which is read from the `retro list` row of the same id ([interface: Order of checks](interface.md#order-of-checks)). Only `state: done`, `superseded: false` rows. The pinned minimum is stated once in the interface ([Draft launch](interface.md#draft-launch), [Upgrades](interface.md#upgrades)) and is set when session-audit ships `retro show --json`. |
| claude-session | A (hard) | `start`, the adopting `exec`, `status`, `resume`, `stop`, `forget --purge` for drafting runs ([interface: Draft launch](interface.md#draft-launch)). The pinned minimum is stated once in the interface ([Draft launch](interface.md#draft-launch), [Upgrades](interface.md#upgrades)); it needs `--tools`, `--no-implicit-done`, `forget --purge` and a `report` that works in the sandbox after a normal install. |

Not used: agent-spawn, wtctl, termctl, memoctl, any notifier, the forge and ticket tools, `awkit`. Consumers: none in the architecture; awkit reads `list --json`, `validate --json`, `audit.jsonl` and the cron block through its adapter (a later awkit issue).

## Decisions

| ID | Decision | Rationale |
|---|---|---|
| D1 | One proposal per retro candidate and target; stored, reviewable, pinned by a plan hash. The dedupe key uses the retro id, not the retro content key (user decision on review pass 1). | Review in your own time; the harness can show "waiting" without applying; automation can draft but never accept. A re-run retro (`retro run --again`) keeps its content key but gets new candidate lists whose ids mean different things, so the content key would return an unrelated proposal. See [Proposal lifecycle](interface.md#proposal-lifecycle). |
| D2 | The first draft is headless; `edit` resumes the same session, re-stages the inputs with a fresh run token for every run, and passes the skill and sandbox explicitly. A revision is a run on a `ready` or `no_change` proposal and never moves it to `failed`. | Spike A: a headless resume keeps the artifact contract, costs about a cent and 20 s, and needs no agent-spawn or tmux. Resume does not carry `--skill`, and inputs given on resume land in a new `inputs/<run>/`, so the tool stages them again. A failed revision leaves the proposal as it was and records `last_revision_error`. See [edit](interface.md#edit) and [Slot contract](interface.md#slot-contract). |
| D3 | The target is always named by a person or the user's own config, never by the model; no scan of directories. | A model-chosen path is the main injection channel of an edit tool. See [Target resolution](interface.md#target-resolution). |
| D4 | Three tiers decided by the path and the execute bit: ordinary, sensitive (extra terminal-only flag), refused. A `targets` entry can raise a tier (`sensitive: true`) and never lowers one. | A permission-profile or hook edit can widen what later trusted sessions may do; review-watch's off-limits default covers the normal targets and cannot be reused. A dotfiles real path such as a settings file stays sensitive whatever the config says. See [Tiers](interface.md#tiers). |
| D5 | The drafting session is isolated (third-party-reading, sandbox required, no setting sources, MCP none) and receives a staged copy of the target. | The tool applies edits, so isolation costs nothing; spike A: outside reads fail 4/4 and no profile lifts the rule, staged input worked 17/17. See [Draft launch](interface.md#draft-launch). |
| D6 | One edit kind: exact find/replace pairs. No unified diff, no full-file output. | Spike A: smallest, cheapest, every `old` unique; the diff scope drifted once. Spike B: own applier has a tiny surface, `git apply` is not safe to call. See [Edit rules](interface.md#edit-rules). |
| D7 | `accept` is accept and apply in one step, confirmed on `/dev/tty` or with `--yes --expect-plan` and a `human:` actor; sensitive targets need a terminal-only flag. | Fewer states; keeps a human on edits that later sessions will load. See [Accept order](interface.md#accept-order). |
| D8 | Apply sequence: lock, no-follow open, hash check, same-directory temp, `RENAME_EXCHANGE` with a post-check. Pre-image copy under the state directory; `undo` refuses if the target changed. | Spike B: symlink and hard-link targets break atomic replace, a git-based undo destroys work, the exchange closes the rename race. |
| D9 | Manual `propose` plus an opt-in `scan` that polls stored retros; no `post-retro` hook driver. | Polling loses nothing; a hook is best-effort and at-most-once. See [scan](interface.md#scan). |
| D10 | Every write to a user file is audited with an `attempted` and an outcome record; drafting is only logged. | session-audit's and awkit's notes ask for audited edits; awkit pairs them by `attempt_id` without a special adapter. See [Audit](interface.md#audit). |
| D11 | Hooks `pre-draft`, `pre-accept`, `post-accept`, `post-attention`; no `post-pending`. | Gates and notices need them; proposals are the pending store. See [Hooks](interface.md#hooks). |
| D12 | `--mode headless` only in 0.x; bare skill names refused; no TUI; home-only config. | Recorded deviations; see [Deviations from the design guide](interface.md#deviations-from-the-design-guide). |
| D13 | Plugin-cache and marketplace targets are refused with the plugin, version and source named. | An update overwrites the edit; the right place is the plugin's source repository. See [Target resolution](interface.md#target-resolution). |
| D14 | No creation of new files and no `git`-based apply or undo in 0.x. | Smaller surface; a new-skill proposal is a 1.x item. |
| D15 | Before staging, the target and every context file are scanned for secret shapes and masked environment values; a hit refuses the target (`contains_secret`) and `new` text is checked with the same function (user decision on review pass 1). | A staged copy goes to the model provider and stays in the session transcript, which `forget --purge` does not remove. A settings or MCP file that holds a literal token cannot be a target until the token moves out. See [Edit rules](interface.md#edit-rules) and [Safety](interface.md#safety). |
| D16 | `accept` and `undo` record an intent row before the `attempted` audit record; a crash is settled at the next state-changing command by comparing the target's hash with the recorded before and after hashes. Recovery fires no hooks. | One source of recovery, no scan of the audit file, hooks stay at most once. See [Accept order](interface.md#accept-order) and [Concurrency](interface.md#concurrency). |
| D17 | In a markdown target an edit may not change frontmatter keys outside `name`, `description` and `argument-hint`, and may not add `!` command lines or `@` imports. A structured target gets the `trust_key_changed` warning. | The ordinary tier is defined as "runs no code"; frontmatter hooks and command lines make a markdown file run code. An allowlist holds whichever keys Claude Code acts on. See [Edit rules](interface.md#edit-rules). |
| D18 | Paths are normalized lexically and opened by a no-follow walk from `/`; the leaf must be a regular file; ancestors are compared by device and inode with the refused and sensitive roots. | Closes symlinked ancestors, FIFOs and bind mounts. See [Target resolution](interface.md#target-resolution) and [Tiers](interface.md#tiers). |
| D19 | Config is read under the password-database home with the symlink and ownership checks of session-audit; drop-ins merge in lexical order with `hooks.<name>` lists appended. | The config selects hook commands and the files the tool writes. The guide and awkit wire hooks through a drop-in, so lists must append. See [Config](interface.md#config). |
| D20 | Setting sources are always empty for the drafting run; there is no `draft.setting_sources` key. | A third-party-reading run must load none (claude-session fails the launch otherwise). See [Deviations from the design guide](interface.md#deviations-from-the-design-guide). |
| D21 | Scan keeps per-candidate bookkeeping and three budgets (`scan.max_per_run`, `scan.max_per_day`, `scan.max_per_retro`); `scan.max_per_day` counts inside `limits.drafts_per_day`. | Bounded cost, and manual drafts are never starved. See [scan](interface.md#scan). |
| D22 | One condition table fixes exit code, envelope `code`, reason and proposal state; `propose` handles every selected candidate and prints a result object on non-zero. | The schemas derive from one table. See [Exit codes](interface.md#exit-codes) and [Refusal reasons](interface.md#refusal-reasons). |
| D23 | The confirmation and `--yes` are not a boundary against a same-user caller; the `accept --dry-run --json` document carries hashes and counts and no text, and a dashboard hands over the terminal. | A process with a pty answers the prompt, and `--actor` is a claim. The boundary is that the drafting session cannot run the tool. See [Safety](interface.md#safety) and [`--json` outputs](interface.md#--json-outputs). |
| D24 | `undo` verifies the pre-image copy against the stored hash first and takes `--allow-sensitive` under the same rule as `accept`. | One rule for both write commands; a corrupt copy is refused before anything is written. See [undo](interface.md#undo). |
| D25 | Retention is per state, and a draft session is kept only while the proposal is `ready` or `no_change`; an open proposal expires after `retention.open`. | Bounds the store and the retained sessions. See [Proposal lifecycle](interface.md#proposal-lifecycle) and [Retention](interface.md#retention). |

### Decisions changed since approval

- Sensitive targets and staging: files holding a secret are not staged (D15).
- Size caps: the changed-line ratio is `invalid_edit` at draft time and `too_large` at accept time; `display.max_lines` defaults to 600; the other caps keep their approved defaults.
- Audit results: there is no `refused` audit result, because a refusal writes nothing.
- Dedupe: the key uses the retro id, not the retro content key (D1).
- Labels: `repo=` is a caller label only.
- Tiers: "ordinary means runs no code" is enforced by the frontmatter and import rule (D17), not changed.
- Reference files of a skill target and the per-retro scan cap are now implemented (staged `references/` files, `scan.max_per_retro`).

## Resolved open topics

From architecture §20 and the seed: *what is audited, when, output, how improvements are applied.*

- **Inputs.** A stored retro's candidates through `retro show --json` (binding A). Candidate text is untrusted; the tool uses the tool-written `risk` and `sessions`, never a risk read from text.
- **When.** On demand (`propose`) and, only when enabled, by `scan`; never at session end.
- **Output.** A stored proposal with a tool-computed unified diff; `list --json` rows for harness attention.
- **How applied.** Only by `accept` after a human confirmation; the tier decides how loud; no automatic apply in 0.x (architecture §12.1 keeps auto-apply of low-risk targets as a post-v1 option, still unscheduled and still a non-goal of v1 in §19.2).
- **Off-limits.** The refused tier (credentials, the tool's own config and state, claude-session's state root) and the sensitive tier, by path; review-watch's off-limits list is not reused because it covers the normal targets.

## Live-verification table

| Behavior relied on | Status | Where |
|---|---|---|
| Find/replace pairs validate and apply; `old` unique; `no_change` is used for vague or oversized candidates | verified (24 haiku runs) | spike A, becomes a `live` test |
| A session reads only staged inputs; outside reads fail under every profile tried | verified | spike A Q2 |
| Injection in candidate text was not followed (2 of 2) | verified, small n | spike A Q3, a `live` regression |
| Headless `resume --prompt-file` keeps the artifact contract; a tainted session may resume in the same kind | verified | spike A Q5 |
| Exchange rename closes the verify-to-replace race for rename-style writers; an open-descriptor writer can still lose a save | verified on Linux ext4 | spike B Q2, process test |
| Symlink, hard link, mode, `EXDEV` and read-only-bit behavior of atomic replace | verified | spike B Q1 |
| The injective ASCII escape round-trips 3000 of 3000 random edits | verified with the spike's alphabet | spike B Q4; re-test with the 0.x alphabet (every backslash written `\\`) as a unit test, including the colliding cases |
| macOS fallback (no `RENAME_EXCHANGE`); 9p and network mounts; another-owner file | not verified | marked in Platform notes; conservative default (refuse or fall back and warn) |
| Headless `resume` with `--input-file` and `--skill` re-invokes the skill and keeps the artifact contract | not verified | becomes a `live` test |
| Frontmatter keys and `!`/`@` syntax Claude Code acts on | needs a live check | the allowlist rule in [Edit rules](interface.md#edit-rules) is conservative whatever the exact key set is; a new mechanism would pass as ordinary ([Residual risks](interface.md#residual-risks)) |
| A drafting session's behavior on a very large skill or a heavily nested config | not verified | cap by size; first live suite checks |
| Retro candidate quality on real transcripts | not verified | live checklist issue |

## Modules (planned)

| Module | Responsibility |
|---|---|
| `cli.py` | argparse commands, global flags, exit-code mapping, JSON envelopes. |
| `config.py` | Load and validate config with ownership and symlink checks, drop-in merge (lists under `hooks` append), safety-key rules; the password-database home is read by one gateway function that tests patch. |
| `store.py` | SQLite proposals, runs, intent rows, scan bookkeeping, draft slots, leases, migrations. |
| `retro.py` | Read and validate retros through `session-audit` (gateway). |
| `targets.py` | Path normalization, target resolution, the safe open walk, target checks, tier matching (glob dialect and root identity compare). |
| `secrets.py` | Secret shapes and masked environment values, carried as this tool's own code from session-audit's redaction rule (a change there must be mirrored); used on staged text and on `new`. |
| `parse.py` | The safe parsers for JSON, YAML and TOML with the node and depth limits, and the frontmatter reader. |
| `draft.py` | Preflight, launch plan check, staging, prompt templates (`prompts/draft.md`, `prompts/revise.md`), start/adopt, resume, outcome mapping, draft and edit recovery. |
| `scan.py` | Per-candidate scan bookkeeping and budgets. |
| `artifact.py` | Strict artifact reading and validation. |
| `edit.py` | Pair rules, character rules, apply to bytes, result checks, changed-line share, the frontmatter and import rule. |
| `apply.py` | The write sequence, pre-image copies, `undo`, apply recovery by hash comparison. |
| `render.py` | The review view, diff, injective escape. |
| `audit.py`, `log.py` | `audit.jsonl` and `log.jsonl` writers. |
| `hooks.py` | Hook runner, payload builders. |
| `cron.py` | Block build, install, uninstall. |
| `gateway.py` | The only place that runs subprocesses (claude-session, session-audit, crontab, hooks). |

Dependencies: Python 3.12 or later and the standard library (`sqlite3`, `difflib`, `hashlib`, `unicodedata`, `ctypes` for `renameat2`); YAML through `pyyaml` (`safe_load` only; a lint test bans every other loader); schema models through Pydantic only where the guide requires. No dependency for the diff, the applier or the escape.

## Data model

SQLite (`state.db`, WAL):

- `proposals`: id, `retro_id`, `retro_key`, `candidate_id`, candidate snapshot (text fields, untrusted), `dedupe_key` (`sha256(retro_id, candidate_id, target_path or "")`, with a unique index; `retro_key` is display data), `state`, `reason`, `last_revision_error`, target path, tier, `pre_sha256`, `post_sha256`, `source` (flag, config, resolve; null while `unresolved`), `revision` (0 while `unresolved`), `outcome`, `rationale`, `replacements` JSON, `plan_sha256`, labels JSON (array of `k=v`), `csession_id`, `cost_usd`, `num_turns`, timestamps.
- `runs`: proposal id, revision, run number, `kind` (`draft` or `edit`), `run_token`, `csession_id`, state, outcome, end reason, artifact sha256.
- `applies` (intent rows): `attempt_id`, proposal id, action (`accept` or `undo`), target path, parent, `pre_sha256`, `post_sha256`, temporary name, pre-image copy path, `audit_offset`, `phase` (`planned` or `writing`), holder pid and start time, `created_at`.
- `draft_slots`: UTC day, source (`manual` or `scan`), count; a slot is reserved at row insert or at the start of an edit run and never released.
- `scan_candidates`: retro id, candidate id, status, updated time; `scanned_retros`: retro id, scanned time, counts; `scan_state`: consecutive failures and first and last failure times.
- `leases`: object id (a proposal id, or `scan`), holder pid and start time, expiry.
- `meta`: schema version.

Intent rows are the only source of apply recovery. Migrations are additive. A newer database exits 20 `store_newer`.

## Concurrency and recovery

Per-proposal lease (30 s renewal, 120 s expiry, compare-and-set takeover), per-target flock, `BEGIN IMMEDIATE` with a dedupe re-check and a daily-limit reservation before any hook, and no lease or lock held while waiting for a person. Drafting and edit runs are recovered by the stored `csession_id` and run number; `accept` and `undo` are recovered from their intent rows by hash comparison, under the target lock, at the start of every state-changing command and never under `--dry-run`. See [Concurrency](interface.md#concurrency), [Accept order](interface.md#accept-order) and [Draft launch](interface.md#draft-launch).

## Error handling

Fail fast with a closed `reason`. A refusal writes no user file and is not audited (a refused target is stored as a `refused` proposal); a failure after the model ran stores the proposal in `failed` with the reason so a person can see why. A failure of cleanup never changes a result. The tool never exits 1 on a validation failure; unexpected exceptions in artifact reading become `artifact_invalid`.

## Test strategy

- All subprocess calls go through `gateway.py`; tests use fakes for claude-session and session-audit with pinned copies of their schemas under `schemas/pinned/` (session-audit's `retro show` and `retro list`, and claude-session's `start`, `status`, `outcome`, `exec-dry-run` and `resume` shapes).
- Golden files for every `--json` output and the review view; schema validation of real output; a differential test that the injective escape round-trips.
- Safe-parser tests: duplicate JSON keys, `NaN`, deep nesting, YAML alias expansion over the node limit, `RecursionError` counted as unparsable, and a lint test that bans `yaml.load`.
- The escape round trip is a unit test of the exact 0.x alphabet, including the colliding cases (a literal backslash followed by `t`, `x0a` or `s`), run as a differential test against the decoder.
- The password-database home is read by one gateway function that tests patch.
- Contract tests pipe every hook payload through `notice-route test --stdin-json`; a test runs a drop-in hook list plus a user hook and checks both run.
- Property tests for the pair rules and the character rules; the spike B corpus (213 cases, in the three formats it covered) becomes the starting set for the pair applier.
- Process tests for the apply sequence: a symlink swap, a hard link added, an edit between plan and apply, two instances racing, `undo` refusals; marked `process`. Crash-recovery process tests kill the tool at each of the five points (after the intent insert, after `attempted`, after the pre-image copy, after the exchange, after the outcome record) and check the state, the audit records and that no hook fires. FIFO and symlinked-ancestor tests check that a FIFO target never blocks and that an ancestor which is a refused root is caught by identity.
- `live` tests (never in CI): the real drafting round trip with a cheap model, the injection candidate, a vague candidate giving `no_change`, a headless `edit`, the skill preflight and sandbox check, the apply race. Each spike result becomes a regression.
- A packaging test that the wheel contains `.claude-plugin/plugin.json`, the skill, the marker file and the profile.
- CI is Linux only (Python 3.12 and 3.13). macOS is a documented, untested fallback.

## Cross-tool facts to carry

- session-audit owns the candidate vocabulary and the retro; self-improve reads the pinned fields only and ignores additions.
- claude-session accepts any artifact as `done`, so this tool checks `report.implicit`, the artifact name, size and hash, identity echoes and the marker itself.
- A profile for every session that reads untrusted text (not only sandboxed ones) must deny `accept` and `undo` and writes to this tool's config and state directories (note for claude-session and agent-spawn).
- awkit's adapter reads `list --json` (attention: `ready` and recent `failed`), `validate --json`, `audit.jsonl` (paired by `attempt_id`) and the cron block, and should hand over the terminal for `accept` and `undo` (`terminal: true`), so the tool's own review view shows the diff; a recipe action with `--yes` must be limited to the ordinary tier and show the dry-run `summary` counts. `list --attention --json` returns `ready` and recent `failed` rows, and `awkit audit --label` does not match this tool's records because the audit `target` is a path. The sensitive tier is terminal-only by design.
