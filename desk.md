
# Zoho Desk Report — E2E Product Dev

Fetch and analyse all support tickets from the E2E Product Dev department for a given period (week or sprint), then render a structured report.

## Fixed IDs (never ask the user)
- Department: **E2E Product Dev**, ID: `236167000034148333`
- Org: **E2E Networks Limited**, ID: `658568929`
- Sprint team ID (for --sprint cross-reference): `667504989`
- Sprint project ID (Discovery): `8253000000154001`

## Arguments
- No argument → current ISO week (Monday of current week → today)
- `--sprint` → auto-fetch active sprint from Zoho Sprints and use its start/end dates
- `--quick` → summary tables only (skip conversation fetching and per-assignee listing — fast)
- `--exec` → executive one-page view: health score, alerts, top services, assignee snapshot (skips conversations and per-assignee detail)
- `--web` → after generating the text report, also render a standalone HTML file and open it in the browser
- `--week YYYY-WNN` → specific ISO week, e.g. `--week 2026-W21`
- `YYYY-MM-DD:YYYY-MM-DD` → custom date range, e.g. `/desk 2026-05-01:2026-05-21`
- Sprint name fragment → passed with `--sprint <name>` to match a named sprint instead of the active one
- All flags can combine: `/desk --sprint --exec --web`, `/desk --quick --web`, `/desk --sprint --quick`

---

## Step 1 — Determine Date Range

Parse the argument to set `period_start` and `period_end` (ISO 8601 dates):

| Mode | period_start | period_end | period_label |
|------|-------------|------------|--------------|
| No arg | Monday of current ISO week | Today | `Week NN (Mon DD MMM – today)` |
| `--week YYYY-WNN` | Monday of that ISO week | Sunday of that ISO week | `Week NN (Mon DD MMM – Sun DD MMM YYYY)` |
| `YYYY-MM-DD:YYYY-MM-DD` | First date | Second date | `DD MMM – DD MMM YYYY` |
| `--sprint` (no name) | From Step 2 | From Step 2 | `<sprint name> (DD MMM – DD MMM YYYY)` |
| `--sprint <name>` | From Step 2 | From Step 2 | `<sprint name> (DD MMM – DD MMM YYYY)` |

Set `period_days = (period_end − period_start).days + 1`.

---

## Step 2 — Fetch Sprint Dates (only if `--sprint` was passed)

Call `ZohoSprints_GetSprints` with:
- teamId: `667504989`
- projectId: `8253000000154001`
- query: `action=data, index=1, range=50, type=[2]`
- headers: `x-za-ui-version=v2, X-convert-response=true`

**If no sprint name was given:** use the first active sprint returned. Set `period_start = sprintStartDate`, `period_end = sprintEndDate`.

**If a sprint name fragment was given:** match against sprint names (case-insensitive substring). If multiple match, print a one-line list and ask which to use.

Note `sprintName`, `sprintStartDate`, `sprintEndDate`, `sprintId` — include sprint name in the report header.

---

## Step 3 — Fetch All Tickets (paginated)

Fetch all tickets from E2E Product Dev created **or** modified within the period. Two passes:

### Pass A — Tickets created in period
Call `ZohoDesk_getTickets` (or `ZohoDesk_searchTickets`) with:
- `departmentId`: `236167000034148333`
- `from`: pagination offset, start at `0`
- `limit`: `100`
- `createdTimeRange`: `{"from": "<period_start>T00:00:00.000Z", "to": "<period_end>T23:59:59.000Z"}`
- `sortBy`: `createdTime`

**Pagination rule:** increment `from` by `100` each page. Stop when the response returns fewer than 100 tickets or `hasMore=false`.

### Pass B — Tickets closed in period (may have been created before period)
Call `ZohoDesk_searchTickets` (or `ZohoDesk_getTickets`) with:
- `departmentId`: `236167000034148333`
- `closedTimeRange`: `{"from": "<period_start>T00:00:00.000Z", "to": "<period_end>T23:59:59.000Z"}`
- Same pagination

Deduplicate by `ticketId` across both passes. Label each ticket with:
- `opened_in_period`: true if `createdTime` falls within the period
- `closed_in_period`: true if `closedTime` falls within the period

**Fields to extract per ticket:**
| Field | Notes |
|-------|-------|
| `ticketNumber` | Display ID shown in Zoho Desk UI |
| `id` | Internal ticket ID |
| `subject` | Full subject line — used for service/type inference |
| `description` | First 500 chars — used for service/type inference |
| `status` | Open / In Progress / On Hold / Closed / etc. |
| `priority` | High / Medium / Low / None |
| `createdTime` | ISO 8601 |
| `closedTime` | ISO 8601 or null |
| `modifiedTime` | ISO 8601 |
| `dueDate` | ISO 8601 or null |
| `assigneeId` | Agent ID |
| `assigneeName` | Agent display name (from `assignee.name` or `assignee.firstName + lastName`) |
| `contactId` | Submitter contact ID |
| `contactName` | Submitter name |
| `channel` | Email / Web / Phone / Chat / etc. |
| `threadCount` | Number of conversation threads |
| `commentCount` | Number of comments |
| `category` | If set in Zoho Desk |
| `subCategory` | If set |
| `customFields` | Any custom field map |

