# proposal.json contract

The artifact is one JSON object with exactly these keys and no others. Keep it under 65536 bytes.

| Key | Rule |
|---|---|
| `proposal_id` | Copy of `request.json`. |
| `run_token` | Copy of `request.json`. |
| `target_sha256` | Copy of `request.json`. |
| `marker` | Copy the line from `references/marker.txt`. |
| `outcome` | `edit` or `no_change`. |
| `rationale` | 1 to 1000 characters, plain text. |
| `replacements` | `edit`: 1 to `max_pairs` objects `{"old": "...", "new": "..."}`; `no_change`: `[]`. |

Each `old` is non-empty, occurs exactly once in the staged target and is copied verbatim. `new` may be empty to delete text. `run_token` and `target_sha256` come from the `request.json` of the current run. Copy the marker line from `references/marker.txt`, which is its only source; do not type it from memory.

Example:

```json
{
  "proposal_id": "sp-abcd2345",
  "run_token": "copied-from-request",
  "target_sha256": "64 hex characters from request.json",
  "marker": "the line copied from references/marker.txt",
  "outcome": "edit",
  "rationale": "Adds the missing rule the candidate describes.",
  "replacements": [{"old": "exact existing text", "new": "replacement text"}]
}
```
