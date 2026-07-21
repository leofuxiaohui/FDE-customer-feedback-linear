---
name: PM-feedback-triage-linear
description: Use when a PM wants to understand, triage, or aggregate Agentforce customer feedback that FDEs/SEs have shared on Linear (AFP-* issues) — without a meeting. Reads the feedback issues and their in-app evidence documents, parses the machine-readable feedback-record blocks written by the companion FDE-customer-feedback-linear skill, answers the PM's questions with citations, builds cross-customer rollups (by feature, gap type, severity, impact), and posts follow-up questions back on the thread asynchronously. Also records the PM's decision back onto the issue as a machine-readable `triage_record` block and drives the loop to a defined closed state.
---

# PM Feedback Triage ← Linear

The PM-side companion to `FDE-customer-feedback-linear`. FDEs/SEs publish structured feedback (an in-app evidence Document + summary comment, ending in a machine-readable `feedback_record: v1` block). This skill lets a PM's coding agent **consume** that feedback: answer questions about a single item, roll up patterns across many, and close remaining gaps by commenting on the thread — so neither side needs a meeting.

## When to use

- "What is AFP-123 actually about / is it real / what's the ask?"
- "Across all Testing Center feedback, what are the top asks?" / "Which items are adoption-blocking?" / "What did customer X report this quarter?"
- "Draft my follow-up questions for AFP-123" (posted on the thread, not asked in a call)
- "Build a triage rollup doc for this month's feedback"