---

## Step 4 — Service & Category Classification

**Run before Step 5. Classify every ticket.**

### Pre-filter: System Auto-Notifications
Skip tickets where subject starts with `[##` — these are Zoho auto-acknowledgement emails
(e.g. "Your ticket has been created", "Your reply has been received"). Label them `Uncategorised`
and exclude from all breakdowns and recurring issue analysis.

### Service Detection

Use the **E2E official services list** below. Check subject + description + the custom field
`Internal Ticket Categorization` (if set, use it as a strong signal). Apply FIRST match (top to bottom).

| Service | Keywords / Signals |
|---------|-------------------|
| TIR | tir, training, inference, triton, model serving, jupyter, notebook, deepstream, ai cloud, ai platform, ml platform, inference billing |
| GPU Cloud | gpu, a100, h100, v100, l4, cuda, nvidia, accelerator, gpu instance, gpu plan |
| k8s | kubernetes, k8s, cluster, node pool, kubectl, helm, master upgrade, terminating state, creating state |
| Nodes | \bnode\b, vm, virtual machine, instance, server, cpu instance, bandwidth limit, reinstall, recovery mode, upgrade, stuck in stopped, deprovisioning |
| Volumes (Linstor) | volume, block storage, linstor, linstor, pvc, persistent volume, volume billing, volume creation, storage pool |
| SFS | sfs, scalable file system, shared file system, sfs tir |
| Snapshots | snapshot, drbd sync, snapshot fail, snapshot recovery |
| CDP Backups | cdp, backup, restore, restoration, cdp backup, hcp driver, backup ticket |
| Saved Image | saved image, image creation, image stuck, image deletion, image not visible, creating image, delete image, image export |
| EOS | eos, object storage, bucket, s3, minio, aistore, blob |
| Reserve IP | reserved ip, reserve ip, reserve ipv4, floating ip, static ip, secondary ip |
| VPC | vpc, delete vpc, unable to delete vpc |
| Firewall (Fortigate) | firewall, fortigate, security group, firewall rule, firewall creation |
| Load Balancer | load balancer, nlb, alb, nlb backend, load balance |
| Networking | network, vpc peering, subnet, route table, bandwidth, nat appliance, tailscale, vpn, wireguard |
| Existing Billing Maintenance | billing, invoice, payment, autopay, overcharg, wrong charg, credit, razorpay, billed, charges after delete, invoice not raised, payment sync, billing address |
| My Account DR | my account, kyc, gstin, account deletion, deprovisioning, suspended account, account email, crn, account dashboard, account owner |
| Dashboard | dashboard, console, web interface, usage report, project name in report |
| Monitoring | monitoring, alert, grafana, prometheus, metric, alert not received, zabbix |
| IAM / Auth | iam, auth, permission, role, access key, mfa, 2fa, sso, pbac, saml |
| Support Section | support portal, django ticketing, ticketing system, support section |
| Container Registry | container registry |
| DNS | dns |
| DBaaS | dbaas, database as a service |
| API / SDK / CLI | api, sdk, cli, terraform, endpoint, api key, rest |
| SSL / License Management | ssl, bitninja, license |
| CDN | cdn |
| Sign up / Onboarding | sign up, signup, onboarding, kyc, new user, international customer |
| Open Nebula | nebula, opennebula, stale entries in nebula |
| Uncategorised | (fallback — no match above) |

**Classification priority:** `Internal Ticket Categorization` custom field > subject keyword match > description keyword match.
**Fuzzy matching:** treat near-identical subjects (e.g. "Unable to Delete VPC", "help to delete vpc") as the same pattern.

### Ticket Type Classification

| Type | Keywords |
|------|----------|
| Bug / Error | bug, error, issue, broken, fail, crash, not working, incorrect, wrong, regression, unexpected, stuck, mismatch, discrepancy, not visible, not reflected, not enforced |
| Outage / Downtime | outage, down, unavailable, unreachable, offline, degraded, incident |
| Performance Issue | slow, latency, timeout, performance, lag, bottleneck, throughput |
| Access / Permission | access denied, permission denied, unauthorized, cannot access, kyc, authorization error, 403, 401 |
| Feature Request | feature request, feature:, enhancement, feasibility, add support, improve, real-time progress, enable os, improvement, requirement for |
| Billing Issue | billing, invoice, payment, overcharg, wrong charg, credit, billed, charges, razorpay, autopay |
| Configuration / Setup | config, configuration, misconfigured, setup, settings, update not reflect, attaching, not enforced, parameter |
| Deployment / Provisioning | deploy, provisioning, launch, spin up, stuck in creat, stuck in terminat, creating state, terminating |
| General Support / Clarif. | clarification, guidance, help, how to, question, investigation required, review, verify, assistance, regarding, unable to delete (non-crash) |

