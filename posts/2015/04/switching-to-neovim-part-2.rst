.. title: Switching to NeoVim (Part 2)
.. slug: switching-to-neovim-part-2
.. date: 2015-04-20 21:48:26 UTC-04:00
.. tags: vim
.. link: 
.. description: In which I describe how I parameterized hardcoded Vim paths in my vimrc.
.. type: text

`Now that my initial NeoVim configuration is in place <link://slug/switching-to-neovim-part-1>`_, I'm ready to get to work, right? Well, almost. In my excitement to make the leap from one editor to another, I neglected a portion of my attempt to keep Vim and NeoVim isolated - the local runtimepath (usually :code:`~/.vim`).

"But Aru, if NeoVim is basically Vim, shouldn't they be able to share the directory?" Usually, yes. But I anticipate, as I start to experiment with some of the new features and functionality of NeoVim, I might add plugins that I want to keep isolated from my Vim instance.

I'd like Vim to use :code:`~/.vim` and NeoVim to use :code:`~/.nvim`.  Accomplishing this is simple - I must first detect whether or not I'm running NeoVim and base the root path on the outcome of that test:

.. code:: vim

  if has('nvim')
      let s:editor_root=expand("~/.nvim")
  else
      let s:editor_root=expand("~/.vim")
  endif

With the root directory in a variable named :code:`editor_root`, all that's left is a straightforward find and replace to convert all rtp references to the new syntax.

e.g. :code:`let &rtp = &rtp . ',.vim/bundle/vundle/'` |srarr| :code:`let &rtp = &rtp . ',' . s:editor_root . '/bundle/vundle/'`

.. |srarr|     unicode:: U+02192

With those replacements out of the way, things *almost* worked. Almost.

I use `Vundle <https://github.com/gmarik/Vundle.vim>`_. I think it's pretty rad. `My vimrc file is configured to automatically install it and download all of the defined plugins in the event of a fresh installation <https://github.com/arusahni/dotfiles/blob/45c6655d46d1f672cc36f4e81c2a674484739ebc/vimrc#L42>`_.  The first time I launched NeoVim with the above changes didn't result in a fresh install - it was still reading the :code:`~/.vim` directory's plugins.

Perplexed, I dove into the Vundle code. Sure enough, it appears to default to installing plugins to :code:`$HOME/.vim` if a directory isn't passed in to the script initialization function.  It appears that I was reliant on this default behavior. Thankfully, this was easily solved by passing in my own bundle path:

.. code:: vim

  call vundle#rc(s:editor_root . '/bundle')

And with that, my Vim and NeoVim instances were fully segregated.
