# Desk Skill for Claude Code

A Claude Code slash command that fetches **live data** from Zoho Desk and generates a structured support ticket report — text output in the terminal and optionally a self-contained HTML file you can share or present.

Built for the **E2E Networks – E2E Product Dev** department. Supports weekly reports, sprint-aligned reports (auto-matched with Zoho Sprints dates), and executive briefings.

---

## What it does

Every `/desk` run fetches fresh ticket data from the Zoho Desk API, classifies each ticket by E2E's actual service list, computes all metrics via a Python script, and prints a full report covering:

- Ticket health score (🟢 ON TRACK / 🟡 AT RISK / 🔴 BEHIND)
- Volume overview — opened, closed, pending, backlog cleared, closure rate vs pace target
- Service breakdown — mapped to E2E's real services (TIR, GPU Cloud, k8s, Volumes/Linstor, SFS, EOS, CDP Backups, Saved Image, VPC, Reserve IP, Firewall, Billing, My Account DR, and 30+ more)
- Ticket type breakdown — Bug, Feature Request, Billing Issue, General Support etc.
- Priority breakdown — High / Medium / Low / None
- Assignee snapshot with workload scoring
- Needs Attention — overdue, unassigned, high-priority open tickets
- Recurring issue patterns — services and ticket types appearing ≥2 times
- Momentum — tickets closed in the last 7 days, per agent
- Operational insights — KB gap detection, automation candidates, backlog trends
- Per-assignee ticket listing with descriptions (full mode)

---

## Installation