---

## Step 5 — Compute All Metrics with a Python Script

**Run immediately after Steps 3 and 4. Write to `/tmp/desk_process_<period_slug>.py` and execute.**

`period_slug` = period label sanitised: lowercase, spaces→underscores, special chars removed. E.g. `week_21_2026` or `sprint_may_2026`.

The script reads the raw ticket JSON saved to disk, applies the service/type classification from Step 4, computes every metric below, and writes `/tmp/desk_metrics_<period_slug>.json`.

**All subsequent steps (6, 7, 8) read exclusively from this JSON file.**

### Metrics to compute

```
period_start, period_end, period_label, period_days, today
sprint_name (if --sprint mode, else null)

opened_count      = tickets where opened_in_period = true
closed_count      = tickets where closed_in_period = true
open_count        = tickets where status != Closed AND opened_in_period = true  (still open at end of period)
in_progress_count = tickets where status == "In Progress" (all fetched, not just period)
on_hold_count     = tickets where status == "On Hold" (all fetched)
pending_count     = open_count + in_progress_count + on_hold_count  (non-closed)

closure_rate_pct  = (closed_count / max(opened_count, 1)) × 100
pace_target_pct   = (days_elapsed / period_days) × 100   where days_elapsed = today − period_start (min 1, max period_days)
days_elapsed, days_remaining = today − period_start, period_end − today (min 0)

overdue_count     = tickets where status != Closed AND dueDate < today AND dueDate is set
sla_breach_count  = overdue_count  (use as SLA proxy until SLA API confirmed)
unassigned_count  = tickets (opened_in_period) where assigneeId is null or empty
avg_resolution_hrs = mean(closedTime − createdTime in hours) for closed tickets in period
high_priority_open = count of opened_in_period tickets where priority == High AND status != Closed

# Health
health:
  RED    if closure_rate_pct < (pace_target_pct − 25)  OR  high_priority_open >= 5  OR  sla_breach_count >= 10
  YELLOW if closure_rate_pct < (pace_target_pct − 10)  OR  high_priority_open >= 2  OR  sla_breach_count >= 5
  GREEN  otherwise
```

### Service breakdown (opened_in_period tickets)
```json
"service_breakdown": {
  "<service>": { "opened": 12, "closed": 8, "pending": 4, "high_priority": 3 }
}
```

### Ticket type breakdown (opened_in_period)
```json
"type_breakdown": {
  "<type>": { "count": 15, "pct": 28.3, "closed": 10, "pending": 5 }
}
```

### Priority breakdown (opened_in_period)
```json
"priority_breakdown": {
  "High": { "opened": 8, "closed": 4, "pending": 4 },
  "Medium": { "opened": 20, "closed": 15, "pending": 5 },
  "Low": { "opened": 10, "closed": 9, "pending": 1 },
  "None": { "opened": 5, "closed": 3, "pending": 2 }
}
```

### Assignee stats (all fetched tickets, not just opened_in_period)
```json
"assignee_stats": {
  "<assigneeId>": {
    "display_name": "...",
    "opened_in_period": 18,
    "closed_in_period": 14,
    "pending": 4,
    "in_progress": 6,
    "high_priority_open": 2,
    "overdue": 1,
    "avg_resolution_hrs": 18.4,
    "avg_response_hrs": 3.2,
    "top_service": "GPU Cloud",
    "workload_score": 7
  }
}
```
`workload_score` = (high_priority_open × 3) + (overdue × 2) + (pending × 0.5) − (closed_in_period × 0.3), clamped to [0, 10], rounded to 1 decimal.

### Recurring issues (opened_in_period)
Group by: (a) identical or near-identical subjects, (b) same service + same type occurring ≥2 times.
```json
"recurring_issues": [
  {
    "pattern": "GPU instance stuck in provisioning",
    "count": 5,
    "service": "GPU Cloud",
    "type": "Deployment / Provisioning",
    "priority_max": "High",
    "ticket_numbers": ["#1234", "#1238", "#1241", "#1255", "#1260"],
    "suggested_action": "Investigate provisioning pipeline for GPU instances"
  }
]
```
Show minimum top 5 patterns. Sort by count descending.

