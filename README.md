# FDE-customer-feedback-linear

Two Claude Code skills that close the loop on **Agentforce customer feedback via Linear** — so FDEs/SEs and PMs don't need a meeting to understand it:

| Skill | Who runs it | What it does |
|---|---|---|
| [`FDE-customer-feedback-linear/`](FDE-customer-feedback-linear/SKILL.md) | FDE / Solution Engineer | Packages detailed feedback onto a customer-feedback issue (`AFP-*`) as **one in-app evidence Document + a summary comment** — including a pre-answered **"Questions a PM will ask"** section and a machine-readable **`feedback_record` block**. |
| [`PM-feedback-triage-linear/`](PM-feedback-triage-linear/SKILL.md) | PM (via their coding agent) | **Consumes** that feedback: answers questions about any item with citations, builds cross-customer rollups (by feature, gap type, severity, impact), and posts follow-up questions back on the thread asynchronously. |

The two sides meet on a shared contract: every evidence doc ends with a fenced `feedback_record: v1` YAML block (fixed vocabulary for `gap_type`, `severity`, `customer_impact`, `repro_status`), defined in both SKILL.md files. Human-readable narrative for people; parseable record for agents.

## The no-meeting flow

1. **FDE/SE** gathers evidence for a product gap (transcripts, test results, traces) and tells their coding agent: *"share this customer feedback on AFP-123 as an in-app doc + summary comment."*
2. The agent (via the FDE skill) reads the issue's intake fields, drafts the structured doc — verdict → evidence → root cause → asks → **pre-answered PM questions** → machine-readable record — links it to the issue, and posts the summary comment.
3. **PM** asks their coding agent (via the PM skill): *"what's AFP-123 about?"*, *"top asks across Testing Center feedback this quarter?"*, or *"draft my follow-ups for AFP-123."*
4. Remaining questions travel as **thread comments**, not calls.

## Install

Each skill is a folder containing a `SKILL.md`. Copy the folder(s) you need into your skills location:

- **Personal:** `~/.claude/skills/FDE-customer-feedback-linear/` and/or `~/.claude/skills/PM-feedback-triage-linear/`
- **Team-shared:** your team's synced skills repo or Claude Code plugin skills directory.

Each user needs the **Linear MCP** connected. Feedback issues live in the `eventsmobileapp` workspace (team "Agentforce Feedback") — if `get_issue AFP-###` fails, reconnect via `/mcp` → linear → pick `eventsmobileapp`.

## Use

```
/FDE-customer-feedback-linear     # FDE/SE: share feedback onto an AFP issue
/PM-feedback-triage-linear        # PM: understand / roll up / follow up on feedback
```

Or in plain language: *"share this customer feedback on AFP-123…"* / *"build a rollup of this month's Agentforce feedback."*

## Requirements

- Claude Code with skills enabled.
- Linear MCP connected (with access to the target workspace).
- `curl` (FDE skill only, and only for binary evidence like screenshots).

## Notes / known constraints

- Linear renders markdown (tables, code, `<details>`) but **not** HTML attachments inline — deliverables are always markdown docs.
- The Linear MCP has **no delete/archive-document** capability; retiring a doc means renaming it to a redirect stub and archiving in-app.
- Always sanitize org IDs / customer PII / tokens before posting. Rollups may name customers — keep them internal.
