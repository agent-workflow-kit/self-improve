# self-improve: design

Status: Seeded from awkit architecture v1.4; not yet designed.

Section references (§N) point to `../awkit-docs/architecture.md` unless marked "design guide" (`../awkit-docs/design-guide.md`).

## Role and layer

Turn audit findings into proposed edits to skills, CLAUDE.md files or config, presented for acceptance.

Layer 3 (workflow). Must work fully standalone; no upward awareness.

Post-v1: not part of the v1 scope (§4, §19). It may be built in parallel if there is capacity.

Skill-wrapper tool (§5.5, design guide §11). Default slot: `improve` (bundled as `self-improve:improve` in `plugin/`).

## Responsibilities

From §5.3 (`self-improve`), verbatim:

Skill-wrapper: turn audit findings into proposed edits to skills, CLAUDE.md files or config, presented for acceptance. Default mode interactive.

## Uses

None specified in the architecture yet; to be decided in the design session (binding: TBD (A/B/C)).

## Key flow

No flow is defined for this tool in §10 (post-v1).

## Relevant cross-cutting sections

- §3.2, §3.3: layering rules and edge bindings
- §5.5: capability slots: `improve`, kind `improve`
- §8: session model
- §12.1: public default: proposed edits, user accepts
- §13: security
- §14: audit
- design guide §11: skill-wrapper pattern

## Constraints (decided)

From §20 (`session-audit, self-improve`):

none.

Packaging: The bundled plugin is shipped inside the package and located via `importlib.resources.files('self_improve') / 'plugin'`.

## Open topics

From §20 (`session-audit, self-improve`):

what is audited, when, output, how improvements are applied.

## Decisions

| Decision | Rationale | Date |
|---|---|---|

## Questions for the design session

1. Which audit findings are inputs, and in what format?
2. When does it run?
3. What does a proposal look like (output format, diff presentation)?
4. How are accepted improvements applied, and which targets could ever be auto-applied?
