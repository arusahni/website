.. tags: code, python, webdev, flask
.. date: 2013-10-06 15:31:00
.. slug: wdb-attachment
.. title: Running wdb alongside your development server
.. description: In which I share a script that I wrote to start and stop alongside a development webserver.
.. comments: true

We've been using the wonderful `wdb <https://github.com/Kozea/wdb>`_ WSGI middleware tool at work to aid in debugging our Flask app.  The features it brings to the table have helped immensely with development, and it has quickly established itself as a integral part of my toolbox.

One minor issue I had with it, though, was the fact that one needs to manually start the :code:`wdb.server.py` script before launching a development server.  If only I could start and stop wdb whenever Flask did.  Some Google-fu introduced me to `bash's signal trapping features <http://www.ibm.com/developerworks/aix/library/au-usingtraps/>`_.  After some experimentation, I devised the following wrapper script.

.. gist:: 5976676

This has been working quite reliably for us.  Here's hoping it helps you!