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
neovim]({% link _posts/2026-08-08-nvim-editor-mcp.md %}), I flipped the
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

To make this practical, the setup needed four things:

1. A terminal environment built on persistent sessions and multi-client flexibility,
   where terminal windows are decoupled views into long-running tasks.
2. Active agent lifecycle awareness and programmatic orchestration for multi-agent
   coordination.
3. A single source of truth for agent skills and steering instructions,
   independent of whatever agent harness is used (Gemini, Codex, OpenCode, Kiro
   CLI).
4. Seamless, decoupled editor interaction so agents running across different
   workspaces can still interact with the right Neovim instance without
   cross-talk.

Here is how I tied this together using `herdr`, `chezmoi`, and a Herdr-native
integration of `nvim-editor-mcp`.

## The Problem

- **Ephemeral sessions and coupled terminal views**: In my previous setup,
  terminal emulators (`foot`, `kitty`) launched `tmux` directly. While
  session persistence is not a fundamental limitation of `tmux` per se, the
  way it was configured treated each terminal window as an ephemeral, isolated
  session. Closing a window terminated running processes, creating constant
  friction when running long-lived agentic tasks or test suites. Furthermore,
  attaching multiple terminals to a single `tmux` session mirrors the active
  view across all windows by default, making multi-monitor workflows clumsy.
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

### 1. Workflow evolution: persistent sessions and multi-terminal attach

The first step was rethinking the terminal session model itself.

In my previous workflow, terminal emulators were configured with `shell=tmux`.
While `tmux` has full support for persistent server sessions and detachment, in
practice my terminal configuration treated sessions as non-persistent and
ephemeral: opening a terminal window launched a standalone session, and closing
the window tore it down. When running autonomous agents that execute multi-step
refactorings or run test loops that take significant time, this model breaks down.
An accidental keystroke or window close terminates the agent mid-flight.

