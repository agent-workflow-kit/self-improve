# self-improve: interface

Interface version: 0.1.0-design. This document is the single normative source for the tool's rules. `design.md`, the README and the architecture link to its headings and never restate its tables. Written before implementation (architecture §18.1).

Section references (§N) point to the architecture (`../awkit-docs/architecture.md`) unless marked "design guide" (`../awkit-docs/design-guide.md`).

## Overview

`self-improve` turns the candidates of a stored `session-audit` retro into **proposals**: each proposal is an exact edit to one local target file (a skill, an instructions file such as `CLAUDE.md`, a config file, a permission profile). A model session drafts the edit; a person reviews a diff that the tool computes; only `accept` writes to the target, and only after a human confirmation.

- One proposal per retro candidate and target. The candidate text is untrusted model output derived from transcripts, so the drafting session is isolated and the tool, not the session, resolves the target path, validates the edit and applies it.
- The drafting run is headless. Revising a draft (`edit`) resumes the same session with the person's instruction. There is no interactive mode in 0.x (see Deviations from the design guide).
- Nothing is applied automatically in 0.x. `scan` (opt-in) and cron can only create drafts; `accept` always needs a human.
- Every write to a user file is audited (`attempted` then outcome record). Every edit has a pre-image copy so `undo` can restore it.
- It reads retros only through `session-audit retro list --json` and `retro show --json` (binding A) and runs sessions only through `claude-session` (binding A). It uses no agent-spawn, wtctl, termctl, memoctl or notifier, never runs `awkit`, and does not know who calls it.

## Concepts

- **Retro, candidate**: as defined by session-audit's Retro object. A candidate has `id` (`c1`..`c20`, unique only inside its retro), `target_class` (`skill`, `instructions_file`, `config`, `permission_profile`, `workflow`), `target_hint`, `proposal` (text), `findings` (finding ids), tool-written `risk` (`high` or `medium`) and `sessions`. `target_hint` and `proposal` are untrusted free text.
- **Proposal**: the stored unit of work (id `sp-` plus 8 base32 characters, matching `^sp-[a-z2-7]{8}$`; a prefix of at least 4 characters after `sp-` is accepted, ambiguity exits 21). See Proposal lifecycle.
- **Target**: the one existing regular file a proposal edits. It is named by a person (`--target`) or by the user's own config (`targets`), never by the model. See Target resolution.
- **Tier**: `ordinary`, `sensitive` or `refused`, decided from the target path only (see Tiers).
- **Edit**: a list of exact find/replace pairs against the target text the session was given (see Edit rules). Or no edit: outcome `no_change` with a rationale.
- **Pre-image**: the target's exact bytes before an accepted edit, copied under the state directory so `undo` can restore it.
- **Plan**: what `accept` would write, identified by `plan_sha256` (see Plan hash).
- **Revision**: the draft counter of a proposal. `edit` creates a new revision of the same proposal; a new draft of the same candidate (`propose --again`) is a new proposal.
- **Actor**: `human:<user>`, `auto:<name>` or `auto:unknown` as in design guide §6. `<user>` fits `[A-Za-z0-9._@-]{1,64}`.

## Global flags

`--json`, `--config PATH`, `--verbose`, `--quiet`, `--dry-run`, `--yes`, `--version` on every command that is meaningful for it. Commands that write a user file or the audit log also take `--actor ACTOR` (default `human:<user>` from a terminal, else `auto:unknown`) and repeatable `--decision-id ID`. `--version` prints the package version from package metadata, never a constant.

Untrusted or model-written text is never an argument: instructions and reasons are passed with `--<name>-file PATH` or `-` (standard input). Errors go to standard error and, with `--json`, also to standard output as `{"error": {"code", "message", "details"}}`.

## Exit codes

| Code | Meaning |
|---|---|
| 0 | Success. `propose` and `scan` count a finished draft as success even when its outcome is `no_change`. |
| 1 | Internal error. |
| 2 | Usage error or invalid config (including `mode_unsupported`, `confirmation_required`, malformed `--skill`, `--candidate`, `--target`). |
| 10 | Not found (retro, candidate, proposal) or already in the requested state (`reject` of a rejected proposal, `undo` of an undone one; `details.state` names it). |
| 11 | A `pre-draft` or `pre-accept` hook vetoed. |
| 12 | Failed or unavailable (reasons below; `details.retryable`). |
| 13 | Needs a human: no terminal for a confirmation, a hook escalation, a session that reported `blocked` or `needs-input`, a sensitive target without `--allow-sensitive`. |
| 14 to 18 | The drafting session's outcome passed through (`details.outcome`, `details.proposal_id`): 14 exited unreported, 15 killed, 16 timeout, 17 lost, 18 reported failed. |
| 20 | Refused (`details.reason`; see Refusal reasons). |
| 21 | Ambiguous id prefix (`details.matches`, at most 20). |
| 22 | Nothing to do (`scan` found no new candidates or is disabled; `propose --all` found no candidates). |
| 129, 130, 143 | Stopped by a signal (no envelope). |

Reasons for exit 12 (`details.reason`): `claude_session_missing`, `claude_session_too_old`, `session_audit_missing`, `session_audit_too_old`, `store_unwritable`, `skill_not_found`, `unsafe_plugin_dir`, `launch_settings_mismatch`, `session_not_started`, `claude_session_error`, `session_audit_error`, `no_artifact`, `artifact_invalid`, `identity_mismatch`, `marker_mismatch`, `invalid_edit`, `write_failed`, `crontab_unavailable`. A failure after the model ran stores the proposal in state `failed` with the same reason.

## Refusal reasons

Exit 20 with `details.reason`:

| Reason | Meaning |
|---|---|
| `refused_target` | The target is in the refused tier (`details.rule` names the rule) or fails a target check (symlink, hard link, not a regular file, foreign owner, special mode bits, not UTF-8, over the size cap). |
| `plugin_cache` | The target is inside an installed plugin's cache directory; `details.plugin`, `details.version` and `details.source` name where the edit belongs. |
| `not_ready` | The proposal is not in state `ready` (`details.state`). |
| `stale` | The target's hash differs from the proposal's recorded pre-image hash. The proposal moves to `stale`. |
| `plan_mismatch` | `--expect-plan` differs from the plan the tool computed now. |
| `sensitive_not_allowed` | A sensitive target without `--allow-sensitive`, or `--allow-sensitive` given with `--yes`. |
| `actor_not_human` | `--yes` with an `auto:` actor. |
| `target_changed` | `undo` found the target differs from the recorded post-image. |
| `backup_expired` | `undo` needs a pre-image that was pruned. |
| `session_gone` | `edit` needs the drafting session but it was forgotten or purged. |
| `too_large` | The edit exceeds a cap or changes more than the allowed share of lines. |
| `daily_limit` | `limits.drafts_per_day` was reached. |
| `already_running` | Another process holds the proposal's lease or the target's lock. |
| `scan_disabled` | `scan` ran while `scan.enabled` is false (exit 22 under `--quiet`, see `scan`). |
| `config_unsafe` | The config file is a symlink, owned by another user or writable by group or others. |
| `store_newer` | `state.db` was written by a newer version; every command except `validate` refuses. |
| `cron_block_malformed` | The crontab holds a block with this tool's first line but no valid end line. |
| `retro_not_done` | The retro is not in state `done`, or is superseded (use `--allow-superseded` for the latter). |
| `declined` | A person answered anything but `y` to the confirmation. |

