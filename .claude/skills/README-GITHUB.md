# GitHub Operations Skills

Central reference for GitHub-related skills and when to use them.

All review and comment-addressing workflows are deeply tied to Linear issues. PRs should always reference a Linear ID (e.g. `ECC-123`) in the branch name, title, or description. Skills will fetch the issue automatically and use its requirements as the source of truth for necessity, completeness, and push-back decisions.

## Quick Reference

| Task                    | Use Skill               |
| ----------------------- | ----------------------- |
| Create PR review        | `github-ops`            |
| Reply to comments       | `github-ops`            |
| Fetch PR data           | `github-ops`            |
| Address review feedback | `address-pr-comments`   |
| Perform code review     | `review-code`           |

## Skill Details

### github-ops

**Use when:**

- Creating PR reviews (inline comments)
- Replying to review comments
- Resolving review threads
- Fetching PR data or comments

**Why use this instead of direct `gh` commands:**

- Ensures draft review creation (safety)
- Provides tested API patterns with correct API version header
- Handles common error cases

### address-pr-comments

**Use when:**

- Analyzing unresolved review comments
- Responding to reviewer feedback
- Making code changes based on reviews

**Note:** This skill automatically uses `github-ops` patterns for API operations.

### review-code

**Use when:**

- Performing structured code review
- Analyzing PRs for necessity/correctness/completeness

**Important:** When posting findings to GitHub, must invoke `github-ops` skill. Each issue includes a ready-to-paste fix prompt for the author.

## Common Mistakes to Avoid

1. **Using `gh pr review --comment`** - This immediately posts. Use the skill to create drafts.
2. **Using `gh api` directly for reviews** - Bypasses safety checks. Use the skill.
3. **Using `in_reply_to` on the create comment endpoint** - Use the `/replies` endpoint instead.
4. **Using `position` field** - Deprecated. Use `line` + `side`.
5. **Skipping deduplication** - Always check existing threads before posting new comments.
