---
description: "Open the current repo for autonomous Claude work: Edit/Write + build/test + git add/commit. Still blocks git push, reset --hard, installs, and sensitive reads."
argument-hint: "[--local]"
user-invocable: true
---

# /sane-repo

Open the **current repo** for autonomous work. Claude can edit, write, delete files inside the repo, run builds and tests, stage and commit — but the plugin's hooks still block `git push`, `git reset --hard`, shell-exec escapes, and the rest of the wall. Installs (`npm install`, `pip install`, etc.) are deliberately **not** allowlisted and will still prompt per call.

## Target file

- **Default (no flag)**: write to `./.claude/settings.json` — **committed to the repo**, shared with the team. The "sane baseline every teammate inherits" reading.
- **`--local`**: write to `./.claude/settings.local.json` instead — gitignored, personal to this checkout only. Use when you don't want to commit the permissive config.

Check `$ARGUMENTS` for `--local`. If present, target is `settings.local.json`, otherwise `settings.json`.

## Steps

1. **Verify you are in a git repository root.** If not, refuse and tell the user to `cd` to a repo root first.

2. If targeting `settings.local.json`: verify `.claude/settings.local.json` is in `.gitignore`. If `.gitignore` exists and doesn't mention it, offer to append `.claude/settings.local.json` to `.gitignore`.

3. **Read** the target file if it exists. If it does, back it up to `<target>.bak-<YYYYMMDD-HHMMSS>`.

4. **Merge** the fragment below:
   - `permissions.allow`: **union** with existing
   - Leave every other key untouched

5. **Write** the merged result back.

6. **Report**:
   - Which file was written (`settings.json` or `settings.local.json`)
   - Backup path (if any)
   - Count of new allow entries added
   - Remind the user: "`git push`, `git reset --hard`, and installs are **still** blocked — by the plugin's hooks and by `/sane-global`'s denylist. This command does not and cannot unlock them."

## Fragment to merge

```json
{
  "permissions": {
    "allow": [
      "Edit",
      "Write",
      "MultiEdit",
      "NotebookEdit",
      "Bash(mkdir:*)",
      "Bash(touch:*)",
      "Bash(mv:*)",
      "Bash(cp:*)",
      "Bash(rm:*)",
      "Bash(ln:*)",
      "Bash(chmod:*)",
      "Bash(npm run:*)",
      "Bash(npm test:*)",
      "Bash(npm exec:*)",
      "Bash(npx:*)",
      "Bash(pnpm run:*)",
      "Bash(pnpm test:*)",
      "Bash(pnpm exec:*)",
      "Bash(pnpm dlx:*)",
      "Bash(yarn run:*)",
      "Bash(yarn test:*)",
      "Bash(pytest:*)",
      "Bash(python -m pytest:*)",
      "Bash(python -m pip show:*)",
      "Bash(uv run:*)",
      "Bash(uv sync)",
      "Bash(cargo build:*)",
      "Bash(cargo test:*)",
      "Bash(cargo run:*)",
      "Bash(cargo check:*)",
      "Bash(cargo clippy:*)",
      "Bash(cargo fmt:*)",
      "Bash(go build:*)",
      "Bash(go test:*)",
      "Bash(go run:*)",
      "Bash(go vet:*)",
      "Bash(make:*)",
      "Bash(tsc:*)",
      "Bash(eslint:*)",
      "Bash(prettier:*)",
      "Bash(ruff:*)",
      "Bash(mypy:*)",
      "Bash(git add:*)",
      "Bash(git commit:*)",
      "Bash(git checkout:*)",
      "Bash(git stash:*)",
      "Bash(git stash pop:*)",
      "Bash(git restore:*)",
      "Bash(git merge --no-ff:*)",
      "Bash(git merge --ff-only:*)",
      "Bash(git switch:*)",
      "Bash(docker compose up:*)",
      "Bash(docker compose down:*)",
      "Bash(docker compose logs:*)",
      "Bash(docker compose ps:*)",
      "Bash(docker build:*)"
    ]
  }
}
```

## Deliberately absent

The following are **not** on the allowlist and will still prompt for confirmation every time:

- `npm install`, `pnpm install`, `pip install`, `cargo add`, `go get`, `brew install`, `apt-get install` — adding new dependencies is a supply-chain trust decision, stays gated.
- `docker pull`, `docker run` (non-compose) — same reason.
- `git push`, `git reset --hard`, `git clean -fd` — blocked by plugin hooks, not unlockable here.
- `sudo` anything — blocked by plugin hooks.
- `python -c` / `node -e` / `bash -c` / `eval` — blocked by plugin hooks.
- `curl | sh`, `wget | sh` — blocked by plugin hooks.