## Commands

The command set (every data command takes `--json`):

```
self-improve propose RETRO_ID [--candidate cN]... [--all] [--target PATH] [--context-file PATH]... [--again]
                              [--skill NAME] [--mode headless] [--model M] [--label K=V]... [--allow-superseded]
self-improve resolve PROPOSAL_ID --target PATH [--context-file PATH]...
self-improve scan [--limit N]
self-improve list [--state S]... [--retro ID] [--target PATH] [--label K=V]... [--limit N]
self-improve show PROPOSAL_ID [--diff]
self-improve edit PROPOSAL_ID --prompt-file PATH|-
self-improve accept PROPOSAL_ID [--expect-plan HASH] [--allow-sensitive]
self-improve reject PROPOSAL_ID [--reason-file PATH|-]
self-improve undo PROPOSAL_ID [--expect-plan HASH]
self-improve prune
self-improve validate [--probe]
self-improve export [--since WHEN]
self-improve cron install|uninstall [--scan-every DURATION]
```

### propose

Drafts proposals for one retro's candidates. `RETRO_ID` is a session-audit retro id (`ra-` plus 8 characters). The candidate is selected with `--candidate cN` (repeatable) or `--all`; exactly one of the two is required. `--target PATH` is allowed only with exactly one candidate and names the target for it. `--context-file PATH` (repeatable, at most 4) stages extra files beside the target as read-only context (for example a sibling skill showing house style); each file follows the Target checks and the size cap and is never edited. `--again` makes a new proposal for a candidate that already has one (see Dedupe). `--label K=V` adds labels to the proposal. `--skill`, `--model` and `--mode` override the config for this call; `--mode interactive` exits 2 `mode_unsupported`. With `--dry-run` it prints what would be drafted (candidates, resolved targets, tiers, the launch plan) and writes nothing.

Order of checks, per candidate: see Order of checks. Result (`--json`): `{"proposals": [Proposal...], "skipped": [{"candidate", "reason"}]}`. Exit 0 when every selected candidate produced a stored proposal (any state), 22 when none was selected, otherwise the first failing candidate's code with the successful ones still stored and listed.

### resolve

Gives an `unresolved` proposal its target (`--target PATH`, plus optional `--context-file`) and drafts it. Only `unresolved` proposals may be resolved; exit 20 `not_ready` otherwise. The path goes through Target resolution, Tiers and the target checks.

### scan

Cron-safe batch drafting. Off unless `scan.enabled` is true: when it is off, exit 22 with `details.reason: scan_disabled` (printed only without `--quiet`). It reads `session-audit retro list --state done --json`, takes retros with `superseded: false` that it has not scanned, and for each candidate that passes the `scan.risk` and `scan.classes` filters and resolves to a target through `targets` runs the `propose` flow. Candidates without a target are stored as `unresolved` proposals without a model run. At most `scan.max_per_run` model runs per call and `limits.drafts_per_day` per day; `--limit N` lowers the per-call cap. `scan` never accepts anything, never prompts, and is safe under concurrent invocation (a second caller exits 0 with `already_running` noted, per design guide §4.3). A scanned retro is recorded in the store so it is not scanned twice.

### list and show

`list` is read-only and prints `{"proposals": [Row...], "truncated": bool}`, newest first, store only (no liveness check). Rows carry no model text. `--state` is repeatable; `--limit` defaults to 100 (1 to 1000). `show` prints one Proposal; text mode renders the Review view; `--json` prints the object; `--diff` adds the rendered diff string to `--json`. Exit 10 for an unknown id, 21 for an ambiguous prefix.

### edit

Revises a `ready` or `no_change` proposal. `--prompt-file` holds the person's instruction (trusted text from the invoking person; it is delivered in the prompt file, not fenced as untrusted, but never put in argv). The tool re-hashes the live target: if it differs from the staged hash the proposal becomes `stale` and the command exits 20 `stale`. Otherwise it resumes the drafting session (`claude-session resume <id> --mode headless --prompt-file ...`) and validates the new artifact exactly as for a first draft. On success the revision counter rises and the new edit replaces the old one; on any failure the previous edit stays. If the drafting session was forgotten, exit 20 `session_gone` (use `propose --again`). `edit` has a per-proposal cap of `limits.revisions` (default 5).

### accept

The only command that writes a target (with `undo`). Steps are in Accept order. Without a terminal and without `--yes`, exit 13. `--dry-run --json` prints the plan and writes nothing. `--yes` is allowed only with a `human:` actor and `--expect-plan HASH`. `--allow-sensitive` is required for a sensitive target, is terminal-only (refused with `--yes`) and must be typed by a person who has seen the full diff.

### reject

Moves a non-terminal proposal to `rejected` (optionally storing a short reason from `--reason-file`, at most 500 characters). Idempotent: rejecting a rejected proposal exits 10 with `details.state`.

### undo

Restores the pre-image of an `applied` proposal. It refuses when the target's current hash differs from the recorded post-image hash (`target_changed`) or the pre-image was pruned (`backup_expired`). It uses the same confirmation, plan and write sequence as `accept` (a plan over the restore) and the same audit records with action `undo`.

### prune

Removes terminal proposals older than `retention.proposals`, pre-images older than `retention.backups` for `applied` proposals, frozen session artifacts and finished draft sessions (`claude-session forget --purge` after the results are stored), stale temp and lock files, and scanned-retro records older than `retention.scanned`. Safe under concurrent invocation.

### validate

Offline and read-only: config safety and schema, state directory ownership and modes, claude-session and session-audit presence and versions, the bundled plugin and profile, the skill preflight, the crontab block state. `--probe` adds live checks (a `claude-session start --dry-run` plan check and `session-audit retro list --json --limit 1`). Prints `{"ok", "checks": [{"name", "ok", "reason"}], "warnings": []}`. Exit 0 ok, 2 invalid config, 12 a failed check; a non-empty `warnings` list with exit 0 means "ok with warnings".

