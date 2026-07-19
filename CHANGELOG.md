# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.2.0] - 2026-07-19

### Added

**Reasoning depth**
- T0 Premise Distillation — the Decomposer now surfaces the question's foundational unstated
  premise as thesis T0, voted like any other thesis; a panel that disputes T0 is challenging
  the frame itself, and the synthesis leads with that.
- Structured vote tags — `FLIP:` (mandatory on every `DISPUTED` vote, names the evidence that
  would reverse it), `ANCHOR: "quote"` (grounds a vote in a specific piece of the question or
  context), and `[impact: critical|moderate|minor]`. The 4-status vocabulary is unchanged; the
  Judge weights disputes carrying `FLIP:` higher, flags unanchored votes, and prioritizes
  critical-impact conflicts.
- `CONSENSUS_STRENGTH` — the synthesis now opens with `Strong consensus` / `Working consensus`
  / `Narrowly carried` / `Contested`.
- Tripwires — the Prerequisites section now also lists `Revisit if: <observable event> →
  re-examine T<n>`, derived from `FLIP:` tags, turning the record into a living monitoring
  document instead of a one-time snapshot.
- Minority Report — a role that loses a vote by roughly 3:1 with substantive dissent gets a
  named block in Unresolved, tracked as a risk rather than buried as a footnote.

**Artifacts**
- `AUDIT_BLOCK` — every run record now opens with a machine-readable block (date, mode,
  roster, consensus strength, verdict, key assumptions), so the record can be pasted into any
  external model for a post-hoc audit, independent of which provider ran the panel.
- Decision journal — runs append one line (date, question, strength, verdict, link) to
  `./consensus-runs/INDEX.md`, turning accumulated runs into a browsable ledger.

**Run control**
- `--plan` flag — dry-run checkpoint that runs Triage and Decompose only, shows the parsed
  intent and theses, and waits for approval or edits before any panel votes are cast.
- `--roles` flag — one-run roster override: `+Shelf-role` enables a shelf role for this run
  only, `-Role` disables an active one, and `+name(focus:'...')` mints an ephemeral persona.

**Experimental**
- `--emergent` flag ("Schrödinger mode") — the problem defines its own panel: the Decomposer
  generates the 5-7 strongest distinct perspectives the question calls for, retroactively mints
  an ephemeral persona to champion each, then runs the normal per-thesis vote flow. Adversarial
  Presence is still enforced. Panel composition is non-deterministic by design — a deliberate
  novelty-vs-reproducibility trade, labeled experimental.

## [1.1.0] - 2026-07-19

- Provider-agnostic external critic — any CLI, or manual paste-through to whatever model you
  have — was gemini-only before.
- Agents inherit the current session's model — was hardcoded to `opus` before.
- Verifiable Decision Record written to `./consensus-runs/` on by default.
- Install block mkdir fix — `~/.claude/agents/` was never created, breaking first-time installs.
- Cost warning and "When to use this skill" section added.
- Uninstall/update docs added.
- Community files added: CONTRIBUTING.md, weak-verdict issue template.
- Affiliation disclaimer added.

## [1.0.0] - 2026-07-19

Initial public release.
