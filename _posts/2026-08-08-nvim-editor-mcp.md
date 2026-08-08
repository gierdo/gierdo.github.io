---
title: "AI: Let your agent drive neovim"
categories:
  - programming
tags:
  - vim
  - ai
  - mcp
  - python
---

# AI: Let your agent drive neovim

<script src="https://asciinema.org/a/1262569.js" id="asciicast-1262569" async="true"></script>

In my [previous post about agentic vim]({% link
_posts/2026-04-23-agentic-vim.md %}), I wrote about how to integrate AI agents
into my `nvim` environment using the Agent Client Protocol (ACP) and
`codecompanion`. In that setup, `nvim` acts as the client, prompting the agent
and bringing the responses back into the editor buffers.

But what if we turn things around?

What if the AI agent is the driver, and we want to let it interact with our
running editor session? E.g., if I'm running an external agent CLI or a
pair-programming assistant, and I want it to be able to open a file at a
specific line, check LSP diagnostics to see if its changes compile, populate my
quickfix list, or send me notifications and let my entire vim flash by toggling
color schemes?

I didn't find a solution, so I built
[`nvim-editor-mcp`](https://github.com/gierdo/nvim-editor-mcp).

## The Problem

- External AI agents or CLI assistants operate blindly, isolated from my
  running editor session.
- When working on shared coding tasks, the agent cannot see what I see. It
  doesn't know my active buffer, cursor position, or the LSP diagnostics
  currently flashing on my screen.
- Instead of a shared, interactive coding session, the workflow becomes
  disjointed: the agent outputs file paths and line numbers in the terminal,
  and I have to manually navigate to them.
- I want a seamless, bi-directional partnership where the agent can interact
  directly with the Vim API to inspect my context in real time and guide me
  through the code.
- And, of course, I want this integration to be loose and lightweight, without
  forcing the entire agent runner to live inside a heavy `nvim` plugin.

## Solution

An MCP server that exposes `nvim` operations as tools, bridging the agent's
requests to `nvim` via RPC.

I created [`nvim-editor-mcp`](https://github.com/gierdo/nvim-editor-mcp). It is a lightweight Python server.

Building it was pretty straightforward, mainly because of two very useful
abstractions.

- **[`fastmcp`](https://github.com/jlonge4/fastmcp)** abstracts away the entire
  MCP boilerplate. It handles JSON-RPC serialization, protocol lifecycles, and
  tool schema generation. You define your tools with a simple `@mcp.tool`
  decorator, and `fastmcp` takes care of the rest.
- **[`pynvim`](https://github.com/neovim/pynvim)** provides the Python
  interface to `nvim`'s Msgpack-RPC API. Instead of worrying about socket-level
  message passing, it gives us a clean, high-level API to command the editor,
  run Lua code, and query buffer/diagnostic states directly.

Together, they make bridging the agent protocol and editor state a breeze.

### How it connects

`nvim` communicates over a Unix socket. When you open a terminal buffer inside
`nvim`, it automatically sets the `$NVIM` environment variable containing the
path to that socket.

`nvim-editor-mcp` is designed to be zero-config:

1. If started from inside `nvim` (e.g., via a terminal buffer or an integrated
   runner), it automatically reads `$NVIM`.
2. If started externally, it falls back to connecting to a specified socket,
   which can be identified by the agent

### The Tools

Once connected, the server exposes the following tools to the AI agent:

| Tool | Description |
| ------- | ------------- |
| `attach_nvim` | Connect to a `nvim` instance by socket path |
| `open_file` | Open a file at a specific line |
| `set_quickfix` | Populate the quickfix list |
| `get_diagnostics` | Fetch LSP diagnostics |
| `nvim_exec_lua` | Execute arbitrary Lua in `nvim` |
| `notify` | Show a notification in `nvim` |

Because we expose `nvim_exec_lua`, the agent has a powerful fallback to do
basically anything it wants inside our editor session. For example, to check
the path of the current buffer, the agent can call `nvim_exec_lua` with:

```lua
return vim.api.nvim_buf_get_name(0)
```

### Client Configuration

To hook it up to an MCP client like Claude Desktop or any other MCP-enabled AI
CLI runner, just add it to your configuration file:

```json
{
  "mcpServers": {
    "nvim-editor-mcp": {
      "command": "nvim-editor-mcp"
    }
  }
}
```

(Ensure `nvim-editor-mcp` is on your `$PATH`, which you can do easily by
installing it with `uv tool install
git+https://github.com/gierdo/nvim-editor-mcp.git`).

## tl;dr

- Exposing local development environments to AI agents via MCP enables
  real-time, interactive pair-programming sessions between you and your agent.
- `nvim-editor-mcp` lets external agents drive your editor and see what you see
  by querying the `nvim` API directly (buffers, cursors, diagnostics).
- `fastmcp` and `pynvim` act as clean abstractions, making it simple to bridge
  the Model Context Protocol to `nvim`'s RPC interface.
- Zero-config connection discovery over local Unix sockets handles the plumbing
  automatically.