### Overdue & attention items
```json
"overdue_detail": [
  { "ticketNumber": "#1234", "subject": "...", "assigneeName": "...", "priority": "High",
    "dueDate": "YYYY-MM-DD", "days_overdue": 3, "service": "GPU Cloud", "status": "In Progress" }
],
"unassigned_detail": [
  { "ticketNumber": "#1235", "subject": "...", "priority": "High", "createdTime": "YYYY-MM-DD",
    "service": "Billing", "status": "Open" }
],
"high_priority_open_detail": [
  { "ticketNumber": "#1236", "subject": "...", "assigneeName": "...", "service": "TIR",
    "createdTime": "YYYY-MM-DD", "dueDate": "YYYY-MM-DD", "days_open": 5, "status": "Open" }
]
```

### Momentum — recently closed (last 7 days within period)
```json
"momentum": {
  "recently_closed_count": 12,
  "by_assignee": {
    "<assigneeId>": { "display_name": "...", "closed_count": 5, "avg_resolution_hrs": 12.0,
                      "tickets": [{ "ticketNumber": "...", "subject": "...", "closedTime": "YYYY-MM-DD" }] }
  }
}
```

### Full output schema — `/tmp/desk_metrics_<period_slug>.json`
```json
{
  "period_start": "YYYY-MM-DD",
  "period_end": "YYYY-MM-DD",
  "period_label": "Week 21 (Mon 19 May – Thu 29 May 2026)",
  "period_days": 11,
  "today": "YYYY-MM-DD",
  "days_elapsed": 11,
  "days_remaining": 0,
  "pace_target_pct": 100.0,
  "sprint_name": null,
  "health": "GREEN",
  "overview": {
    "opened_count": 53,
    "closed_count": 41,
    "open_count": 8,
    "in_progress_count": 6,
    "on_hold_count": 2,
    "pending_count": 16,
    "closure_rate_pct": 77.4,
    "overdue_count": 3,
    "sla_breach_count": 3,
    "unassigned_count": 2,
    "high_priority_open": 4,
    "avg_resolution_hrs": 22.5,
    "avg_response_hrs": 4.1,
    "total_threads_fetched": 38
  },
  "service_breakdown": {},
  "type_breakdown": {},
  "priority_breakdown": {},
  "assignee_stats": {},
  "recurring_issues": [],
  "overdue_detail": [],
  "unassigned_detail": [],
  "high_priority_open_detail": [],
  "momentum": {}
}
```

---

## Step 5.5 — Backlog & Operational Insights (compute in Python script)

Add these to the metrics JSON:

```
backlog_closures      = closed_in_period tickets where opened_in_period = false
long_running_tickets  = opened_in_period tickets where status != Closed AND
                        (today - createdTime) > 7 days
stale_tickets         = opened_in_period tickets where status != Closed AND
                        (today - modifiedTime) > 5 days  [if modifiedTime available]

top_service_by_volume = service with highest opened count
top_service_pending   = service with highest pending% (min 2 tickets)
automation_candidates = services/types with recurring count >= 3 AND type == "General Support / Clarif."
                        (suggests KB article or self-service flow would deflect tickets)
```

Add to JSON:
```json
"backlog": {
  "backlog_closures": 38,
  "long_running_count": 5,
  "long_running_detail": [{ "ticketNumber": "...", "subject": "...", "days_open": 14, "service": "..." }],
  "stale_count": 0
},
"insights": {
  "top_service_by_volume": "TIR",
  "top_service_pending": "Existing Billing Maintenance",
  "automation_candidates": ["VPC", "Existing Billing Maintenance"],
  "clarification_pct": 46.8
}
```

---

## Step 6 — --exec mode output

**Print ONLY this compact report when `--exec` is passed. Skip Steps 7 and 8.**

