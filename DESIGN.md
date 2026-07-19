---
created: 2026-05-21
updated: 2026-07-18
type: design
status: mvp
tags: [skill, consensus, multi-agent, claude-only]
related: [[consensus-protocol]]
---

# `consensus-claude` — Design

A Claude Code skill: multi-perspective consensus via subagents with distinct roles. One apex model, different system prompts. Runs within a Claude subscription (no extra API cost). Complements `consensus-protocol` (Claude+Gemini) — doesn't replace it.

## Purpose and boundaries

**What this is:** a cheap, fast multi-perspective review of architecture decisions, code, specs, and strategic choices — powered by a single apex model split across roles.

**What this is NOT:** not a replacement for `consensus-protocol`. They offer different value:
- `consensus-claude` — an internal sanity check, free within your subscription, single model → fast convergence.
- `consensus-protocol` — an external audit with a genuinely different model (Gemini), for cases where you need a **guaranteed** independent voice.

## Architecture decisions (made during brainstorming)

1. A single tool with auto-classification of the task (architecture | code | spec | strategy | other).
2. Classification happens without asking the user for confirmation, with manual override available after it's shown.
3. Panel size is adaptive (3–5 subagents) based on a complexity heuristic.
4. Round structure is hybrid: parallel R1 → judge finds disputes → R2 covers only disputed theses.
5. Anti-groupthink via Gemini — an optional fallback for three specific cases (see section 4).
6. Relationship to `consensus-protocol` — a separate new skill; the old one stays.
7. Debate mode is thesis-based (atomic theses T1..Tn with statuses AGREED/AGREED_WEAK/DISPUTED/NEEDS_CLARIFICATION), same as `consensus-protocol`.

## 1. Flow architecture

```
[User invokes /consensus-claude or the phrase "claude consensus"]
   ↓
[Main Opus] Classifies the task → selects a panel preset
   ↓
[Main Opus] Scores complexity → selects panel size (3-5)
   ↓
Prints: "Classified as: X · Panel: A / B / C / D (+Judge)"
(user can override via !preset / use:preset before R1)
   ↓
[Decomposer subagent] Cuts the task into atomic theses T1..Tn
   ↓
R1 in parallel: N subagents independently assign a status to each thesis
   ↓
[Judge subagent] Aggregates → agreement/dispute table
   ↓
[Anti-groupthink check] Does the Gemini fallback trigger?
   ├─ yes → Gemini runs an independent check → its DISPUTED theses feed into R2
   └─ no → skip
   ↓
R2 in parallel: disputed theses only, agents see each other's R1 positions
   ↓
[Judge] Final synthesis: consensus + nuances + trade-offs + devil's advocate
   ↓
Output: chat summary (always) + ADR file (for arch/sec/infra)
```

**Principles:**
- One apex model throughout. Not Sonnet, not a mix.
- R1 parallelism = isolation. Each subagent sees only: the original question + the list of theses + its own role mandate. It does not see other roles or their positions.
- The judge is a separate subagent — not the main Opus. Clean aggregation context.
- Hard cap: R1 + R2, never R3.

## 2. Role presets

Each role is a separate agent definition in `~/.claude/agents/consensus-claude-<role>.md` (standard Claude Code subagent pattern, following the same format as other agent definitions in `~/.claude/agents/`). **Important:** Claude Code only scans the top level of `~/.claude/agents/` — subdirectories are not supported. That's why all 6 files carry the `consensus-claude-` prefix, to avoid colliding with other agents.

| Task type | Base (4) | Cut (3, simple) | Add (5, complex) |
|---|---|---|---|
| `architecture` | Architect / Skeptic / Security / Pragmatist | drop Security | +Operations-SRE OR +Domain-expert |
| `code` | Optimizer / Skeptic / Security / Maintainability-advocate | drop Maintainability | +Performance-auditor |
| `spec` | Product-thinker / Architect / Skeptic / Scope-cutter | drop Scope-cutter | +User-advocate |
| `strategy` | Strategist / Skeptic / Customer-advocate / Pragmatist | drop Pragmatist | +Numbers-person |
| `other` | Architect / Skeptic / Pragmatist (3) | — | +ad-hoc role |