### export

Prints the audit log as JSON lines, optionally from `--since WHEN` (an ISO timestamp or a duration such as `7d`). Read-only.

### cron

`cron install` writes the managed crontab block (see Cron); `--scan-every DURATION` adds the `scan` line (a duration such as `1h`; bare numbers are refused); without it the block holds only the daily `prune`. `cron uninstall` removes the block. Both take `--dry-run --json` (`{"action": "installed|replaced|unchanged|removed|absent", "block": "<text>"}`; exit 20 `cron_block_malformed` for a block without a valid end line).

## Proposal lifecycle

States: `drafting`, `ready`, `no_change`, `unresolved`, `refused`, `failed`, `stale`, `applied`, `rejected`, `undone`.

| State | Meaning | Leaves by |
|---|---|---|
| `unresolved` | No target is known (no `--target`, no unique `targets` match). No model has run. | `resolve` (to `drafting`), `reject` |
| `refused` | The target is in the refused tier or failed a target check. Terminal; `reason` says why. | `prune` |
| `drafting` | A drafting session runs; a lease holds it (see Concurrency). | `ready`, `no_change`, `failed` |
| `ready` | A validated edit awaits review. Only this state can be accepted. | `accept` (to `applied` or `failed`), `edit` (stays `ready`), `reject`, `stale` |
| `no_change` | The session found nothing concrete to edit; the rationale is stored. | `edit`, `reject`, `prune` |
| `failed` | The draft or the write failed; `reason` is an exit 12 reason or `write_failed`. | `propose --again` (new proposal), `reject`, `prune` |
| `stale` | The live target differs from the recorded pre-image hash. | `propose --again`, `reject`, `prune` |
| `applied` | The edit is written. | `undo` (to `undone`), `prune` |
| `undone` | The pre-image was restored. Terminal. | `prune` |
| `rejected` | Closed by a person or superseded by a new draft (`reason: superseded`). Terminal. | `prune` |

Closed `reason` values of a proposal: for `failed`, any exit 12 reason above plus `session_blocked`, `session_needs_input`, `runner_lost`, `hook_escalated`, `write_failed`; for `refused`, `refused_target` or `plugin_cache`; for `rejected`, `user`, `superseded` or `hook_vetoed`; null otherwise.

Other rules:

- **Dedupe.** A proposal's key is the sha256 of the retro's `key` field (its content key) and the candidate id. `propose` for a key that already has a proposal returns the existing proposal and runs nothing. `--again` creates a new proposal for the key and moves every older proposal of that key that is still `unresolved`, `ready`, `no_change`, `failed` or `stale` to `rejected` with `reason: superseded`; `applied` proposals are kept. A new `done` retro with different content has a different key, so its candidates are fresh.
- **Superseded retros.** A retro with `superseded: true` is refused (`retro_not_done`) unless `--allow-superseded` is given; proposals already drafted from a retro stay valid when it is later superseded.
- **Attention.** Rows for `awkit` and other readers are `ready` proposals plus `failed` proposals changed in the last 7 days; healthy and terminal states are not attention items. `post-attention` fires on entering those (see Hooks).

## Target resolution

The target is a path a person or the user's own config named. Resolution order, per candidate:

1. `--target PATH` (or `resolve --target`): used as given after the checks below.
2. Otherwise the `targets` config list: an entry matches when its `class` equals the candidate's `target_class` and the candidate's `target_hint` (NFKC, case-folded, trimmed) equals the entry's `name` or one of its `aliases`, or equals the base name of the entry's path. A directory entry with `class: skill` matches a hint that fully matches `[a-z0-9][a-z0-9-]{0,63}` when `<dir>/<hint>/SKILL.md` exists. Exactly one match resolves; none or several leave the proposal `unresolved` (no model run).
3. The model never proposes a path, and the hint is never used as a path.