```
═══════════════════════════════════════════════════════════════════════
DESK REPORT: <period_label>  |  Dept: E2E Product Dev
HEALTH: 🟢 ON TRACK  (or 🟡 AT RISK  or 🔴 BEHIND)
═══════════════════════════════════════════════════════════════════════

<2–3 sentence plain-English narrative — write as if briefing a Head of Support:>
<closed_count> of <opened_count> tickets closed this period (<closure_rate_pct>% closure rate,
target <pace_target_pct>%). <high_priority_open> high-priority tickets still open.
<overdue_count> tickets are overdue. Top service by volume: <top_service>.
─────────────────────────────────────────────────────────────────────

Opened  [<30-char bar>] <opened_count> tickets this period
Closed  [<30-char bar>] <closed_count> / <opened_count> (<closure_rate_pct>%) closed  (target: <pace_target_pct>%)

OVERVIEW
  Metric                | Value
  ----------------------|---------------------------------------------------
  Opened this period    | <opened_count>
  Closed this period    | <closed_count>  (<closure_rate_pct>%)
  Pending (open+active) | <pending_count>  (Open: <open_count> · In Progress: <in_progress_count> · On Hold: <on_hold_count>)
  Pace target           | <pace_target_pct>% closed by now → <On Track / Behind N tickets>
  Avg resolution time   | <avg_resolution_hrs>h per ticket
  Avg first response    | <avg_response_hrs>h
  Overdue               | <overdue_count> tickets past due date  ← ⚠ if >0
  Unassigned            | <unassigned_count> tickets with no agent  ← ⚠ if >0
  High-priority open    | <high_priority_open>  ← ⚠ HIGH if ≥5

⚠  NEEDS ATTENTION

  HIGH PRIORITY OPEN  (<high_priority_open> tickets):
  # | Subject | Assignee | Service | Days Open | Due
  --|---------|----------|---------|-----------|----
  <#N> | <subject in full> | <assignee> | <service> | <N> | YYYY-MM-DD
  — (none) — if 0

  OVERDUE  (<overdue_count> tickets):
  # | Subject | Assignee | Service | Days Overdue | Priority
  --|---------|----------|---------|-------------|--------
  <#N> | <subject in full> | <assignee> | <service> | <N> | High
  — (none) — if 0

  UNASSIGNED  (<unassigned_count> tickets):
  # | Subject | Priority | Service | Created
  --|---------|----------|---------|-------
  <#N> | <subject in full> | High | <service> | YYYY-MM-DD
  — (none) — if 0

📊  SERVICE BREAKDOWN (opened this period, sorted by count descending)
  Service              | Opened | Closed | Pending | High-Pri
  ---------------------|--------|--------|---------|--------
  GPU Cloud            |     12 |      9 |       3 |       4
  TIR                  |      8 |      7 |       1 |       1
  (No Match / General) |      5 |      3 |       2 |       0

👥  ASSIGNEE SNAPSHOT (all open + period tickets)
  Agent       | Opened | Closed | Pending | High-Pri Open | Overdue | Avg Resolution | Workload
  ------------|--------|--------|---------|---------------|---------|----------------|--------
  <name>      |     18 |     14 |       4 |             2 |       1 |           18.4h | 🔴 7.0
  <name>      |     12 |     11 |       1 |             0 |       0 |           10.2h | 🟢 0.8

  Workload score: 🔴 HIGH ≥6 | 🟡 MED ≥3 | 🟢 LOW <3

🔁  TOP RECURRING ISSUES (≥2 tickets with same pattern)
  Pattern | Count | Service | Type | Max Priority | Action
  --------|-------|---------|------|-------------|-------
  <pattern> | 5 | GPU Cloud | Deployment | High | Investigate provisioning pipeline
  — (none) — if no recurring issues

🎯  TICKET TYPE BREAKDOWN
  Type                   | Count |   % | Closed | Pending
  -----------------------|-------|-----|--------|-------
  Bug / Error            |    15 | 28% |     12 |       3
  General Support        |    10 | 19% |      8 |       2

🚀  MOMENTUM — Last 7 Days (tickets closed)
  Agent         | Closed | Avg Resolution
  --------------|--------|---------------
  <name>        |      5 |          12.0h
  — (no tickets closed in last 7 days) — if empty
```

💡  OPERATIONAL INSIGHTS

  - Service generating most tickets: <top_service_by_volume> (<N> tickets, <N>% of sprint)
  - Highest pending rate: <top_service_pending> (<pending_pct>% unresolved)
  - Clarification/Support queries: <clarification_pct>% — suggests KB gaps in: <top clarif. services>
  - Backlog cleared this sprint: <backlog_closures> pre-sprint tickets resolved
  - Automation candidates (≥3 same-type clarifications): <automation_candidates>
  - Long-running tickets (>7 days open): <long_running_count>

