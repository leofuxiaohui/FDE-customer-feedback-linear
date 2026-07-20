---
name: PM-feedback-triage-linear
description: Use when a PM wants to understand, triage, or aggregate Agentforce customer feedback that FDEs/SEs have shared on Linear (AFP-* issues) — without a meeting. Reads the feedback issues and their in-app evidence documents, parses the machine-readable feedback-record blocks written by the companion FDE-customer-feedback-linear skill, answers the PM's questions with citations, builds cross-customer rollups (by feature, gap type, severity, impact), and posts follow-up questions back on the thread asynchronously.
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

## Procedure

### A. Understand one feedback item
1. `get_issue AFP-###` — read the intake description and note the customer/feature fields.
2. `list_comments` on the issue — find the summary comment and any thread discussion since.
3. The issue's `documents` list names its evidence docs — `get_document` each; read TL;DR → verdict → root cause → asks → PM-questions section → the record block.
4. Answer the PM's question **with citations**: issue ID, doc title + URL, and section. Distinguish FDE-asserted facts from your inferences.
5. If the PM has follow-ups the doc doesn't answer: check the doc's *open questions* first (the FDE may have flagged exactly that), then draft the questions and — on the PM's confirmation — post them as a `save_comment` on the issue thread.

### B. Aggregate across many items
1. Enumerate candidates: `list_issues` with `team: "Agentforce Feedback"` (filter by `createdAt`/`updatedAt` for a time window, or `query` for a feature keyword). Page through — don't stop at the first page.
2. For each issue: `get_issue` → `get_document` its evidence doc(s) → extract the `feedback_record` block (fallback: infer from prose, mark `(inferred)`).
3. Build the rollup in whatever cut the PM asked for — by `feature_family`, `gap_type`, `severity`, `customer_impact`, `customer`, or recurring `asks`. Count items, list the issue IDs behind every number, and surface: **blockers first**, then adoption-blocking/deal-risk impacts, then clusters of identical asks (same ask from N customers = strongest signal).
4. Deliver as chat, or — if the PM wants it shareable — as a Linear Document (`save_document` on the PM's chosen project/team/issue) using the rollup template below.
5. Collect every record's `open_questions` into a "needs FDE input" list; offer to post each back to its issue thread as a comment.

### C. Close the loop asynchronously
- Post PM follow-ups as comments on the specific issue (`save_comment` with `issueId`) — the FDE answers on the thread.
- If the PM disposition is "accepted / roadmap / duplicate-of-X / needs-more-evidence", offer to post that as a comment too, so the FDE isn't left waiting.

## Rollup document template

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
