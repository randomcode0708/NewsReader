# Specs

Spec Kit working area. One directory per feature: `NNN-short-slug/`.

```
001-source-backoff/
  spec.md     /specify + /clarify output — the WHAT and WHY
  plan.md     /plan output — the technical approach
  tasks.md    /tasks output — the decomposition into PR-sized units
```

Workflow: `../docs/factory/sdlc.md`.
Binding rules every spec must respect: `../docs/factory/constitution.md`.
What happens to the resulting PRs: `../docs/factory/quality-gates.md`,
`../docs/factory/review-policy.md`.

Specs are committed and kept after merge — the spec history records *why* a
change was made, which the git history alone does not.

## Writing a good spec

- **Behaviour, not implementation.** "Summaries must cite their source" is a
  spec. "Add a citations column" is a plan.
- **Acceptance criteria must be individually testable.** If you can't imagine
  the assertion, the criterion is too vague. This is where quality is won.
- **State what's out of scope.** Prevents the most common agent failure mode.
- **Don't skip `/clarify`.** An ambiguity resolved there costs one sentence;
  the same ambiguity found in review costs a rebuild.