**Skeptic is present in every preset** — an anti-groupthink requirement.

**Style-reviewer** (originally in `code`) was replaced with **Maintainability-advocate** based on Gemini feedback: maintainability is broader than style and doesn't overlap with Optimizer.

### Mandate per role (key idea: orthogonal lens + mandatory disagreement)

| Role | Focus | Off-limits | Required action |
|---|---|---|---|
| Architect | Structure, module boundaries, contracts, evolution | Don't speak to cost/security/UX | Find a point of disagreement with one panel member; prefer Pragmatist if present, otherwise anyone except Judge/Skeptic |
| Skeptic | Gaps, contradictions, unrealistic assumptions | — | Surface at least one weakness even on good decisions; if there truly is none, flag edge cases |
| Security | Threat model, attack surface, secrets, authn/authz, supply chain | Stay silent on performance | Find at least one attack vector |
| Pragmatist | What blows the deadline, what can be cut; enemy of over-engineering | Don't speak to code aesthetics | Find a point of disagreement with one panel member; prefer Architect if present, otherwise anyone except Judge/Skeptic |
| Optimizer | Correctness, readability, idiomaticity, simplifications | — | Find a point of disagreement with one panel member; prefer Maintainability-advocate if present, otherwise anyone except Judge/Skeptic |
| Maintainability-advocate | What makes code easy to maintain in 6 months; decomposition, naming, hidden dependencies | Don't duplicate style conventions | Find what will complicate future changes |
| Product-thinker | Whether the feature is needed at all, what pain it solves, who the user is | Don't drift into implementation | Question whether the task should exist |
| Scope-cutter | What to cut from the spec, where the MVP line is | Don't critique architecture | Name at least 1 item to cut |
| Strategist | Market positioning, priorities, success criteria | Don't drift into tactics | Find a point of disagreement with one panel member; prefer Customer-advocate if present, otherwise anyone except Judge/Skeptic |
| Customer-advocate | The voice of the user/customer, what actually hurts | Don't speak to unit economics | Name at least 1 frustration |
| Numbers-person | Unit economics, capacity, concrete figures | Don't speak to vision | Request a missing figure |
| Operations-SRE | Monitoring, failure modes, operability, postmortems | Don't speak to feature scope | Find at least 1 operational risk |
| Domain-expert | Technical depth on a specific topic (chosen ad-hoc) | — | Accepts `--domain-context "Kafka, Postgres 14"` via flag |
| **Judge** | Aggregation, synthesis, trade-offs, devil's advocate | Don't introduce new theses | Must write a "what could go wrong" section; must handle directional theses (criterion-style theses with an implicit direction, not just binary AGREE/DISAGREE) |
| **Decomposer** | Cuts the source task into atomic theses T1..Tn before R1 | Don't evaluate theses; don't propose solutions — decomposition only | Every thesis must be independently AGREEable/DISPUTEable; avoid compound statements; target 3–7 theses for a typical task |

**Why Decomposer is a separate subagent rather than inline in the main Opus.** The quality of the whole process hinges critically on decomposition quality (a foundational vulnerability flagged by Gemini review). A separate subagent with a clean context and a strict single-purpose mandate produces better decomposition than the main Opus, which is holding the entire process in its head. Same logic for the Judge — splitting it into its own subagent is a structural safeguard against role bleed.

## 3. Classification and complexity heuristics

### Classification (cascade, first match wins):

1. Explicit preset in the input (`/consensus-claude --preset architecture`).
2. **Artifact signals:**
   - a diff/patch/code appears in the message → `code`
   - `*-design.md`, `*-plan.md`, `*-spec.md` mentioned or shown → `spec`
   - an ADR or a mention of choosing a database/infra/protocol → `architecture`
   - pricing/marketing/business model/features → `strategy`
3. **Keywords in the phrasing:** "architecture", "choose a database", "trade-off" → `architecture`; "review", "check this code", "PR", "optimize" → `code`; "spec", "plan", "brainstorm" → `spec`; "priority", "positioning", "pricing tier" → `strategy`.
4. **Fallback** → `other`.

