---
title: Amazon Q in vim
categories:
  - programming
  - linux
tags:
  - vim
  - amazon
  - aws
  - ai
  - Q
---

# Amazon Q - Assistant in vim

## The Problem

My employer is a typical German large scale corporation. One of the attributes
that come with that is that we are, of course, interested in generative AI, but
also very sceptical and cautious.

Slowly, however, more and more AI services become available to us, but
basically only web-interface chat solutions without APIs and IDE/development
related integrations.

I was moderately happy when recently it was announced that we can use [_Amazon
Q Developer_](https://aws.amazon.com/q/developer/)! Awesome! Finally a
development focused solution!

There are extensions for JetBrains IDEs, VS Code, Visual Studio and a preview
for Eclipse, and there is a _cli_.

That should be enough, right?

...

Well. Not for all of us neovim users.

I want to have integration to the full potential of generative AI, adhering to
applicable compliance restrictions, and still remain in the tool environment
that makes me as productive and makes my work as enjoyable as possible.

## Possible solution

There are many pretty excellent neovim AI plugins out there, e.g.
[Avante](https://github.com/yetone/avante.nvim) or [Copilot
Chat](https://github.com/CopilotC-Nvim/CopilotChat.nvim).

While those are pretty nice and powerful, most of them are taylored around a
number of the big AI API providers, and are pretty big to work with.

_Amazon Q_ does not provide a nice API, especially not an API that is
compatible with existing clients and integrations. I spent a few minutes
looking through Avante and did not see a quick path to extend the existing (web
API centric) model adaption with Q.

I have been tinkering with [a fork](https://github.com/gierdo/neoai.nvim) of
[neoai.nvim](https://github.com/Bryley/neoai.nvim), extending it a little for
my usecases, focussing on low-overhead integration of local language models
with llama.cpp. I've been very happy with it, especially because the plugin is
still so small that I don't have a hard time understanding what it does and how
to extend it without having to spend too much time. I'm lazy..

So, I decided to simply extend the plugin I'm most familiar with, [my
fork](https://github.com/gierdo/neoai.nvim).

## Solution

As written above, _Amazon Q_ does not provide a nice API, but has a cli, called
`q`. I decided to keep it simple and simply dump my prompt into `q` in
_non-interactive_ mode, parsing its `stdout` output. I hoped that it would
allow me to use the existing way of providing persistent chat sessions and
context, as well as having a simple way to apply existing functionality, e.g.
code recognition for easy pasting with `:put c` etc.

It was surprisingly easy, the core of it all is fifty-ish lines of `lua`:

```lua
local utils = require("neoai.utils")

--- This model definition supports Amazon Q CLI integration
---@type ModelModule
local M = {}

M.name = "Amazon Q"

M._chunks = {}

M.get_current_output = function()
 return table.concat(M._chunks, "")
end

---@param chunk string
---@param on_stdout_chunk fun(chunk: string) Function to call whenever a stdout chunk occurs
M._recieve_chunk = function(chunk, on_stdout_chunk)
 -- Split the input chunk by newlines
 on_stdout_chunk(chunk)
 table.insert(M._chunks, chunk)
end

---@param chat_history ChatHistory
---@param on_stdout_chunk fun(chunk: string) Function to call whenever a stdout chunk occurs
---@param on_complete fun(err?: string, output?: string) Function to call when model has finished
M.send_to_model = function(chat_history, on_stdout_chunk, on_complete)
 -- Format messages for Amazon Q CLI
 local messages_json = vim.json.encode(chat_history.messages)

 chunks = {}

 local command = {}
 command = {
  "chat",
  "--no-interactive",
 }
 for _, v in ipairs(chat_history.params) do
  table.insert(command, v)
 end
 table.insert(command, messages_json)

 -- Execute the Amazon Q CLI command
 utils.exec("q", command, function(chunk)
  M._recieve_chunk(chunk, on_stdout_chunk)
 end, function(err, _)
  -- Clean up the temporary file

  on_complete(err, M.get_current_output())
 end)
end
return M
```

_Q_ behaves a bit weird, it only marks source-code it produces with colors,
using _ansi_ codes, instead of using the de-facto standard backtics. Luckily it
appears to be consistent, so code snippet extraction works like this:

```lua
extract_code_snippets = function(text)
  local matches = {}
  -- Amazon Q marks code by color-coding it green with ansi codes.
  -- Everything between "start green" and "reset color" is assumed to be code.
  for match in string.gmatch(text, "\27%[38;5;%d+m(.-)\27%[0m") do
    table.insert(matches, match)
  end
  return table.concat(matches, "\n\n")
end
```

I run environments that do not have `q` available, so I made my config
sensitive to its availability.

```lua
config = function()
      local q_available = vim.fn.executable("q") == 1

      local models = {}
      local extract_code_snippets

      if q_available then
        extract_code_snippets = function(text)
          local matches = {}
          -- Amazon Q marks code by color-coding it green with ansi codes.
          -- Everything between "start green" and "reset color" is assumed to be code.
          for match in string.gmatch(text, "\27%[38;5;%d+m(.-)\27%[0m") do
            table.insert(matches, match)
          end
          return table.concat(matches, "\n\n")
        end

        table.insert(models, {
          name = "q",
          model = "q",
          params = { "--trust-all-tools" },
        })
      else
        startLlama()

        extract_code_snippets = function(text)
          local matches = {}
          for match in string.gmatch(text, "```%w*\n(.-)```") do
            table.insert(matches, match)
          end

          -- Next part matches any code snippets that are incomplete
          local count = select(2, string.gsub(text, "```", "```"))
          if count % 2 == 1 then
            local pattern = "```%w*\n([^`]-)$"
            local match = string.match(text, pattern)
            table.insert(matches, match)
          end
          return table.concat(matches, "\n\n")
        end

        table.insert(models, {
          name = "openai",
          model = "llama 3",
          params = nil,
        })
      end

      require("neoai").setup({ ---@diagnostic disable-line: missing-fields
        ui = {
          output_popup_text = "AI",
          input_popup_text = "Prompt",
          width = 45, -- As percentage eg. 45%
          output_popup_height = 80, -- As percentage eg. 80%
          submit = "<Enter>", -- Key binding to submit the prompt
        },
        models = models,
        register_output = {
          ["a"] = function(output)
            return output
          end,
          ["c"] = extract_code_snippets,
        },
        inject = {
          cutoff_width = 75,
        },
        prompts = {
          context_prompt = function(context)
            return "Please only follow instructions or answer to questions. Be concise. "
              .. (vim.api.nvim_buf_get_name(0) ~= "" and "This is my currently opened file: " .. vim.api.nvim_buf_get_name(
                0
              ) or "")
              .. "I'd like to provide some context for future "
              .. "messages. Here is the code/text that I want to refer "
              .. "to in our upcoming conversations:\n\n"
              .. context
          end,
          default_prompt = function()
            return "Please only follow instructions or answer to questions. Be concise. "
              .. (
                vim.api.nvim_buf_get_name(0) ~= ""
                  and "This is my currently opened file: " .. vim.api.nvim_buf_get_name(0)
                or ""
              )
          end,
        },
        mappings = {
          ["select_up"] = "<C-k>",
          ["select_down"] = "<C-j>",
        },
        open_ai = {
          url = "http://127.0.0.1:9741/v1/chat/completions",
          display_name = "llama.cpp",
          api_key = { ---@diagnostic disable-line: missing-fields
            value = nil,
            get = function()
              return ""
            end,
          },
        },
      })
    end,

```

## Secret Agents

Reading through the config you might see a dangerous option: `params = {
"--trust-all-tools" },`.

This allows `q` to execute all commands that are available to it in the current
context.

Yes, it's dangerous, probably also a bit stupid. But really powerful, as well.
And a lot of fun, to be honest.

First I was annoyed by _Amazon Q_ not providing a web API, but using the `cli`
as API makes the tool integration agentic, basically for free!

I give the currently open file as context in the system prompt.

```lua
.. (
  vim.api.nvim_buf_get_name(0) ~= ""
    and "This is my currently opened file: " .. vim.api.nvim_buf_get_name(0)
  or ""
)
```

With this, I can refer to the current file and let _q_ interact with it.

Or even the entire project!

I was pretty amazed by pressing `<Alt-a>` in my editor and watching the
integration perform a full _Please run the tests in this project and tell me if
they were a success._ and things like _Create a feature branch hinting that I
created a new UI, commit the latest changes and push them._

## tl;dr

- Amazon Q is a developer centric AI by AWS
- Amazon Q does not provide a classic web API
- Amazon Q provides a `cli`
- The `cli` can be integrated in `neovim`
- The integration provides powerful assistance, through context-handover and
  chat response or by interacting with entire projects through shell command
  execution and interacting with files
