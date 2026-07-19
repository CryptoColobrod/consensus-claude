---
name: consensus-claude
description: Multi-perspective consensus via single-model subagents with distinct roles. Use when the user says "claude consensus" / "consensus panel" or invokes /consensus-claude. Claude-only mode by default; pulls in Gemini CLI as anti-groupthink fallback for architecture/security or when all agents agree unanimously.
---

# `consensus-claude`

Multi-perspective consensus via subagents of a single model with distinct roles. Runs on an explicit user trigger. Complements `consensus-protocol` (Claude+Gemini), does not replace it.

---

## Protocol (canonical — single source of truth)

> Reading only this block, both a human and the orchestrator LLM understand the whole skill:
> what it does, what it's made of, what makes it unique, what rules it operates under.
> Everything below in the file is execution of this contract, not new rules.

### What it is — one line

`consensus-claude` runs a question through a panel of roles from **a single model** (not a dialogue
between different models) and produces a synthesis with disagreement explicitly preserved.

### Differentiator (the one explicit uniqueness claim)

Voting happens **per atomic thesis** (per-thesis), not on the answer as a whole
(holistic, as with comparable tools). This yields: precise localization of disputes, a
machine-parseable decision trail, and protection against a "lowest common denominator" verdict.

### Phases

`Triage → Decompose → Vote (R1) → Judge(aggregate) → Vote (R2, disputed only) → Judge(synthesize)`

- **Triage** — is the question within the skill's scope? (see Scope below) If not — decline, panel does not run.
- **Decompose** — the Decomposer produces `Canonical Intent` (the gist of the task in 1–2 sentences) + atomic theses T1..Tn.
- **Vote R1** — panel roles vote on each thesis in parallel and independently (context isolation).
- **Judge aggregate** — tally the votes, find disputed theses, detect groupthink.
- **Vote R2** — disputed theses only; roles see each other's R1 positions, must contribute new substance.
- **Judge synthesize** — holism check, separating tension from error, final synthesis with disagreement preserved.

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

### Panel invariants (the panel's constitution)

- **Adversarial Presence** — the panel MUST contain ≥1 adversarial role (Skeptic / Security /
  Scope-cutter). Default invariant-holder is `Skeptic`. (Replaces the narrow rule
  "Skeptic can't be removed" with the general principle.)
- **Size** — 3 to 7 active roles (diversity of opinion vs. cognitive overload and cost).
- **No redundancy** — do not include roles with heavily overlapping mandates simultaneously
  (e.g. Optimizer + Performance-hawk), see `conflicts_with` in role metadata.

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

### Rounds & stop conditions

- **Hard cap: R1 + optional R2. Never R3.** If a single model hasn't converged after 2 rounds —
  that's a signal to escalate to the external `consensus-protocol` (Claude+Gemini), not to spin the panel further.
- R2 only runs if disputed theses remain after aggregation; otherwise — early finalize.
- Stagnation (R2 arguments are a semantic rehash of R1) → the thesis is marked `STAGNATED`, R3 is not invoked.

### Anti-groupthink

- The Adversarial Presence invariant (see above) — a structural safeguard.
- `GROUPTHINK_FLAG` — the Judge raises this if ALL theses are AGREED/AGREED_WEAK across ALL roles.
- Optional external voice: Gemini CLI as a triggered fallback (conditions — see the Gemini fallback step).
  This is the only point where the skill touches a second model, and only as a hedge against
  unanimity, not as a default mode (multi-model dialogue is `consensus-protocol`, a separate skill).

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
- User phrases: "claude consensus", "consensus panel", "run a consensus".
- Slash command: `/consensus-claude <question>` (flags optional, see below).

**First action** (mandatory, before any subagent calls): print to the user:

> Using consensus-claude: checking scope applicability.

Immediately after — Triage (Step 0 below).

---

## Flags

| Flag | Effect |
|---|---|
| `--with-gemini` | Force the Gemini CLI fallback regardless of GROUPTHINK_FLAG |
| `--save-adr` | Save an ADR even for a standard run (by default no ADR is written) |

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

- Question is in scope → continue to Step 1.
- Question is out of scope → polite decline, panel does NOT run. Template: `This is outside consensus-claude's scope (see Scope) — you need <a lawyer/other specialist>, not a decision-review panel.`

### Step 1 — Print classification line

Immediately after the greeting and successful Triage, print the classification line:

```
🧠 consensus-claude · Panel: <active roster> (+Decomposer +Judge)
```

Where `<active roster>` is the list of roles from Protocol · Roster (dynamic, not a literal in this file).

Save the user's parsed flags (`--with-gemini`, `--save-adr`) for use in later steps.

### Step 2 — Decomposer dispatch

Call ONE subagent:

