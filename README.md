# self-improve

Turn the candidates of a stored `session-audit` retro into reviewed, exact edits to your skills, `CLAUDE.md` files, config files and permission profiles. A model session drafts each edit in isolation; you review a diff the tool computes; only `accept` writes, and only after you confirm.

## Status

Design complete (interface 0.1.0-design); not implemented yet. Post-v1: not part of the v1 scope (architecture §19).

## Layer

L3 (workflow). Works standalone at a terminal; it never calls the harness (`awkit`) and does not know who calls it.

## What it does

- `propose RETRO_ID` drafts one proposal per retro candidate: an exact find/replace edit to one target file you named (or your config named), or `no_change` with a reason.
- `show` prints the review view with the tool-computed diff; `edit` revises a draft with your instruction; `accept` applies it after a confirmation on your terminal; `undo` restores the pre-image; `reject` closes it.
- Targets are tiered by path and execute bit: ordinary (skills, `CLAUDE.md`), sensitive (settings files, permission profiles, tool configs, hook scripts; needs `--allow-sensitive`) and refused (credentials, the tool's own state). A `targets` entry in your config never changes a tier. Files that contain secrets are never staged to the model.
- Nothing is applied automatically. Every write to a file is audited, and every edit keeps a pre-image copy.
- Optional `scan` (off by default) drafts for new retros from cron, only for ordinary-tier targets and only with the sandbox required; it never accepts anything.

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

See [Exit codes](docs/interface.md#exit-codes) and [Refusal reasons](docs/interface.md#refusal-reasons) in the interface.

## Environment variables

See [Environment variables](docs/interface.md#environment-variables) and [Config](docs/interface.md#config) in the interface.

## Documentation

- System architecture: [`../awkit-docs/architecture.md`](../awkit-docs/architecture.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/architecture.md))
- Tool design guide: [`../awkit-docs/design-guide.md`](../awkit-docs/design-guide.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/design-guide.md))
- This tool's design: [`docs/design.md`](docs/design.md)
- This tool's interface: [`docs/interface.md`](docs/interface.md)

## License

MIT — see LICENSE.
