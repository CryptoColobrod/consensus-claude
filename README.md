# consensus-claude

A multi-perspective consensus panel for [Claude Code](https://claude.com/claude-code) that votes on each atomic thesis of a question separately — not on the answer as a whole.

## The differentiator

Most review tools score a decision holistically: one pass, one verdict. `consensus-claude` decomposes the question first and votes claim by claim.

- **Per-thesis decomposition.** An isolated Decomposer agent splits the question into 3–7 atomic decision criteria. A binary "A or B" question becomes independent criteria to evaluate — never a mirror pair like "T1: choose A / T2: choose B."
- **4-status voting per thesis** — `AGREED` / `AGREED_WEAK` / `DISPUTED` / `NEEDS_CLARIFICATION` — plus a `CONDITION:` tag for conditional agreement ("yes, if this external dependency holds"), instead of bloating the status vocabulary with a 5th state.
- **Hard-capped rounds.** R1 (parallel, isolated) + a targeted R2 on disputed theses only. Never R3 — two rounds without convergence is a signal to escalate to external review, not to spin the same panel again.
- **Single-model by design.** The panel's independent perspectives come from role isolation within one apex model — distinct mandates, clean contexts, no cross-talk in R1 — not from paying for three API keys across three providers. An external critic enters only as an anti-groupthink tripwire when the panel goes suspiciously unanimous, and it's provider-agnostic: configure any CLI in one line, or fall back to a copy-paste prompt handoff to whatever model you already have — never the default mode.
- **Critics with biographies, not a generic skeptic.** Criticism quality scales with critic specificity — in duel mode, each explored direction gets a critic with a biography: an antagonist shaped by the failures of exactly that idea's class, not a fixed adversarial role stretched to fit every idea.

**vs. alternatives:** tools like [agent-review-panel](https://github.com/wan-huiyan/agent-review-panel) review a document holistically, as one blob. Multi-model council tools require multiple model providers by default. `consensus-claude`'s unit of judgment is the individual claim, and it needs exactly one model to do it.

**Heads-up on cost:** one full run = ~6-11 calls on your strongest available model (decomposer + 4 role votes + judge, plus targeted round-2 re-votes when theses are disputed). Expect several minutes and meaningful quota use on subscription plans. Built for decisions that deserve it — not for quick questions.

## When to use this skill

Reach for the panel when:

- The decision is complex and multi-faceted — e.g. "monolith → microservices?"
- You need explicit trade-off exploration — e.g. "Rust vs Go for this service"
- You're de-risking a choice that's critical or hard to reverse
- You want to surface unknown unknowns in a plan before committing to it

For quick questions, just ask directly — the panel is deliberately heavyweight. On an expensive or ambiguous question, run with `--plan` first to steer the frame before you pay for a full panel — this matters most in duel mode, where `--plan` lets you steer which perspectives get minted before paying for duels.

---

## Example: PostgreSQL → Redis session storage?

The full run is in [`examples/session-storage-demo.md`](examples/session-storage-demo.md) — real, unedited, run against the prompts shipped in this repo. Here's what the machinery actually produces.

**Question:** *Should we migrate our web app's session storage from PostgreSQL to Redis, or keep sessions in Postgres and only add a cache layer for hot paths?*

### Decomposition

```
CANONICAL INTENT: User is choosing between a full session-store migration and a lower-risk
incremental fix (keep Postgres, cache hot paths); the real trade-off is whether current
session-storage pain is severe enough to justify a migration's cost and operational risk
when a cheaper targeted alternative exists.

T1: Session storage in PostgreSQL is currently a real performance or scaling bottleneck.
T2: Adding a cache layer only for hot paths resolves that bottleneck sufficiently.
T3: Redis materially outperforms PostgreSQL for this application's session workload.
T4: The team can operate Redis in production without unacceptable added operational burden.
T5: Introducing Redis as a new stateful component is justified by the expected gains.
T6: Keeping sessions in PostgreSQL avoids data-consistency and durability complications that
    Redis would introduce.
```

**→ note:** the binary question became 6 independent criteria — no `T1: pick Redis / T2: pick Postgres` mirror pair. The Canonical Intent rides along with every thesis to every voter, so no role loses the whole while judging its slice.

### Votes that show the machinery working

**Skeptic on T5 — catching circular justification:**

```
T5: DISPUTED Self-contradiction with T1/T3 — this thesis ("gains justify the new component")
    cannot be true if T1 (bottleneck is real) and T3 (Redis materially outperforms) are
    themselves unproven. T5 presumes established "expected gains" while T1/T3 supply no
    measured baseline from which any gain could be computed. Justification is circular
    absent data.
```
**→ note:** the Skeptic isn't objecting to Redis — it's flagging that the justification thesis depends on two other theses that are themselves unproven. That's a structural catch a holistic "thumbs up/down" review would miss.

**Security abstaining out-of-lens on T1–T3, then landing a CONDITION on T4:**

```
T1: AGREED — outside my lens, deferring to the relevant role.
T2: AGREED — outside my lens, deferring to the relevant role.
T3: AGREED — outside my lens, deferring to the relevant role.
T4: AGREED_WEAK Redis's default-no-auth network boundary is an authz gap (Unvalidated trust
    at a boundary). CONDITION: Redis bound to private network/localhost with requirepass or
    ACLs+TLS, never default no-auth.
```
**→ note:** Security correctly stays out of performance theses it has no lens on — but on T4 it names the exact failure: default-no-auth Redis reachable on a network means an attacker can `KEYS */GET` to dump and replay live session tokens, i.e. session hijacking. This is the mandatory attack vector the Security role must always produce.

**Optimizer's T2 CONDITION tag:**

```
T2: AGREED_WEAK Read-through cache on existing Postgres reuses a well-understood pattern and
    reduces added state vs. a store swap. AGREED_WEAK not AGREED because "sufficiently" is
    load-dependent. CONDITION: holds if the hot-path read/write ratio is skewed enough that
    caching absorbs the contention (i.e., sessions are read far more than written).
```
**→ note:** conditional agreement expressed as a tag inside the rationale, not a 5th status — the Judge extracts every `CONDITION:` line across all votes and surfaces them as prerequisites in the final synthesis.

**A DISAGREEMENT block — Maintainability-advocate vs. Optimizer on T5:**

<details>
<summary>Maintainability-advocate's counter (click to expand)</summary>

```
DISAGREEMENT:
TARGET: Optimizer
THESIS: T5
COUNTER: Optimizer will likely favor Redis on idiomaticity/performance grounds, but the
decisive cost is structural: adding Redis as a second system of record permanently splits
session truth across two stores, so every future change to session semantics (expiry, shape,
failure handling) becomes a dual-store consistency edit — a coupling tax paid on every
change, not a one-time migration cost, and it's being justified by T1/T3 gains that remain
unmeasured.
```
</details>

**→ note:** "coupling tax on every change, not a one-time cost" — this is the mandatory structured disagreement format (`TARGET`/`THESIS`/`COUNTER`), citing a concrete mechanism instead of a slogan.

### The Judge's Holism check

<details>
<summary>Full Holism check finding (click to expand)</summary>

```
The recommended path (T2) partially erodes the very property that makes the recommendation
safe (T6): T6's value is "one system of record ⇒ one place to reason about session truth,"
and T2 introduces a second piece of runtime state (the cache) that can diverge from that
record — the classic stale-session-after-logout bug. The whole is *not* the sum of its
parts: "Postgres + cache" preserves T6's single-truth guarantee **only if** the cache is
pure read-through, source-of-truth-in-Postgres, invalidated on write/logout. The moment it
becomes write-through/write-back, it reintroduces exactly the dual-store coupling the panel
rejected Redis to avoid. This constraint is load-bearing, not cosmetic.
```
</details>

**→ note:** the panel's own recommended path (T2, the cache) partially eroded the property that made it safe in the first place (T6, single source of truth). The Judge caught it and promoted a soft nuance into a load-bearing constraint. This is what per-thesis voting plus an explicit holism guard buys you — a check that individual per-claim votes, by construction, don't perform on their own.

### Net verdict

The panel favors **keep Postgres + add a scoped read-through cache** over a Redis migration — conditional on confirming the pain is real and read-dominated *first*. If the bottleneck (T1) is never substantiated, even the cache work is premature optimization.

**Concrete unblocking step:** capture five numbers on the current Postgres — session-table p99 read/write latency, QPS, read:write ratio, connection-pool saturation, autovacuum/WAL pressure from session-row updates. Those five numbers decide T1, populate T3, and tell you whether the answer is "cache the hot reads," "fix the write pattern" (a cache is the wrong tool), or "no action needed yet."

Full unabridged run: [examples/session-storage-demo.md](examples/session-storage-demo.md)

*Honesty note: the demo's optional targeted R2 was not run — the Judge finalized from R1, handling still-disputed theses through the Unresolved and Prerequisites sections, exactly as the mandate allows. A production run re-votes disputed theses in R2 before finalizing.*

LLM output is non-deterministic — your run on the same question will differ. What the skill guarantees is the structured process (isolated votes, forced disagreement, holism check), not identical prose. Weak verdict? Open an issue and attach the run record.

---

## How it works

```
Triage → Decompose → Vote R1 (parallel, isolated) → Judge aggregate
        → Vote R2 (disputed theses only) → Judge synthesize
```

After decomposition, one triage question forks the flow into two ceremonies: **panel** (vote on a formed decision, the flow above) or **duels** (explore an open problem — see [Duel mode](#duel-mode-explore-dont-just-evaluate) below). `--mode` forces the choice instead of relying on triage.

Decomposition itself is no longer a single pass — an internal Lumper-vs-Splitter negotiation over thesis grain settles per seam, on whichever argument is stronger, and the result surfaces as an always-visible trace line:

```
GRAIN: 6 theses (Splitter proposed 7, Lumper proposed 6) — ...
```

The full negotiation goes into the run record, not the trace line. `--grain fine|coarse` biases the negotiation toward more or fewer theses.

**Golden-standard roster** (4 roles, active by default):

| Role | Lens |
|---|---|
| `Optimizer` | correctness, readability, idiomaticity, simplification |
| `Skeptic` | holes, contradictions, unrealistic assumptions *(adversarial)* |
| `Security` | threat model, attack surface, secrets, authn/authz, supply chain |
| `Maintainability-advocate` | future-change cost, decomposition, naming, hidden coupling |

Plus two utility agents outside the vote: `Decomposer` (splits the question into theses, and surfaces the question's own unstated premise as T0 — a foundational thesis voted like any other) and `Judge` (aggregates, then synthesizes).

**Votes carry structured tags.** Statuses stay at 4 — tags ride alongside them. `FLIP:` is mandatory on every `DISPUTED` vote and names the evidence that would reverse it; `ANCHOR: "quote"` grounds a vote in a specific piece of the question or context instead of a bare opinion; `[impact: critical|moderate|minor]` ranks how much the outcome hinges on that thesis. The Judge weights disputes carrying a `FLIP:` tag higher, flags votes with no `ANCHOR:`, and prioritizes critical-impact conflicts first.

**Panel invariants** (the panel's constitution):
- **Adversarial Presence** — the panel must contain ≥1 adversarial role (Skeptic by default).
- **Size** — 3 to 7 active roles.
- **No redundancy** — no two roles with heavily overlapping mandates active at once.

**Output** is an 8-section synthesis: Consensus, Holism check, Trade-offs, Blockers, Prerequisites, Nuances, Unresolved, Devil's advocate. It now opens with `CONSENSUS_STRENGTH` — `Strong consensus` / `Working consensus` / `Narrowly carried` / `Contested` — so you know how much weight the verdict can bear before reading a single section.

Every run writes a Verifiable Decision Record — a structured markdown transcript (question → intent → votes → conditions → dissent → verdict) — to `./consensus-runs/`, opt out with `--no-record`. The record is the artifact to attach when reporting a weak verdict. Each record opens with an `AUDIT_BLOCK` — a machine-readable summary (date, mode, roster, consensus strength, verdict, key assumptions) — so you can paste the whole record into any external model and ask it to audit the panel's reasoning, no re-run required. Every run also appends one line — date, question, strength, verdict, link — to `./consensus-runs/INDEX.md`, turning your engineering decisions into a browsable ledger over time.

`SKILL.md` is self-describing — its canonical Protocol block at the top **is** the specification the orchestrator executes. Read that one block and you know the whole system.

---

## Composable panel

The golden-standard roster above is active by default. An optional shelf of roles ships disabled:

| Role | Mandate | when_to_enable |
|---|---|---|
| `Performance-hawk` | Latency, throughput, resource consumption | Perf-critical / real-time code |
| `Product-thinker` | User value, JTBD, why this should exist | Product and spec decisions |
| `Scope-cutter` | Trims MVP excess, names what to cut | Risk of over-build, scope creep |
| `Simplicity-advocate` | Complexity beyond the value it delivers is a bug | Author/panel prone to over-engineering |
| `Domain-expert` | Pluggable domain lens | A narrow, specialized domain |

Enabling a role means editing the roster list in `SKILL.md`'s Protocol block — markdown is the config, there's no loader and no runtime machinery to route around. Skeptic is the default satisfier of the Adversarial Presence invariant; `Security` and `Scope-cutter` also qualify if you swap Skeptic out.

**Tuning roles:** the agent files ARE the source — editing mandates and operational heuristics in `~/.claude/agents/consensus-claude-*.md` is a supported customization path, not a hack. Keep the vote-format contract (statuses + `CONDITION:` tag + `DISAGREEMENT` block) intact so the Judge can still parse it.

Full role metadata (`when_to_enable` / `conflicts_with`) is in [DESIGN.md · Role Library](DESIGN.md).

---

## Duel mode (explore, don't just evaluate)

When triage reads the question as exploratory rather than a formed decision, it routes here instead of the panel. The Decomposer generates the 3-5 strongest perspectives the question actually calls for, then mints a pair for each: a Champion to argue it, and a Tailored Critic — a critic with a biography, purpose-built for that specific idea (not a generic skeptic; an antagonist shaped like "you have watched three rewrites die" for a rewrite-the-service idea), FLIP-disciplined against strawmen. Each pair gets one isolated exchange.

The Judge grades survival on four levels — `clean` / `scarred` / `gutted` / `unproven` — feeding into the same `CONSENSUS_STRENGTH` line the panel uses. A Champion's footnote (one sentence from the Judge on how the Champion would have answered its fatal flaw) fixes last-word bias at zero extra cost, and the Judge also names the un-championed direction — the option nobody was minted to defend. Rejected perspectives don't just disappear: their `FLIP:` tags invert into `Revisit if...` tripwires.

The duel structure follows the author's psychodrama practice: every idea deserves its own antagonist, and the stage decides who enters.

Cost is comparable to a full run (~9-13 calls). `--plan` is recommended here even more than in panel mode — steering which perspectives get minted before paying for duels is cheaper than re-running them. `--emergent` is now a deprecated alias for `--mode duels`.

A full real duel run — four critics with biographies independently converging on a verdict that broke the question's own premise: [`examples/duel-demo.md`](examples/duel-demo.md).

---

## Install

**Requirements:** Claude Code (the skill orchestrates panel roles via its subagent tool).

```bash
git clone https://github.com/CryptoColobrod/consensus-claude.git && cd consensus-claude
mkdir -p ~/.claude/skills/consensus-claude ~/.claude/agents
cp SKILL.md DESIGN.md ~/.claude/skills/consensus-claude/
cp agents/consensus-claude-*.md ~/.claude/agents/
```

> Agent files must land at the **top level** of `~/.claude/agents/` — Claude Code does not scan subdirectories for agent definitions.

> A full panel run makes several calls on your strongest model and takes a few minutes — see the cost note above before your first run.

**Model:** the agents inherit your current session's model — there's no separate Opus key or config to set. The skill works best on the strongest apex model available in your session and degrades gracefully on smaller ones.

**Usage:**

```
/consensus-claude <your question>
```

or say "claude consensus" / "consensus panel" in a session.

**Flags:**

| Flag | Effect |
|---|---|
| `--with-external` | Force the external critic regardless of unanimity |
| `--save-adr` | Write an ADR file for this run (off by default) |
| `--no-record` | Skip writing the run record file |
| `--plan` | Dry-run checkpoint — Triage + Decompose only, then stop for approval before any votes are cast |
| `--roles` | One-run roster override — `+Shelf-role`, `-Role`, `+name(focus:'...')` for an ephemeral persona |
| `--mode` | Force `panel` or `duels` instead of relying on triage |
| `--grain` | `fine` or `coarse` — biases the Lumper/Splitter grain negotiation during decomposition |
| `--emergent` | Deprecated alias for `--mode duels` |

**External critic (optional, any model):** used only as the anti-groupthink fallback when the panel goes fully unanimous on a security-adjacent thesis. Reached via a small ladder: configure any CLI that takes a prompt and prints text (one line in `SKILL.md`'s Protocol block — the [`gemini` CLI](https://github.com/google-gemini/gemini-cli) works out of the box as the default worked example), or skip the install entirely and use the built-in copy-paste handoff — the skill prints the critic prompt for you to paste into any other model you have, then paste the reply back. Decline either way and the run continues gracefully, with the status reported honestly (e.g. `skipped (user_declined)`) rather than failing.

---

## Update / Uninstall

**Update:**

```bash
cd consensus-claude && git pull
```

then re-run the two `cp` commands from the install block above.

**Uninstall:**

```bash
rm -rf ~/.claude/skills/consensus-claude
rm ~/.claude/agents/consensus-claude-*.md
```

The `consensus-claude-*` prefix on every agent file makes the second glob safe — it won't touch any unrelated agent files in `~/.claude/agents/`.

**Footprint:** the skill adds 6 prefixed entries to your global agent list (`~/.claude/agents/consensus-claude-*.md`) plus `SKILL.md`/`DESIGN.md` under `~/.claude/skills/consensus-claude/`. Runs also write records to `./consensus-runs/` in whatever working directory you ran the skill from — those are plain markdown files and can be deleted freely at any time.

---

## Honest scope

v1 ships **one** deeply-built panel — the code/engineering-decision preset. That's deliberate: a small, complete spike beats a wide, half-built system.

What's designed but not built — a dynamic role library with auto-assembly, automated relevance-weighting, a dedicated scope-triage agent, an `ANTIBODIES.md` failure-memory file, and additional presets (architecture / spec / strategy) — lives in [DESIGN.md · Future Arc](DESIGN.md) as a visible roadmap, not as half-wired code.

Output follows the user's language; internal status tokens (`AGREED`, `DISPUTED`, etc.) always stay English.

---

## License

MIT — see [LICENSE](LICENSE).

`consensus-claude` is a community project, not affiliated with or endorsed by Anthropic.
