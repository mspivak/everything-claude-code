---
name: github-ops
description: GitHub PR operations via gh CLI. Use when creating reviews, replying to comments, or fetching PR data. CRITICAL - Always create reviews as drafts by omitting the event field.
allowed-tools: Bash
---

# GitHub PR Operations

Comprehensive reference for GitHub pull request operations using the `gh` CLI and API.

For detailed command patterns and syntax, see [Command Reference](./COMMANDS.md).

## CRITICAL: Always Create Draft Reviews

When creating a new PR review with inline comments, **ALWAYS create it as a pending/draft review first**.

**DO NOT** include an `event` field when creating the review. Omitting `event` creates a **pending review** (draft state) that the user can review and submit from the GitHub UI.

### Event field values

**CRITICAL**: Only include `event` field when user explicitly requests submission.

| Value               | Behavior                                   | When to Use                   |
| ------------------- | ------------------------------------------ | ----------------------------- |
| Omit field          | Creates draft/pending review               | **DEFAULT - Always use this** |
| `"PENDING"`         | Creates draft/pending review               | Same as omitting              |
| `"COMMENT"`         | **Submits immediately** as comment         | Only if explicitly requested  |
| `"APPROVE"`         | **Submits immediately** as approval        | Only if explicitly requested  |
| `"REQUEST_CHANGES"` | **Submits immediately** requesting changes | Only if explicitly requested  |

## Review Content Guidelines

### Voice and Tone

- **Write as the reviewer, not as an AI**: Use natural, conversational language
  - Good: "Looks good — all files moved, imports updated, CI green."
  - Avoid: "## Verdict: APPROVE\n\n### Necessary: YES\n### Correct: YES"

- **Keep it conversational**: Frame observations as you would in a normal code review

### Inline Comments

**Only post inline comments for actionable items** — things the author should change, consider, or respond to.

**Write in plain language** — no severity prefixes, no bold labels.

**Do NOT post:**

- Affirmative comments ("good catch", "verified this works", "looks correct")
- Observations with no action needed
- Praise or agreement

**If there are no actionable items**, a body-only approval is the correct review.

### Internal vs. External Output

The `review-code` skill's Output Format is an **internal analysis framework**. Do NOT copy it verbatim into a GitHub review. Summarize findings in natural language instead.

## Best Practices

1. **Always create drafts first** - Let users review before submission
2. **Use suggestion blocks** - Format code suggestions with ```suggestion blocks
3. **Be specific** - Include file paths and line numbers in review summaries
4. **Deduplicate before commenting** - Query existing threads before adding new ones to avoid re-commenting on already-addressed issues
5. **Batch comments** - Accumulate all inline comments into one pending review to avoid per-comment notifications
6. **Handle errors gracefully** - Check API responses before assuming success