[`herdr`](https://herdr.dev/) fundamentally changes this by placing full
persistency at the core of its architecture:

- **Client-server separation with automatic attach**: Herdr runs as a persistent
  background daemon (`herdr.sock`). In my terminal configs (`foot.ini` and
  `kitty.conf`), the shell is set to `shell=herdr`. Launching a new terminal
  window automatically attaches to the running Herdr session instead of
  spawning a throwaway environment. If the server is not running yet, it starts
  seamlessly in the background.
- **Decoupled multi-terminal views**: With standard `tmux`, having multiple
  terminal windows attached to the same session causes them to mirror each other:
  switching windows or scrolling in one terminal forces the same view onto the
  other. Herdr natively decouples attached clients: **different terminal
  windows can display different tabs or workspaces simultaneously**. I can have
  one terminal window on an auxiliary monitor monitoring an agent working in
  Workspace 1, while another terminal window on my main display is actively
  navigating code in Workspace 2 or viewing a separate tab.
- **True persistence across disconnections**: Closing a terminal window (or
  closing all of them) has zero impact on running processes. Background agent
  loops, compilation jobs, and Neovim buffers remain untouched and active on the
  Herdr daemon. Opening a new terminal window immediately brings back the live
  state.

#### Preserving muscle memory: tmux-compatible keybindings

A major risk when switching multiplexers is fighting ingrained muscle memory.
To make the transition seamless, Herdr's keybindings in
`dot_config/herdr/config.toml` are configured to maintain maximum compatibility
with the previous `tmux` setup:

- **Prefix chord**: The prefix remains `ctrl+b`, keeping `ctrl+h/j/k/l` free for
  direct navigation.
- **Splits**: Splitting follows standard tmux bindings - `prefix+%` for vertical
  splits and `prefix+"` for horizontal splits - combined with `new_cwd = "follow"`
  so newly split panes automatically inherit the current working directory
  (matching tmux's `-c "#{pane_current_path}"`).
- **Tabs**: `prefix+c` creates a new tab, while `prefix+n` and `prefix+p` cycle
  through tabs.
- **Workspaces**: Workspace navigation uses fast, direct chords:

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

#### Seamless split and tab navigation in conjunction with Neovim

In my tmux setup, [`smart-splits.nvim`](https://github.com/mrjones2014/smart-splits.nvim)
allowed fluid directional navigation between tmux panes and Neovim splits.
Losing that borderless movement was not an option.

In Herdr, this integration is handled by
[`herdr-splits.nvim`](https://github.com/lmilojevicc/herdr-splits.nvim), paired
with the `herdr-splits` plugin declared in chezmoi's `.chezmoidata/packages.toml`:

```toml
[packages.herdr]
plugins = [
  "jeffarese/herdr-bar",
  "lmilojevicc/herdr-splits.nvim",
  "njs14/herdr-kiro-pulse",
]
```

This pairs a Neovim plugin with a Herdr multiplexer plugin to achieve seamless,
two-way navigation and resizing:

- **Directional navigation (`ctrl+h/j/k/l`)**: When inside Neovim, pressing
  `ctrl+h/j/k/l` navigates between internal Vim splits. When the cursor reaches
  the outermost edge of Neovim, the plugin delegates directly to Herdr
  (`herdr pane focus --direction <dir>`), stepping into the adjacent terminal
  pane without any mode switch. Conversely, pressing `ctrl+h/j/k/l` in a terminal
  pane moves between Herdr panes or steps right back into Neovim splits.
- **Seamless resizing (`alt+h/j/k/l`)**: Resizing works identically across the
  boundary, resizing Neovim windows natively when multiple splits exist, or
  resizing the Herdr terminal pane when at the edge.

Combined with tab navigation (`prefix+n`, `prefix+p`) and workspace switching,
the terminal multiplexer and Neovim behave as a single, coherent canvas.

### 2. Multi-agent orchestration and lifecycle tracking

Beyond persistency and navigation, the primary motivation for switching to Herdr
was its agent-native architecture.

Classic multiplexers treat panes as dumb byte streams. They cannot distinguish
an idle shell prompt from an agent writing code or a process blocked on user
approval. `herdr` organizes terminals into workspaces, tabs, and panes, but with
a critical difference: it recognizes coding agents running inside panes and
tracks their lifecycle states (`idle`, `working`, `blocked`, `done`).

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
  prompts it, and waits for a settled state before advancing the workflow.

This turns the terminal multiplexer into an active coordination layer rather
than just a passive window splitter.

### 3. Harness-agnostic skills and instructions with chezmoi

 Having the skills and instructions in the dotfiles repository allows me to
 develop and track them version-controlled and independent from my work
 environment.

#### Why agent-independent?

Why bother building an agent-independent setup rather than standardizing on a
single agent harness?

Because I have different harnesses in different environments with different
conditions. At work, enterprise environments and access policies require
`kiro-cli` and `codex`. For personal use and open exploration, I prefer
`antigravity` (Gemini) and `opencode`. Not necessarily because they are better,
but I have economically interesting access 😉.

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

### 4. Decoupled editor interaction: nvim-editor-mcp meets Herdr

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

- **Persistent session workflow**: Moving from an ephemeral, window-tied tmux
  setup to Herdr makes persistence foundational. New terminals automatically
  attach to the running daemon, while different windows can independently show
  different workspaces and tabs without fighting over active views.
- **Muscle memory parity**: Keybindings stay fully compatible with tmux
  conventions (`ctrl+b`, splits, tabs), while `herdr-splits.nvim` provides
  seamless directional navigation (`ctrl+h/j/k/l`) and resizing (`alt+h/j/k/l`)
  across Neovim and Herdr panes.
- **Agent lifecycle orchestration**: `herdr` acts as the coordination backbone:
  managing workspaces, tracking agent states (`idle`, `working`, `blocked`,
  `done`), and allowing agents to orchestrate sibling panes programmatically.
- **Single source of truth**: `chezmoi` manages global instructions (`AGENTS.md`)
  and reusable skills in one place, automatically distributing them across
  Gemini, Codex, OpenCode, and Kiro CLI.
- **Decoupled editor interaction**: `nvim-editor-mcp` pairs with Herdr's session
  topology to bind agents to the Neovim instance in their current workspace and
  directory, keeping editor interactions conflict-free.
