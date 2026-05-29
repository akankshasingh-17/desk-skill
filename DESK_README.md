# Desk Skill — Report Guide

This guide explains every section of the `/desk` report and how to interpret the numbers.

---

## Health Score

The first thing you see at the top of every report.

| Score | Meaning |
|-------|---------|
| 🟢 ON TRACK | Closure rate is keeping up with pace target. No major issues. |
| 🟡 AT RISK | Closure rate is falling behind, or 2–4 high-priority tickets are open, or 5–9 tickets are overdue. |
| 🔴 BEHIND | Closure rate is significantly behind pace, or 5+ high-priority tickets open, or 10+ tickets overdue. |

**How it is calculated:**
- RED if closure rate < (pace target − 25%) OR high-priority open ≥ 5 OR overdue ≥ 10
- YELLOW if closure rate < (pace target − 10%) OR high-priority open ≥ 2 OR overdue ≥ 5
- GREEN otherwise

---

## Progress Bars

```
Opened  [██████████████████████████████]  47 tickets this sprint
Closed  [██████████████████████████████]  65 closed  (138.3% — includes pre-sprint backlog)
```

- **Opened bar** is always full — it is the reference (100% = all tickets opened this period).
- **Closed bar** fills based on closure rate. Capped at 100% visually even if rate exceeds 100%.
- A closure rate **above 100%** means pre-sprint backlog tickets were also resolved during this period — that is a good sign.

---

## Overview Table

| Metric | What it means |
|--------|--------------|
| **Opened this period** | Tickets created within the report date range |
| **Closed this period** | All tickets resolved during the period — includes tickets opened before the period (backlog) |
| **Sprint-opened closed** | Only the tickets opened *and* closed within the same period |
| **Pending** | Tickets opened in the period that are still not closed |
| **Pace target** | % of the period elapsed so far. If 60% of the sprint has passed, you should ideally have closed ~60% of tickets. |
| **Closure rate** | (Closed ÷ Opened) × 100. Above 100% = backlog was also cleared. |
| **Avg resolution time** | Mean time from ticket creation to closure. High numbers (100h+) usually mean old backlog tickets were closed — not that new tickets are slow. |
| **Overdue** | Tickets past their due date that are still open. Always investigate these. |
| **Unassigned** | Tickets with no agent assigned. These risk falling through the cracks. |
| **High-priority open** | High-priority tickets still not resolved. These need immediate attention. |

---

## Needs Attention

This section only shows items that require action. If all three sub-sections show "— none —", the sprint is clean.

### High Priority Open
Tickets marked **High** priority that are not yet closed. Includes days open and due date so you can prioritise which to tackle first.

### Overdue
Tickets where the due date has passed but the ticket is still open. Days Overdue tells you how far past the deadline each ticket is.

### Unassigned
Tickets with no agent assigned. These will not be worked on until someone picks them up. Should always be zero.

---

## Service Breakdown

Every ticket is classified against E2E's product and service list using the subject line, description, and the `Internal Ticket Categorization` field in Zoho Desk.

| Column | What it means |
|--------|--------------|
| **Opened** | Tickets for this service opened in the period |
| **Closed** | Of those, how many were resolved |
| **Pending** | Still open at the end of the period |
| **Pending %** | Pending ÷ Opened. A high % means the service has unresolved work piling up. |
| **High-Pri** | High-priority tickets for this service |

**What to look for:**
- Services with **Pending % > 60%** need attention — tickets are coming in but not getting closed.
- Services appearing repeatedly in Recurring Issues are likely product bugs or missing documentation.
- **Uncategorised** tickets are system auto-notifications (`[##...##]` subjects) — they do not represent real customer issues and are excluded from analysis.

---

## Ticket Type Breakdown

Each ticket is classified by what kind of issue it represents:

| Type | What it covers |
|------|---------------|
| **Bug / Error** | Something is broken or not working as expected |
| **Outage / Downtime** | Service unavailable or degraded |
| **Performance Issue** | Slow response, timeouts, latency problems |
| **Access / Permission** | Cannot log in, KYC issues, permission denied |
| **Feature Request** | Customer asking for new functionality |
| **Billing Issue** | Invoice problems, payment failures, incorrect charges |
| **Configuration / Setup** | Misconfigured services, settings not reflecting |
| **Deployment / Provisioning** | Resource stuck in creating/terminating state |
| **General Support / Clarif.** | Clarification questions, guidance requests, how-to queries |

**What to look for:**
- **High "General Support / Clarif." %** (above 40%) is a signal that self-service documentation is missing. Customers are raising tickets for things they should be able to resolve themselves.
- **High "Bug / Error" %** across the same service in multiple sprints signals a product quality issue that needs an RCA.
- **Feature Requests** should be routed to the product backlog and not left open in the support queue indefinitely.

