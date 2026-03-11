---
name: review-code
description: Perform FAANG-level code review analyzing necessity, correctness, and completeness of changes
disable-model-invocation: true
---

# FAANG-Level Code Review

Perform a rigorous code analysis and review of the specified changes. Evaluate whether the changes are **necessary**, **correct**, and **complete**.

## Arguments

- `$ARGUMENTS`: PR URL, branch name, commit SHA, or file path(s) to review

## Review Framework

### 0. Load Linear MCP tools

Before reviewing, load:

```
select:mcp__claude_ai_Linear__get_issue
select:mcp__claude_ai_Linear__list_issues
```

### 1. Understand the Context

**Find the linked Linear issue(s)**:

1. Extract the Linear ID from the PR — check branch name, PR title, and PR description for patterns like `ECC-123` or `[A-Z]+-\d+`
2. Call `mcp__claude_ai_Linear__get_issue` with the extracted ID
3. If no ID is found, call `mcp__claude_ai_Linear__list_issues` with key terms from the PR title to find the likely issue
4. If still not found, note it as a gap and proceed without — but flag it in the review

From the Linear issue, extract:
- **Requirements**: What the issue says must be done
- **Acceptance criteria**: Any explicit criteria listed in the description
- **Priority and type**: Bug fix vs. feature vs. improvement

Before reviewing code, understand:

- What is the associated Linear issue/s?
- What problem is being solved?
- What is the expected behavior change?
- Who are the stakeholders (users, other services, etc.)?

### 2. Necessity Analysis

Determine if the changes are **necessary**:

- [ ] Does the PR description or branch reference a Linear issue?
- [ ] Do the changes directly address what the Linear issue describes?
- [ ] Is this the right layer/component to make this change?
- [ ] Could this be achieved with existing code or a simpler approach?
- [ ] Are there any unnecessary changes (formatting, refactoring) bundled in?
- [ ] Is every added line of code essential?

### 3. Correctness Analysis

Determine if the changes are **correct**:

**Functional Correctness**

- [ ] Does the code do what it claims to do?
- [ ] Are edge cases handled (null, empty, boundary values)?
- [ ] Is error handling appropriate and consistent?
- [ ] Are there any race conditions or concurrency issues?
- [ ] Is state managed correctly?

**Security**

- [ ] Input validation present where needed?
- [ ] No injection vulnerabilities (SQL, XSS, command)?
- [ ] Secrets/credentials handled properly?
- [ ] Authentication/authorization checked?
- [ ] No sensitive data exposure in logs or responses?

**Performance**

- [ ] No unnecessary database queries or N+1 problems?
- [ ] Appropriate use of caching?
- [ ] No blocking operations in hot paths?
- [ ] Memory usage reasonable?
- [ ] Algorithm complexity appropriate for expected data size?

**Reliability**

- [ ] Failures handled gracefully?
- [ ] Retries implemented where appropriate?
- [ ] Timeouts configured for external calls?
- [ ] No silent failures that could cause data inconsistency?

### 4. Completeness Analysis

Determine if the changes are **complete**:

- [ ] Does the implementation fully address every requirement in the linked Linear issue?
- [ ] Are all acceptance criteria from the Linear issue satisfied?
- [ ] If the Linear issue describes multiple sub-tasks, are all of them covered?
- [ ] Are all code paths tested?
- [ ] Are tests meaningful (not just coverage padding)?
- [ ] Is documentation updated if needed (API docs, README)?
- [ ] Are database migrations included if schema changed?
- [ ] Are feature flags or rollback mechanisms in place for risky changes?
- [ ] Are monitoring/alerting considerations addressed?

### 5. Code Quality

Evaluate maintainability:

- [ ] Is the code readable and self-documenting?
- [ ] Are names descriptive and consistent with codebase conventions?
- [ ] Is complexity manageable (functions not too long, nesting not too deep)?
- [ ] Are abstractions at the right level?
- [ ] Is there unnecessary duplication?
- [ ] Does it follow existing patterns in the codebase?

## Output Format

Structure your review as:

