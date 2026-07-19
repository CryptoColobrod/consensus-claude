---
name: psychodrama-protocol
description: The Psychodrama Protocol — adversarial decision review via single-model subagents with distinct roles. Put your decision on stage: per-thesis panel votes for formed decisions, duels with critics-with-biographies for open problems. Use when the user says "psychodrama" / "consensus panel" or invokes /psychodrama-protocol. Pulls in an external critic (any CLI, e.g. Gemini) as anti-groupthink fallback.
---

# The Psychodrama Protocol

Put your decision on stage: a panel of adversarial roles from a single model debates it — thesis by thesis, with critics built for each idea. Complements `consensus-protocol` (Claude+Gemini), does not replace it.

---

## Protocol (canonical — single source of truth)

> Reading only this block, both a human and the orchestrator LLM understand the whole skill:
> what it does, what it's made of, what makes it unique, what rules it operates under.
> Everything below in the file is execution of this contract, not new rules.

### What it is — one line

The Psychodrama Protocol puts a question on stage through a panel of roles from **a single model**
(not a dialogue between different models) and produces a synthesis with disagreement explicitly
preserved.

### Differentiator (the one explicit uniqueness claim)

Voting happens **per atomic thesis** (per-thesis), not on the answer as a whole
(holistic, as with comparable tools). This yields: precise localization of disputes, a
machine-parseable decision trail, and protection against a "lowest common denominator" verdict.

### Phases

Two modes share Triage and Decompose: `panel` (evaluate a formed decision — per-thesis votes, the
backbone) and `duels` (explore an open problem — per-perspective duels).

`Triage → Decompose → [fork on mode] → panel: Vote (R1) → Judge(aggregate) → Vote (R2, disputed only) → Judge(synthesize) | duels: Perspectives → Duels → Judge(synthesize)`

