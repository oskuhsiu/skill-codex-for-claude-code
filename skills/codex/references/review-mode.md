# Code Review with Codex CLI

Codex can review code changes at various scopes: uncommitted changes, branch diffs, or specific commits.

## Basic review commands

### Review uncommitted changes (most common)
```bash
codex exec --sandbox read-only "review the uncommitted changes in this repo for bugs, security issues, and code quality" 2>/dev/null
```

### Review changes against a branch
```bash
codex exec --sandbox read-only "review the diff between the current branch and main. Focus on breaking changes and test coverage" 2>/dev/null
```

### Review a specific commit
```bash
codex exec --sandbox read-only "review commit abc1234 for correctness and style" 2>/dev/null
```

## Review patterns

### Security-focused review
```bash
codex exec --sandbox read-only "perform a security audit of the uncommitted changes. Check for injection vulnerabilities, auth issues, and data exposure" 2>/dev/null
```

### Performance review
```bash
codex exec --sandbox read-only "review the recent changes for performance issues: unnecessary allocations, N+1 queries, missing indexes" 2>/dev/null
```

### API compatibility review
```bash
codex exec --sandbox read-only "check if the uncommitted changes break any public API contracts or backward compatibility" 2>/dev/null
```

## Tips for effective reviews

1. **Always use `read-only` sandbox** — reviews should never modify code
2. **Be specific about focus areas** — "review for security" yields better results than "review this code"
3. **Provide context** — mention the project type, framework, or standards to follow
4. **Use with Claude's own review** — run Codex review alongside Claude's analysis for a second opinion
5. **Resume for follow-up** — after a review, resume the session to ask about specific findings

## Combining with Claude's review

A powerful pattern: have both Claude and Codex review the same changes independently, then compare findings.

1. Claude reviews the changes directly (using Read/Grep tools)
2. Codex reviews via `codex exec --sandbox read-only "review..." 2>/dev/null`
3. Compare the two reviews, highlighting where they agree and disagree
4. Let the user decide on disputed items
