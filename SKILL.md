---
name: consensus-claude
description: Multi-perspective consensus via single-model subagents with distinct roles. Use when the user says "claude consensus" / "consensus panel" or invokes /consensus-claude. Claude-only mode by default; pulls in an external critic (any CLI, e.g. Gemini) as anti-groupthink fallback for architecture/security or when all agents agree unanimously.
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
- **Decompose** — the Decomposer produces `Canonical Intent` (the gist of the task in 1–2 sentences) + `T0` (Premise Distillation — the question's foundational unstated premise, voted like any thesis) + atomic theses T1..Tn.
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

**Other rationale tags.** Alongside `CONDITION`, a vote's rationale may carry: `FLIP: <observable
evidence that would reverse this vote>` — mandatory on every `DISPUTED` vote; it turns a dispute into
a testable position, and the Judge derives revisit-Tripwires from FLIP tags; `ANCHOR:
"<quote>"` — a verbatim quote from the question/spec/code the vote is grounded in (required when the
vote rests on a specific claim in the source); `[impact: critical|moderate|minor]` — the voter's own
severity read, mandatory whenever the vote is `DISPUTED` or `AGREED_WEAK`. All three are optional
elsewhere and, like `CONDITION`, live inside the rationale text rather than expanding the status
vocabulary.

### Rounds & stop conditions

- **Hard cap: R1 + optional R2. Never R3.** If a single model hasn't converged after 2 rounds —
  that's a signal to escalate to the external `consensus-protocol` (Claude+Gemini), not to spin the panel further.
- R2 only runs if disputed theses remain after aggregation; otherwise — early finalize.
- Stagnation (R2 arguments are a semantic rehash of R1) → the thesis is marked `STAGNATED`, R3 is not invoked.
- The final synthesis opens with `CONSENSUS_STRENGTH` (`Strong` / `Working` / `Narrowly carried` / `Contested`) — the Judge's own one-word read on how solid the verdict is, emitted before the rest of Phase C.

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
- User phrases: "claude consensus", "consensus panel", "run a consensus".
- Slash command: `/consensus-claude <question>` (flags optional, see below).

**Deliberateness gate.** The two trigger paths carry different intent signal:
- **Slash command** (`/consensus-claude <question>`) is always a deliberate, explicit invocation → proceed directly to Triage. The Step 1 estimate line is sufficient; no confirmation is needed.
- **Trigger phrase in conversation** ("claude consensus" etc.) may be a casual aside, not a deliberate request for a full panel run → before dispatching any subagent, the orchestrator must print ONE confirm line: `A full panel run costs ~6-11 model calls and takes several minutes. Proceed?` and wait for the user's confirmation before continuing to Triage.

**First action** (mandatory, before any subagent calls): print to the user:

> Using consensus-claude: checking scope applicability.

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
| `--emergent` | Experimental Schrödinger mode: the problem defines its own panel (see Emergent panel mode) |

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

Immediately after the greeting and successful Triage, print the classification line, extended with an upfront cost estimate so the user sees the cost before any subagent is dispatched:

```
🧠 consensus-claude · Panel: <active roster> (+Decomposer +Judge) · ~6-11 model calls · several minutes
```

Where `<active roster>` is the list of roles from Protocol · Roster (dynamic, not a literal in this file). The `~6-11 model calls` figure is fixed: 6 = R1 minimum (1 Decomposer + 4 votes + 1 Judge pass); 11 = with a full targeted R2 (+4 re-votes +1 additional Judge pass).

(For the trigger-phrase path, this line is printed only after the user has already confirmed via the Trigger section's deliberateness gate — it is not itself the confirmation.)

Save the user's parsed flags (`--with-external`, `--save-adr`, `--no-record`) for use in later steps.

### Step 2 — Decomposer dispatch

Call ONE subagent:

```
Agent(
  subagent_type="consensus-claude-decomposer",
  description="Decompose into theses",  prompt="Decompose: <verbatim user question>"
)
```

Parse the response. Expected format:

```
CANONICAL_INTENT: <gist of the task in 1–2 sentences>
T0: <Premise Distillation — the question's foundational unstated premise>
T1: ...
T2: ...
...
Tn: ...
```

**Single-thesis fallback:** if the response contains fewer than 2 valid theses (e.g. the Decomposer produced only T1, or nothing parseable at all) — enter single-thesis mode: treat the user's entire original question as one thesis T1 verbatim, and the Canonical Intent as the question itself. This is a lower-value mode, but the skill doesn't fail outright. Print to the user: `ℹ️ Single-thesis fallback: the task didn't decompose, evaluating as a single thesis T1.` (T0 is still produced and voted normally in this fallback — only T1..Tn collapses to one item.)

Record the Canonical Intent and the final list of theses (T0..Tn) in variables — they're needed in every following step.

### Step 3 — `--plan` checkpoint (only if the flag was passed)

If the `--plan` flag was NOT passed → skip this step entirely, continue straight to Step 4.

If `--plan` WAS passed → print the Canonical Intent and the full thesis list (T0..Tn) from Step 2, then stop and print:

> Reply "approve" to dispatch the panel, or reply with edited theses (same T\<n\> format) to use your version.

Wait for the user's next message:
- **"approve"** (or equivalent) → continue to Step 4 using the theses exactly as produced in Step 2.
- **Edited theses** (a reply in the same `T<n>: <text>` format) → replace the in-memory Canonical Intent/theses with the user's version, then continue to Step 4 using the edited set. No re-dispatch of the Decomposer — the user's reply IS the new thesis list.

No subagent calls happen in this step — it's a pure orchestrator print-and-wait, spending zero of the panel's model calls, which is the point of the flag (approve/edit before any panel votes are spent).

### Step 4 — R1 panel: dispatch panel subagents IN PARALLEL (single message)

**Critical:** all roster calls go in ONE message from the main model, as parallel tool-use (parallel function calls). This is required for R1 isolation (each agent in a clean context, none sees another's answers).

**`--roles` override (only if the flag was passed).** Before assembling the dispatch, apply the
override to the active roster from Protocol · Roster:
- `-Role` removes that role from the active roster.
- `+Shelf-role` enables a role from the shelf (DESIGN.md · Role Library names: `Performance-hawk`,
  `Product-thinker`, `Scope-cutter`, `Simplicity-advocate`, `Domain-expert`) into the active roster.
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

**Emergent panel mode (`--emergent`) replaces this entire step** with a different dispatch flow —
see the dedicated subsection below. Skip straight there if `--emergent` was passed; the rest of
this step (and the fixed-roster prompt below) does not apply.

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

#### Emergent panel mode (`--emergent`) — experimental, replaces the fixed-roster dispatch above

**Experimental.** Only runs if the `--emergent` flag was passed; it replaces this entire step's
fixed-roster dispatch with a different flow. Cost note: up to ~9-13 model calls (higher than the
default ~6-11, see Step 1). Honest labeling: the panel composition itself becomes non-deterministic
under this mode — the trade is novelty of perspectives vs. reproducibility of the panel. Print this
trade-off to the user before dispatching (one line) so it isn't a silent cost/determinism change.

Flow:
(a) One extra subagent call, made right after Step 2's Decomposer output (before any voting):

```
Agent(
  subagent_type="consensus-claude-decomposer",
  description="Generate emergent perspectives",  prompt="Generate the 5-7 strongest genuinely distinct perspectives/core arguments on this question — one line each, no personas yet. Question: <verbatim user question>. Canonical Intent: <text from Step 2>."
)
```

(b) The orchestrator retroactively mints an ephemeral persona to champion each returned perspective
— same mechanics as a `--roles +name(focus:'...')` persona (Step 4, `--roles` override, above): a
name, a one-line mandate derived from the perspective, and the full vote-format contract (statuses +
CONDITION/FLIP/ANCHOR/[impact] tags) injected into its prompt. Cap at max 7 personas, min 3 — if
fewer than 3 distinct perspectives came back, pad with golden-roster roles (Skeptic first) to reach
the minimum.

(c) Adversarial Presence check: if none of the generated personas is adversarial by construction,
add `Skeptic` from the golden roster (Protocol · Roster) to the emergent panel — same invariant as
the `--roles` guard, applied here because the roster isn't fixed in advance under this mode.

(d) Proceed with the normal per-thesis R1/R2/Judge flow (Steps 4 onward, i.e. this step's dispatch
mechanics and Steps 5–9) using these emergent personas in place of the golden roster everywhere
"active roster" is referenced.

(e) The run record (Step 10) lists the emergent roster (names + one-line mandates) and marks the
run `mode: emergent` in its `AUDIT_BLOCK`.

### Step 5 — Judge dispatch (Phase A — aggregation)

Call ONE subagent:

```
Agent(
  subagent_type="consensus-claude-judge",
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

Save this data — needed in Steps 6–9.

### Step 6 — External critic decision

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

**Rung 1 — configured CLI.** The Protocol block's configurable external-critic command line defines the CLI to invoke (default worked example: `gemini -p`). Any CLI that accepts a prompt (as an argument or via stdin) and prints text qualifies — `codex exec`, `ollama run <model>`, etc. Run it via the Bash tool:

```bash
timeout 30s <configured command> "<prompt>" 2>/dev/null
```

Handle exit codes:
- **0** → parse stdout with the regex above. If it produces at least one valid line → set `external_status = triggered (cli)`. If nothing parses → set `external_status = invalid_output`, continue without the critic (do not fall through to Rung 2 — a CLI that ran but produced garbage is a parsing failure, not an availability failure).
- **any non-zero / timeout / binary not found** → the CLI is unavailable. Fall through to Rung 2.
- **no command configured at all** → skip straight to Rung 2.

**Rung 2 — manual handoff.** Print the exact critic prompt from above to the user, followed by:

> Paste this into any other model you have (web chat is fine), then paste its reply back — or reply "skip".

Wait for the user's next message.
- If the user pastes a reply → parse it with the SAME regex as the CLI path. If it produces at least one valid line → set `external_status = triggered (manual)`. If nothing parses → set `external_status = invalid_output`, continue without the critic.
- If the user replies "skip" (or equivalent decline) → set `external_status = skipped (user_declined)`, continue without the critic.

**Rung 3 — skip.** Reached only via an explicit user decline in Rung 2. `external_status = skipped (user_declined)`, run continues (graceful degradation as before).

**Cross-language note:** the external model may reply in another language — statuses parse the same regardless (the format is a fixed token, not prose). The Judge translates comments into the user's language during synthesis in Step 8.

**External critic's effect on R2 scope:** if the critic brought back at least one `DISPUTED` on a thesis that R1 had settled as `AGREED` — add that thesis to `DISPUTED_THESES` for R2 (even if the entire roster was unanimous).

### Step 7 — R2 dispatch (only DISPUTED theses)

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

DISPUTED theses to re-evaluate (only these — others are settled):
T<n>: <text>
...

For each disputed thesis, return ONE line in same format. You may stick, change, or refine your R1 position. But provide NEW substance — repeating R1 verbatim is stagnation.
```

Wait for the entire roster. If a role continues to emit a DISAGREEMENT block (Optimizer / Maintainability-advocate) — keep it, pass it to the Judge.

### Step 8 — Judge dispatch (Phase B + C — stagnation check + final synthesis)

Call the Judge a second time:

```
Agent(
  subagent_type="consensus-claude-judge",
  description="Final synthesis",  prompt="Phase B+C. Original question: <q>. Canonical Intent: <intent>. Theses: <T1..Tn full text>. R1 votes:\n<full roster + external critic if it ran>\n\nR2 votes (disputed only):\n<full roster on disputed theses>"
)
```

The Judge does:
- **Phase B** — for each thesis still DISPUTED after R2, compares R2's arguments against R1. If the arguments are a semantic rehash of R1 → mark the thesis `STAGNATED: true`. Do not invoke R3.
- **Phase C** — opens with `CONSENSUS_STRENGTH` (`Strong`/`Working`/`Narrowly carried`/`Contested`, see Protocol · Rounds & stop conditions), followed by the structured final synthesis in the 8-section schema defined in the Judge mandate (rendered as the Step 9 template below): Consensus, Holism check, Trade-offs, Blockers, Prerequisites, Nuances, Unresolved, Devil's advocate — Holism check and Devil's advocate always required.

Save the Judge's output — it goes into Step 9 verbatim.

### Step 9 — Render output to user

Print to the user as a single block (all in the user's language, per Protocol · Output language, except the internal statuses AGREED/DISPUTED/AGREED_WEAK/NEEDS_CLARIFICATION — these stay as technical markers, see Protocol · Status vocabulary, + the CONDITION tag).

The template below is shown in English as the reference; render it in the user's language in practice:

```
🧠 consensus-claude
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

Run record: <./consensus-runs/<file>.md | skipped (--no-record)>
```

Mapping `external_status → Output line`:
- `not_required` → `not_required`
- `triggered (cli)` → `triggered (cli — <reason: groupthink+security | --with-external>)`
- `triggered (manual)` → `triggered (manual — <reason: groupthink+security | --with-external>)`
- `skipped (user_declined)` → `skipped (user_declined)`
- `invalid_output` → `skipped (invalid_output)`

If `Round 2/2` didn't run (early finalize) — omit the section entirely, print nothing about R2.

### Step 10 — Verifiable Decision Record

ON by default; opt out with `--no-record` (see Flags). Rationale: this record is the product's auditability story — the artifact to attach when reporting a weak or disputed verdict, since it lets anyone re-derive the synthesis from the raw votes without re-running the panel.

If `--no-record` was passed → skip this step silently, and print `Run record: skipped (--no-record)` in Step 9's output (already covered by the template above).

Otherwise, write a structured markdown transcript to `./consensus-runs/YYYY-MM-DD-<slug>.md` relative to cwd (create the `consensus-runs/` directory if it doesn't exist). `<slug>` uses the same rule as the ADR step below: the first 5 significant words of the user's question, lowercase, kebab-case, no punctuation. `YYYY-MM-DD` is today's date.

All content below is already in-context from prior steps — this step performs no new computation, only assembly. The record now OPENS with a machine-readable `AUDIT_BLOCK` — a fenced block of `key: value` lines at the very top of the same file, purpose-built so that this block (or the whole record) can be handed to ANY external model for a post-hoc audit, provider-agnostic:

```
AUDIT_BLOCK
date: YYYY-MM-DD
question: <one-line, truncated if needed>
mode: <standard|emergent>
panel: <active roster from Protocol · Roster (or the emergent roster, see Step 4 · Emergent panel mode), +Decomposer +Judge>
consensus_strength: <Strong|Working|Narrowly carried|Contested>
verdict: <one-line summary of the Judge's Phase C consensus>
key_assumptions:
  - <bulleted, drawn from T0 (Premise Distillation) + any accepted CONDITION premises>
  - ...
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

---
Generated by consensus-claude vX · LLM output is non-deterministic; this record documents this specific run.
```

Save the file path — it is printed as the final line of Step 9's output template (`Run record: ./consensus-runs/<file>.md`).

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

### Step 11 — ADR file generation

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