- **Triage** — is the question within the skill's scope? (see Scope below) If not — decline, panel does not run. Also classifies (or asks) which mode applies (see Duels below).
- **Decompose** — the Decomposer produces `Canonical Intent` (the gist of the task in 1–2 sentences) + `T0` (Premise Distillation — the question's foundational unstated premise, voted like any thesis) + atomic theses T1..Tn, settled via an internal Lumper⚔Splitter dialectic with an always-visible `GRAIN` trace line.
- **Vote R1** *(panel mode)* — panel roles vote on each thesis in parallel and independently (context isolation).
- **Judge aggregate** *(panel mode)* — tally the votes, find disputed theses, detect groupthink.
- **Vote R2** *(panel mode)* — disputed theses only; roles see each other's R1 positions, must contribute new substance.
- **Judge synthesize** — holism check, separating tension from error, final synthesis with disagreement preserved (panel mode); or per-duel SURVIVAL + cross-perspective synthesis (duels mode, see Duels below).

### Roster (declared HERE once; steps dispatch "the roster defined in Protocol")

Default active panel (gold standard) — 4 roles:

| Role | Lens (competencies) |
|---|---|
| `Optimizer` | correctness, readability, idiomaticity, simplification |
| `Skeptic` | holes, contradictions, unrealistic assumptions *(adversarial role)* |
| `Security` | threat-model, attack-surface, secrets, authn/authz, supply-chain |
| `Maintainability-advocate` | future-change cost, decomposition, naming, hidden coupling |

Plus two utility agents outside the voting panel: `Decomposer` and `Judge`.

Optional roles (shelved, off by default — see DESIGN.md · Role Library for how to enable):
`Performance-hawk`, `Product-thinker`, `Scope-cutter`, `Simplicity-advocate`, `Domain-expert`.
First shelf role shipped as a file: `Resource-keeper` (opt-in) — the counterweight to an all-critics panel's loss-aversion.

### Panel invariants (the panel's constitution)

- **Adversarial Presence** — the panel MUST contain ≥1 adversarial role (Skeptic / Security /
  Scope-cutter). Default invariant-holder is `Skeptic`. (Replaces the narrow rule
  "Skeptic can't be removed" with the general principle.)
- **Size** — 3 to 7 active roles (diversity of opinion vs. cognitive overload and cost).
- **No redundancy** — do not include roles with heavily overlapping mandates simultaneously
  (e.g. Optimizer + Performance-hawk), see `conflicts_with` in role metadata.

### The figures (deadlock-breakers)

Two non-voting figures live outside the panel and enter only when the stage is stuck: 🕊 the
outside-the-frame voice dissolves a deadlock from above (exits the dispute's terms; output = a
reframe + `dissolves if` lines); 🔥 the transgressive voice breaks it from below (names the
suppressed option nobody dared stage, with its honest price). Triggers: a `REFRAME` cluster (≥2
roles, see Status vocabulary above) / a `STAGNATED` thesis / `CONSENSUS_STRENGTH: Contested`;
`--summon` forces a figure regardless, `--no-figures` suppresses all automatic summoning. One
appearance each; recorded verbatim; neither votes. Cost: +1-2 calls, only on deadlock.

### Status vocabulary (frozen at 4 — do NOT expand)

| Status | Meaning |
|---|---|
| `AGREED` | agrees without reservation |
| `AGREED_WEAK` | agrees, but with a nuance (nuance is mandatory) |
| `DISPUTED` | disagrees (counter-argument mandatory; for roles with a DISAGREEMENT block — an alternative too) |
| `NEEDS_CLARIFICATION` | needs clarification (must state exactly what is unclear) |

**CONDITION tag (instead of a 5th status).** Conditional agreement ("yes, if some external
condition holds") is expressed NOT as a separate status, but as a `CONDITION: <external dependency>`
line inside the rationale of an `AGREED`/`AGREED_WEAK` vote. The Judge must extract all CONDITIONs and
surface them as prerequisites in the synthesis. (Rationale: the tag lowers cognitive load on the voter —
separating the decision itself from its qualification — and parses more precisely than a new status.)

**Out-of-lens.** A role whose lens doesn't fit a thesis states this in its rationale (e.g.
`T3: AGREED — outside my lens, deferring to roles whose competencies match`), rather than
stamping an opinion. The Judge weighs each vote by the relevance of its `competencies` to the thesis.
No separate status is needed for this.

**Other rationale tags.** Alongside `CONDITION`, a vote's rationale may carry: `FLIP: <observable
evidence that would reverse this vote>` — mandatory on every `DISPUTED` vote; it turns a dispute into
a testable position, and the Judge derives revisit-Tripwires from FLIP tags; `ANCHOR:
"<quote>"` — a verbatim quote from the question/spec/code the vote is grounded in (required when the
vote rests on a specific claim in the source); `[impact: critical|moderate|minor]` — the voter's own
severity read, mandatory whenever the vote is `DISPUTED` or `AGREED_WEAK`; `REFRAME: <the dispute
dissolves if the question/frame itself changed>` — a role's signal that the thesis as posed may be
the wrong question, not just a wrong answer. When the Judge's Phase A aggregation finds `REFRAME` on
the same thesis from ≥2 roles (a REFRAME cluster), that's a standing trigger for the 🕊 figure (see
The figures below) — fired early, before R2. All four tags are optional elsewhere and, like
`CONDITION`, live inside the rationale text rather than expanding the status vocabulary.

### Rounds & stop conditions

- **Hard cap: R1 + optional R2. Never R3.** If a single model hasn't converged after 2 rounds —
  that's a signal to escalate to the external `consensus-protocol` (Claude+Gemini), not to spin the panel further.
- R2 only runs if disputed theses remain after aggregation; otherwise — early finalize.
- Stagnation (R2 arguments are a semantic rehash of R1) → the thesis is marked `STAGNATED`, R3 is not invoked.
- The final synthesis opens with `CONSENSUS_STRENGTH` (`Strong` / `Working` / `Narrowly carried` / `Contested`) — the Judge's own one-word read on how solid the verdict is, emitted before the rest of Phase C.
- Deadlock is no longer just a flag: a stuck stage (`STAGNATED` or `Contested`) summons the figures (see The figures).

### Duels (exploration mode)

`duels` mode explores an open problem instead of evaluating a formed decision. The Decomposer's
grain-settled perspectives are minted into pairs: each perspective gets a **Champion** (its
strongest advocate) and a **Tailored Critic** — an antagonist purpose-built for that idea, not a
generic skeptic, disciplined against strawmen by a mandatory `FLIP:` + `ANCHOR:` in its output. Each
pair runs one isolated exchange (Champion's case, then the Critic's fatal-flaw answer) — no
multi-round chat, same hard-cap philosophy as panel mode. The Judge grades each duel's `SURVIVAL`
(`clean` / `scarred` / `gutted` / `unproven`), feeding the same `CONSENSUS_STRENGTH` vocabulary used
by panel mode. Because the Critic strikes last, the Judge appends a one-sentence Champion's footnote
per duel to restore last-word balance.

### Anti-groupthink

- The Adversarial Presence invariant (see above) — a structural safeguard.
- `GROUPTHINK_FLAG` — the Judge raises this if ALL theses are AGREED/AGREED_WEAK across ALL roles.
- Optional external voice: a provider-agnostic external critic as a triggered fallback (conditions —
  see the external-critic step). Reached via a 3-step ladder: (1) a configured CLI command (any
  provider); (2) if no CLI is configured or it fails, a manual paste-through handoff — the
  orchestrator prints the same self-contained critic prompt for the user to paste into any other
  model (web chat included), then paste the reply back; (3) skip, if the user declines. This is the
  only point where the skill touches a second model, and only as a hedge against unanimity, not as
  a default mode (multi-model dialogue is `consensus-protocol`, a separate skill).

**External critic command (the one configurable line):** `gemini -p` — default worked example.
Replace with any CLI that accepts a prompt (as an argument or via stdin) and prints text:
`codex exec`, `ollama run <model>`, etc. all qualify. Leave unset to always use the manual
handoff (Rung 2) when the fallback triggers.

### Run record

Every run writes a Verifiable Decision Record by default (`--no-record` to skip) — see the
dedicated step for the file location and content schema. The record now opens with a
machine-readable `AUDIT_BLOCK`, and every run additionally appends one line to
`./consensus-runs/INDEX.md` — the decision journal across all runs.

### Scope (boundaries; used in the Triage phase)

**In scope:** review and trade-off analysis of architecture/code/spec/strategy decisions described in text;
risk-finding (correctness, security, maintainability, cost).
**Out of scope:** generating turnkey applications; legal/financial/medical advice;
questions outside engineering decision-making. For an out-of-scope question — a polite decline before the panel runs.

### Model

One apex model across all agents (orchestrator + roles + Judge). Model-agnostic: uses whatever
apex model the environment currently has. Not Sonnet, not a mix of models across roles.

### Output language

All user-facing output (greetings, summaries, the final synthesis) is written in the USER'S
language — mirror the language of the question. Internal status markers (AGREED / AGREED_WEAK /
DISPUTED / NEEDS_CLARIFICATION, CONDITION, STAGNATED, GROUPTHINK_FLAG) always stay in English as
technical tokens.

---

## Trigger

The skill activates on any of:
- User phrases: "psychodrama", "put this on stage", "consensus panel" (functional fallback).
- Slash command: `/psychodrama-protocol <question>` (flags optional, see below).

**Deliberateness gate.** The two trigger paths carry different intent signal:
- **Slash command** (`/psychodrama-protocol <question>`) is always a deliberate, explicit invocation → proceed directly to Triage. The Step 1 estimate line is sufficient; no confirmation is needed.
- **Trigger phrase in conversation** ("psychodrama" etc.) may be a casual aside, not a deliberate request for a full panel run → before dispatching any subagent, the orchestrator must print ONE confirm line: `A full run costs ~6-13 model calls (~6-11 panel mode, ~9-13 duels mode), +1-2 calls if the stage deadlocks (figures; --no-figures to suppress), and takes several minutes. Proceed?` and wait for the user's confirmation before continuing to Triage. (The precise mode and its exact estimate are confirmed again in Step 1, once Triage has classified.)

**First action** (mandatory, before any subagent calls): print to the user:

> Using the Psychodrama Protocol: checking scope applicability.

Immediately after — for the trigger-phrase path, the deliberateness gate above; then in both cases, Triage (Step 0 below).

---

## Flags

| Flag | Effect |
|---|---|
| `--with-external` | Force the external critic regardless of unanimity |
| `--save-adr` | Save an ADR even for a standard run (by default no ADR is written) |
| `--no-record` | Skip writing the run record file |
| `--plan` | Stop after Triage+Decompose: show Canonical Intent + theses for user approval/editing before any panel votes are spent |
| `--roles "<spec>"` | One-run roster override: `+Shelf-role` enables a shelf role, `-Role` disables an active one, `+name(focus:'...')` mints an ephemeral persona for this run only |
| `--mode panel\|duels` | Forces the ceremony instead of letting Triage classify (see Step 0) |
| `--grain fine\|coarse` | Biases the Lumper⚔Splitter settlement toward more/fewer theses or perspectives; never silences either voice (see Protocol · Decompose) |
| `--emergent` | Alias for `--mode duels` (deprecated name) |
| `--no-figures` | Never summon the deadlock figures automatically |
| `--summon above\|below\|both` | Force a figure onto the stage regardless of deadlock — 🕊 above, 🔥 below |

Unknown flags: print a warning `⚠️ Unknown flag: <name>, continuing.` and don't stop.

---

## Edge cases (check before starting orchestration)

- **Question too short/non-technical** (fewer than 2 sentences and no technical context) → ask the user to clarify the question, don't run the panel. Template: `Question is too short for a panel — clarify the task in at least 2–3 sentences. What exactly needs reviewing?`
- **User cancelled execution** (Ctrl-C / cancel) → graceful stop. If R1 votes have already been collected — print their partial summary before exiting.
- **A subagent returned garbage/unparseable output** → mark its votes as `invalid`, continue with N-1 = 3 votes. If invalid ≥ 2 → abort and tell the user: `❌ Too many invalid votes in R1 (≥2 of 4). Stopping consensus. Try rephrasing the question.`

---

## Process

The main model follows the steps strictly in order. Each step is an atomic action. Don't skip, don't reorder.

### Step 0 — Triage

Check against Scope (see Protocol · Scope). This is an instruction to the orchestrator — not a code call.

- Question is in scope → continue to mode routing below.
- Question is out of scope → polite decline, panel does NOT run. Template: `This is outside the Psychodrama Protocol's scope (see Scope) — you need <a lawyer/other specialist>, not a decision-review panel.`

**Mode routing.** Once in scope, classify the question into one of the two ceremonies (see Protocol · Phases and Protocol · Duels):

- **"Evaluate a formed decision?"** — a decision, plan, or piece of work already exists and the question is whether it holds up → `panel` mode.
- **"Explore an open problem / choose between directions?"** — no single decision is on the table yet, the question is which direction to take → `duels` mode.

Routing precedence:
1. `--mode panel` or `--mode duels` was passed → that mode, no classification needed.
2. `--emergent` was passed → `duels` mode (deprecated alias, see Flags).
3. Otherwise classify from the question text. If genuinely ambiguous, ask the user ONE question: `Is this evaluating a decision already made, or exploring open directions?` and wait for the reply before continuing.

Save the resolved mode — it drives every step from here on (Step 4 onward forks on it).

### Step 1 — Print classification line

Immediately after the greeting and successful Triage, print the classification line, extended with an upfront cost estimate so the user sees the cost before any subagent is dispatched:

```
🧠 The Psychodrama Protocol · Mode: panel · Panel: <active roster> (+Decomposer +Judge) · ~6-11 model calls · several minutes
```

or, in `duels` mode:

```
🧠 The Psychodrama Protocol · Mode: duels · 3-5 perspectives, each a Champion+Tailored Critic duel (+Decomposer +Judge) · ~9-13 model calls · several minutes
```

Where `<active roster>` is the list of roles from Protocol · Roster (dynamic, not a literal in this file). The `~6-11 model calls` figure is fixed: 6 = R1 minimum (1 Decomposer + 4 votes + 1 Judge pass); 11 = with a full targeted R2 (+4 re-votes +1 additional Judge pass). The `~9-13 model calls` figure for duels mode: 1 Decomposer (dialectic decompose) + 1 perspective generation + 2N duel calls for N=3–5 perspectives + 1 Judge pass. Neither figure includes the figures (deadlock-breakers): +1-2 calls if the stage deadlocks (REFRAME cluster / STAGNATED / Contested), `--no-figures` to suppress — see Protocol · The figures.

(For the trigger-phrase path, this line is printed only after the user has already confirmed via the Trigger section's deliberateness gate — it is not itself the confirmation.)

Save the user's parsed flags (`--with-external`, `--save-adr`, `--no-record`) for use in later steps.

### Step 2 — Decomposer dispatch

Runs identically in both modes — the dialectic decomposition (Protocol · Decompose) is
mode-independent; only what happens with its output forks later (Step 4).

Call ONE subagent:

```
Agent(
  subagent_type="psychodrama-decomposer",
  description="Decompose into theses",  prompt="Decompose: <verbatim user question>[. Grain preference: fine|coarse]"
)
```

Append `Grain preference: fine|coarse` to the prompt only if the user passed `--grain`; omit it
otherwise and let the Lumper⚔Splitter dialectic settle the grain unbiased.

Parse the response. Expected format:

```
CANONICAL_INTENT: <gist of the task in 1–2 sentences>
GRAIN: <n> theses (Splitter proposed <k>, Lumper proposed <m>) — <rationale>
T0: <Premise Distillation — the question's foundational unstated premise>
T1: ...
T2: ...
...
Tn: ...
```

The `GRAIN` line is a feature, not debug info — it is the negotiated-framing trace and MUST be shown
to the user verbatim, both at the `--plan` checkpoint (Step 3) and in the pre-vote/pre-duel output.
The full Splitter/Lumper dialectic transcript (the tagged splitter-proposal / lumper-critique /
settlement blocks) is NOT shown in chat — it goes into the VDR only (Step 11).

**Single-thesis fallback:** if the response contains fewer than 2 valid theses (e.g. the Decomposer produced only T1, or nothing parseable at all) — enter single-thesis mode: treat the user's entire original question as one thesis T1 verbatim, and the Canonical Intent as the question itself. This is a lower-value mode, but the skill doesn't fail outright. Print to the user: `ℹ️ Single-thesis fallback: the task didn't decompose, evaluating as a single thesis T1.` (T0 is still produced and voted normally in this fallback — only T1..Tn collapses to one item.)

Record the Canonical Intent, the GRAIN line, and the final list of theses (T0..Tn) in variables — they're needed in every following step.

### Step 3 — `--plan` checkpoint (only if the flag was passed)

If the `--plan` flag was NOT passed → skip this step entirely, continue straight to Step 4.

If `--plan` WAS passed → print the Canonical Intent, the `GRAIN` line, and the full thesis list (T0..Tn) from Step 2, then stop and print:

> Reply "approve" to dispatch the panel, or reply with edited theses (same T\<n\> format) to use your version.

Wait for the user's next message:
- **"approve"** (or equivalent) → continue to Step 4 using the theses exactly as produced in Step 2.
- **Edited theses** (a reply in the same `T<n>: <text>` format) → replace the in-memory Canonical Intent/theses with the user's version, then continue to Step 4 using the edited set. No re-dispatch of the Decomposer — the user's reply IS the new thesis list.

In `duels` mode, this same checkpoint mechanic applies a second time after perspective generation
(Step 4a) — see that step. The `--plan` behavior above is unchanged in panel mode.

No subagent calls happen in this step — it's a pure orchestrator print-and-wait, spending zero of the panel's model calls, which is the point of the flag (approve/edit before any panel votes are spent).

### Step 4 — R1 panel: dispatch panel subagents IN PARALLEL (single message)

**Mode fork.** This step (and Steps 5–10) is the `panel`-mode flow. If the resolved mode (Step 0) is
`duels` → skip this entire step and Steps 5–10, go straight to **Step 4a** below; Steps 11–12 (VDR,
ADR) resume afterward and apply to both modes.

**Critical:** all roster calls go in ONE message from the main model, as parallel tool-use (parallel function calls). This is required for R1 isolation (each agent in a clean context, none sees another's answers).

**`--roles` override (only if the flag was passed).** Before assembling the dispatch, apply the
override to the active roster from Protocol · Roster:
- `-Role` removes that role from the active roster.
- `+Shelf-role` enables a role from the shelf (DESIGN.md · Role Library names: `Performance-hawk`,
  `Product-thinker`, `Scope-cutter`, `Simplicity-advocate`, `Domain-expert`) into the active roster.
  `+Resource-keeper` now resolves to the shipped `psychodrama-resource-keeper` agent file (not a
  minted persona) — it votes normally alongside the rest of the roster.
- `+name(focus:'<text>')` mints an ephemeral persona, active for this run only (not saved anywhere).
  The orchestrator composes its mandate inline: a one-line mandate derived from the focus text, an
  instruction to vote through that lens only, plus the full vote-format contract (the 4 statuses +
  CONDITION/FLIP/ANCHOR/[impact] tags, see Protocol · Status vocabulary) injected exactly as it is
  for a real role. This persona is dispatched alongside the rest of the roster in this same step —
  it does not get a separate subagent_type, its mandate is inlined into its prompt.

**Invariant guard.** After applying all `--roles` overrides, check the resulting roster against
Protocol · Panel invariants: Adversarial Presence (≥1 of Skeptic/Security/Scope-cutter, or an
ephemeral persona whose focus is explicitly adversarial) and Size (3–7). If either is violated,
the orchestrator says so to the user and keeps `Skeptic` in the roster regardless of the override
(the guard wins over the user's `-Skeptic` in that one case — invariants are non-negotiable).

If `--roles` was NOT passed → the active roster is exactly Protocol · Roster, unchanged.

Dispatch the entire active roster (post-override, see above) in a single message.

Give each the **same** prompt:

```
Original question: <verbatim user question>

Canonical Intent: <text from Step 2>

Panel members in this run: <active roster>

Theses:
T0: <text>
T1: <text>
T2: <text>
...
Tn: <text>
```

(No `model` parameter is passed — role agents inherit the session's model, per Protocol · Model.)

Wait for all roster subagents to return. Each returns:
- **Block 1:** line-by-line votes `T<n>: <STATUS> [comment]` (comment may contain `CONDITION:`/`FLIP:`/`ANCHOR:`/`[impact: ...]` tags, see Protocol · Status vocabulary).
- **Block 2 (Optimizer and Maintainability-advocate only):** a structured `DISAGREEMENT:` block (`TARGET / THESIS / COUNTER`) or fallback `COUNTER: NO_DISAGREEMENT — ...`.

If any agent returned unparseable output → mark its votes as `invalid` (see the Edge cases section above).

### Step 4a — Duel mode: perspective generation (1 call)

**Duels-mode only** (see the mode fork at the top of Step 4). Runs right after Step 2/3, as an
ADDITIONAL call on top of Step 2 (Step 2 still runs in duels mode — its Canonical Intent, T0, and
GRAIN trace are the foundational frame this call builds on; only T1..Tn are unused, since there is
no per-thesis voting in duels mode). This call replaces the roster dispatch of Step 4.

```
Agent(
  subagent_type="psychodrama-decomposer",
  description="Generate perspectives",  prompt="Generate the 3-5 strongest genuinely distinct perspectives/directions on this question — one line each, no personas yet. Settle the count/grain yourself via the same Lumper⚔Splitter dialectic used for thesis decomposition (Protocol · Decompose), and report a GRAIN trace line for the perspective set. Question: <verbatim user question>. Canonical Intent: <text from Step 2>.[ Grain preference: fine|coarse]"
)
```

Append `Grain preference: fine|coarse` only if the user passed `--grain` (same rule as Step 2).

Expected output: a `GRAIN` trace line (`GRAIN: <n> perspectives (Splitter proposed <k>, Lumper
proposed <m>) — <rationale>`) followed by 3-5 perspective lines. Print the GRAIN line to the user
verbatim (Protocol · Decompose — always user-visible).

**`--plan` checkpoint (duels mode, second checkpoint).** If `--plan` was passed, stop here and print
the GRAIN line + the perspective list, then:

> Reply "approve" to mint Champion/Critic pairs and run the duels, or reply with an edited perspective list to use your version.

Wait for the user's reply — "approve" continues to Step 4b unchanged; an edited list replaces the
in-memory perspectives (no re-dispatch). If `--plan` was NOT passed, continue straight to Step 4b.

**Fallback:** if fewer than 3 distinct perspectives come back, pad with a boring-default perspective
("do nothing / keep the status quo") until 3 are reached — do not silently proceed with 1-2.

### Step 4b — Duel mode: pair minting (orchestrator, no calls)

For EACH perspective from Step 4a, mint two ephemeral personas — same mechanics as a `--roles
+name(focus:'...')` persona (Step 4, `--roles` override, above): a name, a one-line mandate, and the
full vote-format contract (CONDITION/FLIP/ANCHOR/[impact] tags) injected into its prompt.

- **Champion** — mandate built directly from the perspective: its strongest possible advocate,
  instructed to make the strongest case (ANCHOR quotes, CONDITION where honest).
- **Tailored Critic** — the antagonist purpose-built for THIS idea. Its mandate is written as a
  biography of contact with this idea-class's failures (e.g. "you have watched three rewrites die
  this way"), NOT generic skepticism copied across perspectives. It is instructed that its output
  MUST contain a `FLIP:` (what evidence would vindicate the idea, i.e. prove the Critic wrong) and
  an `ANCHOR:` quote — a Critic response missing `FLIP:` is invalid.

**Anti-strawman guard.** If a Critic's duel output (Step 4c) comes back without a `FLIP:` line,
re-mint that Critic once with an explicit reminder of the requirement and re-run its duel call. If
the retry still lacks `FLIP:`, stop retrying and mark that duel `unproven` (see the Judge's SURVIVAL
grades) rather than looping further.

### Step 4c — Duel mode: duels (2 isolated calls per perspective)

For each perspective's pair, two calls (Champion then Critic — sequential per pair since the Critic
answers the Champion's case; different perspectives' pairs are independent of each other and MAY be
parallelized across perspectives, but each pair internally is Champion-then-Critic, one exchange,
no chat back and forth):

```
Agent(
  subagent_type="claude",
  description="Champion case: <perspective>",  prompt="<Champion mandate from Step 4b>\n\nOriginal question: <verbatim user question>\n\nCanonical Intent: <text from Step 2>\n\nState your strongest case for this perspective. Use ANCHOR: quotes from the question/spec where you ground a claim, and CONDITION: for anything contingent on an external dependency."
)
```

```
Agent(
  subagent_type="claude",
  description="Critic fatal-flaw case: <perspective>",  prompt="<Tailored Critic mandate from Step 4b>\n\nOriginal question: <verbatim user question>\n\nCanonical Intent: <text from Step 2>\n\nChampion's case:\n<verbatim Champion output>\n\nState the fatal-flaw case against this perspective. FLIP: (what evidence would vindicate the idea) is MANDATORY. Tag [impact: critical|moderate|minor] on every strike. ANCHOR: quotes required where you ground a claim."
)
```

One exchange each — Champion states its case once, Critic answers once. No multi-round chat (Protocol
· Rounds & stop conditions — the same hard-cap philosophy applies to duels).

If a Critic response lacks `FLIP:` → apply the anti-strawman guard from Step 4b (re-mint once, then
mark `unproven`).

### Step 4d — Duel mode: Judge synthesis (1 call)

Call ONE subagent, dispatched with the duel-mode inputs per the judge mandate's Duel mode section
(`agents/psychodrama-judge.md`):

```
Agent(
  subagent_type="psychodrama-judge",
  description="Duel synthesis",  prompt="Duel mode. Original question: <q>. Canonical Intent: <intent>. GRAIN: <perspective GRAIN trace from Step 4a>. Perspectives: <list from Step 4a>.\n\nDuel transcripts:\n\n<perspective 1>:\nChampion:\n<full Champion output>\nCritic:\n<full Critic output>\n\n<perspective 2>:\n...\n(repeat for every perspective)"
)
```

The Judge (per its Duel mode mandate) works through, in order: per-duel `SURVIVAL` grade
(`clean`/`scarred`/`gutted`/`unproven`) + Champion's footnote; the empty chair (the un-championed
direction) check; cross-duel trade-offs/untested hybrids/unanswered points; a unified
`CONSENSUS_STRENGTH` line (same vocabulary as panel mode, fed by SURVIVAL); verdict + prerequisites
+ tripwires (every rejected perspective's Critic-FLIP inverts into a revisit trigger, 1-3
tripwires) + Devil's advocate.

Save the Judge's output — it goes into Step 4e verbatim.

### Step 4e — Duel mode: Figures (deadlock-breakers)

Duels-mode instance of Step 9 — same trigger logic, no early-summon path (REFRAME_CLUSTER is a
per-thesis panel-mode concept; duels mode has no per-thesis voting). Runs after Step 4d (Judge
synthesis) arrives, before render (Step 4f):

- **STAGNATED** — a duel graded `unproven` after the anti-strawman retry, or any duel the Judge
  otherwise flags as stalled → summon 🕊 only.
- **`CONSENSUS_STRENGTH` is `Contested`** → summon BOTH 🕊 and 🔥.
- **`--summon above|below|both`** forces the corresponding figure(s) regardless. Overrides
  `--no-figures`.
- **`--no-figures`** suppresses the two automatic triggers above.
- None apply → `figures: none`, skip straight to Step 4f.

Dispatch (one isolated call per figure, no chat), same subagents and cost discipline as Step 9 —
`psychodrama-outside-frame` (🕊) and `psychodrama-transgressive` (🔥), inputs adapted to duels mode
(deadlocked perspective(s), Champion+Critic transcripts, REFRAME tags if any surfaced in the duel
text):

```
Agent(
  subagent_type="psychodrama-outside-frame",
  description="Dissolve the deadlock",  prompt="Original question: <q>. Canonical Intent: <intent>. Deadlocked perspective(s) with strongest both-side arguments:\n<STAGNATED/Contested duel transcripts, Champion + Critic, verbatim>\n\nExit the dispute's terms. Produce a reframe of what this dispute is actually about, plus `dissolves if:` lines."
)
```

```
Agent(
  subagent_type="psychodrama-transgressive",
  description="Name the suppressed option",  prompt="Original question: <q>. Canonical Intent: <intent>. Deadlocked perspective(s) with strongest both-side arguments:\n<STAGNATED/Contested duel transcripts, Champion + Critic, verbatim>\n\nName the option nobody dared stage, and its honest price."
)
```

One appearance each per run; recorded verbatim; no second Judge pass — figure outputs are appended
to the rendered synthesis (Step 4f) and the VDR (Step 11) directly. Save `figures:
<none|🕊|🔥|both>` and the trigger reason — needed for Step 4f's render and Step 11's VDR.

### Step 4f — Duel mode: render output to user

Print to the user as a single block (user's language, per Protocol · Output language; statuses/grades
stay English), using the Judge's duel-mode output schema (`agents/psychodrama-judge.md` · Duel
mode · Output schema):

```
🧠 The Psychodrama Protocol · Mode: duels
CONSENSUS_STRENGTH: <Strong|Working|Narrowly carried|Contested> — <clause>
GRAIN: <perspective GRAIN trace from Step 4a>

🗺 Field map:
  <perspective> — SURVIVAL: <clean|scarred|gutted|unproven> · <one-line why> · Footnote: <champion's answer>
  ...

🪑 The empty chair (the un-championed direction): <paragraph or defended "field covers the intent">

🔀 Trade-offs (between survivors): <...>
🧪 Untested hybrids: <labeled observations or "none">
📎 Prerequisites: <conditions checklist>  ·  Tripwires: <revisit-if list>
❓ Unanswered points: <named blocks or "none">
😈 Devil's advocate: <two strikes, ≥1 independent>

Verdict: <direction / measure-first / reframe-first + rationale>

<if 🕊 fired (Step 4e):>
🕊 Outside the frame (<trigger: STAGNATED|Contested|--summon>):
<🕊's full output, verbatim>

<if 🔥 fired (Step 4e):>
🔥 The transgressive voice (<trigger: STAGNATED|Contested|--summon>):
<🔥's full output, verbatim>

Run record: <./consensus-runs/<file>.md | skipped (--no-record)>
```

If no figure fired — omit both 🕊/🔥 sections entirely, print nothing about figures.

Then continue to Step 11 (VDR) — duels mode writes the same record structure (AUDIT_BLOCK `mode:
duels`, duel transcripts in place of per-role votes, see Step 11 for the duel-mode field mapping) —
and Step 12 (ADR) exactly as panel mode does; Steps 5–10 do not run in duels mode.

### Step 5 — Judge dispatch (Phase A — aggregation)

**Panel-mode only** (see the mode fork at the top of Step 4). Duels mode uses Step 4d instead.

Call ONE subagent:

```
Agent(
  subagent_type="psychodrama-judge",
  description="Aggregate R1 votes",  prompt="Phase A. Original question: <q>. Canonical Intent: <intent>. Theses: <T1..Tn full text>. R1 votes:\n\nOptimizer:\n<full Optimizer output>\n\nSkeptic:\n<full Skeptic output>\n\nSecurity:\n<full Security output>\n\nMaintainability-advocate:\n<full Maintainability-advocate output>"
)
```

Parse the Judge's output as a `ROUND_SUMMARY`:
- `AGREED: [...]` — theses on which everyone voted AGREED/AGREED_WEAK without substantial reservations.
- `AGREED_WEAK_THESES: [...]` — list of `{thesis, role, nuance}` objects.
- `DISPUTED_THESES: [...]` — theses with DISPUTED from at least one role (or NEEDS_CLARIFICATION).
- `NEEDS_CLARIFICATION: [...]` — separate list.
- `CONDITIONS: [...]` — list of `{thesis, role, condition}` objects, extracted from CONDITION tags (see Protocol · Status vocabulary).
- `GROUPTHINK_FLAG: <true|false>` — true if ALL theses are AGREED or AGREED_WEAK across ALL agents.
- `REFRAME_CLUSTER: [...]` — list of theses where ≥2 roles tagged `REFRAME` in their rationale (see Protocol · Status vocabulary). Non-empty (and `--no-figures` absent) → **early-summon 🕊 now**, before R2 — see Step 9 (Figures), early-summon path. 🕊's output then joins the R2 dispatch context in Step 7.

Save this data — needed in Steps 6–9.

### Step 6 — External critic decision

**Panel-mode only.** Duels mode has no external-critic step — the Tailored Critic already supplies
the adversarial voice per perspective.

Trigger condition (fires if AT LEAST ONE is met):

1. `GROUPTHINK_FLAG == true` **AND** at least one thesis mentions security-adjacent terms: `auth`, `authn`, `authz`, `secret`, `token`, `pii`, `credential`, `credentials`, `encryption`, `encrypt`, `decrypt` (case-insensitive substring match against thesis text).
2. The user passed the `--with-external` flag.

If neither condition is met → skip this step, set `external_status = not_required`, go to Step 7.

(See Protocol · Anti-groupthink — this is the only point where the skill touches a second model, and only as a hedge, not a default mode.)

**If the fallback fires, walk the 3-step ladder below.** The critic prompt and the parsing regex are shared by both the CLI path and the manual path — build the prompt once, reuse it for whichever rung applies.

Prompt for the external critic (used by both the CLI path and the manual path):

```
You are an external critic. For each thesis below, return ONE line in EXACTLY this format:
T<n>: <AGREED|AGREED_WEAK|DISPUTED|NEEDS_CLARIFICATION> [<comment if not AGREED>]

No preamble, no commentary outside this format.

Theses:
T1: <thesis text>
T2: <thesis text>
...
```

Parsing regex (applied to CLI stdout AND to a pasted manual reply, identically): `^T[0-9]+: (AGREED|AGREED_WEAK|DISPUTED|NEEDS_CLARIFICATION)( .*)?$`

**Rung 1 — configured CLI.** The Protocol block's configurable external-critic command line defines the CLI to invoke (default worked example: `gemini -p`). Any CLI that accepts a prompt (as an argument or via stdin) and prints text qualifies — `codex exec`, `ollama run <model>`, etc.

**Consent gate — required before ANY Rung-1 dispatch, no exceptions.** The theses being sent contain the user's question verbatim. Print this line naming the actual configured command, and wait for confirmation before invoking anything:

```
External critic: this will send the theses (which contain your question verbatim) to '<configured command>'. Proceed? (yes / skip)
```

"skip" (or equivalent decline) does NOT set `external_status = skipped (user_declined)` directly — it falls through to the Rung 2 offer, same as an unavailable CLI would. **`--with-external` does not bypass this line.** The flag forces the critic to fire at all; it is not consent to a specific destination — the user still needs to see and approve which command their question is about to be piped to. Only on explicit "yes" proceed to invoke it via the Bash tool:

```bash
timeout 30s <configured command> "<prompt>" 2>/dev/null
```

Handle exit codes:
- **0** → parse stdout with the regex above. If it produces at least one valid line → set `external_status = triggered (cli)`. If nothing parses → set `external_status = invalid_output`, continue without the critic (do not fall through to Rung 2 — a CLI that ran but produced garbage is a parsing failure, not an availability failure).
- **any non-zero / timeout / binary not found** → the CLI is unavailable. Fall through to Rung 2.
- **no command configured at all** → skip straight to Rung 2.

**Rung 2 — manual handoff.** No gate here — the user is the channel, so they see exactly what leaves before it leaves. Print the exact critic prompt from above to the user, followed by:

> Paste this into any other model you have (web chat is fine), then paste its reply back — or reply "skip".
>
> Note: the pasted prompt contains your question verbatim — choose the external model accordingly.

Wait for the user's next message.
- If the user pastes a reply → parse it with the SAME regex as the CLI path. If it produces at least one valid line → set `external_status = triggered (manual)`. If nothing parses → set `external_status = invalid_output`, continue without the critic.
- If the user replies "skip" (or equivalent decline) → set `external_status = skipped (user_declined)`, continue without the critic.

**Rung 3 — skip.** Reached only via an explicit user decline in Rung 2. `external_status = skipped (user_declined)`, run continues (graceful degradation as before).

**Cross-language note:** the external model may reply in another language — statuses parse the same regardless (the format is a fixed token, not prose). The Judge translates comments into the user's language during synthesis in Step 8.

**External critic's effect on R2 scope:** if the critic brought back at least one `DISPUTED` on a thesis that R1 had settled as `AGREED` — add that thesis to `DISPUTED_THESES` for R2 (even if the entire roster was unanimous).

### Step 7 — R2 dispatch (only DISPUTED theses)

**Panel-mode only.** Duels mode has no R2 — each duel is a single isolated exchange (Step 4c).

If `DISPUTED_THESES` is empty after Step 5 + Step 6 → **early finalize**, go straight to Step 8 without R2.

Otherwise — dispatch the same roster IN PARALLEL (single message, as in Step 4) with the R2 prompt:

```
Original question: <verbatim user question>

Canonical Intent: <text from Step 2>

You participated in R1. Here are the OTHER panel members' R1 positions on the DISPUTED theses (and the external critic's verdict if present).

Panel positions from R1:
Optimizer (on disputed):
<Optimizer's vote lines for disputed theses only>
Skeptic (on disputed):
<...>
Security (on disputed):
<...>
Maintainability-advocate (on disputed):
<...>
<if the external critic fired, add:>
External critic:
<external critic's lines for disputed theses only>
<if 🕊 fired early (REFRAME_CLUSTER, see Step 5), add:>
🕊 Outside the frame (reframe of the REFRAME-clustered theses):
<🕊's full output from Step 9's early-summon path, verbatim>

DISPUTED theses to re-evaluate (only these — others are settled):
T<n>: <text>
...

For each disputed thesis, return ONE line in same format. You may stick, change, or refine your R1 position. But provide NEW substance — repeating R1 verbatim is stagnation.
```

Wait for the entire roster. If a role continues to emit a DISAGREEMENT block (Optimizer / Maintainability-advocate) — keep it, pass it to the Judge.

### Step 8 — Judge dispatch (Phase B + C — stagnation check + final synthesis)

**Panel-mode only.** Duels mode's single Judge synthesis call is Step 4d.

Call the Judge a second time:

```
Agent(
  subagent_type="psychodrama-judge",
  description="Final synthesis",  prompt="Phase B+C. Original question: <q>. Canonical Intent: <intent>. Theses: <T1..Tn full text>. R1 votes:\n<full roster + external critic if it ran>\n\nR2 votes (disputed only):\n<full roster on disputed theses>[\n\n🕊 Outside the frame (early summon, if it fired):\n<🕊's full output from Step 9's early-summon path, verbatim>]"
)
```

If 🕊 fired early (Step 5's `REFRAME_CLUSTER` path), its output is appended to this prompt — the Judge weaves it into Phase C, possibly leading with the reframe (per the Judge mandate).

The Judge does:
- **Phase B** — for each thesis still DISPUTED after R2, compares R2's arguments against R1. If the arguments are a semantic rehash of R1 → mark the thesis `STAGNATED: true`. Do not invoke R3.
- **Phase C** — opens with `CONSENSUS_STRENGTH` (`Strong`/`Working`/`Narrowly carried`/`Contested`, see Protocol · Rounds & stop conditions), followed by the structured final synthesis in the 8-section schema defined in the Judge mandate (rendered as the Step 10 template below): Consensus, Holism check, Trade-offs, Blockers, Prerequisites, Nuances, Unresolved, Devil's advocate — Holism check and Devil's advocate always required.

Save the Judge's output — it goes into Step 10 verbatim, and feeds Step 9 (Figures) next for the end-of-run trigger check.

### Step 9 — Figures (deadlock-breakers)

**Runs in both modes**, positioned here (after the final Judge synthesis, before render) for its
end-of-run trigger check; panel mode also has an early-summon path that fires earlier in the run
(see below). Panel mode continues to Step 10 (render) afterward; duels mode's equivalent slot is
**Step 4e** (this same logic, duels-mode inputs) — the duels-mode render step is renumbered to
**Step 4f** below.

Two figures exist outside the voting panel (Protocol · The figures): 🕊 `psychodrama-outside-frame`
(dissolves a deadlock from above) and 🔥 `psychodrama-transgressive` (breaks it from below). Neither
votes. `--no-figures` suppresses every automatic trigger below (a-c); `--summon` (d) still fires even
with `--no-figures`, since it is an explicit user override, not an automatic one.

**(a) Early-summon path — 🕊 only, panel mode only, fires BEFORE R2 (not here).** Already covered at
Step 5: if `REFRAME_CLUSTER` is non-empty after Phase A aggregation, dispatch 🕊 immediately —

```
Agent(
  subagent_type="psychodrama-outside-frame",
  description="Reframe: <clustered theses>",  prompt="Original question: <q>. Canonical Intent: <intent>. REFRAME-clustered theses (≥2 roles tagged REFRAME):\nT<n>: <text>\n  <role>: REFRAME: <rationale>\n  <role>: REFRAME: <rationale>\n...\nBoth-sides arguments on these theses (full R1 vote lines, all roles): <...>\n\nExit the dispute's terms. Produce a reframe of the question these theses are actually fighting over, plus `dissolves if:` lines — the condition(s) under which this dispute stops mattering."
)
```

Its output is included verbatim in the R2 dispatch context (Step 7) so panel roles see the reframe
when re-voting, and again in the Step 8 Judge prompt so the final synthesis weaves it in — the
sensor (REFRAME_CLUSTER) → figure (🕊) → re-vote pipeline. This path does not run again here even
if triggers (b)/(c) below also fire — 🕊 appears once per run (Protocol · The figures).

**(b)/(c) End-of-run path — evaluated here, after the final Judge synthesis (Step 8) arrives**,
unless 🕊 already fired via (a) for this run:

- **(b) any thesis is `STAGNATED`** → summon 🕊 only.
- **(c) `CONSENSUS_STRENGTH` is `Contested`** → summon BOTH 🕊 and 🔥.

**(d) `--summon above|below|both`** forces the corresponding figure(s) regardless of (a)-(c):
`above` = 🕊, `below` = 🔥, `both` = both. Overrides `--no-figures`.

If none of (a)-(d) apply → `figures: none`, skip straight to render (Step 10).

**Dispatch (one isolated call per figure, no chat):**

```
Agent(
  subagent_type="psychodrama-outside-frame",
  description="Dissolve the deadlock",  prompt="Original question: <q>. Canonical Intent: <intent>. Deadlocked theses with strongest both-side arguments:\n<STAGNATED theses, full vote text>\nREFRAME tags present, if any: <...>\n\nExit the dispute's terms. Produce a reframe of what this dispute is actually about, plus `dissolves if:` lines."
)
```

```
Agent(
  subagent_type="psychodrama-transgressive",
  description="Name the suppressed option",  prompt="Original question: <q>. Canonical Intent: <intent>. Deadlocked theses with strongest both-side arguments:\n<STAGNATED theses, full vote text>\nREFRAME tags present, if any: <...>\n\nName the option nobody dared stage, and its honest price."
)
```

One appearance each per run; recorded verbatim. **No second Judge pass** — cost discipline: the
orchestrator appends the figure outputs verbatim to the rendered synthesis (Step 10 / Step 4f) under
their own headers, and into the VDR (Step 11), rather than re-invoking the Judge to weave them in
(the exception is the early-summon path (a), where 🕊 is already inside the Judge's own input, per
above).

Save `figures: <none|🕊|🔥|both>` and the trigger reason (`REFRAME_CLUSTER` / `STAGNATED` /
`Contested` / `--summon`) — needed for Step 10/4f's render and Step 11's VDR.

### Step 10 — Render output to user

**Panel-mode only.** Duels mode's render step is Step 4f.

Print to the user as a single block (all in the user's language, per Protocol · Output language, except the internal statuses AGREED/DISPUTED/AGREED_WEAK/NEEDS_CLARIFICATION — these stay as technical markers, see Protocol · Status vocabulary, + the CONDITION tag).

The template below is shown in English as the reference; render it in the user's language in practice:

```
🧠 The Psychodrama Protocol
Consensus strength: <Strong|Working|Narrowly carried|Contested>
Panel: <active roster>
External critic: <triggered (cli)|triggered (manual)|skipped (user_declined)|invalid_output|not_required>

Round 1/2:
  Agreed: <list of thesis numbers from Judge Phase A AGREED>
  Disputed: <list of thesis numbers from DISPUTED_THESES>
  AGREED_WEAK nuances: <list of thesis numbers from AGREED_WEAK_THESES>
  <if the external critic fired:>
  External critic brought: <list of theses where it returned DISPUTED + brief comments, translated into the user's language if needed>

<if R2 ran:>
Round 2/2 (disputed only):
  T<n> → <accepted|accepted with amendment|stagnated|still DISPUTED>
  ...

<Judge Phase C output verbatim — sections ✅ Consensus / 🔗 Holism check / 🔀 Trade-offs (value-tensions) / 🚫 Blockers (error-catches) / 📎 Prerequisites (conditions) / ⚠️ Nuances / ❓ Unresolved / 😈 Devil's advocate>

<if 🕊 fired (Step 9, path a/b/c/d):>
🕊 Outside the frame (<trigger: REFRAME_CLUSTER|STAGNATED|Contested|--summon>):
<🕊's full output, verbatim>

<if 🔥 fired (Step 9, path c/d):>
🔥 The transgressive voice (<trigger: STAGNATED|Contested|--summon>):
<🔥's full output, verbatim>

Run record: <./consensus-runs/<file>.md | skipped (--no-record)>
```

Mapping `external_status → Output line`:
- `not_required` → `not_required`
- `triggered (cli)` → `triggered (cli — <reason: groupthink+security | --with-external>)`
- `triggered (manual)` → `triggered (manual — <reason: groupthink+security | --with-external>)`
- `skipped (user_declined)` → `skipped (user_declined)`
- `invalid_output` → `skipped (invalid_output)`

If `Round 2/2` didn't run (early finalize) — omit the section entirely, print nothing about R2.
If no figure fired — omit both 🕊/🔥 sections entirely, print nothing about figures.

### Step 11 — Verifiable Decision Record

Runs for both modes. ON by default; opt out with `--no-record` (see Flags). Rationale: this record is the product's auditability story — the artifact to attach when reporting a weak or disputed verdict, since it lets anyone re-derive the synthesis from the raw votes without re-running the panel.

If `--no-record` was passed → skip this step silently, and print `Run record: skipped (--no-record)` in Step 10's (or Step 4f's, in duels mode) output (already covered by the template above).

Otherwise, write a structured markdown transcript to `./consensus-runs/YYYY-MM-DD-<slug>.md` relative to cwd (create the `consensus-runs/` directory if it doesn't exist). `<slug>` uses the same rule as the ADR step below: the first 5 significant words of the user's question, lowercase, kebab-case, no punctuation. `YYYY-MM-DD` is today's date.

**Sensitivity marker — first run only.** On the FIRST run that creates the `./consensus-runs/` directory (i.e., the directory did not exist before this step), also write a `README.txt` into it:

```
These records contain your questions and the panel's full deliberations verbatim. Treat as sensitive. If this directory is inside a git repository, add consensus-runs/ to .gitignore.
```

Additionally, if cwd is inside a git repository (a `.git` directory exists at cwd or an ancestor), print one reminder line suggesting the `.gitignore` addition — suggest only, never edit the user's `.gitignore` silently.

All content below is already in-context from prior steps — this step performs no new computation, only assembly. The record now OPENS with a machine-readable `AUDIT_BLOCK` — a fenced block of `key: value` lines at the very top of the same file, purpose-built so that this block (or the whole record) can be handed to ANY external model for a post-hoc audit, provider-agnostic. **Panel-mode template** (used when the resolved mode is `panel`):

```
AUDIT_BLOCK
date: YYYY-MM-DD
question: <one-line, truncated if needed>
mode: panel
panel: <active roster from Protocol · Roster, +Decomposer +Judge>
consensus_strength: <Strong|Working|Narrowly carried|Contested>
verdict: <one-line summary of the Judge's Phase C consensus>
key_assumptions:
  - <bulleted, drawn from T0 (Premise Distillation) + any accepted CONDITION premises>
  - ...
figures: <none|🕊|🔥|both> (<trigger: REFRAME_CLUSTER|STAGNATED|Contested|--summon|none>)
record_file: ./consensus-runs/YYYY-MM-DD-<slug>.md

## Question
<verbatim user question>

## Canonical Intent
<text from Step 2>

## Panel
<active roster from Protocol · Roster, +Decomposer +Judge> — <note which model session ran the panel, e.g. "run on: <model name/id of this session>">

## Votes
<per-role, verbatim R1 vote lines including CONDITION/FLIP/ANCHOR/[impact] tags and DISAGREEMENT blocks; if R2 ran, include R2 votes per role as well, verbatim>

## External critic
<status from Step 6 (external_status) + its verbatim output if it ran; "not_required" if it didn't>

## Judge synthesis
<the full Phase C output from Step 8, verbatim>

## Figures
<🕊/🔥 verbatim output(s) from Step 9, if any fired; "none" if `figures: none`>

---
Generated by psychodrama-protocol vX · LLM output is non-deterministic; this record documents this specific run.
```

**Duels-mode template** (used when the resolved mode is `duels` — same file location, same
`AUDIT_BLOCK` opening discipline, different field mapping since there are no per-role votes, only
per-perspective duel transcripts):

```
AUDIT_BLOCK
date: YYYY-MM-DD
question: <one-line, truncated if needed>
mode: duels
panel: <perspectives from Step 4a as Champion/Critic pairs, +Decomposer +Judge>
consensus_strength: <Strong|Working|Narrowly carried|Contested>
verdict: <one-line summary of the Judge's verdict>
key_assumptions:
  - <bulleted, drawn from T0/Canonical Intent + any accepted CONDITION premises from surviving perspectives>
  - ...
figures: <none|🕊|🔥|both> (<trigger: STAGNATED|Contested|--summon|none>)
record_file: ./consensus-runs/YYYY-MM-DD-<slug>.md

## Question
<verbatim user question>

## Canonical Intent
<text from Step 2>

## Grain
<the perspective GRAIN trace line from Step 4a, plus the full Splitter/Lumper dialectic transcript
(tagged splitter-proposal / lumper-critique / settlement blocks) from Step 2's Decomposer call and,
if it ran separately, Step 4a's — this is the one place the full dialectic is shown, not the chat output>

## Perspectives
<perspective list from Step 4a>

## Duels
<per-perspective, verbatim Champion output + verbatim Critic output, from Step 4c>

## Judge synthesis
<the full duel-mode output from Step 4d, verbatim, including per-duel SURVIVAL + footnotes>

## Figures
<🕊/🔥 verbatim output(s) from Step 4e, if any fired; "none" if `figures: none`>

---
Generated by psychodrama-protocol vX · LLM output is non-deterministic; this record documents this specific run.
```

Save the file path — it is printed as the final line of Step 10's (or Step 4f's) output template (`Run record: ./consensus-runs/<file>.md`).

**Decision journal append.** After writing the record file, append ONE line to `./consensus-runs/INDEX.md` (create it with a header row if it doesn't exist yet):

```
| Date | Question | Strength | Verdict | Record |
|---|---|---|---|---|
```

The appended line:

```
| YYYY-MM-DD | <question, truncated ~80ch> | <consensus_strength> | <verdict, one line> | [record](<filename>) |
```

This runs regardless of whether `--save-adr` was passed — the journal is the running index of every recorded run, independent of ADR generation.

### Step 12 — ADR file generation

By default, no ADR is generated. Review artifacts live in PR comments / notes, not in `docs/decisions/`.

**The only exception:** the user explicitly passed the `--save-adr` flag. Then — save an ADR using the template from DESIGN.md, section 6, to `docs/decisions/YYYY-MM-DD-<topic>.md` relative to cwd. `<topic>` is a slug from the first 5 significant words of the user's question (lowercase, kebab-case, no punctuation). `YYYY-MM-DD` is today's date.

If the flag is absent — skip this step silently.

---

## Principles (for the main model reading this skill)

- **One apex model everywhere** (see Protocol · Model). Not Sonnet, not a mix.
- **R1 = isolation.** The entire roster starts in a single message (parallel tool-use), each in a clean context. No cross-talk between roles in R1.
- **The Judge is a separate subagent.** Not the main model. A clean context for aggregation.
- **Hard cap: R1 + R2. Never R3** (see Protocol · Rounds & stop conditions).
- **Don't introduce new theses mid-flight.** Theses are fixed by the Decomposer before R1 and don't change.
- **Canonical Intent accompanies the panel every round** (see Protocol · Phases) — protection against locally optimizing a thesis in isolation from the task's actual intent.
</content>