```
Agent(
  subagent_type="consensus-claude-decomposer",
  description="Decompose into theses",
  model="opus",
  prompt="Decompose: <verbatim user question>"
)
```

Parse the response. Expected format:

```
CANONICAL_INTENT: <gist of the task in 1–2 sentences>
T1: ...
T2: ...
...
Tn: ...
```

**Single-thesis fallback:** if the response contains fewer than 2 valid theses (e.g. the Decomposer produced only T1, or nothing parseable at all) — enter single-thesis mode: treat the user's entire original question as one thesis T1 verbatim, and the Canonical Intent as the question itself. This is a lower-value mode, but the skill doesn't fail outright. Print to the user: `ℹ️ Single-thesis fallback: the task didn't decompose, evaluating as a single thesis T1.`

Record the Canonical Intent and the final list of theses in variables — they're needed in every following step.

### Step 3 — R1 panel: dispatch panel subagents IN PARALLEL (single message)

**Critical:** all roster calls go in ONE message from the main model, as parallel tool-use (parallel function calls). This is required for R1 isolation (each agent in a clean context, none sees another's answers).

Dispatch the entire active roster (see Protocol · Roster) in a single message.

Give each the **same** prompt:

```
Original question: <verbatim user question>

Canonical Intent: <text from Step 2>

Panel members in this run: <active roster>

Theses:
T1: <text>
T2: <text>
...
Tn: <text>
```

(The `model: opus` parameter is set in the agent definitions themselves; no need to pass it in the Agent call.)

Wait for all roster subagents to return. Each returns:
- **Block 1:** line-by-line votes `T<n>: <STATUS> [comment]` (comment may contain `CONDITION: <...>`, see Protocol · Status vocabulary).
- **Block 2 (Optimizer and Maintainability-advocate only):** a structured `DISAGREEMENT:` block (`TARGET / THESIS / COUNTER`) or fallback `COUNTER: NO_DISAGREEMENT — ...`.

If any agent returned unparseable output → mark its votes as `invalid` (see the Edge cases section above).

### Step 4 — Judge dispatch (Phase A — aggregation)

Call ONE subagent:

```
Agent(
  subagent_type="consensus-claude-judge",
  description="Aggregate R1 votes",
  model="opus",
  prompt="Phase A. Original question: <q>. Canonical Intent: <intent>. Theses: <T1..Tn full text>. R1 votes:\n\nOptimizer:\n<full Optimizer output>\n\nSkeptic:\n<full Skeptic output>\n\nSecurity:\n<full Security output>\n\nMaintainability-advocate:\n<full Maintainability-advocate output>"
)
```

Parse the Judge's output as a `ROUND_SUMMARY`:
- `AGREED: [...]` — theses on which everyone voted AGREED/AGREED_WEAK without substantial reservations.
- `AGREED_WEAK_THESES: [...]` — list of `{thesis, role, nuance}` objects.
- `DISPUTED_THESES: [...]` — theses with DISPUTED from at least one role (or NEEDS_CLARIFICATION).
- `NEEDS_CLARIFICATION: [...]` — separate list.
- `CONDITIONS: [...]` — list of `{thesis, role, condition}` objects, extracted from CONDITION tags (see Protocol · Status vocabulary).
- `GROUPTHINK_FLAG: <true|false>` — true if ALL theses are AGREED or AGREED_WEAK across ALL agents.

Save this data — needed in Steps 5–8.

### Step 5 — Gemini fallback decision

Trigger condition (fires if AT LEAST ONE is met):

1. `GROUPTHINK_FLAG == true` **AND** at least one thesis mentions security-adjacent terms: `auth`, `authn`, `authz`, `secret`, `token`, `pii`, `credential`, `credentials`, `encryption`, `encrypt`, `decrypt` (case-insensitive substring match against thesis text).
2. The user passed the `--with-gemini` flag.

If neither condition is met → skip this step, set `gemini_status = not_required`, go to Step 6.

(See Protocol · Anti-groupthink — this is the only point where the skill touches a second model, and only as a hedge, not a default mode.)

**If the fallback fires — run a bash call via the Bash tool:**

Prompt for the CLI:

```
You are an external critic. For each thesis below, return ONE line in EXACTLY this format:
T<n>: <AGREED|AGREED_WEAK|DISPUTED|NEEDS_CLARIFICATION> [<comment if not AGREED>]

No preamble, no commentary outside this format.

Theses:
T1: <thesis text>
T2: <thesis text>
...
```

Command:

```bash
timeout 30s gemini -p "<prompt>" 2>/dev/null
```

Handle exit codes:
- **0** → parse stdout. Expected format: one line per thesis, regex `^T[0-9]+: (AGREED|AGREED_WEAK|DISPUTED|NEEDS_CLARIFICATION)( .*)?$`. Set `gemini_status = triggered`.
- **42** → invalid input (empty/malformed prompt). Log `gemini_invalid_input`. Set `gemini_status = invalid_input`. Continue without Gemini.
- **any other non-zero / timeout** → log `gemini_cli_unavailable`. Set `gemini_status = cli_unavailable`. Continue without Gemini.

**Cross-language note:** if the user wrote in a non-English language, Gemini will still return English output. This is fine for parsing. The Judge translates comments into the user's language during synthesis in Step 7.

**Gemini's effect on R2 scope:** if Gemini brought back at least one `DISPUTED` on a thesis that R1 had settled as `AGREED` — add that thesis to `DISPUTED_THESES` for R2 (even if the entire roster was unanimous).

### Step 6 — R2 dispatch (only DISPUTED theses)

If `DISPUTED_THESES` is empty after Step 4 + Step 5 → **early finalize**, go straight to Step 7 without R2.

Otherwise — dispatch the same roster IN PARALLEL (single message, as in Step 3) with the R2 prompt:

```
Original question: <verbatim user question>

Canonical Intent: <text from Step 2>

You participated in R1. Here are the OTHER panel members' R1 positions on the DISPUTED theses (and Gemini's verdict if present).

Panel positions from R1:
Optimizer (on disputed):
<Optimizer's vote lines for disputed theses only>
Skeptic (on disputed):
<...>
Security (on disputed):
<...>
Maintainability-advocate (on disputed):
<...>
<if Gemini fired, add:>
Gemini (external critic):
<Gemini's lines for disputed theses only>

DISPUTED theses to re-evaluate (only these — others are settled):
T<n>: <text>
...

For each disputed thesis, return ONE line in same format. You may stick, change, or refine your R1 position. But provide NEW substance — repeating R1 verbatim is stagnation.
```

Wait for the entire roster. If a role continues to emit a DISAGREEMENT block (Optimizer / Maintainability-advocate) — keep it, pass it to the Judge.

### Step 7 — Judge dispatch (Phase B + C — stagnation check + final synthesis)

Call the Judge a second time:

```
Agent(
  subagent_type="consensus-claude-judge",
  description="Final synthesis",
  model="opus",
  prompt="Phase B+C. Original question: <q>. Canonical Intent: <intent>. Theses: <T1..Tn full text>. R1 votes:\n<full roster + Gemini if it ran>\n\nR2 votes (disputed only):\n<full roster on disputed theses>"
)
```

The Judge does:
- **Phase B** — for each thesis still DISPUTED after R2, compares R2's arguments against R1. If the arguments are a semantic rehash of R1 → mark the thesis `STAGNATED: true`. Do not invoke R3.
- **Phase C** — structured final synthesis in the 8-section schema defined in the Judge mandate (rendered as the Step 8 template below): Consensus, Holism check, Trade-offs, Blockers, Prerequisites, Nuances, Unresolved, Devil's advocate — Holism check and Devil's advocate always required.

Save the Judge's output — it goes into Step 8 verbatim.

### Step 8 — Render output to user

Print to the user as a single block (all in the user's language, per Protocol · Output language, except the internal statuses AGREED/DISPUTED/AGREED_WEAK/NEEDS_CLARIFICATION — these stay as technical markers, see Protocol · Status vocabulary, + the CONDITION tag).

The template below is shown in English as the reference; render it in the user's language in practice:

```
🧠 consensus-claude
Panel: <active roster>
Gemini fallback: <triggered|skipped|invalid_input|cli_unavailable|not_required>

Round 1/2:
  Agreed: <list of thesis numbers from Judge Phase A AGREED>
  Disputed: <list of thesis numbers from DISPUTED_THESES>
  AGREED_WEAK nuances: <list of thesis numbers from AGREED_WEAK_THESES>
  <if Gemini fired:>
  Gemini brought: <list of theses where it returned DISPUTED + brief Gemini comments, translated into the user's language if needed>

<if R2 ran:>
Round 2/2 (disputed only):
  T<n> → <accepted|accepted with amendment|stagnated|still DISPUTED>
  ...

<Judge Phase C output verbatim — sections ✅ Consensus / 🔗 Holism check / 🔀 Trade-offs (value-tensions) / 🚫 Blockers (error-catches) / 📎 Prerequisites (conditions) / ⚠️ Nuances / ❓ Unresolved / 😈 Devil's advocate>
```

Mapping `gemini_status → Output line`:
- `triggered` → `triggered (<reason: groupthink+security | --with-gemini>)`
- `not_required` → `not_required`
- `invalid_input` → `skipped (invalid_input)`
- `cli_unavailable` → `skipped (cli_unavailable)`

If `Round 2/2` didn't run (early finalize) — omit the section entirely, print nothing about R2.

### Step 9 — ADR file generation

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
