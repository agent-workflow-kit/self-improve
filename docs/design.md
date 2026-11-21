# self-improve: design

Status: designed (interface 0.1.0-design). The normative rules are in [`interface.md`](interface.md); this document gives the reasons, the decisions and the internals. It links to interface headings and never restates their tables.

Section references (§N) point to `../awkit-docs/architecture.md` unless marked "design guide" (`../awkit-docs/design-guide.md`).

## Role and layer

Turn the candidates of a stored `session-audit` retro into proposals: exact edits to one local file (a skill, an instructions file, a config file, a permission profile), drafted by an isolated model session, reviewed as a tool-computed diff and written only by a human-confirmed `accept`. Layer 3 (workflow), post-v1, fully standalone at a terminal. Skill-wrapper tool (§5.5, design guide §11); default slot `improve`, bundled as `self-improve:improve` in `plugin/`.

## Uses

| Tool | Binding | What for |
|---|---|---|
| session-audit | A (hard) | `retro show --json` to read a stored retro and `retro list --json` for the `superseded` flag, which is read from the `retro list` row of the same id ([interface: Order of checks](interface.md#order-of-checks)). Only `state: done`, `superseded: false` rows. The pinned minimums are stated once in the interface [Overview](interface.md#overview). |
| claude-session | A (hard) | `start`, the adopting `exec`, `status`, `resume`, `stop`, `forget --purge` for drafting runs ([interface: Draft launch](interface.md#draft-launch)). It needs `--tools`, `--no-implicit-done`, `forget --purge` and a `report` that works in the sandbox after a normal install. |

Not used: agent-spawn, wtctl, termctl, memoctl, any notifier, the forge and ticket tools, `awkit`. Consumers: none in the architecture; awkit reads `list --json`, `validate --json`, `audit.jsonl` and the cron block through its adapter (a later awkit issue).

## Decisions

| ID | Decision | Rationale |
|---|---|---|
| D1 | One proposal per retro candidate and target; stored, reviewable, pinned by a plan hash. The dedupe key uses the retro id, not the retro content key; its index is plain, and the lookup inside the write transaction is the only guard. | Review in your own time; the harness can show "waiting" without applying; automation can draft but never accept. A re-run retro (`retro run --again`) keeps its content key but gets new candidate lists whose ids mean different things, so the content key would return an unrelated proposal. A unique index would block `--again`. See [Proposal lifecycle](interface.md#proposal-lifecycle). |
| D2 | The first draft is headless; `edit` resumes the same session, re-stages the inputs with a fresh run token for every run and passes the full flag set again, so every run uses the current config. A revision is a run on a `ready` or `no_change` proposal and never moves it to `failed`. | A design spike (not published): a headless resume keeps the artifact contract, costs about a cent and 20 s, and needs no agent-spawn or tmux. A flag given on resume replaces the carried value, so no safety setting depends on a carried value. Inputs given on resume land in a new `inputs/<run>/`, so the tool stages them again. A failed revision leaves the proposal as it was and records `last_revision_error`. See [edit](interface.md#edit) and [Slot contract](interface.md#slot-contract). |
| D3 | The target is named by a person or the user's own config; the model selects only among paths a person listed, never a path of its own. | A model-chosen path is the main injection channel of an edit tool. See [Target resolution](interface.md#target-resolution). |
| D4 | Three tiers decided by the path and the execute bit: ordinary, sensitive (extra terminal-only flag), refused. A `targets` entry never changes a tier; `sensitive_paths` is the way to make a path sensitive. | A permission-profile or hook edit can widen what later trusted sessions may do; review-watch's off-limits default covers the normal targets and cannot be reused. The tier is a pure function of the path, the execute bit and the lists, so a dotfiles real path such as a settings file stays sensitive whatever a `targets` entry says. See [Tiers](interface.md#tiers). |
| D5 | The drafting session is isolated (third-party-reading, sandbox required, no setting sources, MCP none) and receives a staged copy of the target. | The tool applies edits, so isolation costs nothing; in the design spike outside reads failed 4 of 4 and no profile lifted the rule, and staged input worked 17 of 17. See [Draft launch](interface.md#draft-launch). |
| D6 | One edit kind: exact find/replace pairs. No unified diff, no full-file output. | In the design spike this was the smallest and cheapest form and every `old` was unique, while a diff scope drifted once; an own applier has a tiny surface and `git apply` is not safe to call. See [Edit rules](interface.md#edit-rules). |
| D7 | `accept` is accept and apply in one step, confirmed on `/dev/tty` or with `--yes --expect-plan` and a `human:` actor; sensitive targets need a terminal-only flag. The integrity rules are re-checked at `accept`, the caps only at draft time. | Fewer states; keeps a human on edits that later sessions will load. A cap that was met at draft time cannot make a confirmed edit fail, while the diff length is still checked so a reviewer never confirms a cut diff. See [Accept order](interface.md#accept-order). |
| D8 | Apply sequence: lock, no-follow open, hash check, same-directory temp, `RENAME_EXCHANGE` with a post-check. Pre-image copy under the state directory; `undo` refuses if the target changed. | The design spike showed that symlink and hard-link targets break atomic replace (and a read-only bit does not stop it), a git-based undo destroys work, and the exchange closes the rename race. |
| D9 | Manual `propose` plus an opt-in `scan` that polls stored retros; no `post-retro` hook driver. `scan` drafts only ordinary-tier targets and only with `draft.sandbox: required`. | Polling loses nothing; a hook is best-effort and at-most-once. Unattended runs have the weakest secret screen (cron's environment holds few masked values) and nobody watching a staged copy of a sensitive file go to the model provider. See [scan](interface.md#scan). |
| D10 | Every write to a user file is audited with an `attempted` and an outcome record; drafting is only logged. | session-audit's and awkit's notes ask for audited edits; awkit pairs them by `attempt_id` without a special adapter. See [Audit](interface.md#audit). |
| D11 | Hooks `pre-draft`, `pre-accept`, `post-accept`, `post-attention`; no `post-pending`. A `pre-draft` veto or escalation on a first draft stores the proposal `failed`. | Gates and notices need them; proposals are the pending store. One transition for both outcomes keeps scan's failure streak free of special cases. See [Hooks](interface.md#hooks). |
| D12 | `--mode headless` only in 0.x; bare skill names refused; no TUI; home-only config. | Recorded deviations; see [Deviations from the design guide](interface.md#deviations-from-the-design-guide). |
| D13 | Plugin-cache and marketplace targets are refused with the plugin, version and source named. | An update overwrites the edit; the right place is the plugin's source repository. See [Target resolution](interface.md#target-resolution). |
| D14 | No creation of new files and no `git`-based apply or undo in 0.x. | Smaller surface; a new-skill proposal is a 1.x item. |
| D15 | Before staging, the target and every context file are scanned for secret shapes and masked environment values; a hit refuses the target (`contains_secret`) and `new` text is checked with the same function. A masked name matches by word for `KEY`, `KEYS`, `AUTH` and `AUTHORIZATION`, and a value that is wholly an absolute path is not masked. | A staged copy goes to the model provider and stays in the session transcript, which `forget --purge` does not remove. A settings or MCP file that holds a literal token cannot be a target until the token moves out. Whole-word matching keeps `GIT_AUTHOR_NAME` and `SSH_AUTH_SOCK` from refusing every file that holds a name or an e-mail address, and the result no longer differs between a terminal and cron. See [Edit rules](interface.md#edit-rules) and [Safety](interface.md#safety). |
| D16 | `accept` and `undo` record an intent row before the `attempted` audit record; a crash is settled at the next state-changing command by one hash decision, the same function that ends a live write. Recovery fires no hooks. | One source of recovery, no scan of the audit file, hooks stay at most once, and a live write and a crash at the same moment give the same state. See [Accept order](interface.md#accept-order) and [Concurrency](interface.md#concurrency). |
| D17 | In a markdown target an edit may change only lines of a closed set of forms: fence lines, the lines up to the first fence line and the first non-blank line are frozen, between fences only one-line `description` and `argument-hint` values may change, and no added line anywhere (frontmatter included) may hold a `!` command or an `@` followed by a character, except a whole email address. A structured target gets the `trust_key_changed` warning. | The ordinary tier is defined as "runs no code"; frontmatter hooks and command lines make a markdown file run code. The tool does not decide trust by parsing frontmatter and comparing keys, so the frontmatter part holds whichever keys Claude Code acts on. The body part is a denylist. See [Edit rules](interface.md#edit-rules). |
| D18 | Paths are normalized lexically and opened by a no-follow walk from `/`; the leaf must be a regular file with the owner write bit; ancestors and named leaves are compared by device and inode with the refused and sensitive roots. | Closes symlinked ancestors, FIFOs, bind mounts and the hard-linked or renamed settings file. See [Target resolution](interface.md#target-resolution) and [Tiers](interface.md#tiers). |
| D19 | Config is read under the password-database home with the symlink and ownership checks of session-audit; drop-ins merge in lexical order with `hooks.<name>` lists appended. | The config selects hook commands and the files the tool writes. The guide and awkit wire hooks through a drop-in, so lists must append. See [Config](interface.md#config). |
| D20 | Setting sources are always empty for the drafting run; there is no `draft.setting_sources` key. | A third-party-reading run must load none (claude-session fails the launch otherwise). See [Deviations from the design guide](interface.md#deviations-from-the-design-guide). |
| D21 | Scan has one unit, a reserved slot, for `scan.max_per_run`, `scan.max_per_retro`, `scan.max_per_day` and the failure streak; `scan.max_per_day` counts inside `limits.drafts_per_day`. It stops early when a budget is spent and reads at most 20 retros and runs at most 30 minutes per call. | Bounded cost and bounded work per tick, and manual drafts are never starved. There is no per-candidate table: a candidate with any proposal is done and filters are recomputed. See [scan](interface.md#scan). |
| D22 | One condition table fixes exit code, envelope `code`, reason and proposal state; `propose` handles every selected candidate and prints a result object on non-zero. | The schemas derive from one table. See [Exit codes](interface.md#exit-codes) and [Refusal reasons](interface.md#refusal-reasons). |
| D23 | The confirmation and `--yes` are not a boundary against a same-user caller; the `accept --dry-run --json` document carries hashes and counts and no text, and a dashboard hands over the terminal. | A process with a pty answers the prompt, and `--actor` is a claim. The boundary is that the drafting session cannot run the tool. See [Safety](interface.md#safety) and [`--json` outputs](interface.md#--json-outputs). |
| D24 | `undo` verifies the pre-image copy against the stored hash first, checks the file it replaces structurally only, and takes `--allow-sensitive` under the same rule as `accept`. | One rule for both write commands; a corrupt copy is refused before anything is written, and the bytes written are the stored, already-screened pre-image. See [undo](interface.md#undo). |
| D25 | Retention is per state, and a draft session is kept only while the proposal is `ready` or `no_change`; an open proposal expires after `retention.open`. The terminal states are listed in the interface. | Bounds the store and the retained sessions. See [Proposal lifecycle](interface.md#proposal-lifecycle) and [Retention](interface.md#retention). |

### Decisions changed since approval

By topic, against the choices first approved:

- Scan reach: `scan` drafts only ordinary-tier targets and exits 12 `sandbox_lowered` unless the drafting sandbox is required; a person drafts other targets with `propose`.
- Changed-line cap: the cap is the larger of 6 lines and 30% of the file's lines, so small files can be edited; the count is bounded and linear, with no whole-file diff.
- Masked environment names: `KEY`, `KEYS`, `AUTH` and `AUTHORIZATION` match as whole words only, and a value that is wholly an absolute path is not masked (a prefix test would exempt secrets that start with `/`).
- Markdown trust rule: closed line forms instead of a parsed frontmatter key comparison; `name` is no longer editable.
- Tiers: a `targets` entry never changes a tier; `sensitive_paths` does that job.
- Dedupe: the key uses the retro id, its index is plain, and the lookup in the write transaction is the guard.
- Size caps: caps are checked at draft time only, the diff length also at accept time; `display.max_lines` defaults to 600, and a hunk line over 2000 bytes is refused so one huge line cannot hide a change.
- Recovery: hash-only, one decision for the live end of a write and for recovery; no phase or holder fields.
- Counters: one run counter; `revision` is the run number of the stored edit, and `edit` stages the live target.
- Hooks: a `pre-draft` veto on a first draft stores `failed`, like an escalation.
- Stored candidate data: the snapshot keeps the cited findings' title and detail, so no later command reads the retro.
- Sensitive staging: files holding a secret are not staged.
- Audit results: there is no `refused` audit result, because a refusal writes nothing.
- Labels: `repo=` is a caller label only.
- Reference files of a skill target and the per-retro scan cap are implemented (staged `references/` files, `scan.max_per_retro`).

## Resolved open topics

From architecture §20 and the seed: *what is audited, when, output, how improvements are applied.*

- **Inputs.** A stored retro's candidates through `retro show --json` (binding A). Candidate text is untrusted; the tool uses the tool-written `risk` and `sessions`, never a risk read from text. The cited findings' text is copied into the proposal at creation.
- **When.** On demand (`propose`) and, only when enabled, by `scan`; never at session end.
- **Output.** A stored proposal with a tool-computed unified diff; `list --json` rows for harness attention.
- **How applied.** Only by `accept` after a human confirmation; the tier decides how loud; no automatic apply in 0.x (architecture §12.1 keeps auto-apply of low-risk targets as a post-v1 option, still unscheduled and still a non-goal of v1 in §19.2).
- **Off-limits.** The refused tier (credentials, the tool's own config and state, claude-session's state root) and the sensitive tier, by path; review-watch's off-limits list is not reused because it covers the normal targets.

## Live-verification table

| Behavior relied on | Status | Where |
|---|---|---|
| Find/replace pairs validate and apply; `old` unique; `no_change` is used for vague or oversized candidates | verified (24 haiku runs) | design spike (not published); becomes a `live` test |
| A session reads only staged inputs; outside reads fail under every profile tried | verified | design spike (not published) |
| Injection in candidate text was not followed (2 of 2) | verified, small n | design spike (not published); a `live` regression |
| Headless `resume --prompt-file` keeps the artifact contract; a tainted session may resume in the same kind | verified | design spike (not published) |
| Exchange rename closes the verify-to-replace race for rename-style writers; an open-descriptor writer can still lose a save | verified on Linux ext4 | design spike (not published); process test |
| Symlink, hard link, mode, `EXDEV` and read-only-bit behavior of atomic replace (a 0444 file is replaced by the atomic sequence) | verified | design spike (not published) |
| The injective ASCII escape round-trips 3000 of 3000 random edits | verified with the spike's alphabet | re-test with the 0.x alphabet (every backslash written `\\`) as a unit test, including the colliding cases |
| macOS fallback (no `RENAME_EXCHANGE`); 9p and network mounts; another-owner file | not verified | marked in Platform notes; conservative default (refuse or fall back and warn) |
| Headless `resume` with `--input-file` and `--skill` re-invokes the skill and keeps the artifact contract | not verified | becomes a `live` test |
| A resume dry run reports `previous_run` for exactly the keys the check accepts (`cwd`, `repo`, the always-carried settings in claude-session's carry-over table) | not verified | becomes a `live` test |
| `plan.permission_profile` is `file:` plus a prefix of the SHA-256 of the profile file's bytes (claude-session documents the label form, not which bytes are hashed) | not verified | becomes a `live` test; the check is `^file:[0-9a-f]{12,64}$` and a prefix of the SHA-256 of the bytes read once through a no-follow descriptor before the dry run |
| Markdown frontmatter rule | conservative by construction (fence-like lines, everything up to the first one and the first non-blank line are frozen) | [Edit rules](interface.md#edit-rules) |
| The `!` and `@` syntaxes Claude Code acts on in a markdown body | needs a live check before 1.0 | the body rule is a denylist; a new mechanism would pass as ordinary ([Residual risks](interface.md#residual-risks)) |
| A drafting session's behavior on a very large skill or a heavily nested config | not verified | cap by size; first live suite checks |
| Retro candidate quality on real transcripts | not verified | live checklist issue |

## Modules (planned)

| Module | Responsibility |
|---|---|
| `cli.py` | argparse commands, global flags, exit-code mapping, JSON envelopes. |
| `config.py` | Load and validate config with ownership and symlink checks, drop-in merge (lists under `hooks` append), safety-key rules; the password-database home is read by one gateway function that tests patch. |
| `store.py` | SQLite proposals, runs, intent rows, scanned retros, draft slots, leases, migrations. |
| `retro.py` | Read and validate retros through `session-audit` (gateway). |
| `targets.py` | Path normalization, target resolution, the safe open walk, target checks, tier matching (glob dialect and root identity compare). |
| `secrets.py` | Secret shapes and masked environment values, carried as this tool's own code from session-audit's redaction rule with the whole-word rule for `KEY`, `KEYS`, `AUTH` and `AUTHORIZATION` (a change there must be mirrored); used on staged text and on `new`. |
| `parse.py` | The safe parsers for JSON, YAML and TOML with the node and depth limits, and the frontmatter zone reader. |
| `draft.py` | Preflight, launch plan check, staging, prompt templates (`prompts/draft.md`, `prompts/revise.md`), start/adopt, resume, outcome mapping, draft and edit recovery. |
| `scan.py` | The scan loop, slots and budgets. |
| `artifact.py` | Strict artifact reading and validation. |
| `edit.py` | Pair rules, character rules, apply to bytes, result checks, the bounded changed-line count and hunks, the markdown line-form rule. |
| `apply.py` | The write sequence, pre-image copies, `undo`, apply recovery by one hash decision. |
| `render.py` | The review view, diff, injective escape. |
| `audit.py`, `log.py` | `audit.jsonl` and `log.jsonl` writers. |
| `hooks.py` | Hook runner, payload builders. |
| `cron.py` | Block build, install, uninstall. |
| `gateway.py` | The only place that runs subprocesses (claude-session, session-audit, crontab, hooks). |

Dependencies: Python 3.12 or later and the standard library (`sqlite3`, `hashlib`, `unicodedata`, `ctypes` for `renameat2`); YAML through `pyyaml` (`safe_load` only; a lint test bans every other loader); schema models through Pydantic only where the guide requires. No dependency for the diff, the applier or the escape.

## Data model

SQLite (`state.db`, WAL). Tables and columns only; the meaning of each proposal column is in the interface Proposal object ([Proposal lifecycle](interface.md#proposal-lifecycle)).

- `proposals`: id, `retro_id`, `retro_key`, `candidate_id`, candidate snapshot JSON (text fields and the cited findings' id, title and detail, all untrusted), `dedupe_key` (`sha256(retro_id, candidate_id, target_path or "")`, with a plain index; several rows may share a key and the in-transaction lookup is the only guard; `retro_key` is display data), `state`, `reason`, `refusal` JSON, `last_revision_error`, target path, `context_paths` JSON, tier, target `sha256` and `size`, `pre_sha256`, `post_sha256`, `source`, `revision`, `runs`, `outcome`, `rationale`, `replacements` JSON, `plan_sha256`, labels JSON, `csession_id`, `cost_usd`, `num_turns`, timestamps.
- `runs`: proposal id, run number, `kind` (`draft` or `edit`), `run_token`, `csession_id`, state, outcome, end reason, artifact sha256.
- `applies` (intent rows): `attempt_id`, proposal id, action, target path, parent, `pre_sha256`, `post_sha256`, temporary name, pre-image copy path, `audit_offset`, `created_at`.
- `draft_slots`: UTC day, source, count.
- `scanned_retros`: retro id, complete, slots, failures, scanned time. `scan_state`: consecutive failures, first and last failure times, `wedged_lease`.
- `leases`: object id (a proposal id, `scan` or `prune`), holder pid and start time, started time, expiry.
- `meta`: schema version.

Intent rows are the only source of apply recovery. Migrations are additive. A newer database exits 20 `store_newer`.

## Concurrency and recovery

Per-proposal lease (30 s renewal, 120 s expiry, compare-and-set takeover), per-target flock, `BEGIN IMMEDIATE` with a dedupe re-check and a daily-limit reservation before any hook, a compare-and-set on the expected source state for every state write, and no lease or lock held while waiting for a person. Drafting and edit runs are recovered by the stored `csession_id` and run number (a running session is stopped first); `accept` and `undo` are recovered from their intent rows by the hash decision, under the target lock, only for rows whose proposal lease expired, at the start of every state-changing command and never under `--dry-run`. A process that does not hold the lease (supersede, expiry, prune) skips any proposal with a live lease or an intent row. See [Concurrency](interface.md#concurrency), [Accept order](interface.md#accept-order) and [Draft launch](interface.md#draft-launch).

## Error handling

Fail fast with a closed `reason`. A refusal writes no user file and is not audited (a refused target is stored as a `refused` proposal); a failure after the model ran stores the proposal in `failed` with the reason so a person can see why. A failure of cleanup never changes a result. The tool never exits 1 on a validation failure; unexpected exceptions in artifact reading become `artifact_invalid`.

## Test strategy

- All subprocess calls go through `gateway.py`; tests use fakes for claude-session and session-audit with pinned copies of their schemas under `schemas/pinned/` (session-audit's `retro show` and `retro list`, and claude-session's `start`, `status`, `outcome`, `exec-dry-run` and `resume` shapes).
- Golden files for every `--json` output and the review view; schema validation of real output, including one envelope per row of the condition table; a differential test that the injective escape round-trips.
- Safe-parser tests: duplicate JSON keys, `NaN`, deep nesting, YAML alias expansion over the node limit, `RecursionError` counted as unparsable, and a lint test that bans `yaml.load`.
- The escape round trip is a unit test of the exact 0.x alphabet, including the colliding cases (a literal backslash followed by `t`, `x0a` or `s`), run as a differential test against the decoder.
- The password-database home is read by one gateway function that tests patch.
- Contract tests pipe every hook payload through `notice-route test --stdin-json`; a test runs a drop-in hook list plus a user hook and checks both run.
- Markdown line-form tests, run against the rule as written and not against a parser: a one-line `description` edit that adds `---` (or `+++`, `...`) inside the value is refused; `!` and `@` lines added inside frontmatter lines are refused; a frontmatter block preceded by blank lines, a whitespace-only line, an HTML comment line or a BOM-prefixed heading is still frozen outside `description` and `argument-hint`; a fence-like line anywhere in the first 20 lines opens the zone; `@"~/x"`, `@'/x'`, `@-x.md`, `@é.md`, `@$HOME/x` and `x@~/.ssh/y` are refused while a whole email token passes; an existing `@` line never blocks an edit that does not add one.
- Property tests for the pair rules and the character rules, run over a corpus of at least 200 pair cases covering LF, CRLF, final newline, overlaps and Unicode, which is also the starting set for the pair applier. A property test checks that the changed-line count and the hunks match a reference minimal diff within the stated bound on random edits, and a timing test uses a file of 65535 repeated lines. Templates are tested with braces in every untrusted field.
- Process tests for the apply sequence: a symlink swap, a hard link added, an edit between plan and apply, two instances racing, `undo` refusals; marked `process`. Crash-recovery process tests kill the tool at each of the five points (after the intent insert, after `attempted`, after the pre-image copy, after the exchange, after the outcome record) and check the state, the audit records and that no hook fires against the outcome table of Apply recovery. FIFO and symlinked-ancestor tests check that a FIFO target never blocks and that an ancestor which is a refused root is caught by identity.
- `live` tests (never in CI): the real drafting round trip with a cheap model, the injection candidate, a vague candidate giving `no_change`, a headless `edit`, the skill preflight and sandbox check, the resume dry run's `origins` keys, the apply race. Each verified behavior in the table above becomes a regression.
- A packaging test that the wheel contains `.claude-plugin/plugin.json`, the skill, the marker file and the profile.
- CI is Linux only (Python 3.12 and 3.13). macOS is a documented, untested fallback.

## Cross-tool facts to carry

- session-audit owns the candidate vocabulary and the retro; self-improve reads the pinned fields only and ignores additions. Drafting transcripts (labelled `owner=self-improve`) can be ingested by a later retro; session-audit is asked to exclude them by default.
- claude-session accepts any artifact as `done`, so this tool checks `report.implicit`, the artifact name, size and hash, identity echoes and the marker itself. Which `origins` keys a resume reports as `previous_run` is not documented beyond its carry-over table; this tool accepts only `cwd` and `repo` and fails closed otherwise, and it passes `--kind improve` and every other per-call flag on resume so nothing else is carried.
- A profile for every session that reads untrusted text (not only sandboxed ones) must deny `accept` and `undo` and writes to this tool's config and state directories (note for claude-session and agent-spawn).
- awkit's adapter reads `list --json` (attention: `ready` and recent `failed`), `validate --json`, `audit.jsonl` (paired by `attempt_id`) and the cron block. An ordinary `accept` works as a recipe (dry run, `plan_sha256` passed to `--expect-plan`, `yes: true`, `when: {tier: ordinary}`) or as a terminal action, which shows the tool's own review view; sensitive `accept`, `undo`, `edit`, `propose` and `resolve` are CLI-only because awkit passes no `--allow-sensitive`. The `scan_failing` timestamps are usable in `where` only, the cron interval is set as a string such as `1h`, and `awkit audit --label` does not match this tool's records because the audit `target` is a path.
