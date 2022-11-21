.. title: Migrating my Neovim Config to Lua
.. slug: neovim-lua-config
.. date: 2022-11-20 18:04:45 UTC-05:00
.. tags: vim
.. link: 
.. description: In which I describe how I migrated my VimL-based Neovim config file to Lua.

I've been a happy [Neovim](https://neovim.io) user for the past several years.
The pace of development, quality of the product, and energy of the community
have made it enjoyable to use. Recently, the project has introduced
[Lua](https://en.wikipedia.org/wiki/Lua_(programming_language)) as a first-class
citizen in the editor. In places where you may otherwise be forced to wrangle
the mess that is VimL (aka Vimscript), you can instead use a saner, faster
scripting language.

I initially refrained from making the jump over to Lua as I held the ideal of
being able to return to Vim whenever I wanted. With the release of Vim version
8, which doubled down on VimL, I realized my folly. Vim wasn't going to
suddenly adopt the good parts of Neovim, nor was that community going to
change overnight. With that, I sat down one Saturday to get to porting. After a
few weeks of on-and-off experimentation (and exploring both /r/neovim and
GitHub to see how other people did things) , I landed on a stable, Lua-first
config that's been humming along for several weeks now.

Let's dive in!

<!-- TEASER_END -->

## Structure of a Config

One thing Lua brings to the table is a nice import system. This incentivizes
smaller, more logical chunks of config. There's quite a variety of practices
and preferences in the community. What I ended up with (some elisions) is:

```text
.
├── after
│   └── plugin
│       └── colors.lua
├── init.lua
└── lua
    ├── configs.lua
    ├── functions
    │   └── format-json.lua
    ├── functions.lua
    ├── keybindings.lua
    ├── plugins
    │   ├── comment.lua
    │   ├── lualine.lua
    │   ├── test.lua
    │   └── treesitter.lua
    ├── plugins.lua
    └── utils.lua
```

Let's start at the beginning.

### `init.lua` - The Entry Point

This is the first part of the config to be loaded. Its sole purpose is to tell
Neovim which modules it should load:

```lua
require("configs")
require("plugins")
require("keybindings")
require("functions")
```

In this case, I chose to split things out into four categories: configs,
plugins, keybindings, and functions. The order here matters, as the later
imports depend on the earlier ones. Imports in `init.lua` are resolved to
top-level `.lua` files in the `lua/` subdirectory.


### `lua/configs.lua` - Core Config

This file contains all my core Neovim configuration. From tab widths, to
highlighting behavior.

It starts with an import

```lua
local U = require("utils")
```

This loads the export of `lua/utils.lua` into the module-local variable `U`.
`utils.lua` contains a few helper functions that are reused throughout my
config.

Next, I alias some verbose Neovim commands to shorter variants to make the
config easier to read and update:

```lua
local exec = vim.api.nvim_exec -- execute Vimscript
local api = vim.api -- neovim commands
local autocmd = vim.api.nvim_create_autocmd -- execute autocommands
local set = vim.opt -- global options
local cmd = vim.cmd -- execute Vim commands
```

With that out of the way, I can start writing my config! This should look
fairly familiar to you:

```lua
vim.g.mapleader = ","
set.termguicolors = true
set.bg = "dark"
set.tabstop = 4
set.shiftwidth = 4
set.expandtab = true
set.mouse = "a"
-- additional config snipped for brevity
```

One thing I immediately grew to appreciate was being able to configure boolean
settings via `true`/`false` assignment, rather than `set expandtab` or `set
noexpandtab`. As a software engineer, this feels more ergonomic.

```lua
set.listchars = {
    nbsp = '⦸',
    extends = '»',
    precedes = '«',
    tab = '▷─',
    trail = '•',
    space = ' '
}
```

Being able to express my config via a real type system improves readability.
Who needs a comma-and-colon delimited list when you can use a table?

Of course, one nice thing about scriptable config is being able to introduce
function calls and macros for improved readability. Defining a autocmd
requires: `autocmd(<type>, args)`. Verbosity can be reduced (and readability
improved) with some helper functions

```lua
function filetype_autocmd(filetype, cmd, params)
    autocmd("FileType", { pattern = filetype, command = cmd .. ' ' .. params })
end

function buffer_autocmd(pattern, cmd, params)
    autocmd("BufRead", { pattern = pattern, command = cmd .. ' ' .. params })
end

function hold_autocmd(pattern, cmd)
    autocmd("CursorHold", { pattern = pattern, command = cmd })
end

filetype_autocmd("html", "setlocal", "ts=4 sts=4 sw=4 omnifunc=htmlcomplete#CompleteTags")
buffer_autocmd("*.cls", "set", "ft=apex syntax=java")
hold_autocmd("*", "silent call CocActionAsync('highlight')")
-- lots of autocmds excluded
```

Remember that utils import from earlier? Here's where I use it. I query the
platform to determine where it should find the system's Python installation.

```lua
if U.is_linux() then
    vim.g.python3_host_prog = "/bin/python"
elseif U.is_mac() then
    vim.g.python3_host_prog = "/usr/local/bin/python3"
end
```

### `lua/plugins.lua` - Plugins and Plugin Accessories

Plugin configuration is where the migration to Lua really shines. This file
contains the entry point for my plugin manager (Packer), along with
self-bootstrapping code.

```lua
local install_path = fn.stdpath("data") .. "/site/pack/packer/start/packer.nvim"
if vim.fn.empty(fn.glob(install_path)) > 0 then
  packer_bootstrap =
    fn.system({"git", "clone", "--depth", "1", "https://github.com/wbthomason/packer.nvim", install_path})
end
-- Register Packer with vim
vim.api.nvim_cmd({
    cmd = "packadd",
    args = {"packer.nvim"},
}, {})
```

This will pull down the repo if no trace of Packer is found, and install it.
Next, we hook into Packer's startup and provide a callback for plugin
installation.

```lua

return require("packer").startup {
  function(use)
    -- Packer can manage itself
    use "wbthomason/packer.nvim"

    -- Plugins go here

    -- Install and compile plugins
    if packer_bootstrap then
      require("packer").sync()
    end
  end
}
```

If you've used other Vim package managers (like vim-plug) the above `use`
syntax should seem familiar. Let's add some plugins:

```lua
use "simnalamburt/vim-mundo"

use "tpope/vim-repeat"

use {
  "nkakouros-original/numbers.nvim",
  config = [[ require("plugins/numbers") ]]
}
```

Yup, familiar. But what's going on with that `config` parameter? Let's take a
look at the file it references, `lua/plugins/numbers.lua`.

```lua
require "numbers".setup {
  excluded_filetypes = {
    "tagbar",
    "gundo",
    "minibufexpl",
    "nerdtree"
  }
}
```

This syntax is the standard ceremony for Lua-based Neovim plugins. It starts
with requiring the plugin module, invoking the `setup` function, and supplying
a table of configuration. These ergonomics are one of the reasons I'm so excited
about what the migration to Lua brings to the table. This spares me from the
configuration bit rot that leads to minutes spent tracking down the source of
unusual behavior. For non-Lua plugins, you can define them as usual:

```lua
-- lua/plugins/test.lua

-- set a variable using a traditional vim command
vim.cmd([[
    let test#strategy = "neoterm"
]])
```

### `lua/keybindings.lua` - Keybinding Configuration

Now that plugins are out of the way, we need to bind their actions to keys,
alongside Neovim's built-in ones.

It starts with a helpful abstraction for mapping.

```lua
function map(mode, lhs, rhs, opts)
    local options = { noremap = true }
    if opts then
        options = vim.tbl_extend("force", options, opts)
    end
    vim.keymap.set(mode, lhs, rhs, options)
end
```

* `mode` - the editor mode for the mapping (e.g., `i` for "insert" mode).
* `lhs` - the keybinding to detect.
* `rhs` - the command to execute.
* `opts` - additional options for the configuration (e.g., `silent`).

Let's put it to use!

```lua
-- Use jj to exit insert mode
map("i", "jj", "<Esc>")

-- use leader-nt to toggle the NvimTree plugin's visibility in normal mode
map("n", "<leader>nt", ":NvimTreeToggle<CR>")

-- use leader-t to run the unit test under my cursor, but don't display the command in the UI
map("n", "<leader>t", ":TestNearest<CR>", { silent = true })
```

Remember `utils.lua`? Its platform detection helpers make another appearance here:

```lua
-- use `gx` to open the path under the cursor using the system handler
if U.is_linux() then
    map("n", "gx", "<Cmd>call jobstart(['xdg-open', expand('<cfile>')])<CR>")
elseif U.is_mac() then
    map("n", "gx", "<Cmd>call jobstart(['open', expand('<cfile>')])<CR>")
end
```

Handy!

### `lua/functions.lua` - Arbitrary Code Entry Point

Vim's extensibility is a huge boon for resourceful engineers. Sometimes I want
to automate frequent actions with a straightforward Vim command.
`lua/functions.lua` is where it starts. It contains imports:

```lua
require("functions/format-json")
```

which reference standalone function modules:

```lua
-- lua/functions/format-json.lua

-- run the current through Python's builtin JSON formatter
vim.cmd([[ com! FormatJSON %!python -m json.tool ]])
```

Now, I can execute the `:FormatJSON` command any time!

### `after` - What's next?

As the name suggests, Vim's `after/` directory is loaded _after_ the init and
plugin phases. This code is always run on startup, which makes it a good
location for commands that need to override default vim state. Presently, I
only use it for color scheme configuration in `after/plugin/colors.lua`:

```lua
-- set my color scheme, and define some specific highlight overrides for search and matching parentheses
vim.cmd([[
let base16colorspace=256
colorscheme base16-onedark

hi Search ctermfg=237 ctermbg=13
hi MatchParen cterm=underline
]])
```

Now, when I load Vim, the color scheme and highlight overrides are set.

## The Future

That's it! I've really appreciated how Lua has enabled me to decompose my
`.vimrc`, and some of the increased scripting capabilities that come along for
the ride. As you can see, there are still a few places where I'm using `cmd`.
When will I be able to replace them? The Neovim developers have expressed:

> Duplicating every random Vim command in Lua achieves nothing, except a lot of
> useless, manually-written code.

`nvim_cmd`, released with 0.8, cleaned up a few of those remaining occurrences.
I'm not losing sleep over the remaining ones.

[You can see my full configuration here](https://github.com/arusahni/dotfiles/tree/fd6bf6435763f6d29e95ec3c42cdc33aa2cf6952/nvim).