The first output line is always: `Classified as: architecture · Complexity 4/5 · Panel: Architect / Skeptic / Security / Pragmatist / Operations-SRE (+Judge)`.

**Manual override:** the user can override the classification in a single line **before R1 starts**: `!preset code`, `use:spec`, `--preset strategy`. Once R1 has started it's too late — you have to restart.

### Adaptive size (signal counter):

| Signal | +1 |
|---|---|
| Mentions security / auth / secrets / PII | ✓ |
| Mentions scaling / capacity / latency | ✓ |
| ≥3 atomic theses after decomposition | ✓ |
| Explicit trade-off with multiple options (≥2) | ✓ |
| Domain-specific (database engine, distributed systems, ML) | ✓ |

| Score | Size |
|---|---|
| 0–1 | **3** (cut) |
| 2–3 | **4** (base) |
| 4–5 | **5** (add) |

## 4. Anti-groupthink (layered defense)

The single-model echo chamber is the **main irreducible risk.** It can't be fully eliminated, but it's mitigated by six layers:

**Layer 1 — R1 isolation.** Each subagent in R1 sees only the original question + theses + its own mandate. It doesn't see other roles or their answers. Enforced by the `Agent` tool (each subagent runs in a clean context).

**Layer 2 — Mandatory disagreement (dynamic, structured).** Every role is instructed: "find a point of disagreement with one panel member; prefer <specific role>; if absent, anyone except Judge/Skeptic." The dynamic phrasing survives panel cuts. This isn't passive criticism — it's an active search for contradictions.

**Disagreement output format — a machine-parseable block** (validated in PoC 0.2):
```
DISAGREEMENT:
TARGET: <role name>
THESIS: T<n>
COUNTER: <1–2 sentences citing a concrete mechanism — coupling/missing risk/unproven abstraction, not a slogan>
```
If there truly is no disagreement, the fallback is `COUNTER: NO_DISAGREEMENT — T<n> is robust because: (1) <criterion>, (2) <criterion>, (3) <criterion>` (an escape hatch of last resort).

Pragmatist was given an **operational definition of over-engineering**: "an abstraction is over-engineered if its specific need hasn't materialized ≥2 times in real code/incidents." This turns YAGNI slogans into auditable claims.

**Layer 3 — Skeptic must deliver.** Skeptic can never return "everything's fine." At minimum, one weakness or a set of named edge cases.

**Layer 4 — Devil's advocate section from the Judge.** The final report always includes a "What could go wrong" section — 1–2 strong arguments against the resulting consensus.

**Layer 5 — Gemini fallback.** Triggers if **any one** of these holds:
- All theses are AGREED **or** AGREED_WEAK after R1 (100% "weak or strong" agreement is a groupthink red flag, per Gemini review feedback).
- Task type is `architecture` OR the task mentions security/PII.
- Explicit `--with-gemini` flag.

If Gemini fires and returns DISPUTED theses, those theses automatically enter R2 — even if the Opus agents were unanimous.

**Implementation — via the Gemini CLI (not MCP).** The main Opus calls `timeout 30s gemini -p "<prompt>" 2>/dev/null` via the Bash tool (wrapper validated in PoC 0.3). Reasons:
- **Cheaper** — OAuth via a Google AI subscription plan, not per-call API billing. Both resources (the Claude subscription and the Google AI subscription) run on a flat-rate model.
- **Newer model** — `gemini-3.1-pro-preview` (the CLI default) vs. `gemini-2.5-pro` in the MCP path.
- **Less overhead** — no auto-save to a reviews directory, which an internal fallback doesn't need.

Prompt template for the CLI:
```
You are an external critic. For each thesis below, return ONE line in format:
T<n>: <AGREED|AGREED_WEAK|DISPUTED|NEEDS_CLARIFICATION> [<comment if not AGREED>]

Theses:
T1: ...
T2: ...
```

Response parsing is a regex over the statuses. **Inside `consensus-claude`, all Gemini calls go through the CLI only.** The MCP tools (`mcp__gemini-tools__*`) are not used by this skill.

