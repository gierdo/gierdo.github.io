---
title: "AI: Agentic vim - local, cloudy, losely coupled?"
categories:
  - linux
  - programming
tags:
  - vim
  - ai
  - acp
  - agent
---

# AI: Agentic vim - local, cloudy, losely coupled?

I have been playing around with / working productively with _AI_ in vim for a
while now. For religious and practical reasons [with local llms]({% link
_posts/2024-03-02-llama-igpu-amd-container.md %}), but also with _AI agents_,
namely [`kiro-cli`](https://kiro.dev/cli/),
[`opencode`](https://opencode.ai/de) and [`gemini
cli`](https://geminicli.com/), integrated into my workflow and environment with
[`sidekick`](https://github.com/folke/sidekick.nvim).

Through `sidekick`, the different AI agents were integrated in a separate
buffer but with common and uniform keybindings, context integration (file
content, diagnostics, ..) etc.

Still, every AI agent has its own interaction model, UI details, look and feel
etc, and environment integration by `sidekick`, while impressive and mostly
surprisingly stable, was rather hacky.

For my personal environment, I've been using `gemini`. My employer only
provides `kiro-cli` and (limited) `opencode`, and I want to have an as-uniform
experience as possible. So, for a while I would have wanted to have an
abstraction over the cli agents in my environment.

Theoretically, that's what
[_ACP_](https://agentclientprotocol.com/get-started/introduction) is for.
However, _ACP_ support in `kiro-cli` has not been present from the start, and I
became rather familiar with the setup I had in the meantime.

Until I had a bit of time at my hands to touch the setup again last week.

## The Problem

- I use `nvim`
- I (have to) use different AI agents (`kiro-cli`, `opencode`, `gemini`)
- I want a uniform look and feel, configuration, capabilities, etc. in my
  `nvim` environment, no matter which AI agent powers my AI workflow
- In most cases, the environment integration is more important to me than the
  agent specific capabilities and advantages that might be lost with it,
  especially when it comes to agent autonomy and orchestration

## Possible solution

The Agent Client Protocol
[(_ACP_)](https://agentclientprotocol.com/get-started/introduction)
standardizes communication between code editors/IDEs and coding agents.

If all used agents support the _ACP_, a single _ACP_ client configuration could
provide the solution I want.

Sadly, when I started the fun with "my" main agent `kiro-cli`, it has not
supported _ACP_ yet.

## Solution

A while ago, `kiro-cli` added support for _ACP_ 🥳!

No more reason not to try it out.

After a bit of research, I decided to use
[_codecompanion_](https://codecompanion.olimorris.dev/) to replace `sidekick`
for the AI integration.

The full configuration can be found
[here](https://github.com/gierdo/dotfiles/blob/8992099423a5cbea84bc48d51918e50ea826955d/.config/nvim/lua/plugins/ai.lua),
but I will write about a few details of the configuration.

### Keybindings

I had the following `sidekick` keybindings:

`A-a` to open/toggle the agent window, including the selection as context, and
`A-i` to toggle with the predefined prompt catalogue.

```lua
  keys = {
    {
      "<A-a>",
      function()
        require("sidekick.cli").toggle({ filter = { installed = true } })
      end,
      desc = "Sidekick Toggle",
      mode = { "n", "t", "i" },
    },
    {
      "<A-a>",
      function()
        require("sidekick.cli").send({
          filter = { installed = true },
          msg = "The current file is {file}, this is the current selection: \n\n```\n{selection}\n```",
        })
      end,
      desc = "Sidekick Toggle with context",
      mode = { "x" },
    },
    {
      "<A-i>",
      function()
        require("sidekick.cli").prompt({ filter = { installed = true } })
      end,
      desc = "Sidekick Select Prompt",
      mode = { "n", "t", "i", "x" },
    },
  },
```

For `codecompanion`, the same can be achieved with these keybindings:

```lua
    keys = {
      {
        "<A-a>",
        "<cmd>CodeCompanionChat Toggle<cr>",
        desc = "CodeCompanion Toggle",
        mode = { "n", "t", "i", "x" },
      },
      {
        "<A-i>",
        "<cmd>CodeCompanionActions<cr>",
        desc = "CodeCompanion Actions",
        mode = { "n", "t", "i", "x" },
      },
    },
```

### Buffer guard

The `codecompanion` buffer behaves like a normal buffer. Which means that if I
jump in the jumplist while being in it, it would be replaced. Or if a custom /
weird LSP action is triggered. Or...

So, I wanted to protect the `codecompanion` buffer, as identified through its
virtual filetype.

```lua
init = function()
      -- If new buffers are opened or jumps being triggered while focus is on codecompanion, the action should be triggered outside of the codecompanion buffer
      vim.api.nvim_create_autocmd("FileType", {
        pattern = "codecompanion",
        callback = function(args)
          vim.bo[args.buf].buflisted = false
          local win = vim.fn.bufwinid(args.buf)
          if win == -1 then
            return
          end
          -- Guard this window: redirect any non-cc buffer that lands here
          vim.api.nvim_create_autocmd("BufWinEnter", {
            callback = function(ev)
              if not vim.api.nvim_win_is_valid(win) then
                return true
              end
              if vim.api.nvim_get_current_win() ~= win then
                return
              end
              if ev.buf == args.buf then
                return
              end
              -- Restore cc buffer in this window, move new buffer elsewhere
              vim.api.nvim_win_set_buf(win, args.buf)
              for _, w in ipairs(vim.api.nvim_tabpage_list_wins(0)) do
                if w ~= win and vim.bo[vim.api.nvim_win_get_buf(w)].filetype ~= "codecompanion" then
                  vim.api.nvim_win_set_buf(w, ev.buf)
                  vim.api.nvim_set_current_win(w)
                  return
                end
              end
              vim.cmd("split | buffer " .. ev.buf)
            end,
          })
        end,
      })
    end
```

## tl;dr

- _ACP_ is nice for Agent/Client integration
- setting up an work environment to use agent integration through ACP allows
  for the definition of common behaviour and user experience, even with
  different agents underneath it all
- nvim can be set up to use ACP with plugins, e.g. codecompanion
- ?
- profit
