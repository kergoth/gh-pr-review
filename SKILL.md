---
name: gh-pr-review
description: Use when working with GitHub PR review threads and inline review comments, especially when agents need structured context, explicit repository scoping, and confirmation before write operations
---

# gh-pr-review

A GitHub CLI extension that provides complete inline PR review comment access from the terminal with LLM-friendly JSON output.

## When to Use

Use this skill when you need to:

- View inline review comments and threads on a pull request
- Reply to review comments programmatically
- Resolve or unresolve review threads
- Create and submit PR reviews with inline comments
- Access PR review context for automated workflows
- Filter reviews by state, reviewer, or resolution status

This tool is particularly useful for:
- Automated PR review workflows
- LLM-based code review agents
- Terminal-based PR review processes
- Getting structured review data without multiple API calls

## Installation

First, ensure the extension is installed and trusted according to your local policy:

```sh
gh extension install agynio/gh-pr-review
```

## Security and Data Handling

- Treat pull request review data as potentially confidential. Inline comments, replies, file paths, review summaries, and thread metadata may expose proprietary code, internal design discussions, operational details, or security-relevant context.
- Do not send raw review output to external or unapproved models, services, logs, or tools unless you are explicitly authorized to do so for that repository and data class.
- Always scope commands to the intended repository with `-R owner/repo`.
- Minimize retrieved data. Prefer `--unresolved`, `--not_outdated`, `--reviewer <login>`, and `--tail 1` unless broader context is necessary.
- Prefer quoting or summarizing only the minimum necessary review text when handing context to another tool or agent.
- Treat any command that mutates GitHub state as requiring explicit user confirmation before execution.

## Safety Model

Read-only operations:
- `review view`
- `threads list`

Mutating operations:
- `comments reply`
- `threads resolve`
- `threads unresolve`
- `review --start`
- `review --add-comment`
- `review --submit`

Required behavior for mutating operations:
- Confirm the target repository and PR number explicitly.
- Confirm the intended action before execution.
- Prefer inspecting current thread state with a read-only command immediately before higher-impact writes, or when thread context is unclear.

## Core Commands

### 1. View All Reviews and Threads

Get complete review context with inline comments and thread replies:

```sh
gh pr-review review view -R owner/repo --pr <number>
```

**Useful filters:**
- `--unresolved` - Only show unresolved threads
- `--reviewer <login>` - Filter by specific reviewer
- `--states <APPROVED|CHANGES_REQUESTED|COMMENTED|DISMISSED>` - Filter by review state
- `--tail <n>` - Keep only last n replies per thread
- `--not_outdated` - Exclude outdated threads

**Output:** Structured JSON with reviews, comments, thread_ids, and resolution status. Treat this output as potentially sensitive review data.

### 2. Reply to Review Threads

This is a mutating operation. Confirm intent before running it.

Reply to an existing inline comment thread:

```sh
gh pr-review comments reply <pr-number> -R owner/repo \
  --thread-id <PRRT_...> \
  --body "Your reply message"
```

### 3. List Review Threads

Get a filtered list of review threads:

```sh
gh pr-review threads list -R owner/repo <pr-number> --unresolved --mine
```

### 4. Resolve/Unresolve Threads

This is a mutating operation. Confirm intent before running it.

Mark threads as resolved:

```sh
gh pr-review threads resolve -R owner/repo <pr-number> --thread-id <PRRT_...>
```

### 5. Create and Submit Reviews

These are mutating operations. Confirm intent before running them.

Start a pending review:

```sh
gh pr-review review --start -R owner/repo <pr-number>
```

Add inline comments to pending review:

```sh
gh pr-review review --add-comment \
  --review-id <PRR_...> \
  --path <file-path> \
  --line <line-number> \
  --body "Your comment" \
  -R owner/repo <pr-number>
```

Submit the review:

```sh
gh pr-review review --submit \
  --review-id <PRR_...> \
  --event <APPROVE|REQUEST_CHANGES|COMMENT> \
  --body "Overall review summary" \
  -R owner/repo <pr-number>
```

## Output Format

All commands return structured JSON optimized for programmatic use:

- Consistent field names
- Stable ordering
- Omitted fields instead of null values
- Essential data only (no URLs or metadata noise)
- Pre-joined thread replies

Example output structure:

```json
{
  "reviews": [
    {
      "id": "PRR_...",
      "state": "CHANGES_REQUESTED",
      "author_login": "reviewer",
      "comments": [
        {
          "thread_id": "PRRT_...",
          "path": "src/file.go",
          "author_login": "reviewer",
          "body": "Consider refactoring this",
          "created_at": "2024-01-15T10:30:00Z",
          "is_resolved": false,
          "is_outdated": false,
          "thread_comments": [
            {
              "author_login": "author",
              "body": "Good point, will fix",
              "created_at": "2024-01-15T11:00:00Z"
            }
          ]
        }
      ]
    }
  ]
}
```

## Best Practices

1. **Always use `-R owner/repo`** to specify the repository explicitly
2. **Use `--unresolved` and `--not_outdated`** to focus on actionable comments
3. **Save thread_id values** from `review view` output for replying
4. **Filter by reviewer** when dealing with specific review feedback
5. **Use `--tail 1`** to reduce output size by keeping only latest replies
6. **Parse JSON output** instead of trying to scrape text
7. **Confirm intent before every write** including replies, resolution changes, and review submission

## Common Workflows

### Get Unresolved Comments for Current PR

```sh
gh pr-review review view --unresolved --not_outdated -R owner/repo --pr $(gh pr view --json number -q .number)
```

### Reply to All Unresolved Comments

This workflow performs writes. Confirm intent before executing replies or thread resolution.

1. Get unresolved threads: `gh pr-review threads list --unresolved -R owner/repo <pr>`
2. For each thread_id, reply: `gh pr-review comments reply <pr> -R owner/repo --thread-id <id> --body "..."`
3. Optionally resolve: `gh pr-review threads resolve <pr> -R owner/repo --thread-id <id>`

### Create Review with Inline Comments

This workflow performs writes. Confirm intent before starting or submitting a review.

1. Inspect current PR state: `gh pr-review review view --not_outdated -R owner/repo --pr <pr>`
2. Start: `gh pr-review review --start -R owner/repo <pr>`
3. Add comments: `gh pr-review review --add-comment -R owner/repo <pr> --review-id <PRR_...> --path <file> --line <num> --body "..."`
4. Submit: `gh pr-review review --submit -R owner/repo <pr> --review-id <PRR_...> --event REQUEST_CHANGES --body "Summary"`

## Important Notes

- All IDs use GraphQL format (PRR_... for reviews, PRRT_... for threads)
- Commands use pure GraphQL (no REST API fallbacks)
- Empty arrays `[]` are returned when no data matches filters
- The `--include-comment-node-id` flag adds PRRC_... IDs when needed
- Thread replies are sorted by created_at ascending

## Documentation Links

- Usage guide: docs/USAGE.md
- JSON schemas: docs/SCHEMAS.md
- Agent workflows: docs/AGENTS.md
- Blog post: https://agyn.io/blog/gh-pr-review-cli-agent-workflows
