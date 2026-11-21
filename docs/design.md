# self-improve: design

Status: designed (interface 0.1.0-design). The normative rules are in [`interface.md`](interface.md); this document gives the reasons, the decisions and the internals. It links to interface headings and never restates their tables.

Section references (§N) point to `../awkit-docs/architecture.md` unless marked "design guide" (`../awkit-docs/design-guide.md`).

## Role and layer

Turn the candidates of a stored `session-audit` retro into proposals: exact edits to one local file (a skill, an instructions file, a config file, a permission profile), drafted by an isolated model session, reviewed as a tool-computed diff and written only by a human-confirmed `accept`. Layer 3 (workflow), post-v1, fully standalone at a terminal. Skill-wrapper tool (§5.5, design guide §11); default slot `improve`, bundled as `self-improve:improve` in `plugin/`.

## Uses

| Tool | Binding | What for |
|---|---|---|
| session-audit | A (hard) | `retro list --json` and `retro show --json` to read stored retros ([interface: Order of checks](interface.md#order-of-checks)). Only `state: done`, `superseded: false` rows. Pinned minimum is set when session-audit ships `retro show --json`. |
| claude-session | A (hard) | `start`, the adopting `exec`, `status`, `resume`, `stop`, `forget --purge` for drafting runs ([interface: Draft launch](interface.md#draft-launch)). Pinned minimum 0.45.2 (`--tools`, `--no-implicit-done`, `forget --purge`, a `report` that works in the sandbox after a normal install). |

Not used: agent-spawn, wtctl, termctl, memoctl, any notifier, the forge and ticket tools, `awkit`. Consumers: none in the architecture; awkit reads `list --json`, `validate --json`, `audit.jsonl` and the cron block through its adapter (a later awkit issue).

## Decisions

| ID | Decision | Rationale |
|---|---|---|
| D1 | One proposal per retro candidate and target; stored, reviewable, pinned by a plan hash. | Review in your own time; the harness can show "waiting" without applying; automation can draft but never accept. See [Proposal lifecycle](interface.md#proposal-lifecycle). |
| D2 | The first draft is headless; `edit` resumes the same session with the person's instruction. | Spike A: a headless resume keeps the artifact contract, costs about a cent and 20 s, and needs no agent-spawn or tmux. This keeps "interactive" as the revise loop without a second code path. See [edit](interface.md#edit). |
| D3 | The target is always named by a person or the user's own config, never by the model; no scan of directories. | A model-chosen path is the main injection channel of an edit tool. See [Target resolution](interface.md#target-resolution). |
| D4 | Three tiers decided by path only: ordinary, sensitive (extra terminal-only flag), refused. | A permission-profile or hook edit can widen what later trusted sessions may do; review-watch's off-limits default covers the normal targets and cannot be reused. See [Tiers](interface.md#tiers). |
| D5 | The drafting session is isolated (third-party-reading, sandbox required, no setting sources, MCP none) and receives a staged copy of the target. | The tool applies edits, so isolation costs nothing; spike A: outside reads fail 4/4 and no profile lifts the rule, staged input worked 17/17. See [Draft launch](interface.md#draft-launch). |
| D6 | One edit kind: exact find/replace pairs. No unified diff, no full-file output. | Spike A: smallest, cheapest, every `old` unique; the diff scope drifted once. Spike B: own applier has a tiny surface, `git apply` is not safe to call. See [Edit rules](interface.md#edit-rules). |
| D7 | `accept` is accept and apply in one step, confirmed on `/dev/tty` or with `--yes --expect-plan` and a `human:` actor; sensitive targets need a terminal-only flag. | Fewer states; keeps a human on edits that later sessions will load. See [Accept order](interface.md#accept-order). |
| D8 | Apply sequence: lock, no-follow open, hash check, same-directory temp, `RENAME_EXCHANGE` with a post-check. Pre-image copy under the state directory; `undo` refuses if the target changed. | Spike B: symlink and hard-link targets break atomic replace, a git-based undo destroys work, the exchange closes the rename race. |
| D9 | Manual `propose` plus an opt-in `scan` that polls stored retros; no `post-retro` hook driver. | Polling loses nothing; a hook is best-effort and at-most-once. See [scan](interface.md#scan). |
| D10 | Every write to a user file is audited with an `attempted` and an outcome record; drafting is only logged. | session-audit's and awkit's notes ask for audited edits; awkit pairs them by `attempt_id` without a special adapter. See [Audit](interface.md#audit). |
| D11 | Hooks `pre-draft`, `pre-accept`, `post-accept`, `post-attention`; no `post-pending`. | Gates and notices need them; proposals are the pending store. See [Hooks](interface.md#hooks). |
| D12 | `--mode headless` only in 0.x; bare skill names refused; no TUI; home-only config. | Recorded deviations; see [Deviations from the design guide](interface.md#deviations-from-the-design-guide). |
| D13 | Plugin-cache targets are refused with the plugin, version and source named. | An update overwrites the edit; the right place is the plugin's source repository. |
| D14 | No creation of new files and no `git`-based apply or undo in 0.x. | Smaller surface; a new-skill proposal is a 1.x item. |

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
| The injective ASCII escape round-trips 3000 of 3000 random edits | verified | spike B Q4 |
| macOS fallback (no `RENAME_EXCHANGE`); 9p and network mounts; another-owner file | not verified | marked in Platform notes; conservative default (refuse or fall back and warn) |
| A drafting session's behavior on a very large skill or a heavily nested config | not verified | cap by size; first live suite checks |
| Retro candidate quality on real transcripts | not verified | live checklist issue |

## Modules (planned)

| Module | Responsibility |
|---|---|
| `cli.py` | argparse commands, global flags, exit-code mapping, JSON envelopes. |
| `config.py` | Load and validate config with ownership checks, drop-ins, safety-key rules. |
| `store.py` | SQLite proposals, revisions, runs, scanned retros, leases, migrations. |
| `retro.py` | Read and validate retros through `session-audit` (gateway). |
| `targets.py` | Target resolution, target checks, tier matching (glob dialect). |
| `draft.py` | Preflight, launch plan check, staging, prompt writing, start/adopt, outcome mapping, recovery. |
| `artifact.py` | Strict artifact reading and validation. |
| `edit.py` | Pair rules, character rules, apply to bytes, result checks, changed-line share. |
| `apply.py` | The write sequence, pre-image copies, `undo`. |
| `render.py` | The review view, diff, injective escape. |
| `audit.py`, `log.py` | `audit.jsonl` and `log.jsonl` writers. |
| `hooks.py` | Hook runner, payload builders. |
| `cron.py` | Block build, install, uninstall. |
| `gateway.py` | The only place that runs subprocesses (claude-session, session-audit, crontab, hooks). |

Dependencies: Python 3.12 or later and the standard library (`sqlite3`, `difflib`, `hashlib`, `unicodedata`, `ctypes` for `renameat2`); YAML through `pyyaml`; schema models through Pydantic only where the guide requires. No dependency for the diff, the applier or the escape.

## Data model

SQLite (`state.db`, WAL):

- `proposals`: id, `retro_id`, `retro_key`, `candidate_id`, candidate snapshot (text fields, untrusted), `dedupe_key`, `state`, `reason`, target path, tier, `pre_sha256`, `post_sha256`, `source` (flag, config, resolve), `revision`, `outcome`, `rationale`, `replacements` JSON, `plan_sha256`, labels JSON, `csession_id`, `cost_usd`, `num_turns`, timestamps.
- `runs`: proposal id, revision, run number, `run_token`, `csession_id`, state, outcome, end reason, artifact sha256.
- `scanned_retros`: retro id, retro key, scanned time, counts.
- `leases`: object id, holder pid and start time, expiry.
- `meta`: schema version.

Migrations are additive. A newer database exits 20 `store_newer`.

## Concurrency and recovery

Per-proposal lease (30 s renewal, 120 s expiry, compare-and-set takeover), per-target flock, `BEGIN IMMEDIATE` with a daily-limit reservation before any hook, recovery by stored `csession_id` after a label sanity check. See [Concurrency](interface.md#concurrency) and [Draft launch](interface.md#draft-launch) item 12.

## Error handling

Fail fast with a closed `reason`. A refusal writes nothing and is not audited; a failure after the model ran stores the proposal in `failed` with the reason so a person can see why. A failure of cleanup never changes a result. The tool never exits 1 on a validation failure; unexpected exceptions in artifact reading become `artifact_invalid`.

## Test strategy

- All subprocess calls go through `gateway.py`; tests use fakes for claude-session and session-audit with pinned copies of their schemas (`schemas/cli/retro-show.json` and the `start --dry-run`, `status`, `forget` shapes).
- Golden files for every `--json` output and the review view; schema validation of real output; a differential test that the injective escape round-trips.
- Property tests for the pair rules and the character rules; the spike B corpus (213 cases, in the three formats it covered) becomes the starting set for the pair applier.
- Process tests for the apply sequence: a symlink swap, a hard link added, an edit between plan and apply, two instances racing, a crash at both points of the intent record, `undo` refusals; marked `process`.
- `live` tests (never in CI): the real drafting round trip with a cheap model, the injection candidate, a vague candidate giving `no_change`, a headless `edit`, the skill preflight and sandbox check, the apply race. Each spike result becomes a regression.
- A packaging test that the wheel contains `.claude-plugin/plugin.json`, the skill, the marker file and the profile.
- CI is Linux only (Python 3.12 and 3.13). macOS is a documented, untested fallback.

## Cross-tool facts to carry

- session-audit owns the candidate vocabulary and the retro; self-improve reads the pinned fields only and ignores additions.
- claude-session accepts any artifact as `done`, so this tool checks `report.implicit`, the artifact name, size and hash, identity echoes and the marker itself.
- A sandboxed profile that wraps this tool must deny `accept` and `undo` and writes to its config and state directories (note for claude-session and agent-spawn).
- awkit's adapter reads `list --json` (attention: `ready` and recent `failed`), `validate --json`, `audit.jsonl` (paired by `attempt_id`) and the cron block, and never runs `accept` itself except through its confirmation dialog with `--yes --actor human:<user> --expect-plan`; the sensitive tier is terminal-only by design.
