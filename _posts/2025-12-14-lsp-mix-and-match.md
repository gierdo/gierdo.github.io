---
title: "LSP mix and match - cherry-pick capabilities from different LSP servers"
categories:
  - linux
  - programming
tags:
  - vim
  - lsp
  - configuration
  - capabilities
  - ty
  - python
  - pyrefly
---

# LSP mix and match - cherry-pick capabilities from different LSP servers

As I have written about [here]({% link _posts/2024-09-19-nvim-lsp.md %}), for
most of my file edititing / programming / note-taking / ... I use [vim
(neovim)](https://neovim.io).

Not a bare-bones neovim, though. I use many plugins, and I have even more of
them set up. Too many? Maybe. Probably. Especially considering the ever growing
number of software supply chain attacks..

Anyways, that is a different topic for a different day.

Besides convenient plugins, the main building block for developer experience
enhancement and efficiency are [_language
servers_](https://microsoft.github.io/language-server-protocol/implementors/servers/),
which provide stuff like _diagnostics_, _code completion_, _navigation
enhancements_ and more.

Besides playing around with its config, I try to actually use vim for
productive work, including programming in `python`.

My go-to language server for python has been
[`basedpyright`](https://github.com/DetachHead/basedpyright), a fork of
microsoft's [`pyright`](https://github.com/microsoft/pyright), which basically
adds the functionality of the proprietary `pylance` extension to `pyright` and
adds some more bells and whistles.

While `basedpyright` is stable, helpful, convenient and generally excellent,
there are a few new kids on the block:

There is a new generation of `python` language servers, written in `Rust`,
targeting high performance and productivity in large and complex code bases:

There is Astral's [`ty`](https://docs.astral.sh/ty/), Meta's
[`pyrefly`](https://pyrefly.org/) and David Halter's (Jedi's original author)
[`zuban`](https://zubanls.com), and probably some more.

I have been intrigued by them, mainly hoping for a smoother and more performant
workflow in large and growing codebases. So, I have played around with them for
a while now, eventually ending up using `ty` only for type checking while still
having to rely on `basedpyright` for completion, auto-import, symbol renaming,
navigation etc.

_Why?_

In single capabilities, none of the new lsp servers match `basedpyright` in my
workflow. Yet.

E.g.:

- `pyrefly` has great completion, navigation and even auto-import, but it's
  really bad in renaming symbols. It can rename variables, but no functions.
- `ty` has limited support of `pydantic`, which I use a lot, completion is not
  as powerful as `pyrefly`'s, but renaming works great.

_Why not use both?_

If you enable several language servers with the (nominally) same capabilities
for the same file type, you get duplicate results/actions for every capability.
Go to definition? Select where you want to jump to from these two identical
alternatives!

My goal was to mix-and-match, e.g. use `ty` for renaming and `pyrefly` for
everything else. However, while especially `pyrefly`'s configuration is very
flexible, the lsps don't support tailored enabling and disabling of select
capabilities.

I tried for an unsuccessful while to configure the lsp server by restricting
the handed-over client capabilities, but that didn't go anywhere.

The only almost compromise that worked for me with the configuration through
the lsp servers themselves was _disabling_ `basedpyright`'s type checking and
_disabling_ `ty`'s language server capabilities, besides type checking. Like
this, at least a little bit of the heavy lifting could be offloaded from
`basedpyright` to `ty`, but I was not happy.

Now, I sort of am happy! I found out how to disable specific lsp capabilities
for specific lsp servers through [`nvim`'s lsp
configuration](https://neovim.io/doc/user/lsp.html). Instead of trying to get
the lsp server to not provide the functionality, the _client_ ignores it!

## The Problem

I like working with Python, also with larger and growing code bases. There are
LSP servers that offer excellent performance. However, none match the full
functionality of `basedpyright` when used independently. Unfortunately,
`basedpyright` lacks in performance. Combining multiple LSP servers introduces
duplicate functionality and a bit of a mess. Such combination could be feasible
if we could selectively disable or enable conflicting capabilities. However, it
is not possible to instruct LSP servers to withhold certain functionalities
unless they provide an interface for that, e.g. with dedicated configuration
options.

## The solution

Enable conflicting lsp servers, but tell the lsp client (_nvim_) that it should
not use specific lsp server's capabilities through an `on_init` hook 🥳!

The configuration is probably not fully final and will change, but this is the
general thing.

```lua
  local lsp_capabilities = require("blink.cmp").get_lsp_capabilities(nil, true)

  ---@param client vim.lsp.Client
  local ty_on_init = function(client)
    client.server_capabilities.callHierarchyProvider = false
    client.server_capabilities.completionProvider = false
    client.server_capabilities.inlineCompletionProvider = false
    client.server_capabilities.inlineValueProvider = false
    client.server_capabilities.declarationProvider = false
    client.server_capabilities.definitionProvider = false
    client.server_capabilities.implementationProvider = false
    client.server_capabilities.inlayHintProvider = false
    client.server_capabilities.signatureHelpProvider = false
    client.server_capabilities.hoverProvider = false
    client.server_capabilities.referencesProvider = false
    client.server_capabilities.codeActionProvider = false
  end
  vim.lsp.config("ty", {
    capabilities = lsp_capabilities,
    on_init = { ty_on_init },
    settings = {
      ty = {
        disableLanguageServices = false,
        diagnosticMode = "workspace",
        experimental = {
          rename = true,
        },
      },
    },
  })

  ---@param client vim.lsp.Client
  local pyrefly_on_init = function(client)
    client.server_capabilities.renameProvider = false
  end
  vim.lsp.config("pyrefly", {
    capabilities = lsp_capabilities,
    on_init = { pyrefly_on_init },
    settings = {
      python = {
        pyrefly = {
          disableLanguageServices = false,
          displayTypeErrors = "force-on",
        },
        analysis = {
          diagnosticMode = "workspace",
          inlayHints = {
            callArgumentNames = "all",
            variableTypes = true,
            functionReturnTypes = true,
            pytestParameters = true,
          },
        },
      },
    },
  })

```

## tl;dr

- Some lsp servers are not powerful enough to fully replace alternatives but
  would be great enhancements or replacements for single capabilities
- _nvim_'s lsp config API allows for specifying different hooks, `on_init` and
  `on_attach` etc.
- It's possible to disable specific capabilities in those hooks
