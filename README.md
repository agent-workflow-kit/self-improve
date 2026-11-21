# self-improve

Turn the candidates of a stored `session-audit` retro into reviewed, exact edits to your skills, `CLAUDE.md` files, config files and permission profiles. A model session drafts each edit in isolation; you review a diff the tool computes; only `accept` writes, and only after you confirm.

## Status

Design complete (interface 0.1.0-design); not implemented yet. Post-v1: not part of the v1 scope (architecture §19).

## Layer

L3 (workflow). Works standalone at a terminal; it never calls the harness (`awkit`) and does not know who calls it.

## What it does

- `propose RETRO_ID` drafts one proposal per retro candidate: an exact find/replace edit to one target file you named (or your config named), or `no_change` with a reason.
- `show` prints the review view with the tool-computed diff; `edit` revises a draft with your instruction; `accept` applies it after a confirmation on your terminal; `undo` restores the pre-image; `reject` closes it.
- Targets are tiered by path and execute bit: ordinary (skills, `CLAUDE.md`), sensitive (settings files, permission profiles, tool configs, hook scripts; needs `--allow-sensitive`) and refused (credentials, the tool's own state). A `targets` entry in your config can raise a tier, never lower it. Files that contain secrets are never staged to the model.
- Nothing is applied automatically. Every write to a file is audited, and every edit keeps a pre-image copy.
- Optional `scan` (off by default) drafts for new retros from cron; it never accepts anything.

## Commands

`propose`, `resolve`, `scan`, `list`, `show`, `edit`, `accept`, `reject`, `undo`, `prune`, `validate`, `export`, `cron install|uninstall`. Full flags, exit codes and JSON shapes: [`docs/interface.md`](docs/interface.md).

## Dependencies

| Tool | Binding | Install |
|---|---|---|
| session-audit (minimum version: see the interface) | A (hard; `retro list --json`, `retro show --json`) | `uv tool install git+https://github.com/agent-workflow-kit/session-audit` |
| claude-session (minimum version: see the interface) | A (hard) | `uv tool install --link-mode=copy git+https://github.com/agent-workflow-kit/claude-session` |

Minimum versions are stated once, in [`docs/interface.md`](docs/interface.md) (Overview). Linux and WSL2 also need `bubblewrap` and `socat` for the sandboxed drafting session. A missing sibling gives an error that names its install command.

## Install

```sh
uv tool install git+https://github.com/agent-workflow-kit/self-improve
```

## Quick start

```sh
self-improve validate
self-improve propose ra-xxxxxxxx --candidate c1 --target ~/.claude/skills/my-skill/SKILL.md
self-improve show sp-xxxxxxxx --diff
self-improve accept sp-xxxxxxxx
```

Pre-register your usual targets in `~/.config/self-improve/config.yaml` (`targets:`) so a candidate resolves without `--target`. See `config.example.yaml`.

## Exit codes

0 success; 2 usage or invalid config; 10 not found or already in that state; 11 hook veto; 12 failed or unavailable; 13 needs a human; 14 to 18 the drafting session's outcome; 20 refused; 21 ambiguous id; 22 nothing to do. The condition table with every reason is in [`docs/interface.md`](docs/interface.md) (Exit codes and Refusal reasons).

## Environment variables

Only non-safety keys have overrides: `SELF_IMPROVE_DRAFT_TIMEOUT`, `SELF_IMPROVE_DRAFT_MODEL`, `SELF_IMPROVE_LIMITS_DRAFTS_PER_DAY` and `SELF_IMPROVE_RETENTION_PROPOSALS`. Config and state are read from the home in the password database; `XDG_*` variables do not move them. The adopting `claude-session exec` needs the variables named by `required_env_names` (cron lines source your env file). See [`docs/interface.md`](docs/interface.md) (Environment variables).

## Documentation

- System architecture: [`../awkit-docs/architecture.md`](../awkit-docs/architecture.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/architecture.md))
- Tool design guide: [`../awkit-docs/design-guide.md`](../awkit-docs/design-guide.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/design-guide.md))
- This tool's design: [`docs/design.md`](docs/design.md)
- This tool's interface: [`docs/interface.md`](docs/interface.md)

## License

MIT — see LICENSE.