Do **not** use this to author FDE feedback (that's `FDE-customer-feedback-linear`) or to change issue states/priorities without the PM explicitly asking.

## Prerequisites — Linear workspace

Feedback issues (`AFP-*`) live in the **`eventsmobileapp`** workspace, team **"Agentforce Feedback"**. The Linear MCP connects to one workspace at a time — verify with `get_issue AFP-###` or `list_issues` with `team: "Agentforce Feedback"`. If access fails, have the PM reconnect via `/mcp` → linear → pick **eventsmobileapp**.

## How feedback is structured (the contract)

Each feedback item is one `AFP-*` issue carrying:
1. **Intake description** — customer name, feature family/group, feedback theme, business outcome (the original form fields).
2. **Summary comment** — TL;DR, ask, open questions, link to the doc.
3. **Evidence Document** (linked to the issue) — the deep dive. Its last section is a fenced ```yaml block starting `feedback_record: v1`:

```yaml
feedback_record: v1
issue: AFP-###
date: YYYY-MM-DD
customer: <name | internal-demo>
feature_family: <e.g. Testing Center>
gap_type: bug | limitation | feature-request | docs-gap
severity: blocker | workaround-exists | inconvenience
customer_impact: adoption-blocking | deal-risk | productivity-loss | confidence-loss   # may be a comma-separated list
repro_status: reproduced-live | reported-only | intermittent | not-reproducible
root_cause_known: true | false
root_cause: <one line>
asks: [<primary>, <alternative>]
workaround: <one line | none>
evidence_doc: <url>
open_questions: [<…>]
notes: <optional>
```

The doc also contains a **"Questions a PM will ask"** section with pre-answered follow-ups and explicitly flagged open questions — read it before drafting your own questions; most will already be answered.

**Fallback:** older or hand-written docs may lack a record block. Parse the prose instead and infer the same fields; mark inferred values as `(inferred)` in any rollup so they aren't mistaken for FDE-asserted facts.

## The triage_record schema (v1) — what the PM side emits

The FDE→PM direction is machine-readable (`feedback_record`); the PM→FDE direction must be too. Every triage disposition comment ends with exactly one fenced ```yaml block:

```yaml
triage_record: v1
issue: AFP-###
date: YYYY-MM-DD
disposition: planned | shipped | wontfix | duplicate | needs-evidence   # exactly one — terminal states of the loop
accepted_asks:
  - <which of the record's asks were accepted, verbatim; empty list if none>
declined_asks:
  - <asks not taken, with a short "why" in parens; omit if none>
duplicate_of: <AFP-### — only when disposition: duplicate>
target: <epic/project/release the work lands in, or "unscheduled">
tracking:                       # register ID -> delivery work issue; added at planned/shipped
  A2: <issue-id or url>
epic: <roadmap parent name/url, or "unscheduled">
owner: <who carries the next action — a person, "FDE", or "PM">
next_action: <one line — the single next concrete step>
loop_status: closed | open-with-FDE | open-with-PM | open-with-customer   # who the ball is with
decided_date: YYYY-MM-DD
notes: <optional nuance>
```

**Enum meanings:**
- `disposition` — `planned` (accepted, scheduled on roadmap/epic), `shipped` (fix released), `wontfix` (declined, reason required in the comment), `duplicate` (fold into `duplicate_of`), `needs-evidence` (parked; ball with FDE until evidence arrives).
- `loop_status` — `closed` (all open_questions resolved AND disposition committed to issue fields AND stated on thread), else `open-with-<role>` naming who owes the next move.

`tracking` is filled once the decision becomes tracked work (Procedure E); before that, omit it.

## Procedure

### The register (ask/answer ledger)

Every **item** in the loop — an ask, a question, a follow-up, a decision — lives in a **register table with a stable ID and a status token**, never as a loose bullet. Prose is for rationale *around* the register. The same table shape runs from the first ask to the close, and the **closing comment carries the complete register** so the outcome reads in one scan.

| ID | Item | Owner | Status | Resolution |
|---|---|---|---|---|
| A1 | \<the item, one line\> | PM | ⏳ open | — |

**Status tokens:** ⏳ open · 📋 accepted · 🔎 sized (accepted in principle, lives in an epic) · 🔴 blocked (needs customer / external input) · ✅ resolved · ❌ declined (reason in prose)

**ID prefixes:** `A#` = FDE ask · `Q#` = FDE evidence-limit question (about the boundary of the FDE's own investigation) · `F#` = PM follow-up to the FDE · `P#` = PM-owned triage action (e.g. a telemetry pull)

IDs are also the join key to the machine records: an `A1` in the register is the same `A1` referenced in `feedback_record`/`triage_record`.

#### Keep the register readable

The register is an **index, not the content** — every cell is a short **label (≤ ~6 words)**, never a sentence. The full ask, answer, and rationale live in the prose around the table. Two layouts, by round:

- **Posing a round** (asks/questions, no answers yet): a terse **4-column** table — omit `Resolution` entirely until answers exist:

  | ID | Item | Owner | Status |
  |---|---|---|---|
  | A1 | Execute prior turns to persist state | PM | ⏳ open |

- **Answering / closing** (resolutions carry reasoning): use **stacked entries**, one per item, so each gets full comment width:

  > **A1 · 🔎 sized · owner PM** — Execute prior turns to persist state
  > → *Resolution:* accepted in principle; sized in the multi-turn epic.

Re-post only the rows that **changed** each round; the **complete register appears once, at the close**, in stacked form. If a cell needs a clause to be understood, that's the signal it belongs in prose, not the table.

### A. Understand one feedback item
1. `get_issue AFP-###` — read the intake description and note the customer/feature fields.
2. `list_comments` on the issue — find the summary comment and any thread discussion since.
3. The issue's `documents` list names its evidence docs — `get_document` each; read TL;DR → verdict → root cause → asks → PM-questions section → the record block.
4. Answer the PM's question **with citations**: issue ID, doc title + URL, and section. Distinguish FDE-asserted facts from your inferences.
5. If the PM has follow-ups the doc doesn't answer: check the doc's *open questions* first (the FDE may have flagged exactly that), then draft the questions **as register rows** (`F#`, Owner = FDE or customer, Status ⏳ open or 🔴 blocked if customer-dependent) — not loose bullets — and, on the PM's confirmation, post them as a `save_comment` on the issue thread.
6. **Prevalence is a PM question.** When triaging, raise blast-radius / prevalence questions yourself as `P#` register items (e.g. "pull telemetry on how many suites are affected") — do not expect the FDE to supply prevalence; the FDE only has their one case.

### B. Aggregate across many items
1. Enumerate candidates: `list_issues` with `team: "Agentforce Feedback"` (filter by `createdAt`/`updatedAt` for a time window, or `query` for a feature keyword). Page through — don't stop at the first page.
2. For each issue: `get_issue` → `get_document` its evidence doc(s) → extract the `feedback_record` block (fallback: infer from prose, mark `(inferred)`).
3. Build the rollup in whatever cut the PM asked for — by `feature_family`, `gap_type`, `severity`, `customer_impact`, `customer`, or recurring `asks`. Count items, list the issue IDs behind every number, and surface: **blockers first**, then adoption-blocking/deal-risk impacts, then clusters of identical asks (same ask from N customers = strongest signal).
4. Deliver as chat, or — if the PM wants it shareable — as a Linear Document (`save_document` on the PM's chosen project/team/issue) using the rollup template below.
5. Collect every record's `open_questions` into a "needs FDE input" list; offer to post each back to its issue thread as a comment.

### C. Close the loop asynchronously
- Post PM follow-ups as register rows (`F#`, Status ⏳ open) in a comment on the specific issue (`save_comment` with `issueId`) — the FDE answers on the thread by updating the row's status.
- If the PM disposition is "accepted / roadmap / duplicate-of-X / needs-more-evidence", offer to post that as a comment too, so the FDE isn't left waiting.

### D. Record the decision on the issue
1. Draft the disposition (using the enums above) and show it to the PM. **Only on the PM's explicit confirmation** (this preserves the "don't silently mutate" guardrail — see Guardrails below) apply it to the issue itself via `save_issue`: set `state` (e.g. Backlog → Planned/Canceled per disposition), `priority`, and add a label for the feature family if one exists. The issue's own fields are the source of truth; the comment is the rationale.

   | `disposition` | Linear issue state |
   |---|---|
   | `planned` | Todo (or In Progress once work starts) |
   | `shipped` | Done |
   | `wontfix` | Canceled |
   | `duplicate` | Duplicate |
   | `needs-evidence` | Triage |

   Also set **`assignee`** to the current owner of the next action, so the board shows who's up — reassign as ownership moves across rounds.
2. Post the disposition comment **threaded under the FDE's summary comment** (`save_comment` with `parentId: <FDE comment id>`), ending with the fenced `triage_record: v1` block.
3. If `disposition: planned`, link the target epic/project in the comment; if `wontfix`, state the reason in prose above the record; if `duplicate`, also post a one-liner on the duplicate target issue pointing back.
4. Report what was changed on the issue and what the record says.

### E. From decision to delivery (close the real loop)

A `planned`/`shipped` disposition is a decision, not a delivery. The loop only truly closes when the decision becomes **tracked work** and the outcome reaches the customer. **Only on the PM's explicit confirmation** — same guardrail as Procedure D — before creating any issues:

1. **Spin up tracked work** — one work issue per accepted deliverable, titled with its register ID (`[AFP-###·A2] …`), created in the delivery team's epic/project. Feedback issues live in `eventsmobileapp`; eng work lives in the dev workspace and **sub-issues cannot cross workspaces** — so create the delivery issues in the eng team and **cross-link by URL** (a Linear relation), or, when tracking inside the feedback team, create them as sub-issues of the feedback issue. State which you did.
2. **Record the mapping** in the `triage_record` `tracking:` block (register ID → work issue) and set `epic:`.
3. **Roll up** — when every tracked child is Done, move the feedback issue to Done.
4. **Close back to the customer** — post a short comment addressed to CS summarizing what shipped (by register ID), so CS can relay it to the customer. This is the real close; a disposition alone leaves the customer uninformed.

## When is the loop closed?

- **Two different closures.** The **FDE↔PM triage loop** closes when open items resolve and a disposition is committed to the issue; **the feedback is fully closed** only when the tracked work ships and the customer is informed (Procedure E). Terminal `disposition` states remain as defined below; a `planned` disposition implies delivery tracking follows via Procedure E.
- **Terminal states:** `planned`, `shipped`, `wontfix`, `duplicate` are **closed**; `needs-evidence` is **parked** (open, ball with FDE).
- **The three-part closure test** (all must hold): (1) every `open_questions` entry from the `feedback_record` is answered on the thread; (2) the disposition is committed to the issue's own fields (state/priority/label), not just prose; (3) a closing comment states it so CS/FDE/customer can see the outcome.
- **Converge, don't count.** Each exchange must *shrink the open set* in the register. If open items aren't trending to zero — or a round adds more than it closes — that's the signal to escalate to a sync. The skills remove *routine* meetings, not genuine deadlocks. (Most items settle in one or two exchanges; treat that as observation, never a pass/fail bar.)

## Rollup document template

Per-item registers posted in issue comments follow CANON D (labels, not sentences) — see *Keep the register readable* above. The rollup below is a different artifact (a cross-issue index for a PM audience), and its columns may stay tabular.

```markdown
> **Feedback rollup** — <scope: time window / feature / team> · generated <date> · <N> items reviewed

## Headlines
- <the 2–4 things a PM must know, each citing issue IDs>

## By severity
| Severity | Count | Issues |
|---|---|---|
| blocker | n | AFP-…, AFP-… |
| workaround-exists | n | … |
| inconvenience | n | … |

## By feature / gap type
| Feature | bug | limitation | feature-request | docs-gap |
|---|---|---|---|---|
| <family> | n | n | n | n |

## Recurring asks (strongest signal)
1. **<ask>** — requested in AFP-…, AFP-… (<customers>)
2. …

## Needs FDE input (open questions found in records)
- AFP-### — <question>

## Item index
| Issue | Customer | Feature | Gap | Severity | Impact | Repro | Ask (primary) |
|---|---|---|---|---|---|---|---|
| AFP-### | … | … | … | … | … | … | … |

*(values marked `(inferred)` were parsed from prose, not FDE-asserted records)*
```

## Guardrails

- **Cite everything.** Every claim in an answer or rollup traces to an issue/doc. No uncited aggregates.
- **Don't silently mutate.** Never change issue state, priority, assignee, or labels unless the PM explicitly asks.
- **Respect the record.** Where a record and your prose-reading disagree, the record wins; note the discrepancy.
- **Confidentiality.** Rollups may name customers — treat outputs as internal; don't paste them to external channels.
- **Every hand-off names an owner.** Follow-up questions posted to the thread must say who owes the answer (`loop_status: open-with-FDE` / `open-with-customer`); a question that requires customer input must be flagged `(customer-dependent)` so the FDE knows to schedule a customer touch rather than answer from the desk.
- **Prevalence is yours.** How-many-customers / how-widespread questions are PM triage actions (`P#`), never FDE asks — the FDE sees one customer.
- **Never invent specifics.** Don't manufacture customer detail, quantified impact, or telemetry to make a triage look complete. If a number isn't known, say so and make obtaining it a `P#` action. Any illustrative/simulated value must be labelled — and preferred qualitative over a fake-precise figure.

## Worked example

Two filled instances to imitate — a single-item triage, and the cross-feature rollup that shows this skill spans **all** Agentforce feedback (Testing Center below is just the example, not the scope).

### 1. Single-item triage (Procedure A + D)

**Scenario:** the PM asks their agent *"What's AFP-### about, and draft my disposition."* The agent reads the intake → summary comment → evidence doc → parses the `feedback_record`. (Same demo item as the FDE skill's worked example — a multi-turn Testing Center gap, customer `internal-demo` — so the two examples visibly connect as one loop.)

**Triage verdict**

| Dimension | Verdict |
|---|---|
| Classification | `limitation` — agree with the FDE, not a bug |
| Severity | `workaround-exists` — but the workaround (manual `sf agent preview` runs) isn't CI-viable |
| Impact | `confidence-loss, productivity-loss` |
| Repro | `reproduced-live`, root cause proven via the Data Cloud trace — evidence quality high, no further validation needed |

**Register**

> **A1 · 🔎 sized · owner PM** — Execute prior turns to persist state (execution-model fix)
> → *Resolution:* Routed to the multi-turn testing epic — the real fix, but heavier.
>
> **A2 · 📋 accepted · owner PM** — Warn when a value isn't in the transcript
> → *Resolution:* Ships fastest — detectable at authoring time.
>
> **A3 · ✅ resolved · owner PM** — Document the state-replay limitation
> → *Resolution:* Immediate; ships alongside A2.
>
> **P1 · ⏳ open · owner PM** — Pull telemetry on affected multi-turn suites

**Disposition:** Accepted, per the register above. A1 is the real fix but heavier, so it's routed to the epic rather than blocking the near-term ship; A2 and A3 ship together now. P1 (prevalence) is the PM's own action, not a question for the FDE.

```yaml
triage_record: v1
issue: AFP-###
date: 2026-07-20
disposition: planned
accepted_asks:
  - Warn when a multi-turn case depends on a value not present in the transcript (A2 — near-term, authoring-time check)
  - Document the state-replay limitation so PASS/FAIL is trusted correctly (A3 — immediate)
target: multi-turn testing epic
owner: FDE
next_action: FDE confirms the warning (A2) is feasible at authoring time; PM runs the P1 telemetry pull separately
loop_status: open-with-FDE
decided_date: 2026-07-20
notes: Execution-model ask (A1) accepted in principle; sized separately in the epic, not blocking the near-term ship. A material share of multi-turn suites likely depend on prior-turn action outputs; exact count pending the P1 telemetry pull — no hard number yet.
```

On the PM's confirmation, the agent applied the disposition to `AFP-###` itself per Procedure D: `state` → Todo (per the disposition→state mapping, `planned` → Todo), `priority` raised, label `testing-center` added, and `assignee` set to the FDE — the current owner of A2's next action.

Then, per Procedure E: once A2 and A3 shipped, the agent created delivery issues in the eng team (sub-issues can't cross workspaces from `eventsmobileapp`) and cross-linked them by URL, then updated the `triage_record`:

```yaml
tracking:
  A2: AFL-412
  A3: AFL-413
epic: multi-turn testing epic
```

...and posted a close-back comment addressed to CS on `AFP-###` noting A2 and A3 shipped.

### 2. Cross-feature rollup (Procedure B)

**Scenario:** *"Roll up this month's Agentforce feedback."* The agent enumerates issues across every feature family, not just Testing Center:

| Issue | Customer | Feature | Gap | Severity | Impact | Repro | Ask (primary) |
|---|---|---|---|---|---|---|---|
| AFP-101 | Acme Energy | Testing Center | limitation | workaround-exists | confidence-loss | reproduced-live | Execute prior turns in multi-turn tests |
| AFP-102 | Northwind Bank | Agent Script | feature-request | inconvenience | productivity-loss | reported-only | Loop constructs in Agent Script |
| AFP-103 | Contoso Health | Voice | bug | blocker | adoption-blocking | reproduced-live | Barge-in cuts off variable capture |
| AFP-104 | Acme Energy | Agent Builder | docs-gap | inconvenience | confidence-loss | reported-only | Document topic-routing precedence |

**Headlines:**
- One `blocker` leads the list (AFP-103, Voice barge-in — adoption-blocking, reproduced live).
- `Acme Energy` appears twice (AFP-101, AFP-104) — a repeat reporter across two unrelated feature families.
- Every value above was parsed straight from `feedback_record` blocks — zero prose interpretation.

Same table, any feature family — the record contract is what makes the rollup possible.
