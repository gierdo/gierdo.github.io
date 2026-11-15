---
title: "AI: Pi - The extensible agent harness"
categories:
  - programming
  - linux
tags:
  - ai
  - pi
  - chezmoi
  - vim
  - mcp
  - herdr
---

In my previous posts on [orchestrating agents with herdr, chezmoi, and nvim]({% link
_posts/2026-11-08-orchestrating-agents-herdr-chezmoi.md %}) and [letting your
agent drive neovim]({% link _posts/2026-08-08-nvim-editor-mcp.md %}), I focused
on the coordination layer: how persistent terminal sessions, workspace-aware
RPC, and centralized dotfiles turn separate tools into a coherent multi-agent
environment.

However, one central piece of that setup deserves a closer look: the agent
harness itself.

Most commercial agent CLIs (Claude Code, OpenAI Codex CLI, OpenCode, Kiro CLI)
resemble modern full-fledged IDEs. They arrive with heavy built-in assumptions:
rigid planning loops, opinionated multi-agent abstractions that spawn background
processes you cannot see or control, and proprietary authentication systems
tied to a single vendor.

For my daily workflow, I chose a different path: **[Pi](https://github.com/earendil-works/pi)**
(`@earendil-works/pi-coding-agent`).

Here is why Pi has become my primary harness, how its extensibility model
differs from conventional agent tools, and how the entire configuration is
managed reproducibly with `chezmoi`.

## (N)vim vs. IDEs <-> Pi vs. Monolithic Harnesses

The best way to understand Pi is through an analogy that will feel familiar to
terminal users:

| Conventional / Heavyweight Model                 | Composable / Barebones Model                    |
| ------------------------------------------------ | ----------------------------------------------- |
| **Full-fledged IDE & Forks** (VS Code, Kiro IDE) | **Modular Editor** (Neovim, Vim)                |
| Monolithic features out of the box               | Bare-bones core focused on text editing         |
| Heavy extension host or maintaining a full fork  | Straightforward customization via Lua and RPC   |
| **Monolithic Harness** (Codex, Claude Code)      | **Extensible Harness** (Pi)                     |
| Hardcoded subagents and opinionated loops        | Bare-bones core focused on the agent loop       |
| Heavyweight integration or rigid wrappers        | Straightforward TypeScript, npm, and git        |

To be fair, existing tools can also be extended. VS Code has an enormous
extension marketplace, and when deeper integration is desired, vendors
frequently fork the entire editor codebase (as seen in AI-centric IDEs like
Kiro IDE and others). But there is a fundamental difference in architecture and
overhead. Extending a monolithic IDE involves significant ceremony: isolated
extension-host processes, complex manifest schemas, webview IPC boundaries, and
heavy dependency packaging. When that becomes too restrictive, maintaining a
multi-million-line IDE fork is an immense engineering tax.

A main differentiator of composable systems like Neovim and Pi is the inherent
simplicity of a barebones core explicitly designed from the start to target
customization. Rather than shipping a sprawling feature set and tacking on an
extension API as an afterthought, the core remains minimal. This makes the
extensions, adaptation and configuration themselves dramatically more
straightforward, transparent, and manageable.

This architectural divergence was recently analyzed in depth by Barbaste et al.
in [*Harness Engineering: Anatomy, Architecture, and Evolution of Coding Agents

- A Source-Code Study of Eleven Systems*
(arXiv:2609.00006)](<https://arxiv.org/abs/2609.00006>). Analyzing the source
code of eleven production agent harnesses (including Claude Code, Codex CLI,
OpenCode, Aider, and Pi itself), the authors show how the agent harness has
evolved from a superficial API wrapper into an independent runtime platform
where architectural decisions - tool governance, loop mechanics, and extension
boundaries - matter as much as the underlying foundation model.

Out of the box, Pi does not provide built-in multi-agent workflows, autonomous
plan modes, or anything much really. Instead, it provides:

- A fast, reliable prompt-eval loop with native streaming and token budgeting.
- A clean terminal UI with interactive and non-interactive (print/JSON/RPC) modes.
- Core primitives for tools (`read`, `bash`, `edit`, `write`), context
  management, and session persistence.
- A first-class extension architecture with zero artificial barriers between
  "built-in" and "user-defined" behavior.

Because Pi does not try to be an all-knowing IDE for agents, it leaves room to
build the exact orchestration architecture you want. You shape the harness
around your existing Unix workflow rather than altering your workflow to fit the
harness.

## Flexible Vendor Support: Antigravity, Kiro, and Beyond

A constant pain point with coding agent tools is vendor lock-in. A harness tied
exclusively to one API or authentication model forces you to switch tools
whenever your working context changes.

In my case, enterprise environments require `kiro` (AWS Builder ID and IAM
Identity Center), while personal projects rely on `antigravity` (Google Cloud
Code Assist / Gemini) and various open endpoints.

Pi solves this cleanly through its generic provider architecture. Rather than
hardcoding API clients, Pi allows extensions to register custom providers via
`pi.registerProvider()`. A provider extension owns authentication, token
refresh, OAuth callbacks, dynamic model discovery, and streaming protocols.

Thanks to an active community ecosystem, this means:

- **`pi-antigravity`**: Connects Pi directly to Google Antigravity / Cloud Code
  Assist. It handles browser-based Google OAuth login, token rotation, and model
  routing (including Gemini 3.8 Flash, Gemini 4 Argon, and Claude models hosted
  on Antigravity) without ever shelling out to external CLIs.
- **`pi-provider-kiro` / `pi-kiro`**: Connects Pi to the Kiro API (AWS
  CodeWhisperer/Q infrastructure). It supports AWS Builder ID, IAM Identity
  Center OAuth, or reusing active session credentials from a local `kiro-cli`
  installation, exposing Kiro's verified Claude Opus and Sonnet models inside Pi.
- **Native endpoints & Gateways**: Any standard OpenAI, Anthropic, or local
  Ollama endpoint can be plugged in via `models.json` without writing a single
  line of code.

Having one consistent terminal interface, behavior set, extensions,
integrations and keybinding set across diverse backend providers eliminates the
mental tax of context switching.

## Cat

This has been very dry. To brighten and lighten things up a little, here's an
exageratedly cute cat:

![Resources](/assets/images/2026-11-14-pi/cat.jpeg)

## Straightforward Extensibility

Many tools claim extensibility, but in practice they tend to fall into two
extremes:

1. **Shallow hooks**: Agent CLIs often limit customization to passing system
   prompt markdown or registering external MCP servers over stdio. While useful
   for exposing tools, this does not let you customize loop mechanics, intercept
   permissions, or control UI rendering.
2. **Heavyweight IDE ecosystems and forks**: At the other extreme, platforms
   relying on VS Code extensions or full IDE forks (like Kiro IDE) require
   navigating complex extension-host IPC, elaborate contribution manifests,
   and heavy packaging pipelines. The extensions themselves quickly turn into
   complex, high-maintenance projects.

The differentiator for Pi is the inherent simplicity of a barebones system
engineered specifically for customization. Because the core does not impose
layers of defensive abstraction, the extensions themselves are remarkably
straightforward and manageable:

- **In-process execution**: Extensions are plain TypeScript modules that run
  directly in-process with zero IPC overhead. They have first-class access to
  the agent lifecycle: inspecting events, registering tools and custom
  providers, controlling the terminal widget layer, and intercepting execution
  decisions.
- **Minimal ceremony**: There is no boilerplate, no manifest generation, and
  no separate host runtime. An extension can be a single TypeScript file in
  your dotfiles, a local script, an npm package, or a git repository.
- **Manageable and auditable**: Because extensions remain small, direct, and
  transparent, you can (relatively) easily inspect, debug, and maintain them.

### 1. Harness-level nvim-editor-mcp integration

In my [earlier post]({% link _posts/2026-08-08-nvim-editor-mcp.md %}), I
described how `nvim-editor-mcp` allows agents to drive Neovim over Msgpack-RPC.

Instead of declaring an MCP server, Pi allows installing `nvim-editor-mcp`
directly as a harness package from git:

```json
"packages": [
  "git:https://github.com/gierdo/nvim-editor-mcp"
]
```

Installed this way, Neovim tools (`open_file`, `set_quickfix`,
`get_diagnostics`, `nvim_exec_lua`) become native capabilities of the harness.
The mcp server's capabilities and behaviour sets become configurable and hooked
into the harness, enabling e.g. automatic connection to the most suitable nvim
instance on startup, automatic synchronisation of editor content with edits
through the agent and more.

### 2. Fully asynchronous multi-agent orchestration with Herdr

Some tools handle multi-agent tasks by spawning hidden background threads or
running subagents inside opaque Docker containers. You cannot see what they are
doing, you cannot step into their shell.

By pairing Pi with `pi-herdr-subagents`, subagent execution is delegated to
real, visible terminal panes in `herdr`:

- **Non-blocking delegation**: When the orchestrator calls `subagent()`, the
  call returns immediately. The subagent launches in a dedicated Herdr pane or
  tab, leaving the main Pi session responsive.
- **Parallel execution**: Multiple subagents can be dispatched concurrently (for
  example, two scouts exploring different modules) with simple concurrent calls.
- **Live TUI widget**: Above the input prompt, Pi renders a real-time status bar
  tracking each background agent:

```
╭─ Subagents ──────────────────── 1 active · 1 open ─╮
│ 00:23  Scout: Auth (scout)        active · bash 7m │
│ 00:45  Scout: DB (scout)                waiting 2m │
╰────────────────────────────────────────────────────╯
```

- **Automatic result steering**: When a background subagent finishes its task,
  its final summary is steered back into the parent Pi session as an async event,
  waking the parent agent and triggering the next step of the workflow.

### 3. Permission inheritance: solving background agent stalls

A common frustration in autonomous workflows is the permission prompt. A
security-conscious setup should not allow agents to run arbitrary bash commands
or overwrite sensitive files without confirmation. However, if a subagent
running a test suite in a background Herdr pane pauses every few minutes to ask
for confirmation, autonomous execution is impossible.

I solve this with Pi through `@gotgenes/pi-permission-system` paired with a
custom authorizer link: `subagent-inherit-manual.ts`.

The extension `subagent-inherit-manual.ts` listens to the
`permissions:decision` bus in the parent session. When a human manually approves
an action (for example, approving `git status *` or running tests in a project
directory), the extension records that approved pattern.

When a background subagent forwards a permission ask, the link checks whether
the pattern was already manually approved in this session. If it was, the
request is automatically granted (`allow`). If it represents an unseen or
risky operation, it defers to prompt the user.

Background tasks proceed sort of unhindered, without loosing control.

### 4. Context management with context-mode

Running builds, test suites, or large searches can produce thousands of lines of
terminal output. Dumping that raw data into an LLM context window quickly exhausts
token limits and degrades reasoning quality.

Pi integrates `context-mode`, providing sandboxed subprocess execution
(`ctx_execute`, `ctx_batch`). Instead of returning raw stdout, commands execute
in a local sandbox, return concise answers, and index the full output for on-demand
retrieval.

## Managing the Pi Setup with Chezmoi

A modular harness is only as good as the control over its setup. Every aspect
of my Pi configuration is tracked in my dotfiles repository.

### 1. Automated installation and bootstrapping

In `~/.local/share/chezmoi/.chezmoiscripts/run_onchange_after_07_custom_builds.sh.tmpl`,
Pi is bootstrapped automatically if not already present:

```bash
if ! command -v pi >/dev/null 2>&1; then
  curl -fsSL https://pi.dev/install.sh | sh || true
fi
```

Pi manages its runtime under `~/.pi/agent/install/releases/`, isolating node
dependencies while exposing a clean launcher binary at `~/.pi/agent/bin/pi`.

### 2. Package declarations in packages.toml

Rather than installing extensions manually, all packages and themes are declared
in `.chezmoidata/packages.toml`:

```toml
[packages.pi]
extensions = [
  "npm:context-mode",
  "npm:pi-web-access",
  "npm:pi-herdr-subagents",
  "npm:@gotgenes/pi-permission-system",
  "git:https://github.com/hasit/pi-community-themes",
  "git:https://github.com/gierdo/nvim-editor-mcp",
]
```

A chezmoi run script (`run_onchange_after_09_tool_plugins.sh.tmpl`) iterates
over these declarations and installs them via `pi install "$extension"`.

This doesn't remove extensions again after removing them from the
`packages.toml`, I will probably look into a more capable solution one day.

### 3. Declarative agent personas

Specialized subagents are declared as markdown templates under
`~/.local/share/chezmoi/dot_pi/agent/agents/`:

- **`scout.md`**: Fast, read-only reconnaissance (`read`, `bash`). Maps
  existing patterns and generates a structured `scout-context.md`.
- **`planner.md`**: Interactive design specialist. Clarifies requirements,
  explores approaches, and outputs task breakdowns. Enforces a strict one-question-per-turn
  gate and defers to OpenSpec whenever an `openspec/` directory exists.
- **`worker.md`**: Implementation specialist (`read`, `bash`, `write`, `edit`).
  Executes tasks inside isolated git worktrees, adheres to lazy-senior engineering
  principles, and verifies test suites before exiting.
- **`reviewer.md`**: Read-only review agent. Compares diffs against specifications
  and delivers findings straight into Neovim's quickfix list.

### 4. Shared instructions, skills, and MCP tools

Chezmoi keeps Pi aligned with other tools on the machine:

- **Instructions**: `run_onchange_after_11_agent_instructions.sh.tmpl` copies
  the root `AGENTS.md` directly into `${HOME}/.pi/agent/AGENTS.md`.
- **Skills**: `run_onchange_after_10_agent_skills.sh.tmpl` mirrors shared skills
  (`software-engineering`, `using-git-worktrees`, `glab`, `workflow-executor`)
  into `${HOME}/.pi/agent/skills/`.

## tl;dr

Pi is nice, if you like building your development environment out of sharp,
focused tools that compose well over one-size-fits-all suites.

- **Inherent simplicity**: A lean, barebones core explicitly designed for
  customization, avoiding both monolithic feature bloat and the maintenance
  overhead of full IDE forks.
- **Straightforward, manageable extensions**: In-process TypeScript modules that
  remain small and easy to write, audit, and evolve, without the ceremony and
  accidental complexity of IDE extension platforms.
- **Vendor agility**: Seamless support for diverse models and authentication
  schemes, from Google Antigravity to enterprise Kiro.
- **Deep extensibility**: Full lifecycle access enabling genuine integrations