```
## Linear Issue
[ID and title, e.g. "ECC-123: Add rate limiting to API endpoints"]
[If no linked issue found: "No linked Linear issue found — PR should reference one."]

## Summary
[1-2 sentence overview of what the changes do]

## Verdict: [APPROVE | REQUEST_CHANGES | NEEDS_DISCUSSION]

### Necessary: [YES | PARTIAL | NO]
[Does the PR match what the Linear issue asks for?]

### Correct: [YES | PARTIAL | NO]
[Brief explanation with specific issues if any]

### Complete: [YES | PARTIAL | NO]
[Does the PR satisfy all acceptance criteria from the Linear issue? What's missing?]

## Critical Issues (blocking)
[List any issues that must be fixed before merge]

For each issue, include a suggested fix prompt:
> **Fix prompt:** "In `path/to/file.py`, fix [specific issue]. [One sentence of context about what correct behavior looks like.]"

## Suggestions (non-blocking)
[List improvements that would be nice but aren't required]

For each suggestion, include a suggested prompt:
> **Prompt:** "In `path/to/file.py`, [what to do]. [One sentence of context.]"

## Questions
[List any clarifying questions for the author]
```

## Fix Prompt Guidelines

When an issue is identified, always include a **Fix prompt** — a ready-to-paste agent instruction the author can use to address it immediately.

**Format:**

> **Fix prompt:** "In `{file}:{line}`, {imperative verb} {what}. {One sentence of relevant context.}"

**Examples:**

> **Fix prompt:** "In `src/api/auth.py:42`, validate that `user_id` is a non-empty string before passing it to `get_user()`. The function currently raises an unhandled `KeyError` when called with an empty string."

> **Fix prompt:** "In `src/worker/processor.py:87`, add a timeout to the `httpx.get()` call. Without it, a slow upstream will stall the worker indefinitely."

**Rules:**

- Be specific: include file path and line number
- Use imperative mood ("add", "remove", "replace", "validate")
- Keep to one sentence of context — not a lecture
- Only for actionable issues, not for questions or praise

## Review Principles

1. **Be specific**: Point to exact lines/files. Use `file:line` format.
2. **Explain why**: Don't just say "this is wrong" - explain the impact.
3. **Suggest solutions**: When identifying problems, propose fixes when possible.
4. **Prioritize**: Distinguish blocking issues from nice-to-haves.
5. **Stay objective**: Focus on the code, not the author.
6. **Assume good intent**: The author may have context you don't.
7. **Be thorough but efficient**: Don't nitpick style if there are logic bugs.

## Review Tone for GitHub

When posting reviews to GitHub, write as a human colleague — not an AI with a classification system. The structured analysis above is for local display only.

- **Frame suggestions as questions**: "Is the intent..." or "Would you consider..." rather than "You should..."
- **No severity prefixes**: Don't label comments with `nit:`, `suggestion:`, `question (low):`, etc.
- **Explain why**: Link reasoning to concrete impacts (performance, maintainability, correctness)
- **Assume good intent**: The author likely has context you don't — pose alternatives as questions

**Good:**

```
Since `max_concurrent_activities` is per-worker, and both this and the workflow
semaphore are set to 40 — is the intent that a single workflow can use full capacity?
```

**Avoid:**

```
**question (low):** Since `max_concurrent_activities` is per-worker...
```

## Severity Levels

Use these for internal analysis only. Do not include severity labels in GitHub-posted comments.

- **Critical**: Security vulnerabilities, data loss risks, breaking changes
- **High**: Bugs that will cause failures in production
- **Medium**: Performance issues, missing error handling, incomplete features
- **Low**: Code style, minor improvements, documentation gaps

## Before Posting to GitHub

**Pre-flight checklist** (verify all before posting):

- [ ] Using `gh api` patterns from `github-ops` skill
- [ ] Review will be created as DRAFT (omit the `event` field)
- [ ] File paths and line numbers are accurate
- [ ] Comments are written in plain conversational language (no severity prefixes)

If any item is unchecked, DO NOT POST THE REVIEW.

## Creating GitHub Reviews

When posting review findings to GitHub, use the `github-ops` skill for correct API patterns.

**CRITICAL**: Reviews MUST be created as drafts (omit the `event` field) so the user can review before submission.