**Layer 6 — Trade-offs in the final output.** The Judge must explicitly spell out key trade-offs: "T2 — Architect chose extensibility, Pragmatist flagged a +2 week schedule risk. Architect's call was adopted, risk acknowledged." This prevents quietly burying the losing side of an argument.

**Known limitations (stated explicitly):**
- The `Agent` tool in Claude Code doesn't let you vary temperature per subagent → one classic anti-groupthink technique is unavailable.
- Every agent is the same model → shared blind spots can't be fully ruled out. The Gemini fallback closes this gap partially, not completely.

## 5. Stop conditions and stagnation

Priority, top to bottom:

| Condition | Action |
|---|---|
| After R1: all AGREED/AGREED_WEAK + Gemini fallback didn't trigger OR Gemini also agrees | **Early finalize**, skip R2 |
| After R1: all AGREED/AGREED_WEAK, but Gemini surfaced DISPUTED theses | R2 runs **only** on the theses Gemini disputed |
| After R1: some theses are DISPUTED/NEEDS_CLARIFICATION | R2 runs on those theses |
| After R2: all AGREED/AGREED_WEAK | **Finalize.** AGREED_WEAK nuances → "Nuances to consider" |
| After R2: some theses still DISPUTED | **Finalize with a flag.** "Unresolved" → its own section, arguments from both sides, escalate to the user |

**Stagnation in R2:** if a thesis's agent arguments just repeat R1's arguments with no new substantive information (detected by the Judge via semantic overlap), that's stagnation. The Judge does not run another round — it marks the thesis "stagnated" and escalates to the user.

**Hard cap:** R1 + R2 maximum. We do not attempt R3.

**Rationale for the hard cap:** if a single-model panel hasn't converged after two rounds, that's a signal to pursue **external** review via `consensus-protocol` — not to repeat the same panel again.

### Edge cases

- **The task doesn't decompose into atomic theses** (e.g., "what do you think of this code?") → fall back to *single-thesis* mode: the whole task becomes one thesis T1, the panel responds freely, and the Judge aggregates. Less valuable, but doesn't break.
- **A subagent returns garbage or an invalid status** → the Judge marks that vote `invalid`; the thesis is scored on N-1 votes. If invalid votes exceed 50%, abort and notify the user.
- **Gemini CLI is unavailable** (non-zero exit, expired OAuth token, binary not found, timeout) → skip the fallback, log `gemini_cli_unavailable`, proceed to R2 with Opus agents only. Report this to the user in the chat summary: `Gemini fallback: skipped (CLI unavailable)`. Special case: exit code **42** = empty/invalid input (a PoC finding) — detected separately as `gemini_invalid_input`.
- **Cross-language quirk** (PoC 0.3): Russian input → Gemini returns English output with the correct T<n>: STATUS format. This is a feature for anti-groupthink purposes (a different language is a different angle), but if the final output needs comments in the user's language, the Judge must translate them during synthesis.

## 6. Output

### Chat summary (always)

```
🧠 consensus-claude
Classified as: architecture · Complexity: 4/5 → Panel (5):
  Architect / Skeptic / Security / Pragmatist / Operations-SRE (+Judge)
Gemini fallback: triggered (architecture+security)

Round 1/2:
  Agreed: T1, T3, T5
  Disputed: T2, T4
  AGREED_WEAK nuances: T1 (Skeptic), T3 (Pragmatist)
  Gemini surfaced: T2 — DISPUTED (new argument)

Round 2/2 (T2, T4 only):
  T2 → accepted with an amendment
  T4 → stagnated, escalating

✅ Consensus:
  T1: <final thesis>
  T2: <final thesis, amended>
  T3: <final thesis>
  T5: <final thesis>

⚠️ Nuances to consider:
  T1 — <AGREED_WEAK from Skeptic>
  T3 — <AGREED_WEAK from Pragmatist>

🔀 Key trade-offs:
  T2 — Architect chose extensibility, Pragmatist flagged a +2 week schedule risk

❓ Unresolved (needs your call):
  T4 — <gist>
    Pragmatist: <position>
    Security: <position>

😈 Devil's advocate (from the Judge):
  1. <strong counter-argument>
  2. <second strong counter-argument>
```

### ADR file (architecture / security / infrastructure only)

