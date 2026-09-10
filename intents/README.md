# Intent Records

One file per unit of work: `NNNN-short-slug.md`. Each drives exactly one PR.

Format and rules: `../docs/factory/intent-record-template.md`.
What happens to the resulting PR: `../docs/factory/quality-gates.md`
and `../docs/factory/review-policy.md`.

Records are committed and kept after merge — the intent history records *why*
a change was made, which the git history alone does not.

Status values: `draft` (has open questions, not buildable) -> `ready` ->
`in-progress` -> `in-review` -> `merged` | `abandoned`.
