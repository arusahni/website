.. title: Accessing Webcams with Python
.. slug: webcams-in-python
.. date: 2014-06-30 20:10:53 UTC-04:00
.. tags: python, code, linux, osx
.. link: 
.. description: In which I describe the process by which I access webcams in Linux.
.. type: text

So, I've been working on a tool that turns your commit messages into image macros, named `Lolologist <https://github.com/arusahni/lolologist/>`_.  This was a great learning exercise because it gave me insight into things I haven't encountered before - namely:

1. Packaging python modules
2. Hooking into Git events
3. Using PIL (through Pillow) to manipulate images and text
4. Accessing a webcam through Python on \*nix-like platforms

I might talk to the first three at a later point, but the latter was the most interesting to me as someone who enjoys finding weird solutions to nontrivial problems.

Perusing the internet results in two third-party tools for "python webcam linux": `Pygame <http://www.pygame.org>`_ & `OpenCV <http://sourceforge.net/projects/opencvlibrary>`_. Great! Only problem is these come in at 10MB and **92MB** respectively.  Wanting to keep the package light and free of unnecessary dependencies, I set out to find a simpler solution...

.. TEASER_END

I started by looking for a way to directly access a webcam within the context of the system. Sadly, the ways forward I found were highly specific to certain kinds of devices and presented no "universal" solution.

I then decided to look one step further up in the abstraction chain and discovered some APIs provided by my desktop environment (KDE). The leads were promising, but, having suffered from the poor decisions of other devs who decided to couple their apps to particular DE's \*cough\*Gnome\*cough\*, I looked on.

Through my research I stumbled upon an interesting solution: Most Linux environments have `MPlayer <http://www.mplayerhq.hu/design7/info.html>`_ available, which in turn offers `a unified way to access webcams via the command line <https://wiki.archlinux.org/index.php/Webcam_Setup#MPlayer>`_.  Now we're talking.

To go about implementing this in Python, I determined that I needed to:

1. Call MPlayer
2. Pass it the appropriate arguments
3. Retrieve the captured image

Step one is easy - to execute system calls in Python one can use the :code:`subprocess` module.

Step two, however, required some trial and error.  What I ended up with was:

.. code:: python

  from subprocess import call, STDOUT
  call(['mplayer', 'tv://', '-vo', 'jpeg:outdir={}'.format(
      OUTPUT_DIRECTORY), '-frames', str(WARMUP_TIME)],
      stderr=STDOUT)
  image_path = os.path.join(OUTPUT_DIRECTORY,
      '{0:08d}.jpg'.format(WARMUP_TIME))
      
This will result in the command :code:`mplayer tv:// -vo jpeg:outdir=/path/to/output/ -frames WARMUP_INTEGER` being executed.  

The interesting trick here is in my use of the :code:`frames` argument.  It's a poor-man's warmup time.  Instead of telling the webcam to start capturing and capture the image after *n* seconds, I instead tell the webcam to capture a certain number of frames.  I then take the last frame out of the :code:`OUTPUT_DIRECTORY` and discard the rest.  This produces a usable image. In the event that the delay is insufficient, I made the warmup time user-customizable.

That solved the problem in Linux. This was a good solution until I changed jobs and got stuck with a Macbook Pro. What use is a tool if you can't use it? I set about trying to get things to work.

I quickly realized that there was no straightfoward way to access webcams via the OS without significant pain, and there was no always-there equivalent to MPlayer under OS X.  A cursory Googling turned up `ImageSnap <http://iharder.sourceforge.net/current/macosx/imagesnap/>`_, a command line tool that allows for webcam access. Not only was it public domain, but it was also available on Homebrew, so I could get away without bundling it with Lolologist.

Calling this was just as easy, if not slightly easier:

.. code:: python

  from subprocess import call, STDOUT
  call(['imagesnap', '-w', str(WARMUP_TIME), '-q', outpath],
      stderr=STDOUT)
  image = outpath
      
Notice - I didn't have to do the same frame-junking as I did with MPlayer. Instead, ImageSnap handled the wait on its own.

And *that* is how I got Lolologist to love two platforms simultaneously.

Postscript: `here is the relevant portion of the Lolologist codebase <https://github.com/arusahni/lolologist/blob/3ea78b86fdc61d5d7d76547bbd2baddd42873fdb/lolologist/cameras.py>`_. Happy hacking!