✅  RECOMMENDATIONS
  1. <top recurring issue> — <suggested_action>
  2. <second recurring issue> — <suggested_action>
  3. Create KB articles for top clarification topics to deflect repeat tickets
  4. <any billing/access issues> — investigate root cause for recurring pattern
  5. Filter system auto-notifications ([##...##]) at ticket intake to avoid polluting reports

Column notes for `--exec` mode:
- ASCII bar: 30 chars wide; filled_chars = round(pct/100 × 30); █ for filled, ░ for remainder. For Opened bar: fill = round(opened_count / max(opened_count, 1) × 30) = always full (reference bar).
- Workload score → 🔴 HIGH if ≥6, 🟡 MED if ≥3, 🟢 LOW if <3.
- Pace target = pace_target_pct (% of period elapsed). "On Track" if closure_rate_pct ≥ pace_target_pct − 10, else "Behind N tickets".

---

## Step 7 — Default / --quick output

**Full output order. All data sections must be formatted as markdown tables. Never use plain-text lists for data.**

```
DESK REPORT: <period_label>  |  Dept: E2E Product Dev<sprint_label_if_sprint>

Opened  [<30-char bar>] <opened_count> tickets this period
Closed  [<30-char bar>] <closed_count> / <opened_count> (<closure_rate_pct>%) closed  (target: <pace_target_pct>%)
Health: 🟢 ON TRACK  (or 🟡 AT RISK  or 🔴 BEHIND)

OVERVIEW
  Metric                | Value
  ----------------------|---------------------------------------------------
  Opened this period    | <opened_count>
  Closed this period    | <closed_count>  (<closure_rate_pct>%)
  Pending (open+active) | <pending_count>  (Open: <N> · In Progress: <N> · On Hold: <N>)
  Pace target           | <pace_target_pct>% closed by now → <On Track / Behind N tickets>
  Avg resolution time   | <avg_resolution_hrs>h per ticket
  Avg first response    | <avg_response_hrs>h
  Period duration       | <period_days> days  (<days_elapsed> elapsed, <days_remaining> remaining)
  Overdue               | <overdue_count> tickets past due date
  Unassigned            | <unassigned_count> tickets with no agent
  High-priority open    | <high_priority_open>

⚠  NEEDS ATTENTION  (<total_issues> issues)

  HIGH PRIORITY OPEN  (<N> tickets):
  # | Subject | Assignee | Service | Days Open | Due Date
  --|---------|----------|---------|-----------|--------
  <#N> | <subject in full — no truncation> | <agent> | <service> | <N> | YYYY-MM-DD  ← Last comment if not --quick
  — (none) — if 0

  OVERDUE  (<N> tickets):
  # | Subject | Assignee | Service | Due Date | Days Overdue | Priority
  --|---------|----------|---------|----------|-------------|--------
  <#N> | <subject in full> | <agent> | <service> | YYYY-MM-DD | <N> | High
  — (none) — if 0

  UNASSIGNED  (<N> tickets):
  # | Subject | Priority | Service | Created | Status
  --|---------|----------|---------|---------|------
  <#N> | <subject in full> | High | <service> | YYYY-MM-DD | Open
  — (none) — if 0

STATUS BREAKDOWN (opened this period)
  Status        | Count |   %
  --------------|-------|----
  Open          |    <N>| <N>%
  In Progress   |    <N>| <N>%
  On Hold       |    <N>| <N>%
  Closed        |    <N>| <N>%

SERVICE BREAKDOWN (opened this period, sorted by count descending)
  Service              | Opened | Closed | Pending | High-Pri | % of Total
  ---------------------|--------|--------|---------|----------|----------
  GPU Cloud            |     12 |      9 |       3 |        4 |      22.6%
  TIR                  |      8 |      7 |       1 |        1 |      15.1%
  (No Match / General) |      5 |      3 |       2 |        0 |       9.4%

TICKET TYPE BREAKDOWN (opened this period)
  Type                   | Count |   % | Closed | Pending
  -----------------------|-------|-----|--------|-------
  Bug / Error            |    15 | 28% |     12 |       3
  Outage / Downtime      |     4 |  8% |      3 |       1
  ...

PRIORITY BREAKDOWN (opened this period)
  Priority | Opened | Closed | Pending |   %
  ---------|--------|--------|---------|----
  High     |      8 |      4 |       4 | 15%
  Medium   |     20 |     15 |       5 | 38%
  Low      |     10 |      9 |       1 | 19%
  None     |     15 |     13 |       2 | 28%

CHANNEL BREAKDOWN (opened this period, if data available)
  Channel | Count |   %
  --------|-------|----
  Email   |    <N>| <N>%
  Web     |    <N>| <N>%

🔁  RECURRING ISSUES (≥2 tickets matching same pattern)
Show ALL recurring patterns — no truncation, no "N more" footer.
  Pattern                           | Count | Service   | Type                    | Max Priority | Ticket #s         | Suggested Action
  ----------------------------------|-------|-----------|-------------------------|-------------|-------------------|----------------
  GPU instance stuck in provisioning|     5 | GPU Cloud | Deployment/Provisioning | High        | #1234 #1238 #1241 | Investigate pipeline
  — (no recurring issues) — if none

ASSIGNEE BREAKDOWN (all agents with tickets in period, sorted by opened descending)
  Agent       | Opened | Closed | Pending | In Progress | High-Pri Open | Overdue | Avg Res (h) | Workload | Top Service
  ------------|--------|--------|---------|-------------|---------------|---------|-------------|----------|------------
  <name>      |     18 |     14 |       4 |           3 |             2 |       1 |        18.4 | 🔴 7.0  | GPU Cloud
  UNASSIGNED  |      2 |      0 |       2 |           0 |             1 |       0 |           — |      —   | Billing

MOMENTUM — Last 7 Days (tickets closed in last 7 days of period)

  Summary (sorted by closed count descending):
  Agent         | Closed | Avg Resolution (h)
  --------------|--------|------------------
  <name>        |      5 |               12.0

  Detail (one row per ticket, grouped by agent, same sort order):
  Agent         | #     | Subject                                       | Service    | Closed
  --------------|-------|-----------------------------------------------|------------|----------
  <name>        | #1234 | Subject in full — no truncation               | GPU Cloud  | YYYY-MM-DD
  — (no tickets closed in last 7 days) — if empty
```

Notes:
- Service and type are inferred; tag row with `[I]` if inferred and confidence is low.
- For `--quick`: omit "Last Comment" column from NEEDS ATTENTION and skip Steps 8 and 9.
- ASCII bar for Closed: filled = round(closure_rate_pct / 100 × 30).
- Avg response time: `firstResponseTime − createdTime`; show "—" if `firstResponseTime` not available.
- Workload score → 🔴 HIGH ≥6 | 🟡 MED ≥3 | 🟢 LOW <3.
- Show UNASSIGNED as a pseudo-agent row in the assignee table.

---

## Step 8 — Fetch Conversations for Attention Items

**Skip entirely if `--quick` or `--exec` was passed.**

For tickets in the NEEDS ATTENTION section (high-priority open, overdue, unassigned), fetch the most recent conversation thread:

Call `ZohoDesk_getTicketConversations` (or `ZohoDesk_getThreads`) for each ticket:
- `ticketId`: ticket's internal `id`
- Fetch last 5 threads (or `limit=5, sortBy=createdTime desc`)

Batch in groups of up to 10 parallel calls. Store the most recent thread summary keyed by ticketId.

After fetching, go back and add a **Last Comment** column to the HIGH PRIORITY OPEN table.
Format: `[YYYY-MM-DD] <agent/contact name>: <first 120 chars of message>…`

---

## Step 9 — Per-Assignee Ticket Listing

**Skip entirely if `--quick` or `--exec` was passed.**

After all summary sections, print a detailed breakdown grouped by agent.
Sort agents by opened_in_period count descending (same as assignee table).

For each agent print:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
AGENT: <name>  (<N> opened · <N> closed · <N> pending)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| # | Subject | Status | Priority | Service | Type | Created | Due | Closed | Threads |
|---|---------|--------|----------|---------|------|---------|-----|--------|---------|
| <ticketNo> | <subject in full> | Open | High | GPU Cloud | Bug | YYYY-MM-DD | YYYY-MM-DD | — | <N> |

  ↳ Description: <plain text, strip HTML, max 300 chars, truncate with "…">
  ↳ Last comment: [YYYY-MM-DD] <name>: <text…>  ← only for HIGH priority or overdue; "— (no threads)" if none
```

- Strip HTML tags from description before printing.
- Within each agent, sort by: priority (High first), then status (Open → In Progress → On Hold → Closed), then createdTime ascending.
- Show all tickets — no limit.
- Print UNASSIGNED agent section last.

---

## Step 10 — HTML Browser Report (only if `--web` was passed)

**Skip entirely if `--web` was NOT passed. Runs after all text output is complete.**

Generate a self-contained HTML file, save it, and open in browser.

### File path
```
/tmp/desk_report_<period_slug>.html
```

### After writing, run:
```bash
open /tmp/desk_report_<period_slug>.html
```
Then print:
```
🌐 Opened in browser → file:///tmp/desk_report_<period_slug>.html
```

### HTML template

Same dark-theme aesthetic as the sprint `--web` report. All CSS inline in `<style>` block — no external URLs.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Desk Report: <period_label></title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body   { background: #0f0f1a; color: #d0d0e8; font-family: 'Segoe UI', system-ui, sans-serif;
             font-size: 14px; padding: 24px; line-height: 1.5; }
    h1     { color: #7c9fff; font-size: 1.4em; border-bottom: 1px solid #2a2a4a;
             padding-bottom: 10px; margin-bottom: 6px; }
    h2     { color: #a0b8ff; font-size: 1em; text-transform: uppercase; letter-spacing: 1px;
             margin: 32px 0 10px; }
    h3     { color: #8090c0; font-size: 0.9em; margin: 18px 0 6px; }
    .meta  { color: #555577; font-size: 12px; margin-bottom: 20px; }
    .narrative { background: #13132a; border-left: 4px solid #4c9fff; padding: 12px 16px;
                 border-radius: 4px; margin-bottom: 24px; color: #c0c8e8; }
    .health { font-weight: bold; font-size: 1.1em; margin-bottom: 20px; }
    .green  { color: #4cff91; }
    .yellow { color: #ffd740; }
    .red    { color: #ff5555; }
    .bars   { margin-bottom: 24px; }
    .bar-row { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
    .bar-label { width: 70px; color: #8090c0; font-size: 12px; }
    .bar-outer { background: #1e1e3a; border-radius: 4px; height: 14px; width: 300px; }
    .bar-inner { background: #4c9fff; border-radius: 4px; height: 14px; transition: width 0.3s; }
    .bar-text  { color: #a0b0d0; font-size: 12px; }
    table  { border-collapse: collapse; width: 100%; margin-bottom: 12px; font-size: 13px; }
    th     { background: #1a1a30; color: #6080c0; text-align: left; padding: 7px 10px;
             border-bottom: 2px solid #2a2a50; white-space: nowrap; }
    td     { padding: 5px 10px; border-bottom: 1px solid #1a1a2e; vertical-align: top; }
    tr:hover td { background: #14142a; }
    .chip  { display: inline-block; border-radius: 3px; padding: 1px 6px; font-size: 11px;
             font-weight: 600; white-space: nowrap; }
    .c-closed   { background: #0d2e0d; color: #4cff91; }
    .c-open     { background: #0d1a2e; color: #4c9fff; }
    .c-onhold   { background: #1a1a0d; color: #ffd740; }
    .c-overdue  { background: #2e0d0d; color: #ff5555; }
    .c-high     { background: #2e1a0d; color: #ff9040; }
    .c-med      { background: #1a1a0d; color: #ffd740; }
    .c-low      { background: #0d1a1a; color: #40d0c0; }
    .c-wl-h     { background: #2e0d0d; color: #ff5555; }
    .c-wl-m     { background: #2e2200; color: #ffd740; }
    .c-wl-l     { background: #0d2010; color: #4cff91; }
    .warn  { color: #ffd740; }
    .alert { color: #ff5555; }
    hr { border: none; border-top: 1px solid #1e1e3a; margin: 28px 0; }
  </style>
</head>
<body>

<h1>Desk Report: <period_label></h1>
<p class="meta">E2E Product Dev · <period_start> → <period_end> · Report generated <today><sprint_label_if_sprint></p>
<p class="health <health_color_class>">HEALTH: <health_emoji> <health_label></p>

<!-- Narrative (always shown) -->
<div class="narrative"><narrative_text></div>

<!-- Progress bars -->
<div class="bars">
  <div class="bar-row">
    <span class="bar-label">Closed</span>
    <div class="bar-outer"><div class="bar-inner" style="width: <min(closure_rate_pct,100)>%"></div></div>
    <span class="bar-text"><closed_count>/<opened_count> (<closure_rate_pct>%) closed &nbsp;·&nbsp; target <pace_target_pct>%</span>
  </div>
</div>

<!-- OVERVIEW table -->
<h2>Overview</h2>
<table>…one <tr> per overview row, .warn/.alert on flagged cells…</table>

<!-- NEEDS ATTENTION -->
<h2>⚠ Needs Attention</h2>
<h3>High Priority Open (<N>)</h3><table>…</table>
<h3>Overdue (<N>)</h3><table>…</table>
<h3>Unassigned (<N>)</h3><table>…</table>

<!-- SERVICE BREAKDOWN -->
<h2>📊 Service Breakdown</h2>
<table>…apply .chip.c-high/.c-med to High-Pri cells…</table>

<!-- TICKET TYPE BREAKDOWN -->
<h2>🏷 Ticket Type Breakdown</h2>
<table>…</table>

<!-- PRIORITY BREAKDOWN -->
<h2>🎯 Priority Breakdown</h2>
<table>…apply .chip.c-high/.c-med/.c-low to Priority cells…</table>

<!-- ASSIGNEE BREAKDOWN -->
<h2>👥 Assignee Breakdown</h2>
<table>…apply .chip.c-wl-h/.c-wl-m/.c-wl-l to Workload cells…</table>

<!-- RECURRING ISSUES -->
<h2>🔁 Recurring Issues</h2>
<table>…</table>

<!-- MOMENTUM -->
<h2>🚀 Momentum — Last 7 Days</h2>
<h3>Summary</h3><table>…</table>
<h3>Tickets Closed</h3>
<table>…one row per ticket, grouped by agent…</table>

</body>
</html>
```

### Status → chip class mapping
| Status | Class |
|--------|-------|
| Closed | `.c-closed` |
| Open | `.c-open` |
| In Progress | `.c-open` |
| On Hold | `.c-onhold` |
| Overdue (flag) | `.c-overdue` |

### Priority → chip class mapping
| Priority | Class |
|----------|-------|
| High | `.c-high` |
| Medium | `.c-med` |
| Low | `.c-low` |
| None | (no chip) |

### Notes for HTML generation
- All data from already-computed metrics — no re-fetching.
- Narrative text is always rendered (unlike sprint skill which only shows narrative in --exec --web).
- Progress bar `width` clamped to 100% max.
- If a NEEDS ATTENTION sub-section has 0 items, render `<p style="color:#555577">— none —</p>`.
- Omit per-assignee ticket listing (Step 9) from HTML — the assignee table provides sufficient detail.
- Apply `.warn` (yellow) class to overview cells containing ⚠; `.alert` (red) for critical flags.
