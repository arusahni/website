.. tags: code, python, webdev, flask, draft
.. date: 2013-10-13 12:00:00
.. slug: flask-multithreading
.. title: Multithreading your Flask dev server to safety
.. description: In which I share a quick tip on how to make Flask's dev server not suck when handling multiple users.

`Flask <http://flask.pocoo.org/>`_ is awesome. It's lightweight enough to disappear, but extensible enough to be able to get some niceties such as auth and ACLs without much effort.  On top of that, the Werkzeug debugger is pretty handy (:doc:`not as nice as wdb's, though <wdb-attachment>`).

Things were going swimmingly until our QA server went down.  While the server may have stopped, development didn't, and we needed a way to get testable builds up and running for QA.  One of my fellow developers quickly stood up an instance of our application on a lightly-used box that was nearing the end of its usefulness.  To get around the fact that the machine wasn't outfitted for httpd action, the developer just backgrounded an instance of the Flask development server and established an SSH tunnel for our QA team to use.  This was deemed acceptable by our QA team of one.  We rejoiced and went back to work.

Days passed and our system engineer's backlog remained dangerously full, with the QA server being a low priority.  Thankfully, aside from having to restart the process a few times, the Little Server That Could kept on chugging.  However, disaster soon struck - we hired another tester!  Technically her joining was a great thing; when you bring in fresh blood you get not only another person working the trenches but also an influx of fresh ideas.  However, her arrival brought some unforeseen trouble - the QA server would get more than one user! This wouldn't normally be an issue, but at the moment our QA server was nothing more than a single threaded developer script running as an unmanaged process.  Oops.

Sure enough the complaints from QA started bubbling forth: "Test is down!", "Test isn't loading!", "Test is unusable!"  To preserve harmony in the office, and to hold to our end of the developer-tester bargain, we had to do something (the sysengineer was busy rebuilding a dead Solr cluster). We needed to buy some time, so I started investigating what we could do with what we already had - maybe there was a way to handle the load with the development server.

Flask uses the Werkzeug WSGI library to manage its server, so that would be a good place to start.  Sure enough, `the docs state <http://werkzeug.pocoo.org/docs/serving/#werkzeug.serving.run_simple>`_ that the `wsgiref` wrapper accepts a parameter specifying whether or not the it should use a thread per request.  Now, how to invoke this from within Flask?

This, much like with everything else in the framework, was surprisingly easy. `Flask's app.run function <http://flask.pocoo.org/docs/api/#flask.Flask.run>`_ accepts an `option` collection which it passes on to Werkzeug's `run_simple`.  I updated our dev server startup script...

.. code:: python

  app.run(host="0.0.0.0", port=8080, threaded=True)


... and then threw it back over the wall to QA.  I moseyed on over to test-land shortly thereafter to discover them chipping away at a backlog of tasks, with the webserver serving away as if nothing happened.

A few weeks weeks later, the new test server is almost ready for prime time, and our multithreaded Flask server (**which should never be used for any sort of production purposes**) is still holding down the fort.