Path: `docs/decisions/YYYY-MM-DD-<topic>.md` (or `documentation/decisions/` if that's the project's convention).

```markdown
---
created: YYYY-MM-DD
type: ADR
source: consensus-claude
classification: architecture
complexity: 4
panel: [Architect, Skeptic, Security, Pragmatist, Operations-SRE]
gemini_invoked: true
---

# Decision: <topic>

## Context
<original question>

## Decision
<final agreed-upon theses>

## Nuances to consider during implementation
<all AGREED_WEAK comments>

## Key trade-offs
<trade-offs, attributed to roles>

## Debate history
- R1: agreed on T1/T3/T5, disputed T2/T4
- Gemini fallback: added a DISPUTED flag on T2
- R2: T2 accepted with amendment, T4 stagnated

## Devil's advocate
<strong counter-arguments — even if not adopted>

## Unresolved
<T4 — escalated to the user>

## Rejected alternatives
<what roles proposed but didn't make the cut, and why>
```

For `code` / `spec` / `strategy` — chat summary only. Artifacts for these task types live elsewhere (PR comments, design.md, notes in your knowledge base).

## 7. Triggers and integration

### Slash command

```
/consensus-claude <question>
/consensus-claude --with-gemini <question>
/consensus-claude --domain-context "Kafka, Postgres 14" <question>
/consensus-claude --preset code <question>
```

### Phrase triggers

| Phrase | Skill |
|---|---|
| "claude consensus", "consensus claude" | **`consensus-claude`** |
| "gemini consensus", "consensus with Gemini", "debate this with Gemini" | **`consensus-protocol`** |
| "need a consensus" (no model specified) | **Ask for clarification:** "claude or gemini?" |

**Auto-launch (Claude deciding on its own) is NOT supported.** The skill only starts on an explicit trigger phrase or slash command. This is a deliberate user-control choice — every trigger is explicit.

The phrase mapping lives in the user's global Claude Code configuration, in the "Gemini — phrase-to-tool mapping" section. The existing entry for `consensus-protocol` stays; an entry for `consensus-claude` is added alongside it.

### Integration with other skills

- `superpowers:brainstorming`, after producing a spec, **may suggest** running `/consensus-claude --preset spec` as a self-review step (an alternative to the existing gemini_review path). The user decides.
- `superpowers:writing-plans` — same pattern.
- `superpowers:requesting-code-review` — same pattern, `--preset code`.

Integration means a suggestion, never an auto-launch.

## Open questions for implementation

