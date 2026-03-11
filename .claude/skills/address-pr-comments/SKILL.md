---
name: address-pr-comments
description: Analyze and address unresolved review comments for pull requests with thoughtful responses and code fixes
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write, Grep, Glob, WebFetch, AskUserQuestion, github-ops
context: fork
---

Analyze and address unresolved review comments for the pull request: $ARGUMENTS

**Workflow Principle**: When user provides a specific discussion URL or identifies a particular comment, use targeted queries instead of broad fetching. See GitHub CLI Commands Reference below for extraction methods.

## Phase 0: Load Tools and Fetch Linear Context

**Load required tools first:**

```
select:mcp__claude_ai_Linear__get_issue
select:mcp__claude_ai_Linear__list_issues
```

**Find and fetch the linked Linear issue:**

1. Extract the Linear ID from the PR — check branch name, PR title, and PR description for patterns like `ECC-123` or `[A-Z]+-\d+`
2. Call `mcp__claude_ai_Linear__get_issue` with the extracted ID
3. If no ID found, call `mcp__claude_ai_Linear__list_issues` with key terms from the PR title

Store the Linear issue's requirements and acceptance criteria — you will use them in Phase 1 to evaluate whether reviewer feedback is aligned with the original intent.

## Phase 1: Fetch and Analyze

**STEP 1 - Determine scope and fetch comments**:

- If user provides specific discussion URL (`#discussion_rXXXXXX`): See "Extract comment from discussion URL" in `.claude/skills/github-ops/COMMANDS.md`
- If user provides only PR URL: Extract owner, repo, and PR number, then fetch pending review threads using GraphQL (see COMMANDS.md)

### GitHub CLI Commands Reference

**CRITICAL**: Before executing any `gh api` call, read `.claude/skills/github-ops/COMMANDS.md` for the exact syntax. Do not guess parameter names — the GitHub API has naming asymmetries between read and write endpoints.

For all GitHub API operations (fetching threads, creating reviews, replying to comments, resolving threads), reference the **`github-ops`** skill which provides comprehensive, tested patterns.

**Key operations available via github-ops**:
- Fetch unresolved review threads (GraphQL)
- Extract comment from discussion URL
- Get recent comments sorted by date
- **Create pending/draft reviews** (CRITICAL: always draft first)
- Reply to existing comments via `/replies` endpoint
- Submit pending reviews (only after confirmation)
- Resolve review threads

**CRITICAL**: When creating reviews, always create as drafts by omitting the `event` field.

For each unresolved thread, gather:
- File path and line number
- Author and timestamp
- Comment body and any replies

Read the relevant code context for each comment, then analyze whether the feedback is:
- **Good**: Is it constructive and actionable?
- **True**: Is it technically accurate?
- **Necessary**: Is addressing it important for code quality, correctness, or maintainability?
- **In scope**: Is it consistent with what the Linear issue actually asks for? Reviewer feedback that requests changes beyond the Linear issue scope is a candidate for push back.

For each comment, provide a recommendation:
- **Fix**: The feedback is valid and should be addressed with code changes
- **Push back**: The feedback fails the good/true/necessary/in-scope tests — cite the Linear issue requirements when pushing back on out-of-scope requests
- **Discuss**: Needs clarification or further discussion

Present the analysis in a clear format grouped by file and wait for user feedback.

### Verification Before Reporting (CRITICAL)

Before presenting your analysis:

1. **Count verification**: Explicitly state "Found X unresolved comment(s)"
2. **If X = 0**:
   - Do NOT claim all resolved without user confirmation
   - Say: "My query returned 0 unresolved comments. Can you confirm this is accurate, or provide the specific comment you'd like me to address?"

## Phase 2: Address Comments (after user approval)

Based on user feedback:

1. **For comments marked "Fix"**: Make the necessary code changes
2. **For comments marked "Push back"**: Draft a polite, professional reply explaining the reasoning
3. **For all comments**: Reply to each thread using the `/replies` endpoint (see github-ops COMMANDS.md)
4. **Resolve threads**: Mark each thread as resolved using `gh api graphql` mutation

Stage any code changes with `git add`. Do NOT commit or push automatically.

## Response Tone and Phrasing Guidelines

When drafting replies to PR review comments:

- **Be curious, not declarative**: Frame responses as questions or collaborative explorations
  - Good: "I'm wondering if this approach might have issues with X. Would it make sense to consider Y instead?"
  - Avoid: "This is wrong. You need to do Y."

- **Stay calm and factual**: Focus on technical details and reasoning

- **Acknowledge and respect**: Recognize the reviewer's perspective before presenting an alternative view
  - Good: "That's a good point about performance. I looked into it and found..."
  - Avoid: "That's not accurate. The real issue is..."

- **Use natural, conversational language**: Avoid overly formal or robotic phrasing

- **Ask for clarification when needed**: If a comment is unclear, ask thoughtfully rather than making assumptions

## Important Notes

- **No auto-commit**: Do NOT automatically commit or push changes. Stage changes and let the user decide when to commit.
- **Bot skepticism**: Automated review bots can make mistakes. Always verify suggestions against the actual code before accepting them.

### Error Recovery

If the user indicates your analysis is incorrect or you made a mistake:

1. **Acknowledge immediately**: "You're right, let me focus on that specific comment"
2. **Stop broad fetching**: Don't re-run expensive queries unnecessarily
3. **Ask for specifics** (one of):
   - The comment text/body
   - File path and line number
   - The exact discussion URL
4. **Use targeted approach**: Query only what's needed
5. **Don't over-explain or re-fetch everything**: Fix the issue and move forward
