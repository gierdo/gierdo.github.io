---
title: lua all the things! - lua all the things?
categories:
  - programming
  - linux
tags:
  - vim
---

# Migrating a vimscript configuration to lua

## The Problem

I started using vim a while back around 2016 and started managing my vim config
and general setup with a [dotfiles repo](https://github.com/gierdo/dotfiles)
in 2017. As it goes, the repo grew and grew and grew some more, and not all
things in it are well designed and set up.
Sometimes, those limitations are affected by the used technology and lack of
alternatives. Which is also true for the _configuration of vim_!

[Vimscript](https://en.wikibooks.org/wiki/Learning_the_vi_Editor/Vim/VimL_Script_language),
the default domain specific language to configure, extend and enhance vim is
not bad, and it allows you to configure vim itself very concisely. That's the
main purpose of the language, after all. However, it comes with limitations. If
you want to express complicated logic, ideally in a reusable and maintainable
way, and execute the expressed logic in a performant way, vimscript will reach
its limits.

That's the main reason why [neovim](https://neovim.io), a modern
reimplementation of vim, introduced support for [lua](https://www.lua.org) as
natively supported and interpreted language. It can be used to write plugins,
but also to _configure_ neovim and its plugins.

I have started using neovim a few years ago already. My reason for switching
was that it provided excellent support for asynchronous plugins, e.g. linters
and language servers. However, for a quite long time I kept my setup compatible
with vim and neovim. Even after breaking compatibility, I kept the base of my
old config and extended it from there.

Which meant that

- The configuration was all over the place
- Plugin installation was decoupled from plugin configuration
- Complicated logic was a mess
- Startup time was not ideal (While it was absolutely acceptable for me)

## Why not migrate from vimscript to lua?

Rewriting all that stuff in a language I've never worked with always sounded
like too much work. And I've never really looked into lua, to be honest.. But
recently, I touched lua for the first time, playing around vim
[neoai](./2023-11-24-llama-vim.md), and it wasn't too bad. So I finally
continued with it and rewrote my config.

## Observations

The lua-connoisseurs will probably know solutions, more elegant patterns and
reasons for my Observations, and maybe I will also know more in a few months.
But this was what I observed.

### Not all vimscript is supported in lua

Not everything possible with vimscript is reimplemented in the lua api of neovim.
This is [by
design](https://github.com/neovim/neovim/issues/18201#issuecomment-1107665957),
and that's why vim commands can be executed from lua directly with `vim.cmd()`.
This makes sense and is understandable, but quite often had the questions of
the kind "Can _I_ do this in lua? Should I do this in lua? Do I want to do this
in lua?", and I also had to figure out if what I wanted to do was actually
_possible_ in lua.

I ended up adding support for sourcing vimscript "modules" from my lua modules,
but also grouping vimscript execution in lua modules. I haven't made up my mind
yet what I prefer.

```lua
local exports = {}


local function script_path()
  local str = debug.getinfo(2, "S").source:sub(2)
  return str:match("(.*/)")
end

function exports.load_local_vimscript(script_name)
  local module_directory = script_path()
  vim.cmd('source ' .. module_directory .. script_name)
end

return exports
```

To be used like this:

```lua
local utils = require('utils')

return {
  {
    'altercation/vim-colors-solarized',
    config = function()
      utils.load_local_vimscript("plugins/theme.vim")
    end,
    lazy = false,
    priority = 1000
  },
        .
        .
        .
```

And my general plugin-independent config loads all lua and vimscript files in
the `config_d` directory:

```lua
-- Require all other `.lua` and `.vim` files in the same directory

local info = debug.getinfo(1, 'S')
local module_directory = string.match(info.source, '^@(.*)/')
local module_filename = string.match(info.source, '/([^/]*)$')

-- Apparently the name of this module is given as an argument when it is
-- required, and apparently we get that argument with three dots.
local module_name = ... or "init.d"

local function scandir(directory)
  local i, t, popen = 0, {}, io.popen
  local pfile = popen('ls -a "' .. directory .. '"')
  for filename in pfile:lines() do
    i = i + 1
    t[i] = filename
  end
  pfile:close()
  return t
end

local lua_config_files = vim.tbl_filter(function(filename)
  local is_lua_module = string.match(filename, "[.]lua$")
  local is_this_file = filename == module_filename
  return is_lua_module and not is_this_file
end, scandir(module_directory))

for i, filename in ipairs(lua_config_files) do
  local config_module = string.match(filename, "(.+).lua$")
  require(module_name .. "." .. config_module)
end

local vimscript_config_files = vim.tbl_filter(function(filename)
  local is_config = string.match(filename, "[.]vim$")
  local is_this_file = filename == module_filename
  return is_config and not is_this_file
end, scandir(module_directory))

for i, filename in ipairs(vimscript_config_files) do
  vim.cmd("source " .. module_directory .. "/" .. filename)
end
```

I ended up replacing _most_ of my configuration with lua, but a few things that
did not require any logic and are very specific, I didn't migrate. I didn't see
the benefit, and for some configuration things vimscript is just much less
verbose. It's a DSL for vim configuration, after all..

### Lazy is a pretty great plugin manager

I decided to try out [Lazy](https://github.com/folke/lazy.nvim) for my plugin
management, and it's pretty awesome!

_Why is this significant?_

Most of the functional magic of vim happens in plugins, and that's also where
most of the magic of the orchestration and configuration happens.
And Lazy is a lua-centric plugin manager with great features.

The main things that stand out for me are

- Self-contained, modular definition of plugin installation and configuration
- Definition of explicit plugin dependencies with lazy loading
- Easy bootstrapping

#### Modular plugin installation and configuration

My configuration is structured like this:

```text
~/.dotfiles/.config/nvim/..
+ - lua
  + - config_d
      - basic_behavior.lua
      - filetypes.vim
      - init.lua
  + - plugins
      - ale.lua
      - coc.lua
      - init.lua
      - neoai.lua
      - nvim-tree.lua
      - telescope.lua
      - theme.lua
    - utils.lua
  - init.lua
```

The definition of all installed plugins and their configuration happens in the
`plugins` directory, and the single lua files are completely self-contained.
They contain the installation specification and complete configuration of
plugins with all their dependencies, defined explicitly. Plugins can be added
or removed without remaining artifacts, everything is easy to maintain, extend
and clean up. And `Lazy` loads and orchestrates everything on its own.

The Grouping, btw, is not definitive. I kind of group larger plugins with a
more complex configuration and complex dependencies in their own module,
trivial stuff is defined in `plugins/init.lua`

Example config excerpt for `nvim-tree`:

```lua
local function open_nvim_tree(data)
  local api = require("nvim-tree.api")
  -- buffer is a real file on the disk
  local real_file = vim.fn.filereadable(data.file) == 1

  if real_file then
    return
  end

  -- buffer is a [No Name]
  local no_name = data.file == "" and vim.bo[data.buf].buftype == ""

  if not real_file and not no_name then
    return
  end
  api.tree.toggle({ focus = false, find_file = false, })
end

return {
  {
    'nvim-tree/nvim-tree.lua',
    priority = 1,
    lazy = false,
    dependencies = {
      'nvim-tree/nvim-web-devicons',
    },
    init = function()
      vim.g.loaded_netrw = 1
      vim.g.loaded_netrwPlugin = 1
    end,
    config = function()
      local tree = require("nvim-tree")
      tree.setup(
        {
          on_attach = my_on_attach,
          update_focused_file = {
            enable = false,
          },
          filters = {
            dotfiles = true,
            exclude = { ".config", ".local" },
          },
        }
      )

      vim.api.nvim_create_autocmd("WinClosed", {
        callback = function()
          local winnr = tonumber(vim.fn.expand("<amatch>"))
          vim.schedule_wrap(tab_win_closed(winnr))
        end,
        nested = true
      })
      vim.api.nvim_create_autocmd({ "VimEnter" }, { callback = open_nvim_tree })
      vim.keymap.set('n', '<C-n>', '<cmd>NvimTreeToggle<cr>')
      vim.keymap.set('n', '<A-n>', '<cmd>NvimTreeFindFile<cr>')
    end
  },
  'nvim-tree/nvim-web-devicons',
}
```

## tl;dr

- Context: vim / neovim configuration
- lua is more powerful than vimscript
- Some things are not possible with lua
- Some things are simpler with vimscript
- I migrated (most of) my config from vimscript to lua
- Check it out [here](https://github.com/gierdo/dotfiles/tree/master/.config/nvim)
