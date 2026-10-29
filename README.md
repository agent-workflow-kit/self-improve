# self-improve

Turn audit findings into proposed edits to skills, CLAUDE.md files or config, presented for acceptance.

## Status

Pre-design: interface not yet specified; see docs/design.md. Post-v1: not part of the v1 scope (architecture §19).

## Layer

L3 (workflow)

## What it does

- Skill-wrapper: turns audit findings into proposed edits to skills, CLAUDE.md files or config, presented for acceptance.
- Default mode interactive. Public default: proposed edits that the user accepts; auto-apply of low-risk targets is a post-v1 option (architecture §12.1).

## Indicative commands

Not yet specified in the architecture; to be defined in this tool's design session and finalized in `docs/interface.md`.

## Dependencies

Not yet specified in the architecture; to be decided in this tool's design session (binding: TBD (A/B/C)).

## Install

```sh
uv tool install git+https://github.com/agent-workflow-kit/self-improve
```

## Documentation

- System architecture: [`../awkit-docs/architecture.md`](../awkit-docs/architecture.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/architecture.md))
- Tool design guide: [`../awkit-docs/design-guide.md`](../awkit-docs/design-guide.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/design-guide.md))
- This tool's design: [`docs/design.md`](docs/design.md)
- This tool's interface: [`docs/interface.md`](docs/interface.md)

## License

MIT — see LICENSE.
