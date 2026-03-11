---
name: linear
description: Linear issue tracker operations — creating, updating, triaging, and reviewing issues, projects, cycles, and initiatives via MCP.
origin: ECC
---

# Linear

Linear issue and project management via the Linear MCP server.

## When to Activate

- Creating or updating issues
- Checking active tickets and their status
- Triaging incoming issues
- Managing projects, cycles, or initiatives
- Generating project status updates

## MCP Tools

Always load tools via ToolSearch `select:` before calling them:

```
mcp__claude_ai_Linear__list_teams
mcp__claude_ai_Linear__list_projects
mcp__claude_ai_Linear__list_issues
mcp__claude_ai_Linear__save_issue          # create or update
mcp__claude_ai_Linear__save_comment
mcp__claude_ai_Linear__list_issue_labels
mcp__claude_ai_Linear__list_cycles
mcp__claude_ai_Linear__get_user
mcp__claude_ai_Linear__get_issue
mcp__claude_ai_Linear__list_issue_statuses
```

## Hierarchy

```
Workspace
└── Teams
    ├── Initiatives (cross-team, strategic groupings of projects)
    │   └── Projects (time-bound deliverables)
    │       ├── Milestones
    │       └── Issues
    │           └── Sub-issues
    └── Cycles (per-team sprints, typically 2 weeks)
        └── Issues
```

Every issue belongs to a **team**. Always confirm team via `list_teams` — never guess.

## Creating Issues

### Team and project discovery

1. `list_teams` → present options via AskUserQuestion, never ask user to type freeform
2. `list_projects` filtered to selected team, excluding archived → present relevant projects with "None" as an option
3. If user names a project that doesn't exist, note it — `save_issue` only accepts existing projects

### Deduplication

Before drafting, search for duplicates:

1. Extract 2–3 key terms from the problem
2. `list_issues` with `team` + `query` using those terms
3. Classify results:
   - **DUPLICATE** → ask user: skip, comment with new context, or create anyway
   - **RELATED** → mention when presenting draft so user can link
   - **NO MATCH** → proceed

If duplicate confirmed: `save_comment` on existing ticket with new context, then stop.

### Title conventions

- Imperative mood, under 80 characters
- Describe the outcome, not the activity
- Good: `Consolidate health check to test DB connectivity`
- Bad: `Update health check endpoint`

### Priority mapping

| Value | Label | When to use |
|-------|-------|-------------|
| 1 | Urgent | Production down or severely degraded |
| 2 | High | Impacts users or deployments; fix soon |
| 3 | Normal | Next sprint, standard work |
| 4 | Low | Backlog, nice-to-have |

### Estimates

| Size | Scope | Action |
|------|-------|--------|
| XS | ~1 line change | Create as-is |
| S | ≤1 day of work | Create as-is |
| M | 2–3 days of work | Create as-is |
| L | 3+ days of work | Flag to user — should be split into sub-issues before starting |

Always set an estimate. For L tickets, recommend splitting before creating and offer to draft the sub-issues.

### Description template

```markdown
## Problem

[What's broken or missing. Include context for someone unfamiliar. Reference incidents if applicable.]

## Solution

[Concrete steps with file paths. Specific about what code changes are needed.]

## Verification

[How to confirm the fix — commands to run, endpoints to test, etc.]
```

### Approval before creating

Present to user: title, team, project, priority, full description.
Ask: "I'll create this ticket. Shall I proceed?"

Then `save_issue` and return the ticket URL and identifier (e.g. `DEV-2091`).

## Checking Active Tickets

### Query broadly — never use state filter

`state: started` only returns In Progress and In Review. Triage, Backlog, Todo, and Upcoming are excluded. Always fetch broadly and filter client-side.

Run two parallel queries:
1. `list_issues(assignee: "me", includeArchived: false, limit: 100)`
2. `list_issues(query: "", limit: 50, includeArchived: false)` → filter client-side to `createdBy` email (use `get_user(query: "me")`)

Deduplicate by issue ID. Exclude Done, Canceled, Duplicate statuses.

### Group by status

Present in this order, sorted by priority (Urgent → High → Normal → Low) within each group:

1. In Progress
2. In Review
3. Triage
4. Todo
5. Backlog
6. Upcoming

### Output format

```
### In Progress (2)
| Ticket   | Title                          | Priority | Project        |
|----------|--------------------------------|----------|----------------|
| DEV-2149 | Build Supabase-to-S3 Migration | Medium   | Infrastructure |
| DEV-2100 | Fix health check endpoint      | High     | —              |

### Needs Attention
- **DEV-2150** is in Triage with no assignee (you created it)
- **DEV-2091** is High priority but still in Todo
```

Surface anomalies: unassigned tickets you created, stale Triage tickets, High/Urgent not In Progress.

## Status Conventions

When creating tickets for completed or in-flight work:

| Status | When to use |
|--------|-------------|
| Done | Fully complete and deployed/verified |
| In Review | Awaiting feedback or validation |
| In Progress | Code exists but not deployed or fully verified |

Do not assume merged PR = Done for infrastructure, migration, or ops work.

## Rate Limits

- Linear is generally tolerant but large result sets may exceed tool output limits
- Use `limit: 100` (not 250); paginate if needed
- When parallel batch partially fails with rate limit errors, retry failed calls in next batch

## Cycles

- Check current cycle load before assigning issues (`list_cycles`)
- Never overload a cycle — incomplete issues roll over automatically
- Scope discipline matters more than heroics

## Projects and Initiatives

- Every project needs one **owner**
- Use milestones to track phases; post status updates for async visibility
- Use initiatives only for OKR/strategic alignment — not for operational grouping
- Initiatives group multiple projects under a company objective

## Anti-Patterns

```
# BAD: Guessing team name instead of querying list_teams
# BAD: Creating an issue without checking for duplicates first
# BAD: Using state filter in list_issues queries
# BAD: Fetching with limit: 250 — use 100 and paginate
# BAD: Assuming merged PR = Done for ops/infra work
# BAD: Creating an initiative for operational grouping (use projects)
# BAD: Multiple owners on an issue
# BAD: Creating ticket without user approval
```
