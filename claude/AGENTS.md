# Agent Instructions

This project uses **bd** (beads) for issue tracking. Run `bd onboard` to get started.

## Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --status in_progress  # Claim work
bd comments add <id> "tokens: <count>"  # Log token usage
bd close <id>         # Complete work
bd sync               # Sync with git
```

## Token Tracking

**Track token usage for every issue:**

1. **At start of issue** - Note current token count from context
2. **When closing issue** - Calculate tokens used and log it:
   ```bash
   bd comments add <id> "tokens: <count>"
   bd close <id>
   ```

Example:
```bash
# Started with 1000000 tokens, now at 965744
# Used: 34256 tokens
bd comments add atlas-123 "tokens: 34256"
bd close atlas-123
```

## Landing the Plane (Session Completion)

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until changes are committed locally.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status and log tokens**:
   - Calculate tokens used this session
   - Add comment with token count: `bd comments add <id> "tokens: <count>"`
   - Close finished work: `bd close <id>`
   - Update in-progress items
4. **COMMIT CHANGES** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd sync
   git commit -m "descriptive message"
   git status  # Verify all changes committed
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed locally
7. **Hand off** - Provide context for next session and inform user that changes are ready to push

**CRITICAL RULES:**
- Work is NOT complete until changes are committed locally
- NEVER leave uncommitted changes - commit everything
- DO NOT push to remote - user will push manually after review
- If authentication/push issues occur, stop at commit step and inform user

