---
description: "Write opinionated, read-only-first defaults to ~/.claude/settings.json (safe baseline for every Claude session)"
user-invocable: true
---

# /sane-global

Write a sane, read-only-first permission baseline to `~/.claude/settings.json`. This is the floor that applies to every Claude session across every project: read/search/fetch + git-read-only + gh-read-only, with aggressive denies for destructive shell, code-exec escapes, and sensitive path reads.

## Steps

1. **Read** `~/.claude/settings.json` if it exists. If it does, **back it up** to `~/.claude/settings.json.bak-<YYYYMMDD-HHMMSS>` using the current timestamp. Tell the user the backup path.

2. **Merge** the fragment below into the existing settings:
   - `env.*`: set these keys (do not clobber other env keys the user already has)
   - `enableAllProjectMcpServers`: set to `false` (explain why if the user had it `true`)
   - `permissions.allow`: **union** with existing — add entries that aren't already there, don't remove anything
   - `permissions.deny`: **union** with existing
   - Any other top-level keys the user has (hooks, statusLine, enabledPlugins, etc.): leave untouched

3. **Write** the merged result back to `~/.claude/settings.json`.

4. **Report** to the user:
   - Backup path (if any)
   - Count of new allow entries added, new deny entries added
   - A one-line reminder: "This plugin's hooks (shell-exec, destructive-git, curl|sh, rm -rf /, sudo, gh-api-writes) are active regardless of settings.json."

## Fragment to merge

```json
{
  "env": {
    "DISABLE_TELEMETRY": "1",
    "DISABLE_ERROR_REPORTING": "1",
    "CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY": "1"
  },
  "enableAllProjectMcpServers": false,
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "WebSearch",
      "WebFetch",
      "Bash(ls:*)",
      "Bash(pwd)",
      "Bash(which:*)",
      "Bash(file:*)",
      "Bash(date)",
      "Bash(uname:*)",
      "Bash(whoami)",
      "Bash(env)",
      "Bash(echo:*)",
      "Bash(git status:*)",
      "Bash(git log:*)",
      "Bash(git diff:*)",
      "Bash(git branch:*)",
      "Bash(git show:*)",
      "Bash(git remote -v)",
      "Bash(git remote show:*)",
      "Bash(git rev-parse:*)",
      "Bash(git stash list:*)",
      "Bash(git config --get:*)",
      "Bash(git config --list)",
      "Bash(gh pr view:*)",
      "Bash(gh pr list:*)",
      "Bash(gh pr diff:*)",
      "Bash(gh pr checks:*)",
      "Bash(gh issue view:*)",
      "Bash(gh issue list:*)",
      "Bash(gh repo view:*)",
      "Bash(gh run view:*)",
      "Bash(gh run list:*)",
      "Bash(gh search:*)",
      "Bash(gh auth status)",
      "Bash(gh api repos/:*)"
    ],
    "deny": [
      "Bash(sudo:*)",
      "Bash(rm -rf /*)",
      "Bash(rm -rf ~*)",
      "Bash(rm -rf *)",
      "Bash(rm -fr *)",
      "Bash(mkfs:*)",
      "Bash(dd:*)",
      "Bash(:(){ :|:& };:)",
      "Bash(curl * | sh*)",
      "Bash(curl *|sh*)",
      "Bash(curl * | bash*)",
      "Bash(curl *|bash*)",
      "Bash(wget * | sh*)",
      "Bash(wget *|sh*)",
      "Bash(wget * | bash*)",
      "Bash(wget *|bash*)",
      "Bash(python -c:*)",
      "Bash(python3 -c:*)",
      "Bash(node -e:*)",
      "Bash(node --eval:*)",
      "Bash(deno eval:*)",
      "Bash(bash -c:*)",
      "Bash(sh -c:*)",
      "Bash(zsh -c:*)",
      "Bash(perl -e:*)",
      "Bash(ruby -e:*)",
      "Bash(eval:*)",
      "Bash(git push:*)",
      "Bash(git push --force*)",
      "Bash(git push -f*)",
      "Bash(git reset --hard*)",
      "Bash(git clean -fd*)",
      "Bash(git clean -fdx*)",
      "Read(~/.ssh/**)",
      "Read(~/.gnupg/**)",
      "Read(~/.aws/**)",
      "Read(~/.azure/**)",
      "Read(~/.config/gh/**)",
      "Read(~/.git-credentials)",
      "Read(~/.netrc)",
      "Read(~/.npmrc)",
      "Read(~/.pypirc)",
      "Read(~/.docker/config.json)",
      "Read(~/.kube/**)",
      "Read(~/Library/Keychains/**)",
      "Edit(~/.ssh/**)",
      "Edit(~/.aws/**)",
      "Edit(~/.bashrc)",
      "Edit(~/.zshrc)",
      "Edit(~/.profile)",
      "Write(**/.env)",
      "Write(**/.env.*)",
      "Edit(**/.env)",
      "Edit(**/.env.*)"
    ]
  }
}
```

## Notes for the assistant

- `Bash(cat:*)` is **deliberately absent**. Use the `Read` tool for files — it respects the deny list for sensitive paths, `cat` does not.
- No `mcp__*` wildcards. Each MCP server is a trust decision the user makes themselves.
- The plugin's own PreToolUse hooks (shell-exec, destructive-git, curl|sh, rm -rf /, sudo, gh-api-writes) are active whether or not this command has been run. Denylist entries above are defense-in-depth duplicates where the native matcher can express the rule; the hooks catch the patterns the matcher can't.
- If the user's existing settings.json has conflicting allows (e.g. `Bash(git push:*)`), point them out — **don't** silently remove them. The user may have them on purpose for a specific workflow.
