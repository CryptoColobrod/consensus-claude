# Contributing

## Reporting a weak or wrong panel verdict

Open an issue using the "Weak or wrong panel verdict" template. Attach the run record from
`./consensus-runs/` for that run, and name which thesis and which role you think went wrong
(e.g. "T3, Optimizer missed the N+1 query"). Without the run record, there's no raw vote data
to debug against — it's the single most useful thing you can attach.

## Proposing a new shelf role

Follow the mandate template already used by the existing roles in `agents/`:

- A one-line mandate (what this role is for, in a sentence)
- A lens (what it evaluates, and just as importantly, what it explicitly stays out of)
- Operational heuristics (what makes it vote `DISPUTED` vs `AGREED_WEAK`, concretely)
- The vote-format contract: statuses (`AGREED` / `AGREED_WEAK` / `DISPUTED` /
  `NEEDS_CLARIFICATION`), the `CONDITION:` tag, and a `DISAGREEMENT:` block — the Judge parses
  all three, so a new role has to emit them in the same shape as the existing roles.

See any file in `agents/` for a working example of the shape, and DESIGN.md's Role Library
section for how `when_to_enable` / `conflicts_with` metadata is expressed.

## Pull requests

Welcome for role heuristics (tightening an existing role's lens, fixing a vague DISPUTED
signal) and for docs. Keep PRs scoped to one role or one doc section at a time — easier to
review, easier to revert if it doesn't work out.

## Scope discipline

Feature ideas — new presets, auto-assembly, anything beyond the golden-standard panel — belong
in an issue first, not a PR. Check [DESIGN.md's "Future Arc"](DESIGN.md) section before filing:
it's the visible roadmap of what's deliberately deferred to v2, and your idea may already be on
it with a stated reason it isn't in v1 yet.
