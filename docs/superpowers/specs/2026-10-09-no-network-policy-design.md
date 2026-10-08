# No-Network Policy — Design

**Date:** 2026-10-09
**Status:** Approved in conversation; awaiting written-spec review
**Scope:** Private, company-internal copy of superpowers. Not intended for upstream.

## Goal

Make the plugin comply with two company rules:

1. It never sends information to any URL.
2. It never accesses any URL by default.

"By default" means a network read happens only when the human partner explicitly
asks for that specific access in the current session. Sending data is forbidden
regardless. Company git remotes count as URLs. A localhost-only server the agent
starts itself (the brainstorming visual companion) is allowed, provided the page
it serves loads no remote resources.

## Success criteria

- No runtime file (skills, hooks, harness adapters) instructs the agent to push,
  pull, fetch, clone, open PRs or issues, call `gh`/`curl`/`wget`, or run
  downloading package installs without asking.
- The visual companion page contains no `http(s)://` resource or link to a
  non-local host.
- Every session carries an explicit no-network rule via the bootstrap.
- A guard script fails when a new network mention appears in runtime files
  without an allowlist entry.

## Inventory of network paths (before)

| # | Location | Behavior |
|---|---|---|
| 1 | `skills/brainstorming/scripts/server.cjs` `brandMarkup()` | Loads a logo from `primeradiant.com` with the plugin version in the query string (on unless a telemetry-disable env var is set); links to GitHub |
| 2 | `skills/brainstorming/visual-companion.md` Design Tips | Suggests Unsplash images in mockups |
| 3 | `skills/using-git-worktrees/SKILL.md` Step 2 | Auto-runs `npm install`, `cargo build`, `pip install`, `poetry install`, `go mod download` |
| 4 | `skills/diagnosing-superpowers/` step 5 + `references/github-issues.md` | Searches GitHub issues via `gh`/`curl`; files issues via `gh issue create` |
| 5 | `skills/finishing-a-development-branch/SKILL.md` | `git pull` before merge; push + create PR options |
| 6 | `skills/receiving-code-review/SKILL.md` | Replies to review comments via `gh api` |

Harness tool-mapping references (codex, gemini, hermes, opencode, kimi) only name
tools and are unchanged. README, docs, tests and release scripts do not run at
plugin runtime and are unchanged.

## Design

### 1. Bootstrap rule

Add a `## Network Policy` section to `skills/using-superpowers/SKILL.md`,
immediately before `## User Instructions`. The session-start hook injects this
file into every session. Text:

> Never send data to any URL or remote: no `git push`, no PR or issue creation,
> no `gh`/`curl`/`wget` writes, no uploads. Do not access any URL by default: no
> web fetch or web search tools, no `git pull`/`fetch`/`clone`, no package
> installs that download. When a step needs network access, stop, name the exact
> command, and let your human partner run it. A read is allowed only when your
> human partner explicitly asks for that specific access in this session.
> Localhost-only servers you start yourself are fine.

### 2. Companion server

In `server.cjs`:

- `brandMarkup()` always returns text-only branding:
  `<div class="brand"><span class="brand-copy">Superpowers vX.Y.Z</span></div>`.
  No `<img>`, no `<a href>`.
- Remove `SUPERPOWERS_BRAND_IMAGE_URL`, `TELEMETRY_DISABLE_ENV_VARS` and
  `SUPERPOWERS_TELEMETRY_DISABLED` (and `isTruthyEnv` if it has no other use).
- Leave `frame-template.html` CSS unchanged; the now-unused `.brand-logo` and
  `.brand a` rules are harmless and keeping them keeps the diff small.

Rewrite `tests/brainstorm-server/branding.test.js` to assert: branding shows the
package version as text; the served page contains no `http(s)://` URL to a
non-local host; no `brand-logo` element; branding stays in the header before the
status element; the existing `.brand` / `.brand-copy` layout rules still hold.

### 3. Skill edits

**using-git-worktrees, Step 2 — Project Setup.** Detect the setup command per
manifest as today, but do not run it. Show the command and ask your human
partner whether to run it, since it downloads packages. If dependencies are
already present, proceed to the baseline tests.

**finishing-a-development-branch.**

- Normal repo menu: `1. Merge back to <base-branch> locally`,
  `2. Prepare for push (you push)`, `3. Keep the branch as-is`.
- Detached HEAD menu: `1. Prepare for push as a new branch (you push)`,
  `2. Keep as-is`.
- Option 1: remove `git pull`; note to your human partner that the local base
  may be behind its remote.
- Option 2 (renamed "Prepare for Push"): ensure work is committed on a named
  branch, then print the exact push command (`git push -u origin <branch>` or
  `git push origin HEAD:refs/heads/<new-branch>`) and a draft PR title and body
  following the repo's PR template if present. Never run them. Keep the
  worktree.
- Update the Quick Reference table and the PR/push rows in Common
  Rationalizations to match.

**receiving-code-review.** Replace "GitHub Thread Replies" with "Thread
Replies": draft each reply labelled with the comment it answers, for your human
partner to post in the thread. Remove the `gh api` command.

**diagnosing-superpowers.**

- Step 5 becomes "Issue draft — when report §7 says possible or likely, or your
  partner asks": fill `templates/issue.md`, write it to the workspace, show the
  text and give the path. Your human partner searches for duplicates and files
  it. Never search or file yourself.
- Delete `references/github-issues.md`.
- Leave `templates/issue.md` unchanged; its duplicate-search checkbox is for
  your human partner to tick after searching.
- Hard rules and description otherwise unchanged.

**brainstorming/visual-companion.md, Design Tips.** Replace the Unsplash line
with: use realistic domain copy; images only from local project files or inline
SVG/CSS; never remote images.

### 4. Guard

`tests/no-network/test-no-network.sh` (bash only).

- Scans: `skills/`, `hooks/`, `index.js`, `.opencode/plugins/`,
  `.pi/extensions/`, `.hermes-plugin/`.
- Flags: `https?://` URLs not to `localhost`, `127.0.0.1` or `[::1]`;
  `git (push|pull|fetch|clone)`; `\bgh `; `curl`; `wget`;
  `(npm|pip|poetry) install`; `go mod download`; `fetch\(`; `XMLHttpRequest`;
  `https?\.(request|get)\(`.
- Allowlist: `tests/no-network/allowlist.txt`, lines of
  `path<TAB>regex<TAB>reason`. A hit is allowed when its file matches `path` and
  its line matches `regex`. Comments (`#`) and blank lines ignored.
- Output: every non-allowlisted hit as `file:line: text`; exit 1 if any.
- Self-test: the script accepts an optional root dir argument so a test can
  point it at a temp copy with a planted `curl` line and confirm exit 1.

## Testing

- Guard passes on the modified tree; fails on a planted violation.
- `node tests/brainstorm-server/branding.test.js` passes (built-ins only).
- Other brainstorm-server tests that need no `ws` dependency are run;
  `ws`-dependent tests are skipped because installing `ws` needs network, and
  are reported as skipped.
- `bash tests/hooks/test-session-start.sh` passes.
- Skill prose changes are reviewed, not eval-tested: the upstream eval harness
  requires network/API access.

## Out of scope

- README, docs, release notes, release scripts, CI, and test files other than
  those named above.
- Harness tool-mapping references.
- Upstream contribution.
