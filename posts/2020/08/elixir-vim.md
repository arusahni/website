<!--
    .. title: Elixir Development with Vim
    .. slug: elixir-vim
    .. date: 2020-08-25 13:41:20 UTC-04:00
    .. tags: tech, elixir, vim
    .. link: 
    .. description: In which I describe my NeoVim configuration for Elixir development
    .. type: text
-->

I love Vim (more specifically, [the NeoVim fork](http://neovim.org)). Modal
editing as part of my
[Unix IDE](https://sanctum.geek.nz/arabesque/unix-as-ide-introduction/) brings
an immense amount of productivity and enjoyment to my day-to-day development
activities. As such, whenever I take up a new language or framework, I enjoy
experimenting with how best to integrate it into my existing workflow.

Having started programming in Elixir, I've started the customization journey
for the language.  Much as with Maslow's Hierarchy of Needs, there's a
hierarchy of editor support required for an fulfilling programming experience.
Let's get there with Elixir!

<!-- TEASER_END -->>

### 1. Indentation & Syntax Highlighting

Both smart indentation and syntax highlighting make editing less painful - the
former reduces keystrokes (and occurrences of fighting the editor), while the
latter makes it easier to scan and parse the code displayed. Vim ships with
some presets, but they're either missing, incomplete, or outdated for newer,
less-used languages. The good news is, one can author and share their
language-specific rules. To save myself from this song and dance whenever I
open source code for a bespoke programming language, I use the
[vim-polyglot](https://github.com/sheerun/vim-polyglot) plugin. It bundles the
[vim-elixir](https://github.com/elixir-editors/vim-elixir) plugin (which is
currently searching for new maintainers).

All I need to do is drop an `Plug 'sheerun/vim-polyglot'` into my `init.vim`'s
[vim-plug](https://github.com/junegunn/vim-plug) section, and I'm off to the
races.

### 2. Linting

I've experimented with different levels of linting intrusiveness - from the
blocking, quickfix-window-popping-up-on-save, to manually-invoked lint checks
whenever I want to validate things. As Vim (especially with the advent of
NeoVim) became more async-y, I grew to like near-realtime linting of my code.
Linter integration is challenging - most linters are standalone scripts, which
are in turn wrapped by the editor plugins. They not only have to manage the
script lifecycle, but also parse the output and map it to the code in a Vim
buffer.  This can lead to all sorts of fun and exciting failure states.

Previously, I've accomplished that with Syntastic and ALE. However, since then
(and with credit to VS Code) we've entered a language server renaissance. They
not only enable things like smart code completion (a la Intellisence), but also
runtime linting. Despite being "standardized", however, much like the web, most
language servers are coded to the quirks and functionality of a specific client
- VS Code.

Enter [coc.nvim](https://github.com/neoclide/coc.nvim). This plugin runs VS Code
extensions in NeoVim, and offers advanced levels of integration into the editor.
Not only does it provide gutter icons and highlights for linting errors, but it
can also display the feedback as virtual text (vs. in a quickfix window).
Additionally, if language-based autocompletion is your thing, it's got your
back.

Installing is as straightforward as adding
`Plug 'neoclide/coc.nvim', {'branch': 'release'}` to my `init.vim`, and then
adding a global settings file in `~/.config/nvim/coc-settings.json`. This can
be conveniently opened by entering the `:CocConfig` command.  Then, I install
the Elixir plugin: `:CocInstall coc-elixir`

However, that wasn't _quite_ enough on my system. I quickly discovered that the
language server wasn't working. I dug through the backlog to discover that I
was using a version of Elixir different from what the language server was
compiled with.  Thankfully,
[coc-elixir](https://github.com/elixir-lsp/coc-elixir) can be aimed at a
different LSP.

1. Clone the elixir-ls repository: `git clone https://github.com/elixir-lsp/elixir-ls.git ~/.elixir-ls`
2. Enter the local repository: `cd ~/.elixir-ls`
3. Download all dependencies: `mix deps.get`
4. Build the package: `mix compile`
5. Create a release distribution: `mix elixir_ls.release -o release`

Once built, I updated coc.nvim's global config to point to it:

```js
// ~/.config/nvim/coc-settings.json
{
    "elixir.pathToElixirLS": "~/.elixir-ls/release/language_server.sh"
}
```

_Honorable Mention: There's also [the coc-diagnostic
plugin](https://github.com/iamcco/coc-diagnostic), which can be configured to
use [credo](https://github.com/rrrene/credo)._

### 3. Formatting

Code formatters are useful because they provide a default set of opinions for
code layout. For newcomers to a language, or larger community projects, this is
a boon. When developing in Python I use
[Black](https://black.readthedocs.io/en/stable/). While I disagree with some of
its opinions, having well-formed code be an invocation away is pretty rad.
Elixir ships with one - it can be invoked with `mix format`.

Vim itself supports external formatter integration through use of the
`formatprg` setting. Once set, the formatter can be invoked by using the `gq`
motion over a text object.

Tying these two things together is dead simple. In my `init.vim`:

```vim
autocmd FileType elixir setlocal formatprg=mix\ format\ -
```

This instructs NeoVim to set the `formatprg` for all Elixir files to `mix
format -`. The hyphen indicates that the buffer should be supplied via stdin.
With this setting, I can now run `gggqG` to format an entire file:  `gg` to go
to the first line, `gq` to start the formatter range selection, and `G` to set
the range to the last line.

---

With the above needs met, I now have an enjoyable Elixir-authoring experience.
What Vim customizations do you consider essential for your Elixir life?
