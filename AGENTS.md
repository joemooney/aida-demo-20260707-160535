# AGENTS.md

Guidance for Codex and MCP-compatible coding agents working in this
repository. Read this as instructions-to-self: when you implement work,
coordinate through AIDA, keep the git/aida-store state coherent, and
leave durable traces for the next agent.

The block delimited by HTML comment markers below is auto-generated from
`.claude/AIDA.md` on each `aida scaffold apply`. Leave the markers in
place. Content outside the marked block is project-owned guidance.

## Project Orientation

aida-demo-20260707-160535

Use `OVERVIEW.md` for product/architecture context and
`docs/agents/cross-agent-onboarding.md` for the shared MCP operating
model. Use `docs/agents/codex-mcp-setup.md` when configuring Codex
against AIDA's MCP server. Use `docs/agents/session-communication.md`
for agent pause/abort/defer semantics.

<!-- AIDA-AUTOGEN-BEGIN -->
# AIDA Conventions

This file is the single source of truth for AIDA's coding conventions in
this project. CLAUDE.md imports it via `@.claude/AIDA.md`; AGENTS.md
inlines a copy inside auto-generated delimiters. Edit this file to change
the conventions for both.

## Requirements management

This project tracks requirements with [AIDA](https://github.com/joemooney/aida).
**Do not maintain a separate `REQUIREMENTS.md`** — the requirements DB is
canonical.

Requirements database: distributed git-canonical store at `.aida-store/` (orphan branch `aida-store`, plus a rebuildable SQLite cache at `.aida/cache.db`).

### Daily commands

```bash
aida list                              # list all requirements (cache-backed)
aida list --status draft               # filter by status
aida show <ID>                         # show details (e.g. `aida show FR-0042`)
aida search "<query>"                  # full-text search
aida add --title "..." --type <type> --status draft
aida edit <ID> --status in-progress
aida edit <ID> --status completed
aida comment add <ID> "implementation note..."
aida rel add --from <ID> --to <ID> --type <Parent|Verifies|References>
aida history                           # what was touched recently (digest)
aida statusline                        # one-line: project · role · queue · cache
```

### Requirement-first development

1. **Before coding:** check whether the work has a SPEC-ID. If not, create one
   (`aida add --type <task|story|bug|...> --status approved --title "..."`).
2. **During coding:** add inline trace comments referencing the SPEC-ID.
3. **Before committing:** mark the requirement `completed` (or `in-progress`
   if work continues), and ensure the commit message references it.

## Inline trace comments

Add a comment near the code that implements (or fixes, or verifies) a
requirement:

```rust
// trace:FR-0042 | ai:claude
fn implement_feature() { /* ... */ }
```

Format: `// trace:<SPEC-ID> | ai:<tool>[:<confidence>]`

- `<SPEC-ID>` — e.g. `FR-0042`, `BUG-1-017`, `TASK-0344`
- `<tool>` — `claude`, `codex`, `copilot`, `human`, `aider`, …
- `<confidence>` — optional: `high` (implied), `med` (40-80% AI), `low` (<40% AI)

## Commit message format

```
[AI:tool] type(scope): description (REQ-ID)
```

Examples:

```
[AI:claude] feat(auth): add login validation (FR-0042)
[AI:claude:med] fix(api): handle null response (BUG-0023)
[AI:antigravity+claude] test(hooks): accept mixed authorship (TASK-509)
chore(deps): update dependencies        # no REQ-ID needed
docs: update README                     # no REQ-ID needed
```

Rules:

- `[AI:tool]` required when commit includes AI-assisted code (any file with a
   `// trace:... | ai:tool` comment changed). Use `[AI:tool1+tool2]` for
   mixed-agent authorship, with optional confidence on the whole commit
   (`[AI:tool1+tool2:med]`).
- `type` required: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`,
   `build`, `ci`, `chore`, `revert`.
- `(scope)` optional — component or area affected.
- `(REQ-ID)` required for `feat`/`fix`; optional for `chore`/`docs`.

Set `AIDA_COMMIT_STRICT=true` (or commit through the `/aida-commit` skill) to
enforce; otherwise the commit-msg hook just warns on non-conforming messages.

## Capture proactively, not reactively

The requirements DB is only valuable when it stays in sync with reality.
Treat `/aida-capture` as a habit, not a safety net:

1. **Spec-first when introducing a new theme.** New command, new field on a
   core model, new skill, new architectural pattern — pause and `aida add`
   *before* the implementation commits. ~2 min cost; saves backfill later.
2. **Don't reuse one EPIC as a catchall.** When the work has drifted from
   what the EPIC was originally about, that's a signal to create a new EPIC,
   not stretch the existing one.
3. **Run `/aida-capture` at natural pauses.** End of focused work, before
   compaction, when stepping away. Five-minute pass that catches missed reqs.
4. **Yellow flag at >5 untracked commits.** Five+ feat/fix commits without a
   matching requirement → offer to capture before continuing.
5. **Trace comments must match reality.** A `// trace:EPIC-N` on code that
   has nothing to do with EPIC-N is misinformation that compounds. If you're
   unsure which spec a piece of work belongs to, that's the signal it needs
   its own.

## Glance at the statusbar

`.claude/settings.json` wires `aida statusline` into Claude Code's status
bar. It shows project · active role · queue depth · cache freshness. If the
role you expect isn't there, you forgot to `aida role enter <name>` before
starting the session.

## Git sync & review workflow

- **`aida pull` refusing (divergent branches)?** The code leg is
   `git pull --ff-only` (won't auto-rebase your tree); the store leg
   is `--rebase`. Recovery recipe + the one-time `git config` to make
   raw `git pull` Just Work: `docs/aida/discipline/git-sync-and-review.md`.
- **Reviewing a PR?** `aida review prompt --pr N` lifts each linked
   spec's `## Acceptance` into a review prompt. Needs `gh`/`glab` for
   `--pr` mode; write a `## Acceptance` section in every STORY/BUG so
   there's something to lift. Detail: same discipline doc.
<!-- AIDA-AUTOGEN-END -->


## Codex Operating Discipline

### Storage Model

AIDA's source of truth is the git-canonical spec store, not an ad hoc
notes file. Use MCP tools for spec graph and coordination operations
when available; use shell commands for build, test, git inspection, and
cross-surface verification.

### MCP Server Registration

`aida init` scaffolds `.codex/config.toml` with an `[mcp_servers.aida]`
block that registers AIDA's MCP server (`aida mcp-serve`) for this
project — the Codex-side parallel to the `.mcp.json` Claude Code uses. A
Codex session started from the project root therefore discovers AIDA's
tools out of the box; you do not need to run `codex mcp add aida -- aida
mcp-serve` by hand. If `aida` is not on `PATH`, edit the scaffolded
`command` to the absolute binary path. See
`docs/agents/codex-mcp-setup.md` for verification (`codex mcp list`,
`/mcp`).

### Requirements Management

Before implementing, make sure a requirement exists and read it with
`show_requirement` or `aida show <ID>`. If you file new requirements via
MCP, pass a valid lowercase `type`; AIDA derives the canonical ID prefix
from that type. Do not invent `SPEC-N` IDs.

### Daily-Use Commands

```bash
codex mcp add aida -- aida mcp-serve
aida show <SPEC-ID>
aida list --status approved
aida queue work <SPEC-ID>
aida pr ship
aida brief list --for-agent codex
aida brief ack .aida/agent-briefs/codex/<brief>.md
tests/test_mcp_stdio.sh --skip-agent-contract
tests/test_mcp_doc_consistency.sh
```

### Optional Status Lines

AIDA's bootstrap goal is to make other projects agent-ready without forcing a
house style. The AIDA-aware status line is therefore a convenience, not a
requirement: use `aida statusline --color=always` anywhere your shell, terminal
multiplexer, or agent client can run a command-backed status line.

Claude Code supports that directly through `.claude/settings.json`. Codex CLI
has its own built-in TUI footer, configured with `[tui].status_line` in Codex
`config.toml` or interactively with `/statusline`. Codex's current footer
accepts built-in item IDs; it does not run `aida statusline` as an arbitrary
command. For Codex, use the built-in footer fields as a lightweight companion
to the AIDA-aware shell/statusline command:

```toml
[tui]
status_line = ["model-with-reasoning", "context-remaining", "git-branch", "current-dir"]
```

Put that in `~/.codex/config.toml` for a personal default, or in a trusted
project's `.codex/config.toml` if the whole team wants the same footer.

The built-in footer fields cannot host AIDA's role / queue-depth / inbox-depth
segment, so for in-agent parity wire `aida statusline --title` into your shell
prompt: it emits the same one-liner wrapped in an OSC terminal-title escape, so
the AIDA segment rides the terminal title bar / tmux window name during the
Codex session — the in-agent analog of the command-backed footer Claude Code
runs in `.claude/settings.json`.

```bash
# bash (~/.bashrc)
PROMPT_COMMAND='aida statusline --title 2>/dev/null; '"$PROMPT_COMMAND"

# zsh (~/.zshrc)
precmd() { aida statusline --title 2>/dev/null }
```

Run `aida statusline setup --client codex` for the copy-paste version of both.

### MCP Coordination

Use AIDA MCP for substrate operations: `show_requirement`,
`list_active_leases`, `claim_task`, `release_task`, `file_finding`,
`post_punt`, `list_briefs`, `read_brief`, `ack_brief`, `add_comment`,
and directive tools. Trust MCP `tools/list` for argument names. Current
responses are text envelopes; parse defensively until structuredContent
ships.

For cross-agent communication semantics, especially Claude Code
`PreToolUse` / `PostToolUse`, `continue: false`, `ask`, and `defer`, use
`docs/agents/session-communication.md`. Do not assume a later hook can ask
whether to continue after an earlier hook has halted the run.

### Worktree And Session Discipline

Do implementation work in a sibling worktree. No `.aida-store` symlink is
needed — a sibling worktree resolves the canonical store at the main
worktree automatically (BUG-331). Do not edit another agent's dirty main
worktree. If a branch, lease, or worktree state looks inconsistent, stop
and surface it instead of forcing git.

### Direct Assignment: Implement BUG/TASK-N

When the operator says "implement BUG-N / TASK-N" and there is no queued
brief, follow this path (it's the same one used for TASK-132 and BUG-406):

1. `aida show <SPEC>` — read the spec, acceptance criteria, and any owning plan.
2. If it is Draft and the operator explicitly assigned it, promote it: `aida edit <SPEC> --status approved`.
3. Start an isolated session: `aida session start --owns <SPEC> --role implementer --base origin/main`.
4. Work in the sibling worktree — no `.aida-store` symlink (the store resolves automatically, per Worktree discipline above).
5. Implement; add `// trace:<SPEC> | ai:codex` comments; run targeted tests + `cargo fmt --all -- --check`.
6. Commit `[AI:codex] type(scope): description (<SPEC>)`.
7. `aida pr ship` — watches CI, squash-merges, pulls, and auto-bumps the spec to Completed.
8. End the session; verify the spec reached Completed.
9. Architecture-class work → sketch first and wait for master sign-off (see Sketch-First Protocol).

### Code Traceability

When code implements a spec, add a trace comment in the touched code:

```rust
// trace:TASK-123 | ai:codex
```

Keep spec IDs in developer artifacts: commits, PR titles, trace
comments, and plans. Do not leak internal IDs into user-facing CLI text
unless that output is explicitly developer/operator-facing.

### Commit And PR Format

Use the Codex prefix and put every shipped spec in trailing parens:

```text
[AI:codex] fix(scope): concise description (TASK-123)
[AI:codex] docs(agents): Codex setup integration (STORY-417 TASK-485 TASK-484)
```

The auto-bump scanner reads the trailing parens. If one PR closes
multiple specs, include every spec ID in that group.

### Sketch-First Protocol

Before opening a PR for architecture-class changes, post a sketch on the
owning spec and wait for master sign-off. Architecture-class means file
formats, MCP tool contracts, orchestrator semantics, lease model,
cross-cutting lifecycle vocabulary, or discipline/memory changes.
Bounded tests, docs refreshes, and acceptance-criteria implementation do
not need a sketch unless they introduce a reusable harness or new
project convention.

### Known Codex Pitfalls

- PR-201 missed the trailing spec trailer in the squash subject; that
  incident is why trailing-parens discipline is non-optional.
- Read the `aida pr ship` arc before relying on the wrapper in a new
  environment: SPEC-410, BUG-339, BUG-344, and BUG-345 document subject
  repair, parser alignment, CI startup waiting, and stale-main-worktree
  handling.
- `aida mcp-serve` self-respawns after handled requests when the on-disk
  `aida --version` reports a newer package version or different build
  SHA. If MCP still appears stale, kill that agent's server process and
  let the client respawn it.
- If an instruction from another session sounds inconsistent with the
  branch contents, verify the PR contents and flag the mismatch.