### Prerequisites
- [Claude Code](https://claude.ai/code) installed
- The **Zoho Desk MCP** server configured in your Claude Code settings (`mcp__claude_ai_Zoho_Desk__*` or `mcp__zoho-desk__*`)
- The **Zoho Sprints MCP** server configured if you want `--sprint` mode (`mcp__zoho-sprints__*`)
- Python 3 available in your shell

### Steps

1. Create the skill directory and copy the skill file:
   ```bash
   mkdir -p ~/.claude/skills/desk
   cp desk.md ~/.claude/skills/desk/SKILL.md
   ```

2. Add the frontmatter to `SKILL.md`. Open it and prepend these lines at the very top:
   ```
   ---
   name: desk
   version: 2.0.0
   description: Zoho Desk weekly/sprint report for E2E Product Dev — /desk, /desk --sprint, /desk --exec, /desk --web, /desk --quick
   allowed-tools:
     - Bash
     - Read
     - Write
     - mcp__claude_ai_Zoho_Desk__getTickets
     - mcp__claude_ai_Zoho_Desk__searchTickets
     - mcp__claude_ai_Zoho_Desk__getTicket
     - mcp__claude_ai_Zoho_Desk__getTicketConversations
     - mcp__claude_ai_Zoho_Desk__getThreads
     - mcp__claude_ai_Zoho_Desk__getThread
     - mcp__zoho-desk__ZohoDesk_getTickets
     - mcp__zoho-desk__ZohoDesk_searchTickets
     - mcp__zoho-desk__ZohoDesk_getTicket
     - mcp__zoho-desk__ZohoDesk_getTicketConversations
     - mcp__zoho-sprints__ZohoSprints_GetSprints
   triggers:
     - /desk
     - desk report
     - desk tickets
     - support tickets report
     - zoho desk report
   ---
   ```

3. Open Claude Code and type `/desk`. You're ready.

---

## Commands

| Command | Speed | Best for |
|---------|-------|----------|
| `/desk` | ~30–60s | Current week, full report |
| `/desk --quick` | ~10–15s | Fast weekly check — summary tables only |
| `/desk --exec` | ~15–20s | Executive briefing — one compact page |
| `/desk --web` | ~30–60s | Full report + opens HTML in browser |
| `/desk --exec --web` | ~15–20s | Shareable exec one-pager |
| `/desk --quick --web` | ~10–15s | Fast check + visual HTML |
| `/desk --sprint` | ~30–60s | Active sprint date range, full report |
| `/desk --sprint --exec` | ~15–20s | Sprint period exec briefing |
| `/desk --sprint --exec --web` | ~15–20s | Sprint briefing + HTML |
| `/desk --sprint --quick` | ~10–15s | Sprint summary tables only |
| `/desk --week 2026-W21` | ~30–60s | Specific ISO week |
| `/desk 2026-05-01:2026-05-29` | ~30–60s | Custom date range |

All flags can be combined freely: `--sprint`, `--exec`, `--quick`, `--web`, a week, or a date range.

---

## Report modes explained

### Default (no flag)
Full report — overview, all breakdown tables, needs attention, recurring issues, assignee breakdown, momentum, and a per-assignee ticket listing with descriptions and last comment for high-priority items.

### `--quick`
Summary tables only. Skips fetching conversation threads and the per-assignee ticket listing. Good for a fast daily or end-of-week check.

### `--exec`
Compact one-page executive view:
- 2–3 sentence narrative brief
- Overview metrics table
- Needs Attention section (overdue, unassigned, high-priority open)
- Service breakdown
- Assignee snapshot with workload scores
- Top recurring issues
- Ticket type breakdown
- Momentum (last 7 days)
- Operational insights + recommendations

No per-assignee listing or conversation threads.

### `--web`
After generating the text report, writes a self-contained dark-themed HTML file to `/tmp/desk_report_<period>.html` and opens it in your browser.

### `--sprint`
Instead of using calendar week dates, auto-fetches the active sprint from Zoho Sprints (Discovery project) and uses its start/end dates as the report period. Pass a name fragment to target a specific sprint: `/desk --sprint "18 May"`.

---

## Service classification

Tickets are classified against E2E's real service list using subject + description keyword matching, with the `Internal Ticket Categorization` custom field used as a priority signal when set.

**Services covered:** TIR · GPU Cloud · k8s · Nodes · Volumes (Linstor) · SFS · Snapshots · CDP Backups · Saved Image · EOS · Reserve IP · VPC · Firewall (Fortigate) · Load Balancer · Networking · Existing Billing Maintenance · My Account DR · Dashboard · Monitoring · IAM / Auth · Support Section · Container Registry · DNS · DBaaS · API / SDK / CLI · SSL / License Management · CDN · Sign up / Onboarding · Open Nebula · Uncategorised

System auto-notifications (`[##...##]` subjects) are automatically excluded from all analysis.

---

## Adapting for a different department

At the top of `desk.md`, two IDs are hardcoded:

```
Department: E2E Product Dev  →  ID: 236167000034148333
Org:        E2E Networks Limited  →  ID: 658568929
```

Replace these with your own Zoho Desk department and org IDs. You can find them in your Zoho Desk admin settings under Setup → Departments.

The Zoho Sprints IDs (for `--sprint` mode) are also hardcoded:

```
Team ID:    667504989
Project ID: 8253000000154001   (Discovery project)
```

Update these if you want `--sprint` to sync with a different Zoho Sprints project.

---

## How it works (technical)

1. Parses the command arguments to determine date range (week / sprint / custom)
2. If `--sprint`: calls `ZohoSprints_GetSprints` to get active sprint start and end dates
3. Runs two parallel ticket fetches via `ZohoDesk_searchTickets`:
   - Pass A — tickets **created** in the period (paginated)
   - Pass B — tickets **closed** in the period (paginated)
   - Deduplicates by ticket ID across both passes
4. Classifies every ticket by service and type using keyword rules + custom fields
5. Writes a Python 3 script to `/tmp/desk_process_<period>.py` and executes it — computes all metrics and writes `/tmp/desk_metrics_<period>.json`
6. All output sections are rendered from that JSON
7. In full mode, fetches conversations for Needs Attention tickets (overdue, high-priority, unassigned) and adds a Last Comment column
8. If `--web`, generates a self-contained dark-themed HTML file and opens it in the browser

---

## Files

| File | Purpose |
|------|---------|
| `desk.md` | The skill definition — Claude's instructions for fetching, classifying, and rendering the report |
| `README.md` | This file — installation, usage, and overview |

---

## Related

- [sprint-skill](https://github.com/piyushkapoor-227/sprint-skill) — companion skill for Zoho Sprints live reports (`/sprint`, `/sprint --exec`, `/sprint --web`)
