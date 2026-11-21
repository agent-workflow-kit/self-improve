# proposal.json contract

The artifact is one JSON object with exactly these keys and no others. Keep it under 65536 bytes.

| Key | Rule |
|---|---|
| `proposal_id` | Copy of `request.json`. |
| `run_token` | Copy of `request.json`. |
| `target_sha256` | Copy of `request.json`. |
| `marker` | The line of `references/marker.txt`. |
| `outcome` | `edit` or `no_change`. |
| `rationale` | 1 to 1000 characters, plain text. |
| `replacements` | `edit`: 1 to `max_pairs` objects `{"old": "...", "new": "..."}`; `no_change`: `[]`. |

Each `old` is non-empty, occurs exactly once in the staged target and is copied verbatim. `new` may be empty to delete text. The marker line for this skill is:

si-improve-marker-nb495z8wsxcw

Example:

```json
{
  "proposal_id": "sp-abcd2345",
  "run_token": "copied-from-request",
  "target_sha256": "64 hex characters from request.json",
  "marker": "the marker line above",
  "outcome": "edit",
  "rationale": "Adds the missing rule the candidate describes.",
  "replacements": [{"old": "exact existing text", "new": "replacement text"}]
}
```