---

## Priority Breakdown

Shows how tickets are distributed across High / Medium / Low / None priority.

**What to look for:**
- A large **None** count means agents are not setting priority at ticket intake. This makes triage harder.
- **High priority pending** tickets should always be investigated — they are the ones most likely to escalate.

---

## Assignee Snapshot

One row per agent. Shows their workload during the period.

| Column | What it means |
|--------|--------------|
| **Opened** | Tickets assigned to this agent that were opened in the period |
| **Closed** | Tickets this agent resolved during the period (includes pre-sprint backlog they cleared) |
| **Pending** | Their open tickets at the end of the period |
| **HP Open** | Their high-priority tickets still unresolved |
| **Avg Resolution** | Mean time from creation to closure for tickets they resolved. Treat high numbers with caution — they are often inflated by old backlog tickets. |
| **Workload Score** | A calculated risk score (see below) |
| **Top Service** | The service that appears most in their ticket list |

### Workload Score

```
Score = (HP Open × 3) + (Overdue × 2) + (Pending × 0.5) − (Closed × 0.3)
Clamped between 0 and 10.
```

| Score | Label | Meaning |
|-------|-------|---------|
| ≥ 6 | 🔴 HIGH | Agent has too many unresolved high-priority or overdue items. Needs support. |
| 3–5.9 | 🟡 MED | Some pressure — worth monitoring |
| < 3 | 🟢 LOW | Manageable load |

**Note:** A high Avg Resolution time does not necessarily mean an agent is slow. Agents who clear old backlog tickets will show inflated averages because those tickets were created weeks or months ago.

---

## Recurring Issues

Groups tickets by service + type where the same combination appears 2 or more times. These are patterns — not individual problems.

| Column | What it means |
|--------|--------------|
| **Pattern** | Service + ticket type that repeats |
| **Count** | How many tickets matched this pattern in the period |
| **Max Priority** | Highest priority seen across those tickets |
| **Ticket #s** | The actual ticket numbers so you can investigate |
| **Suggested Action** | Recommended next step |

**What to look for:**
- Patterns with **count ≥ 4** are strong signals of a product bug or missing KB article.
- Patterns of type **General Support / Clarif.** mean customers cannot find answers themselves — a documentation fix will deflect these tickets.
- Patterns of type **Bug / Error** on the same service across multiple sprints need an RCA.

---

## Momentum — Last 7 Days

Shows only tickets closed in the **last 7 days of the report period**. This tells you about recent velocity — not the full period.

Useful for:
- End-of-sprint check — who shipped what in the final push
- Identifying agents who had a strong close-out week
- Spotting agents who opened tickets but did not close any recently

---

## Operational Insights

Auto-generated observations based on the data:

| Insight | What it means |
|---------|--------------|
| **Top service by volume** | The service generating the most tickets this period — may need product attention |
| **Highest pending rate** | The service where the most tickets remain unresolved |
| **Clarification % too high** | If above 40%, too many tickets are how-to questions — needs KB articles |
| **Automation candidates** | Services where 3+ clarification tickets appeared — a self-service flow or FAQ would deflect these |
| **Backlog cleared** | Pre-sprint tickets resolved during this period — shows backlog reduction progress |
| **Long-running tickets** | Tickets open for more than 7 days — may need escalation or re-assignment |

---

## Choosing the right command

| Situation | Use |
|-----------|-----|
| End of sprint review with the team | `/desk --sprint` |
| Quick morning check | `/desk --quick` |
| Briefing a manager or leadership | `/desk --sprint --exec` |
| Sharing a visual report | `/desk --sprint --exec --web` |
| Investigating a specific week | `/desk --week 2026-W21` |
| Comparing a custom period | `/desk 2026-05-01:2026-05-29` |
| Full audit with all ticket details | `/desk --sprint` (full mode) |

---

## Avg Resolution Time — Why is it so high?

You will often see average resolution times of 500h, 1000h, or more. This is **not** a sign of poor performance.

It happens because the report fetches all tickets **closed during the sprint period** — including tickets that were created weeks or months before the sprint started. When a 60-day-old ticket gets resolved, it adds 1440h to the average.

To get a more accurate view of how quickly new tickets are being resolved, compare tickets that were both **opened and closed** within the same sprint. The `Sprint-opened closed` count in the overview gives you this subset.

---

## Tip — Improve classification accuracy

The service classification is based on subject line keywords. If you find tickets landing in **Uncategorised** that clearly belong to a specific service, ask agents to:

1. Set the **Internal Ticket Categorization** custom field in Zoho Desk when creating or updating a ticket — the skill uses this as its highest-priority signal.
2. Use clear service names in ticket subjects (e.g. "TIR: ..." or "k8s cluster...").

Better tagging = more accurate reports with no changes to the skill.
