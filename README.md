# sane-claude-setup

Opinionated, privacy-first Claude Code defaults, shipped as a plugin. Two slash commands and a set of deterministic PreToolUse guardrails that the plugin installs for you.

> **Deny by default, opt in to what's safe.** Not the other way around.

## What you get

| | |
|---|---|
| `/sane-global` | Writes a read-only-first baseline to `~/.claude/settings.json`. Every Claude session in every project inherits it: read/search/fetch, git read-only, gh read-only. Nothing else. |
| `/sane-repo` | Opens **the current repo** for autonomous work: Edit/Write, build, test, `git add`/`commit`. Still blocks push, reset --hard, installs, and sensitive reads. `--local` to write to `settings.local.json` instead of the committed `settings.json`. |
| Hooks (auto-active on install) | `python -c` / `node -e` / `bash -c` / `eval` blocker · destructive git (`push`, `reset --hard`, `clean -fd`) blocker · `curl\|sh` blocker · `rm -rf /` / `~` / `*` blocker · `sudo` blocker · `gh api` write-method blocker |

Three layers of defense, in order:

1. **Hooks** — deterministic, always-on once the plugin is installed. Cannot be unlocked by any `/sane-*` command. This is the floor.
2. **Denylist** — defense-in-depth duplicates of hook rules in `settings.json`, plus sensitive-path `Read`/`Edit` denies (`~/.ssh`, `~/.gnupg`, `~/.aws`, etc.).
3. **Allowlist** — explicit opt-in to each tool category. If it's not on the list, Claude has to ask.

## Install

```bash
# Add the marketplace (one time, inside a Claude Code session)
/plugin marketplace add DominikKanjuh/sane-claude-setup

# Install the plugin from it
/plugin install sane-claude-setup@sane-claude-setup
```

Or clone and install from disk for local development:

```bash
git clone https://github.com/DominikKanjuh/sane-claude-setup.git
# then, inside a Claude Code session:
/plugin marketplace add ./sane-claude-setup
/plugin install sane-claude-setup@sane-claude-setup
```

Once installed, the hooks are **already active**. Restart your Claude session if needed.

> **Caveats worth reading before you ship.** This is v0.1 and genuinely untested at runtime. Known gaps: `node --eval` and `deno eval` slip the shell-exec hook; `rm -rf ~/Desktop` slips the catastrophic-rm hook; `cp leaked.env .env` bypasses the `.env` write deny under `/sane-repo`; hook regexes false-positive on git commit messages containing the blocked patterns (you'll see a block when committing "fix the python -c crash"); the permission allowlist matcher semantics are assumed to be prefix-based and unverified. This plugin defends against **Claude-specific** failure modes (prompt injection, runaway autonomy) — it is **not** a sandbox against a compromised host. Treat it as a sane starting baseline, not a finished security product.

## Use

```bash
# In any Claude session, one time per machine:
/sane-global

# In each repo where you want Claude to work freely:
/sane-repo

# Same but don't commit the permissive config (personal to this checkout):
/sane-repo --local
```

## What's blocked (even after you run everything)

- `git push`, `git push --force`, `git reset --hard`, `git clean -fd` — make pushes a manual human decision
- `python -c`, `node -e`, `bash -c`, `eval` — code-exec flags bypass any allowlist; write to a file and run that instead
- `curl ... | sh`, `wget ... | sh` — classic supply-chain vector
- `rm -rf /`, `rm -rf ~`, `rm -rf *`, `rm -rf $HOME` — catastrophic paths
- `sudo` anything — Claude should never need root
- `gh api -X POST/PUT/DELETE/PATCH` — GitHub API writes stay manual
- Reads of `~/.ssh`, `~/.gnupg`, `~/.aws`, `~/.azure`, `~/.config/gh`, `~/.git-credentials`, `~/.netrc`, `~/.npmrc`, `~/.docker/config.json`, `~/.kube`, macOS Keychains
- Writes/edits to `**/.env`, `**/.env.*`
- Installing new dependencies (`npm install`, `pip install`, `cargo add`, `brew install`, `apt-get install`) — still prompts every time; new deps are a trust decision

## What's allowed after `/sane-global`

Read-only everything: `Read`/`Glob`/`Grep`, `WebSearch`/`WebFetch`, `mcp__context7__*` (free, read-only docs fetcher — install [context7](https://github.com/upstash/context7) separately if you want it), `git status|log|diff|show|rev-parse|branch` (destructive `git branch -d/-D/-m/-M/-c/-C` explicitly denied), `gh pr|issue|repo|run view/list/diff/checks`, `ls|pwd|which|file|tree|date|uname|whoami`, `gh api repos/...` (read).

## What's additionally allowed after `/sane-repo`

Full edit + typical build/test loop: `Edit`/`Write`/`MultiEdit`, `mkdir|touch|mv|cp|rm|chmod`, `npm|pnpm|yarn run/test/exec`, `pytest`, `uv run`, `cargo build/test/run/check/clippy/fmt`, `go build/test/run/vet`, `make`, `tsc`, `eslint`, `prettier`, `ruff`, `mypy`, `git add|commit|checkout|stash|restore|switch|merge --ff-only`, `docker compose up/down/logs/ps`, `docker build`.

## Undoing

Both commands back up the file they touch to `<file>.bak-<timestamp>` before writing. To fully revert:

```bash
# Global
mv ~/.claude/settings.json.bak-<timestamp> ~/.claude/settings.json

# Per-repo
mv .claude/settings.json.bak-<timestamp> .claude/settings.json
```

The hooks are part of the plugin — to disable them, disable or uninstall the plugin:

```bash
/plugin disable sane-claude-setup
# or
/plugin uninstall sane-claude-setup
```

## Philosophy

Claude Code gives you allow/deny/ask and hooks. Most setups on the internet ship either a **blank slate** (every tool call prompts — annoying, teaches you to auto-approve) or a **full auto-accept** (every tool call just runs — one prompt injection from disaster). Both are bad defaults.

The right answer is a tight, opinionated allowlist where Claude can do 80% of real work without prompting, but the 20% that could actually hurt you stays gated. That's what this plugin is.

**Inspired by but distinct from** [`trailofbits/claude-code-config`](https://github.com/trailofbits/claude-code-config). Trail of Bits uses a denylist-only posture and a clone-and-run template. This plugin uses an allowlist-first sandbox, ships as an installable plugin, and splits global (read-only) from per-repo (edit) as two separate commands you opt into explicitly.

## License

MIT