Target checks, applied at `propose`, again before staging, and again by `accept` (see Accept order): the path is absolute after `~` expansion and contains no `..`, no symlink in any component (checked with no-follow opens, ancestors included; a symlinked ancestor such as a dotfiles-managed directory is refused), names an existing regular file with exactly one link, owned by the effective user, without setuid, setgid or sticky bits, valid UTF-8 without NUL, at most `limits.target_max_bytes` (default 262144), and its parent directory is writable by the user. A failure is `refused_target` with `details.rule` naming the failed check. A target inside a plugin cache (a path with a component `plugins/cache`) is refused as `plugin_cache`; `details.plugin`, `details.version` and `details.source` are read from the cache layout and say where the edit belongs (the plugin's source repository), and a person can run `propose --target` against that source file instead.

## Tiers

The tier is decided by the target's path, never by the candidate's `target_class`, `risk` or any model text. Patterns use the glob dialect below and are matched against the absolute target path.

| Tier | Paths | Effect |
|---|---|---|
| refused | self-improve's own config and state directories; claude-session's state root; `~/.ssh/**`, `~/.gnupg/**`, `~/.aws/**`, `~/.config/gh/**`, `~/.config/glab-cli/**`; any file named `.env`, `.env.*`, `.netrc`, `*.pem`, `*.key`, `id_*`, or whose name contains `credentials` or `secret` | No proposal is drafted (`refused`). |
| sensitive | `**/.claude/settings.json`, `**/.claude/settings.local.json`, `**/.mcp.json`, `**/.claude/hooks/**`, `~/.config/**` (every awkit tool's config lives there), any file with an execute bit, and any path the ordinary tier does not match | `accept` needs `--allow-sensitive`, terminal-only. |
| ordinary | `**/SKILL.md`, `**/skills/**/*.md`, `**/CLAUDE.md`, `**/CLAUDE.local.md`, `**/AGENTS.md`, `**/.claude/commands/**/*.md`, `**/.claude/agents/**/*.md`, `**/.claude/rules/**/*.md`, and every file named by a `targets` entry without `sensitive: true` | The normal confirmation. |

The default lists are documented here and extendable in config (`refuse_paths`, `sensitive_paths`, `ordinary_paths`). Config can add to the refused and sensitive lists and to the ordinary list, but a path matching a built-in refused or sensitive default can never be moved to a lower tier by config, and a `targets` entry for such a path keeps its tier. Evaluation order: refused, then sensitive, then ordinary, then sensitive by default.

Glob dialect: `*` and `?` stay inside one path segment; `**` as a whole segment matches zero or more whole segments; a leading `~` is the invoking user's home from the password database; matching of the refused and sensitive tiers is ASCII case-insensitive and ordinary tier matching is case-exact; patterns are validated at config load (a pattern that fails the dialect is `config_invalid`); the path is compared as a string after the target checks, so no filesystem lookups happen during matching.

## Order of checks

For each candidate, `propose` runs these steps in order and stops at the first failure:

1. Parse flags and load config (`config_unsafe`, `store_newer`).
2. `claude-session --version` and `session-audit --version` meet the pinned minimums (exit 12 `claude_session_missing`, `claude_session_too_old`, `session_audit_missing`, `session_audit_too_old`).
3. Read the retro: `session-audit retro show RETRO_ID --json` (exit 10 passes through; `state` other than `done`, or `superseded`, is `retro_not_done`). Validate the fields used against the pinned copy of `retro-show.json`; unknown extra fields are ignored. The candidate must exist (exit 10).
4. Resolve the target (Target resolution). No target: store `unresolved`, stop. Refused or failed checks: store `refused`, stop.
5. Dedupe (Proposal lifecycle). An existing proposal is returned without a model run unless `--again`.
6. Skill preflight (Draft launch, item 1).
7. Launch plan check with `claude-session start ... --dry-run --json` (Draft launch, item 2).
8. `--dry-run` stops here.
9. `BEGIN IMMEDIATE`: insert the proposal in `drafting` and take its lease; reserve one slot of `limits.drafts_per_day` (`daily_limit`); this happens before any hook runs.
10. `pre-draft` hook (veto exits 11 and moves the proposal to `rejected` with `reason: hook_vetoed`; exit 13 stores `failed` with `reason: hook_escalated`).
11. Read the target and any context files once through verified descriptors, hash them, write the staged inputs and the prompt file under the state directory.
12. `claude-session start --json`, then the adopting `exec`, then collect, validate, store (Draft launch and Artifact validation).
13. `post-attention` when the proposal enters `ready` or `failed`.

## Draft launch

The drafting run mirrors session-audit's retro launch. Names of claude-session flags are as in its interface.

1. **Skill preflight.** `--skill` must be `plugin:skill` (bare names exit 2: no setting sources are loaded). The skill is searched in the bundled plugin (`importlib.resources.files("self_improve") / "plugin"`) and `draft.plugin_dirs`. A configured plugin directory may not be, contain or lie inside `$HOME` or an ancestor, `~/.claude`, `~/.config`, `~/.local/state` or the XDG homes, and everything in it must be owned by the user, not group- or other-writable, and a regular file or directory (exit 12 `unsafe_plugin_dir`); the bundled plugin is exempt (it is the tool's own package). The skill file `skills/<name>/SKILL.md` must exist (exit 12 `skill_not_found`).
2. **Launch plan check.** `claude-session start <flags> --dry-run --json` with placeholder files. The plan must show exactly: the configured tool list, `implicit_done` false, third-party-reading true, setting sources empty, MCP none, no environment inheritance, no env names set, no write directories, no notes context, no extra claude arguments, the configured sandbox mode, no repo, and only built-in or flag origins (except `model`, `timeout`, `max_turns`). Anything else is exit 12 `launch_settings_mismatch`.
3. **Per-call settings.** `--kind improve --reads-third-party --no-implicit-done --skill <skill> --plugin-dir <dir>... --prompt-file <prompt> --input-file <each staged input> --cwd session-dir --permission-profile <profile file> --sandbox <draft.sandbox> --setting-sources "" --env-allowlist-only <name>... --tools <draft.tools> --timeout <draft.timeout> --max-turns <draft.max_turns> [--model M] --label owner=self-improve --label proposal=<id>` plus the caller's labels. No report or escalation instructions are added.
4. **Staged inputs** (names fit claude-session's input-name grammar):
   - `request.json`: `{"proposal_id", "run_token", "revision", "target_name", "target_sha256", "target_class", "risk", "limits": {...}, "context": [{"name", "sha256"}]}` (ids, hashes, counts and enums only).
   - `target.<ext>` (the target's text, at most `limits.target_max_bytes`), named by the target's extension and a fixed stem so no path is exposed.
   - `context-N.<ext>` for each `--context-file`.
5. **Prompt file** (under `<state>/prompts/<proposal_id>/`, deleted when the run ends): the trusted instruction text for this call and the candidate material in a fence with a fresh nonce: the candidate's `target_class`, `target_hint` and `proposal`, and the titles and details of the findings it cites, each inside `<untrusted source="retro" id="NONCE">...</untrusted id="NONCE">` after neutralizing any `</untrusted`. No transcript text, no digest and no evidence items are included (the digest may have expired). The staged target and context files are also data; the standing sentence says so.
6. **Bundled profile** (`plugin/profiles/improve-draft.yaml`): adds only denies on top of claude-session's generated rules: `Agent`, `Task`, `WebFetch`, `WebSearch`, `NotebookEdit`, `mcp__*`, `Bash(memoctl:*)`, `Bash(agent-spawn:*)`, `Bash(forgectl:*)`, `Bash(ticketctl:*)`, `Bash(notice-route:*)`, `Bash(awkit:*)`, `Bash(session-audit:*)`, `Bash(self-improve:*)`, and `sandbox.network.allowedDomains: []`. The skill uses literal absolute paths in the `claude-session report` command and reads inputs with the Read tool.
7. **Start and adopt.** `claude-session start --json` (120 s), validate `CSESSION_ID` and the directory it names, store the session id on the proposal in the same transaction as the first run record, then run the printed `argv` plus `--outcome-file <state>/outcomes/<proposal_id>.json --json`. `edit` uses `claude-session resume <id> --mode headless --prompt-file <new prompt> --json` after re-staging nothing (the target must be unchanged, see `edit`).
8. **Deadline and signals.** Deadline is the exec spawn plus `draft.timeout` plus 10 minutes; at the deadline the tool signals the exec child it started, waits 30 s, runs `claude-session stop <id> --grace 5s`, collects and stores `failed` with `claude_session_error`. Call timeouts: `--version` 10 s; `list`, `status`, `forget`, `stop` 60 s; `start --dry-run` and `start` 120 s. Each child runs in its own process group.
9. **Sandbox check.** With `draft.sandbox: required`, a `status --json` whose `run.sandbox_active` is not true stores `failed` with `claude_session_error` and the artifact is not read.
10. **Outcome mapping** (first match wins): own deadline: `failed`, exit 12; signal forwarded: the proposal returns to its previous state and the tool exits 129, 130 or 143; `start` exit 12 with `launch_failed`: `failed`, `session_not_started`; the adopting exec reports a launch-phase failure: `failed`, `session_not_started`; exit 0 `done`: validate the artifact; exit 13 in the session phase: `failed` with `reason: session_blocked` or `session_needs_input`, exit 13; exit 14, 16, 17, 18: `failed`, the code passed through; exit 15 not caused by the tool: `failed`, 15; anything else (including unparseable output): `failed`, `claude_session_error`, exit 12.
11. **Cleanup.** After the result is stored the session is kept while the proposal is `ready` or `no_change` (so `edit` can resume it). It is forgotten with `claude-session forget <id> --purge` when the proposal reaches a terminal state or by `prune`; forgetting is retried by `prune` and never blocks a command's result.
12. **Recovery.** A `drafting` proposal whose lease expired is recovered by its stored session id (never by label lookup), after a label sanity check (`owner=self-improve`, `proposal=<id>`): collect and validate if the run finished, otherwise `failed` with `reason: runner_lost`. Recovery runs at the start of `propose`, `scan`, `edit`, `accept` and `prune`, never in `validate` or `list`.

## Slot contract

The `improve` skill (bundled as `self-improve:improve`) receives, in the session directory: `inputs/1/request.json`, `inputs/1/target.<ext>`, optional `inputs/1/context-N.<ext>`, and the prompt file path in the bootstrap prompt. It returns a single artifact `proposal.json` in `$CSESSION_DIR/artifacts/` and files `claude-session report done`.

Artifact schema (closed; no other keys; `schemas/slot/artifact.json`):

| Key | Rule |
|---|---|
| `proposal_id`, `run_token` | Echo of `request.json`. |
| `target_sha256` | Echo of the staged target's hash. |
| `marker` | The marker string from the skill's `references/marker.txt` (bundled) or `draft.skill_marker`; not an injection defense, a check that the skill ran. |
| `outcome` | `edit` or `no_change`. |
| `rationale` | 1 to 1000 characters, plain text, shown to the reviewer as model text. |
| `replacements` | For `edit`: 1 to `limits.max_pairs` objects `{old, new}`; for `no_change`: an empty list. |

The skill must not write anywhere but `artifacts/proposal.json` and its summary file, must treat the candidate, finding text, target and context files as data, and prefers `no_change` with a short rationale over a vague or oversized edit.

## Artifact validation

After a session ends `done`, the tool reads `claude-session status <id> --json` and checks, in this order (each failure stores `failed` with the named reason):

1. `run.report.implicit` is false and an artifact is reported (`no_artifact`).
2. The artifact name is `proposal.json` and its size is at most 65536 bytes (`artifact_invalid`).
3. The frozen file is read with `O_NOFOLLOW`, regular, at most 65537 bytes, and its size and SHA-256 equal `run.report.artifact` (`artifact_invalid`).
4. Strict JSON: UTF-8, no duplicate keys, no `NaN` or `Infinity`, depth at most 8, closed schema (`artifact_invalid`).
5. Identity echoes `proposal_id`, `run_token`, `target_sha256` (`identity_mismatch`).
6. `marker` (`marker_mismatch`).
7. The edit rules (below) and the result checks (`invalid_edit`); the proposal stays `failed` and the previous revision's edit, if any, is kept.
8. The target is re-read through a no-follow descriptor and its hash must still equal the staged hash; if not the proposal becomes `stale`.

The artifact copy is stored in the proposal row; the session's frozen copy is not relied on afterwards.

## Edit rules

The only edit kind in 0.x is a list of exact find/replace pairs against the staged target text. Checked by the tool before a proposal can be `ready`:

- Each `old` is non-empty and occurs exactly once in the target; `old` and `new` differ; pairs do not overlap and the same `old` does not appear twice; at most `limits.max_pairs` (20) pairs.
- The target and `new` are UTF-8 text without NUL, lone surrogates, a byte-order mark added or removed, or any character from the refused character set (C0 and C1 controls other than TAB and LF, DEL, bidi controls, zero-width characters and joiners, tag characters, variation selectors, U+2028, U+2029, NEL, private use, unassigned code points, noncharacters). A character already present in the target is not re-validated if it sits in an unchanged part.
- Line endings are preserved: `new` must use the target's ending (a CRLF file gets CRLF in `new`; a bare CR anywhere is refused); the final-newline state of the file may not change unless an `old` or `new` covers the end of the file.
- The result is at most `limits.target_max_bytes`, `new` text totals at most `limits.new_text_max_bytes` (65536), and the share of changed lines (changed lines over lines in the target) is at most `limits.max_changed_ratio` (0.30); above that the drafted edit is rejected as `invalid_edit` (the proposal is `failed`, because the model produced it). `too_large` as a refusal reason applies when `accept` re-checks a stored edit against caps that were lowered after the draft.
- A JSON, YAML or TOML target must still parse after the edit, with the same top-level type.
- No `new` text contains a value that matches secret shapes or a configured env var value (`redaction.mask_env` names variables whose values are refused), or a path under `~/.ssh`.

An edit that fails a rule is not applied or shown as an acceptable proposal. `no_change` and an empty edit are stored as `no_change` with the rationale.

## Plan hash

`plan_sha256` is the SHA-256 of the canonical JSON (sorted keys, no whitespace, ASCII) of `{"v": 1, "action": "accept"|"undo", "proposal_id", "revision", "target": {"path", "parent", "tier", "pre_sha256", "size"}, "edit_sha256", "post_sha256"}`, where `edit_sha256` is the hash of the canonical pairs and `post_sha256` the hash of the resulting bytes. It is printed by `accept --dry-run --json` and in the exit 13 envelope and must be passed back with `--yes --expect-plan`. Any change to the target, the edit, the tier or the revision changes it.

## Accept order

`accept PROPOSAL_ID` runs these steps in order:

1. Parse flags, load config, run recovery for this proposal's lease.
2. Load the proposal; it must be `ready` (`not_ready`). Take its lease and the target's lock file (`already_running`).
3. Re-run the Target checks and Tiers on the stored path (`refused_target`, `plugin_cache`). A target that moved into a higher tier since the draft is judged by the new tier.
4. Open the target no-follow, read it through the descriptor, and compare its SHA-256 with the proposal's `pre_sha256`; a difference sets state `stale` and exits 20 `stale`.
5. Re-apply the stored pairs to the bytes just read, re-run the Edit rules and compute the plan (`invalid_edit` is a refusal here, state `failed`).
6. Render the Review view. If the output was truncated by the display cap the command refuses to confirm (`too_large`, nothing written).
7. `--dry-run` prints the plan and stops.
8. Confirmation. For a sensitive target `--allow-sensitive` must be present and `--yes` absent (`sensitive_not_allowed`). Without `--yes`: open `/dev/tty` (no terminal: exit 13 `needs_human`, no fallback to standard input), flush pending input, print the view and read one answer; anything but an explicit `y` declines (exit 20 with `reason: declined`; nothing is written and nothing is audited). With `--yes`: the actor must be `human:<user>` (`actor_not_human`) and `--expect-plan` must equal the plan (`plan_mismatch`).
9. `pre-accept` hook (veto exits 11 and writes nothing; escalation exits 13).
10. Append the `attempted` audit record (if the append fails, nothing is written).
11. Write the pre-image copy (mode 0600, fsynced) and an `applying` intent row, then perform the write sequence below.
12. Append the outcome audit record, set the state (`applied`, or `failed` with `write_failed` and the target unchanged), set `applied_at`, `post_sha256`.
13. `post-accept` hook.

Write sequence (Linux; the target's own directory is pinned with a no-follow directory descriptor and every call uses it): take the per-target lock (flock on `<state>/locks/<hash of path>.lock`); open the target no-follow; check regular, one link, owner, no special bits, size; read and hash through the descriptor and compare with `pre_sha256`; create the temporary file in the same directory (`O_EXCL`, no-follow, name `.si-<pid>-<random>`); write, set the target's own mode with `fchmod`, `fsync`; re-verify that the path still names the verified inode, that size and mtime are unchanged and re-hash; exchange the temporary file and the target with `renameat2(RENAME_EXCHANGE)`; open the swapped-out file and require its device, inode and hash to equal the verified ones, exchanging back on a mismatch (outcome `failed`, `reason: write_failed`, exit 20 `stale`); remove the old file; `fsync` the directory. Where `RENAME_EXCHANGE` is unavailable the tool uses the re-verify step and `os.replace` and prints a warning naming the weaker guarantee (see Platform notes). A writer that already holds an open descriptor to the old file can still lose its save in the last microseconds; `undo` re-checks the hash for exactly that reason.

## Review view

Text output of `show` and of the confirmation screen, built by the tool and never by the model:

- Header: proposal id, retro id and candidate id, target path, tier, the candidate's `risk` (labelled as tool-written), revision, state, the pre-image hash prefix and the plan hash.
- The rationale, labelled "model text", sanitized.
- The candidate text and cited finding titles, labelled "model text from retro", sanitized and truncated for display.
- The diff: unified, three lines of context, computed by the tool from the target bytes and the result bytes with the pairs applied, with counts of added, removed and changed lines and the changed-line share. Warnings for: share above half the cap, a `permissions` allow set that grew (for JSON targets with a `permissions.allow` list), an edit touching a line with an `allow`/`deny` keyword for a profile-like file, `redacted` counts from the retro.
- Display sanitizing: every string from a retro, a model or a file is rendered by one injective ASCII-only escape: a backslash is written `\\` when the next character would otherwise form an escape, TAB `\t`, CR `\r`, a trailing space `\s`, other C0 controls and DEL `\xNN`, every non-ASCII code point `\uXXXX` or `\UXXXXXXXX`. The rendering reconstructs the original text exactly. Lines are never cut silently: a display cap (`display.max_lines`, default 2000) that would cut the diff makes `accept` refuse to confirm, and `show` prints a marked cut.
- Homoglyphs and visually similar letters are not detected; a reviewer must read the diff.

## `--json` outputs

Schemas are in `schemas/cli/`. Timestamps are UTC with `Z`. `--json` output is ASCII (`ensure_ascii`). Fields carrying text from a retro, a model or a file are marked `x-untrusted` in the schemas.

The Proposal object (`propose`, `resolve`, `edit`, `show`; `schemas/defs.json`): `id`, `state`, `reason` (null or the closed reason for `failed`, `refused`, `rejected`), `retro_id`, `retro_key`, `candidate_id`, `candidate` (`{target_class, target_hint, proposal, findings, risk, sessions}`; the text fields are untrusted), `target` (`null` while unresolved, else `{path, tier, sha256, size}`), `source` (`how the target was chosen: flag|config|resolve`), `revision`, `revisions` (a count), `outcome` (`edit|no_change|null`), `rationale` (untrusted or null), `replacements` (list of `{old, new}`, untrusted), `changed_lines`, `changed_ratio`, `plan_sha256` (null unless `ready`), `post_sha256`, `labels`, `csession_id`, `cost_usd`, `num_turns`, `warnings` (closed set), `created_at`, `updated_at`, `drafted_at`, `applied_at`, `undone_at`, `finished_at`.

Rows of `list` carry `id`, `state`, `reason`, `retro_id`, `candidate_id`, `target_path`, `tier`, `risk`, `revision`, `updated_at` and no text. The `accept --dry-run --json` plan: `{"plan": {...fields of Plan hash}, "plan_sha256", "tier", "needs_allow_sensitive", "warnings"}`. An exit 13 or 20 envelope for `accept` carries `details.plan_sha256` when a plan exists.

## Hooks

Hooks are configured only in the tool's own config (`hooks.<name>`: a list of `{cmd, timeout, async}`; `cmd` is a string run by `sh -c` in its own process group, JSON on standard input, no template substitution). Environment: `SELF_IMPROVE_HOOK`, `SELF_IMPROVE_ID`, `SELF_IMPROVE_OBJECT_TYPE`; the hook inherits the environment minus the credential variables. Default timeout 30 s, at most 600 s. A payload carries `tool`, `hook`, `object_type` (`proposal`, `scan`), `object_id`, `event_id` (`ev-` plus 26 lowercase base32) and `labels`; there is no `repo`. No payload carries retro, candidate, model or file text. Payloads carry `reads_third_party: false` and `tainted: false` booleans.

| Hook | When | Payload extras | Effect |
|---|---|---|---|
| `pre-draft` | after the proposal row and lease exist, before any model run | `retro_id`, `candidate_id`, `target_class`, `risk`, `tier`, `model` | exit 0 continue; 13 escalate; other non-zero, timeout or signal veto (exit 11) |
| `pre-accept` | after the confirmation, before the audit record and the write | `proposal_id`, `action` (`accept` or `undo`), `target_path`, `tier`, `plan_sha256` | exit 0 continue; 13 hold (exit 13); other non-zero veto (exit 11) |
| `post-accept` | after the outcome record, for `accept` and `undo`, success or failure | `proposal_id`, `action`, `result` (`applied`, `undone`, `failed`, `stale`), `pre_sha256`, `post_sha256`, `tier` | cannot undo; best effort, at most once; may be `async` |
| `post-attention` | a proposal enters `ready` or `failed`, or `scan` keeps failing | `reason` (`proposal_ready`, `draft_failed`, `scan_failing`), `object_type`, `details` (`{retro_id, candidate_id, failure_reason}` or `{consecutive_failures, first_failed_at, last_failed_at}`) | cannot undo; best effort, at most once |

A pre-hook's standard output may be a JSON object with `decision_id` (a string or a list); it is recorded in the audit entry. There is no `post-pending`: proposals are the pending store and `post-attention` with `proposal_ready` says the same thing with a closed reason. No outbox in 0.x. Hook schemas: `schemas/hooks/<hook>.json`.

## Config

File: `~/.config/self-improve/config.yaml` under the home directory from the password database, plus `config.d/*.yaml` (lists replace, mappings merge). The file must be a regular file owned by the user and not writable by group or others; otherwise `config_unsafe`. `--config PATH` is accepted on commands that run nothing. Schema: `schemas/config.json`; example: `config.example.yaml`. Durations use the guide grammar.

| Key | Default | Meaning |
|---|---|---|
| `me.name` | none | Name for `human:` actors when the terminal user cannot be derived. |
| `draft.skill` | `self-improve:improve` | `plugin:skill` of the drafting skill. |
| `draft.skill_marker` | none | Marker for a custom skill (the bundled skill's marker file is used otherwise). |
| `draft.plugin_dirs` | `[]` | Extra plugin directories (at most 8 absolute paths; see Draft launch item 1). |
| `draft.model` | none | Model for drafts (a cheap model is recommended). |
| `draft.timeout` | `10m` | Session timeout (1 s to 2 h). |
| `draft.max_turns` | `12` | Turn bound. |
| `draft.permission_profile` | bundled `improve-draft.yaml` | Profile file path. |
| `draft.sandbox` | `required` | `required`, `preferred` or `off`; lowering needs a terminal. |
| `draft.setting_sources` | `[]` | Setting sources to load. |
| `draft.env_allowlist` | `[PATH, HOME, USER, LANG, TERM]` | Names passed with `--env-allowlist-only`. |
| `draft.tools` | `[Read, Write, Bash, Skill]` | Passed with `--tools`. |
| `targets` | `[]` | Named targets: `{name, aliases, path, class, sensitive}`; a file or, for `class: skill`, a directory of skills. |
| `refuse_paths`, `sensitive_paths`, `ordinary_paths` | `[]` | Extra patterns (Tiers). |
| `limits.drafts_per_day` | `20` | Model drafts per day (reserved at row insert). |
| `limits.revisions` | `5` | Revisions per proposal. |
| `limits.target_max_bytes` | `262144` | Target size cap. |
| `limits.new_text_max_bytes` | `65536` | Sum of `new` text. |
| `limits.max_pairs` | `20` | Pairs per edit. |
| `limits.max_changed_ratio` | `0.30` | Changed-line share cap. |
| `display.max_lines` | `2000` | Diff lines before `accept` refuses to confirm. |
| `scan.enabled` | `false` | Enables `scan`. |
| `scan.max_per_run` | `5` | Model runs per `scan` call. |
| `scan.risk` | `[high, medium]` | Candidate risks `scan` drafts. |
| `scan.classes` | all five | Candidate target classes `scan` drafts. |
| `redaction.mask_env` | `[]` | Names of variables whose values may never appear in `new` text. |
| `retention.proposals` | `90d` | Terminal proposals. |
| `retention.backups` | `90d` | Pre-images of applied proposals. |
| `retention.scanned` | `180d` | Scanned-retro records. |
| `hooks.<name>` | none | Hook lists (see Hooks). |

Safety keys (`draft.permission_profile`, `draft.sandbox`, `draft.setting_sources`, `draft.env_allowlist`, `draft.tools`, `draft.plugin_dirs`, `targets`, the path lists and `redaction.mask_env`) come from the config file and flags only; there is no environment layer and no per-repo layer for them.

## Environment variables

Only non-safety keys have overrides: `SELF_IMPROVE_DRAFT_TIMEOUT`, `SELF_IMPROVE_DRAFT_MODEL`, `SELF_IMPROVE_LIMITS_DRAFTS_PER_DAY`, `SELF_IMPROVE_RETENTION_PROPOSALS`, `SELF_IMPROVE_RETENTION_BACKUPS`. Hook processes receive `SELF_IMPROVE_HOOK`, `SELF_IMPROVE_ID`, `SELF_IMPROVE_OBJECT_TYPE`. `XDG_*` variables do not move the config or state directories (they are taken from the password database home). The tool needs `claude-session`'s `required_env_names` in its environment for the adopting `exec` (cron lines source the user's env file).

## Labels

A proposal carries `retro=<retro id>`, `candidate=<cN>`, `owner=self-improve`, `kind=improve` and any `--label K=V` pairs (grammar as claude-session: key `^[a-z0-9][a-z0-9_.-]{0,63}$`, value 0 to 256 characters from `A-Za-z0-9._:/#@+=,-`). The drafting session carries `owner=self-improve` and `proposal=<id>`. Labels are lookup aids only; no key has built-in meaning and no label is an authority.

## TUI

None in 0.x (`self-improve browse` is a 1.x item). Review is `show --diff` plus `accept` in a terminal; a harness reads `list --json`.

## Files

State under `~/.local/state/self-improve/` (directories 0700, files 0600), from the home in the password database:

| Path | Content |
|---|---|
| `state.db` | SQLite (WAL, versioned additive migrations, `BEGIN IMMEDIATE` for writes): proposals, revisions, runs, scanned retros, leases. Private; never read by other tools. |
| `audit.jsonl` | Public audit log (Audit). |
| `log.jsonl` | Public hash-only operational log (Log). |
| `backups/<proposal_id>/` | Pre-image copy and metadata (0600). |
| `prompts/`, `inputs/`, `outcomes/`, `tmp/` | Per-run files, deleted when the run ends. |
| `locks/` | Per-target lock files. |

Config under `~/.config/self-improve/`. The bundled plugin (`plugin/`) holds `.claude-plugin/plugin.json`, `skills/improve/SKILL.md`, `skills/improve/references/marker.txt` and `profiles/improve-draft.yaml`; the packaging test must prove the hidden `.claude-plugin` directory ships in the wheel.

## Audit

`audit.jsonl` (schema `schemas/audit.json`): one record per phase for every write to a user file, appended before the write (`attempted`) and after it (the outcome). Fields: `ts`, `actor`, `actor_source`, `stdin_tty`, `decision_id` (string or list, optional), `attempt_id` (shared by the pair), `action` (`accept` or `undo`), `target` (the path), `inputs` (`proposal_id`, `retro_id`, `candidate_id`, `revision`, `plan_sha256`, `tier`, `pre_sha256`, `post_sha256`; hashes and ids only, never text), `result` (`attempted`, `applied`, `undone`, `failed`, `stale`). If the first append fails nothing is written. A refusal, veto, escalation or declined confirmation is not audited (nothing was written). Drafting is not audited: it writes only the tool's own store. `export` prints the file; awkit reads it directly.

## Log

`log.jsonl` (schema `schemas/log.json`) is hash-only and records completed changes of the tool's own state: `proposal_state` (`proposal_id`, `state`, `reason`, `retro_id`, `candidate_id`, `revision`, `target_sha256`, `csession_id`, `cost_usd`, `num_turns`, `decision_ids`), `scan` summaries and `prune` counts. It carries no model, retro or file text.

## Retention

Terminal proposals: `retention.proposals` (90 days). Pre-images of `applied` proposals: `retention.backups` (90 days, then `undo` refuses `backup_expired`). A `failed` or `stale` proposal is terminal for retention after 30 days. Draft sessions: kept while a proposal can be revised, then `forget --purge`. Frozen artifacts are copied into the proposal row and not relied on later. `prune` applies all of these.

## Concurrency

A proposal's lease is `(holder_pid, holder_start_time, expires_at)`, renewed every 30 s and expiring 120 s after the last renewal; takeover is a compare-and-set. The per-target lock serializes two self-improve processes that edit the same file; it does not stop an editor. `scan` and `prune` follow design guide §4.3 (a second caller exits 0 and says "already running"). A signal handler is installed before the first state change; hooks and claude-session children run in their own process groups; a second signal exits at once.

## Cron

`cron install` writes this block (first line `# self-improve managed`, last line `# self-improve managed end`, an explicit `PATH` including `~/.local/bin`, an optional user-owned environment file sourced first):

```
# self-improve managed
<minute> * * * * PATH=... self-improve scan --quiet --actor auto:self-improve-cron   (only with --scan-every)
<minute> 3 * * * PATH=... self-improve prune --quiet --actor auto:self-improve-cron
# self-improve managed end
```

`scan` draws on `scan.enabled`; installing the line does not enable it. Neither line can accept anything.

## Safety

- **Isolation.** The drafting run reads untrusted text and is third-party-reading: sandbox required, no setting sources, MCP none, no environment inheritance, an own empty working directory, a profile that denies every other awkit tool and the network, and tools restricted to `Read Write Bash Skill`.
- **The tool decides the path.** The model never names a target; the artifact carries no path; the staged target's hash is checked before and after; only `accept` writes.
- **Applier, not tools.** The tool applies exact pairs with its own code; it never runs `git apply`, `patch` or a shell on model output.
- **Human acceptance.** `accept` reads a confirmation from `/dev/tty`; `--yes` needs a `human:` actor and the exact plan hash; automation never passes `--yes` to bypass the prompt; the sensitive tier needs a terminal-only flag.
- **Persistence.** An accepted edit becomes input to later trusted sessions. The review diff, the tiers, the changed-line cap and the character rules exist for that reason; a reviewer must still read the diff.
- **Never destroy work.** No `git reset`, `checkout`, `clean` or `restore` on user files; a changed target makes `accept` and `undo` refuse; signals go only to exact PIDs the tool started.
- **Secrets.** No credential values in state, logs, audit or output; `new` text is refused if it contains a masked variable's value.
- **Untrusted text transport.** Candidate, finding, target and model text travel in files or on standard input, never in argv or env; prompt files live under the state directory and are deleted when the run ends.
- **The session's profile must not allow `self-improve` itself**, and a profile that wraps this tool for a sandboxed session must deny `accept`, `undo` and writes to its config and state directories.

## Platform notes

- Linux and WSL2 are first class. Sandboxing needs bubblewrap and socat; claude-session must be installed with copied (not hard-linked) files.
- macOS: `renameat2(RENAME_EXCHANGE)` does not exist. The apply sequence falls back to re-verify plus `os.replace` and prints a warning; the lost-save window is slightly wider. Not tested in 0.x.
- Filesystems where `fsync` or atomic rename behave differently (network and 9p mounts) are untested; targets there are refused with a warning-level `unsafe_filesystem` check by `validate` when detected.
- A target inside a dotfiles-managed tree reached through a symlink is refused (no symlink in any component); the person points `--target` at the resolved real file.

## Residual risks

- A homoglyph, a visually similar letter or a subtly wrong instruction inside a legitimate-looking edit can pass a tired reviewer; the tiers only raise the ceremony.
- An editor that holds an open descriptor to the old file can lose a save in the last microseconds of an apply.
- Draft quality varies run to run (n=1 samples in the spike); a `no_change` result is often right for a vague candidate.
- A retro's evidence may have expired (digests live 7 days, retros 180), so the tool cannot show evidence items; it shows the cited finding text.
- The staged target text is untrusted input to the model too (an installed skill can carry instructions).
- A permission-profile or settings edit can widen what later trusted sessions may do; the warnings and the sensitive tier mitigate and do not prevent it.
- Extended attributes, ACLs and security labels of the target are not carried to the new file.

## Deviations from the design guide

- `--mode interactive` is refused (exit 2 `mode_unsupported`) in 0.x; the headless draft and `edit` replace it (guide §11).
- Bare user or project skill names are refused in 0.x (no setting sources are loaded).
- The `improve` kind row in the architecture (trusted, unsandboxed, all setting sources) stays the fallback for a direct `claude-session exec --kind improve`; every drafting call overrides it per call (guide §11 safety defaults).
- Config and state are read only from the password-database home, with no environment layer or per-repo layer for the safety keys (as for tools whose config chooses what gets written).
- No `post-pending` hook: proposals are the pending store and `post-attention` covers it.
- Confirmation is read from `/dev/tty` (no `--yes` without a `human:` actor), as in ticket-create.

## Upgrades

The interface version and the schemas change together. `state.db` migrations are additive; a database written by a newer version makes every command except `validate` exit 20 `store_newer`. New states, reasons, hooks, config keys and `--json` fields are additive; renamed or removed ones before 1.0 are a `feat` whose title names the break. `schemas/cli/retro-show.json` is a pinned, tolerantly read copy of session-audit's; a sibling change that breaks a field this tool uses is found by `validate --probe` and by the live suite.
