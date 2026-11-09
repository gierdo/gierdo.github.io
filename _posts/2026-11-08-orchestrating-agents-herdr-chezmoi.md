---
title: "AI: Orchestrating agents with herdr, chezmoi, and nvim"
categories:
  - programming
  - linux
tags:
  - ai
  - herdr
  - chezmoi
  - vim
  - mcp
---

In my [earlier post on agentic vim]({% link _posts/2026-04-23-agentic-vim.md
%}), I explored integrating AI agents directly into Neovim using the Agent
Client Protocol (ACP) and `codecompanion`. Then, in [let your agent drive
neovim]({% link_posts/2026-08-08-nvim-editor-mcp.md %}), I flipped the
direction: using [`nvim-editor-mcp`](https://github.com/gierdo/nvim-editor-mcp)
to let external agents drive my running editor session over RPC, inspecting
buffers, populating quickfix lists, and reading diagnostics. Combined with [git
worktrees]({% link _posts/2026-07-22-git-worktrees.md %}), this provided a
solid foundation for parallel development branches.

However, as agentic workflows evolve from quick inline queries into autonomous,
multi-step development tasks - implementing full features, chasing down tricky
bug fixes, or running complex refactorings with test suites - running
everything inside a single editor window or terminal tab quickly hits limits.

Real-world coding tasks can take hours, or at least near hours. They involve
multiple agents acting in concert: a lead agent planning architecture,
sub-agents writing code in isolated git worktrees, and background jobs running
builds and test suites.

To make this practical, the setup needed three things:

1. A terminal environment built for persistent, long-running agentic tasks and
   multi-agent coordination.
2. A single source of truth for agent skills and steering instructions,
   independent of whatever agent harness is used (Gemini, Codex, OpenCode, Kiro
   CLI).
3. Seamless, decoupled editor interaction so agents running across different
   workspaces can still interact with the right Neovim instance without
   cross-talk.

Here is how I tied this together using `herdr`, `chezmoi`, and a Herdr-native
integration of `nvim-editor-mcp`.

## The Problem

- **Multiplexer blindness**: Classic multiplexers like `tmux` treat panes as
  dumb byte streams. They cannot tell whether a process in a pane is an idle
  shell, an agent actively writing code, or an agent blocked waiting for user
  approval.
- **Harness fragmentation**: Depending on context (using `kiro` and `codex` at
  work, `antigravity` and `opencode` for personal use), I have to use different
  harnesses in different environments with different conditions. Each expects
  its own configuration directory, custom instruction filenames, and skill
  locations. Maintaining separate copies of system prompts and skills across
  all of them does not scale.
- **Ambiguous editor targeting**: I want agents to be able to interact with my
  editor and vice versa. In a setup with multiple workspaces and multiple
  Neovim instances open, an external agent running in a background pane cannot
  simply connect to an arbitrary `/tmp/nvim.sock` without risking mutating the
  wrong buffer in the wrong repository.

## Solution

### 1. Replacing tmux with Herdr for agent orchestration

The first change was replacing `tmux` with [`herdr`](https://herdr.dev/) across
my environment (including switching my terminal configs to launch `herdr`
directly).

`herdr` is a terminal multiplexer designed specifically for coding agents. It
organizes terminals into workspaces, tabs, and panes, but with a critical
difference: it recognizes coding agents running inside panes and tracks their
lifecycle states (`idle`, `working`, `blocked`, `done`).

Crucially, agents are not left to guess how to drive the multiplexer via ad-hoc
shell commands. The intended interaction with Herdr is provided to agents
through a tailored api and dedicated skills and instructions embedded directly
within higher-level skills:

- **The `herdr` skill**: A foundational skill that teaches agents how to verify
  their Herdr environment (`HERDR_ENV=1`), resolve caller context
  (`HERDR_WORKSPACE_ID`, `HERDR_TAB_ID`, `HERDR_PANE_ID`), manage layout
  without stealing user focus (`--no-focus`), and safely coordinate agents or
  inspect pane output through structured JSON responses.
- **Skill-driven workflow orchestration**: Higher-level skills compose Herdr
  from within skills. For example, my `workflow-executor` skill defines
  declarative multi-step workflows for feature implementation, bug fixes, and
  refactorings. Rather than running everything sequentially in a single
  process, the workflow orchestrator uses Herdr to delegate sub-tasks: it
  splits a sibling pane, launches a specialized worker or reviewer agent,
  prompts it, and waits for a settled state before advancing the workflow:

Through chezmoi, `herdr` configuration and keybindings are version-controlled
in `dot_config/herdr/config.toml`:

```toml
# --- workspace navigation & management ---
switch_workspace = "prefix+shift+1..9"
previous_workspace = "prefix+["
next_workspace = "prefix+]"

workspace_picker = "prefix+w"
new_workspace = "prefix+shift+n"
rename_workspace = "prefix+shift+w"
close_workspace = "prefix+shift+d"
```

Herdr plugins are declared in `.chezmoidata/packages.toml` and installed
automatically via a chezmoi run script
(`run_onchange_after_09_herdr_plugins.sh.tmpl`):

```toml
[packages.herdr]
plugins = [
  "jeffarese/herdr-bar",
  "lmilojevicc/herdr-splits.nvim",
  "njs14/herdr-kiro-pulse",
]
```

This turns the terminal multiplexer into an active coordination layer rather
than just a passive window splitter.

### 2. Harness-agnostic skills and instructions with chezmoi

 Having the skills and instructions in the dotfiles repository allows me to
 develop and track them version-controlled and independent from my work
 environment.

#### Why agent-independent?

Why bother building an agent-independent setup rather than standardizing on a
single agent harness?

Because I have different harnesses in different environments with different
conditions. At work, enterprise environments and access policies require
`kiro-cli` and `codex`. For personal use and open exploration, I prefer
`antigravity` (Gemini) and `opencode`.

Each harness expects its own configuration folder, distinct system instruction
filenames, and separate skill locations. But my core workflows - how git
worktrees are spun up, how code reviews run, how refactorings are verified -
should remain completely identical regardless of the underlying harness.
Maintaining divergent copies across four different CLI tools is a maintenance
dead end.

Rather than maintaining duplicate skill folders and system prompts for each
agent CLI, everything is centralized in chezmoi and synced on change.

#### Global Instructions: AGENTS.md

A single `AGENTS.md` in the root of the chezmoi repository defines universal
conduct, verification requirements, safety checks, and tool preferences (such
as using `serena` for code navigation and `nvim-editor-mcp` for Neovim
interaction).

A chezmoi `run_onchange_after` script monitors the SHA-256 hash of `AGENTS.md`
and copies it to the expected paths of all installed harnesses:

```bash
#!/usr/bin/env bash
set -euo pipefail

# AGENTS.md hash: {{ include (joinPath .chezmoi.sourceDir "AGENTS.md") | sha256sum }}

SRC="{{ .chezmoi.sourceDir }}/AGENTS.md"

TARGETS=(
  "${HOME}/.codex/AGENTS.md"
  "${HOME}/.config/opencode/AGENTS.md"
  "${HOME}/.gemini/GEMINI.md"
  "${HOME}/.kiro/steering/00-global.md"
)

for target in "${TARGETS[@]}"; do
  mkdir -p "$(dirname "$target")"
  cp "$SRC" "$target"
done
```

Whether Gemini, Codex, OpenCode, or Kiro CLI is launched, every agent
immediately inherits the exact same baseline behavior and safety constraints.

#### Reusable skills

The same pattern applies to agent skills. In `~/.local/share/chezmoi/skills/`,
individual skills are defined with standard `SKILL.md` frontmatter and
instructions. Amongst others, they are:

- `sw-engineer`: Pragmatic software engineering principles (lazy senior
  developer mindset, verification before claiming completion).
- `using-git-worktrees`: Managing isolated worktrees for feature branches.
- `workflow-executor`: Structured state machines for bug fixing, feature
  implementation, and code review.
- `glab` / `glab-stack`: Safe GitLab interaction and stacked merge requests.
- `herdr`: Direct control of Herdr panes and agent coordination.

A matching chezmoi script hashes all skill files and mirrors them to every
harness:

```bash
TARGET_DIRS=(
  "${HOME}/.gemini/config/skills"
  "${HOME}/.codex/skills"
  "${HOME}/.config/opencode/skills"
  "${HOME}/.kiro/skills"
)

for target in "${TARGET_DIRS[@]}"; do
  mkdir -p "$target"
  for skill_dir in "${SKILLS_SRC}"/*/; do
    [ -d "$skill_dir" ] || continue
    skill_name="$(basename "$skill_dir")"
    rm -rf "${target:?}/${skill_name}"
    cp -r "$skill_dir" "${target}/${skill_name}"
  done
done
```

Now, updating or adding a skill in one place automatically equips all agent
runners across the system.

### 3. Decoupled editor interaction: nvim-editor-mcp meets Herdr

#### Why nvim-editor-mcp integration?

Why integrate an external agent directly with Neovim via MCP instead of having
it just edit files on disk or output shell commands?

I want agents to be able to see what I see in my favorite environment to
interact with code, and vice versa.

When a reviewer agent finishes inspecting a git branch, I do not want it to
dump forty lines of terminal output with file paths and line numbers that I
have to manually click through. I want to tell the agent: *"add the findings of
the review to the quickfix window in vim"*, and immediately navigate through
them in my running editor. Similarly, if I have compiler errors or diagnostics
flashing in my editor buffer, I want the agent to read those exact LSP
diagnostics directly rather than guessing.

In [AI: Let your agent drive neovim]({% link
_posts/2026-08-08-nvim-editor-mcp.md %}), I showed how `nvim-editor-mcp`
connects an agent to Neovim over Msgpack-RPC. But in a multi-workspace setup
managed by Herdr, there might be three different Neovim instances open across
separate workspaces and worktrees.

How does an agent running in a Herdr pane know which editor to talk to?

`nvim-editor-mcp` now features native Herdr topology awareness. When started,
it queries the active Herdr session (`$HERDR_SOCKET_PATH`) and correlates each
discovered Neovim socket with its Herdr metadata:

- Calling context: `$HERDR_WORKSPACE_ID`, `$HERDR_TAB_ID`, `$HERDR_PANE_ID`.
- Editor context: Neovim's current working directory, open buffers, and Herdr
  location.

When multiple editor instances exist, `nvim-editor-mcp` automatically resolves
the target:

1. It scopes candidates to Neovim instances inside the **same Herdr
   workspace**.
2. It selects the instance sharing the **same working directory (`cwd`)**,
   breaking ties with the current Herdr tab.
3. If ambiguity remains, the agent can call `list_nvim_instances` to inspect
   the candidate list (with `[recommended]` indicators) and explicitly
   `attach_nvim(socket=...)`.

This also unlocks a clean handoff pattern inside Neovim itself. In
`codecompanion`, when a task is recognized as long-running (such as running a
full test suite or a complex multi-file rewrite), the assistant can hand off
execution to a dedicated background pane in Herdr using the `herdr` skill:

```markdown
- Detecting herdr: inside CodeCompanion, derive the pane id from surviving
  env vars (HERDR_WORKSPACE_ID, HERDR_SOCKET_PATH) matching CodeCompanion.nvim.
- If you run jobs expected to run longer, hand them off to a sub-agent
  in a sibling Herdr pane.
```

The background agent runs independently in its own pane, while
`nvim-editor-mcp` allows it to push file changes, quickfix diagnostics, or
notifications back into the user's active editor when finished.

## tl;dr

- Long-running, multi-agent development tasks need persistent multiplexing and
  process lifecycle awareness that traditional tools like `tmux` do not
  provide.
- `herdr` acts as the coordination backbone: managing workspaces, tracking
  agent states (`idle`, `working`, `blocked`, `done`), and allowing agents to
  orchestrate sibling panes programmatically.
- `chezmoi` provides single-source-of-truth management for agent
  configurations, automatically distributing `AGENTS.md` and skills across
  Gemini, Codex, OpenCode, and Kiro CLI.
- `nvim-editor-mcp` pairs with Herdr's session topology to automatically bind
  agents to the Neovim instance in their current workspace and directory,
  keeping editor interactions decoupled and conflict-free.