(These go into writing-plans; they don't block the design.)

1. How does the Judge detect semantic overlap between R1/R2 arguments for stagnation — a keyword heuristic, or an additional LLM pass?
2. Final prompt-template wording for the Decomposer, the Judge, and each role (worked out in PoC).
3. MVP priority: which preset to start with (Gemini's recommendation — `code`, as the most frequent use case).
4. **Phase 0 (PoC before full implementation):**
   - **PoC-Decomposer:** test decomposition quality on 5–10 real historical tasks (architecture/code-review/specs). Metric: theses are atomic, non-overlapping, and cover the task.
   - **PoC-Disagreement:** with a pair of roles (Architect + Pragmatist), verify that dynamic mandatory disagreement produces a substantive debate, not a formality.
   - **PoC-CLI-fallback:** verify that `gemini -p` reliably returns parseable per-thesis statuses.

## Prior art (for context)

- [agent-review-panel](https://github.com/wan-huiyan/agent-review-panel) — 4–6 reviewers + a supreme judge, anti-groupthink.
- [adversarial-review (ng)](https://github.com/ng/adversarial-review) — Optimizer + Skeptic pairs.
- [adversarial-spec (zscole)](https://github.com/zscole/adversarial-spec) — multi-model (GPT/Gemini/Grok), not a fit here.
- The existing `~/.claude/skills/consensus-protocol/SKILL.md` — the source of the thesis-based protocol and the ADR format.

## Future Arc (v2+ roadmap)

Freezing scope to a single preset (`code`) is a deliberate choice, not an unfinished state. v1 ships a spike: narrow, deep, and carried through to completion — a small, complete solution rather than a scaffold with half-built rooms. Elegance here comes from completeness, not coverage. The section below exists to show the boundary was drawn on purpose, with a clear view of what's beyond it: the orchestration and resilience layer gets built next, in public (build-in-public), not quietly and not because "there wasn't time."

Every item below is tagged with why it's v2 and not v1 — a deliberate-debt signal, not forgotten work.

1. **A full dynamic role library.** Metadata for `when_to_enable` / `pairs_well_with` / `conflicts_with` on every role, plus auto-assembly of the optimal panel for a given request (tailored to its specifics, not just its task type). Why not v1: this requires reliably scoring how relevant a role is to a given phrasing — there isn't enough data from a single preset to calibrate that heuristic without risking an incoherent panel. v1 ships a fixed golden standard (4 roles) plus a static shelf (see Role Library below) that the user edits by hand.

2. **Automated relevance-weighting by competencies.** In v1, the Judge weighs votes through its own reasoning (see Out-of-lens in the Protocol) — it works, but it's opaque and not reproducible across runs. Automating the down-weighting of low-relevance roles (e.g. by matching a role's `competencies` against a thesis's keywords) is v2 work, once there's a corpus of real votes to calibrate the weights against.

3. **A SCOPE.md triage agent.** In v1, triage is the orchestrator reading the Scope section directly out of the Protocol block — it works, but conflates the orchestrator and gatekeeper roles. A dedicated triage agent backed by an extensible `SCOPE.md` (a growing list of in/out patterns that doesn't require editing prompts) is v1.1 — the next step after release, not a full v2.

4. **ANTIBODIES.md.** A declarative list of the skill's own known failure modes — e.g. "trivial decomposition into >7 theses," "groupthink with weak rationales" — injected into the Judge as immune memory during synthesis. This is a category-correct way to "learn" from its own mistakes without writing a line of code: the file grows as real runs surface new findings, and the Judge reads it as a checklist of known failure modes. It's a hallmark of a mature v2 system — v1 captures these lessons as prose in the Post-Phase retrospective (below); ANTIBODIES.md turns that experience into a structured artifact the Judge can actually parse.

5. **The remaining presets (architecture / spec / strategy / other) + auto-classification + adaptive panel sizing 3–5.** Fully designed already (see §2 "Role presets" and §3 "Classification and complexity heuristics" above) — this is an extension of an already-accepted architecture, not a new one. Deliberately excluded from v1: v1 stays frozen on the `code` preset so one panel reaches production quality, instead of five panels reaching mediocre quality. Turning this on is a matter of copying an already-proven pattern onto new roles, not a redesign.

## Role Library (v1: golden standard + optional shelf)

In v1, the panel is assembled in a way that's category-correct for a markdown skill: roles are agent-definition files, not config entries. Enabling or disabling a role means adding or removing it from the roster in the Protocol block of SKILL.md — by editing text, not through a config loader or a runtime flag. This is a deliberate form, not a scaled-down stand-in for a "real" config system — for a markdown-native skill, an editable role list *is* the native way to manage panel composition.

**Golden standard (active by default).** The roster from §2/Protocol: `Optimizer`, `Skeptic`, `Security`, `Maintainability-advocate`. This is the default `code`-preset panel described above — its mandate table isn't repeated here (see §2), only its place in the overall library: 4 roles covering correctness, adversarial pressure, security, and future maintainability — a curated minimum, not a random sample.

**Optional shelf (OFF by default).** Enabled manually, by moving a role from the shelf into the active roster:

| Role | Mandate (1 line) | when_to_enable | conflicts_with |
|---|---|---|---|
| `Performance-hawk` | Latency, throughput, resource consumption | Perf-critical / real-time code | `Simplicity-advocate` |
| `Product-thinker` | User value, JTBD, why this should exist at all | Product and spec decisions | — |
| `Scope-cutter` | Trims the excess from MVP, names what to cut | Risk of over-build, scope creep | — (an adversarial role — can satisfy the Adversarial Presence invariant in place of Skeptic) |
| `Simplicity-advocate` | Any complexity beyond what the value requires is a bug | Author/panel prone to over-engineering | `Performance-hawk` |
| `Domain-expert` | A pluggable domain lens (see `--domain-context` in §2) | A narrow, specialized domain | — |

**Panel invariants** (defined in Protocol · Panel invariants, repeated here for convenience when picking from the shelf):
- **Adversarial Presence** — the active panel must include ≥1 adversarial role; the default is `Skeptic`, but `Security` and `Scope-cutter` also qualify.
- **Size** — 3–7 active roles.
- **No redundancy** — don't enable roles with conflicting mandates at the same time (see the `conflicts_with` column above; e.g. `Performance-hawk` + `Simplicity-advocate` is a direct conflict of priorities, not complementary lenses).

**How to enable one (v1, honest).** Open the Protocol block in SKILL.md, find the Roster section, add the role's name from the shelf to the active list for the relevant preset (or create a panel override). There's no runtime loader, config flags, or role-picker UI — this is a markdown skill, and editing the list as text is the extension seam. To be explicit: what looks like "a missing feature" is in fact v1's architectural extension point.

Deliberate-debt signal: most shelf roles being off by default is curation, not an oversight. An empty or unstructured shelf would suggest the author never thought about extensibility. A shelf with `when_to_enable`/`conflicts_with` metadata on every role shows the opposite: extensibility was designed in, just not forced on as default complexity.

## Known limitations (stated explicitly)

- **Single-model echo chamber.** Every Opus agent is the same model. The Gemini fallback layer closes this gap partially, not completely. This is a deliberate trade-off in exchange for cost and speed.
- **Fragile classification.** The cascade heuristic can misfire. Manual override is mandatory. We're not attempting an ML classifier.
- **Prompt quality = system quality.** 90% of success comes down to the wording of each role's mandate. Requires iterative tuning.
- **No temperature variation.** A technical limitation of the Claude Code Agent tool.

## Post-Phase 1 retrospective (2026-05-22)

**Phase 0 PoC surprises:**
- The Decomposer's initial prompt from PLAN.md failed on binary "A or B" questions — it produced mirror pairs like `T1: choose A / T2: choose B`. Fixed via a criteria-based decomposition rule (3 iterations, up to 10/10).
- Pragmatist initially slid into YAGNI slogans. Fixed via an operational definition: "over-engineered = a specific need hasn't materialized ≥2 times in code/incidents."
- The structured `DISAGREEMENT: TARGET/THESIS/COUNTER` block (validated in PoC 0.2) is machine-parseable and outperforms prose-style disagreement. This structure made it into the production prompts for Optimizer and Maintainability-advocate.
- Gemini CLI cross-language behavior: input in one language → output in English. Initially looked like a bug, turned out to be a feature (a different language is a different angle) — the Judge translates during synthesis.
- Gemini exit code 42 on empty input isn't documented anywhere, but it's detectable as a distinctive sentinel.

**Phase 1 implementation surprises:**
- Subagents must live at the **top level** of `~/.claude/agents/` — subdirectories (`~/.claude/agents/consensus-claude/`) aren't scanned. Solved with a filename prefix. PLAN.md originally had the wrong path — fixed after the fact.
- The list of available subagent_type values is cached at the start of a Claude Code session. New agent files aren't visible until restart. End-to-end smoke testing is deferred to a new session.

**What to watch in Phase 2 (when expanding presets):**
- Skeptic in production might turn out too formulaic — watch for it churning out boilerplate "edge cases: empty input + concurrent write" answers.
- Cross-role mandatory disagreement priority: in `architecture/cut` (3 agents), when Security is dropped, the dynamic fallback looks for disagreement from Pragmatist/Architect instead — already validated in PoC, but watch real runs to make sure this doesn't turn into circular criticism.
- Strategy preset (Phase 2): Gemini Set 2 (criteria-style strategy) returned all NEEDS_CLARIFICATION — which is correct behavior, but the Judge needs to parse this as "insufficient data" and escalate to the user, not flag it as groupthink.
- ADR auto-generation for the `architecture` preset — the path format `docs/decisions/` is already defined, but projects use different conventions (`documentation/decisions/`, `docs/adr/`) — needs a heuristic to detect the existing structure before writing a new ADR.
