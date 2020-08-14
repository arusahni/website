.. title: Switching to NeoVim (Part 1)
.. slug: switching-to-neovim-part-1
.. date: 2015-03-31 22:49:11 UTC-04:00
.. tags: vim, linux, osx
.. link:
.. description: In which I describe the process by which I switched from Vim to NeoVim.
.. type: text

*2016-11-03 Update: Now using the XDG-compliant configuration location.*

`NeoVim <http://neovim.org/>`_ is all the rage these days, and I can't help but be similarly enthused. Unlike other editors, which have varying degrees of crappiness with their Vim emulation, NeoVim *is* Vim.

If it's Vim, why bother switching? Much like all squares are rectangles, but not all rectangles are squares, NeoVim has a different set of aspirations and features.  While vanilla Vim has the (noble and important) goal of supporting all possible platforms, that legacy has seemingly held it back from eliminating warts and adding new features.  That's both a good thing and a bad thing. Good because it's stable, bad because it can lead to stagnation.  A Vim contributor, annoyed with how the project was seemingly hamstrung by this legacy (with its accompanying byzantine code structure, project structure, and conventions), decided to take matters into his hands and fork the editor.

The name of the fork?  NeoVim.

It brings quite a lot to the table, and deserves a blog post or two in its own right.  `I'll leave the diffing as an exercise to the reader <https://github.com/neovim/neovim/wiki/Introduction>`_.  I plan on writing about some of those differences as I do more with the fork's unique features.

.. TEASER_END

So, what did I need to do to switch to NeoVim?  I installed it.  On Kubuntu, all I needed to do was add a PPA and install the :code:`neovim` package (and its Python bindings for full plugin support).

.. code:: console

  $ sudo add-apt-repository -y ppa:neovim-ppa/unstable
  $ sudo apt-get update && sudo apt-get install -y neovim
  $ pip install --user neovim

Next up, configuration - one of Vim's great strengths.  I dutifully keep a copy of `my vimrc file on GitHub <https://github.com/arusahni/dotfiles/blob/master/vimrc>`_, and deploy it to any workstation I use for prolonged periods of time.   It'd be nice if I could carry it over to NeoVim.

Suprise!  It Just |Works (TM)|.  Remember, NeoVim *is* Vim, as such it shares the same configuration syntax.  Since I don't think I'm doing anything too crazy in my vimrc, it should be a drop-in operation.

.. |Works (TM)| unicode:: Works U+2122

.. code:: console

  $ mkdir -p ~/.config/nvim
  $ ln -s ~/.vimrc ~/.config/nvim/init.vim

After that, it's a simple matter of invoking :code:`nvim` from the command line.  Everything loaded and worked for me from the first run!

Almost.

This pleasant detour over, I went to resume `lolologist <https://github.com/arusahni/lolologist>`_ development.  However, when I activated my virtual environment and fired up :code:`nvim`, I got a message stating:

  No neovim module found for Python 2.7.8. Try installing it with 'pip install neovim' or see ':help nvim-python'.

Hm.  That's strange.  `The relevant help docs <https://github.com/neovim/neovim/blob/c47e0d6210a82f3c2a88e2bc937e77f8b2a72b64/runtime/doc/nvim_python.txt>`_, however, tell us all we need to know - the Python plugin needs to be discoverable in our path, and, since I'm using a virtual environment, a different Python instance is being used.  This is easily addressed, as detailed in that help doc.  **However**, since I use this vimrc file on two platforms (Linux & OS X), I need to be a little smarter about hardcoding paths to Python executables.  I added this to my vimrc (it shouldn't negatively impact my Vim use, so it's fine to be in a shared configuration).

.. code:: vim

  if has("unix")
    let s:uname = system("uname")
    let g:python_host_prog='/usr/bin/python'
    if s:uname == "Darwin\n"
      let g:python_host_prog='/usr/local/bin/python' # found via `which python`
    endif
  endif

Restarting NeoVim with that configuration block in place let it find my system Python and all associated plugins.

I'll keep this site updated with any new discoveries and NeoVim experiments! I'm quite eager to see how the client-server integrations flesh out.

*I've written more on this!* |p2link|_.

.. _p2link: link://slug/switching-to-neovim-part-2

.. |p2link| replace:: *Part 2*
