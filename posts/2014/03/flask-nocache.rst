.. tags: code, python, webdev, flask
.. date: 2014-03-08 12:15:00
.. slug: flask-nocache
.. title: Disabling caching in Flask
.. description: In which I share a way to disable browser caching of specific endpoints in Flask.

It was your run of the mill work day: I was trying to close out stories from a jam-packed sprint, while QA was doing their best to break things.  Our test server was temporarily out of commission, :doc:`so we had to run a development server in multithreaded mode <flask-multithreading>`.

One bug came across my inbox, flagging a feature that I had developed. Specifically a dropdown that displayed a list of saved user content wasn't updating.  I attempted to replicate the issue on my machine, no-luck.  Fired up my IE VM to test and, yet again, no luck.  Weird.

I walked over to the tester's desk and asked her to walk me through the bug.  Sure enough, upon saving an item, the contents of the dropdown did not update.  Stumped, I returned to my machine and spoofed her account, wondering if there was an issue affecting her account within our mocked database backend (read: flat file).  I was able to see the "missing" saved content. Now I was getting somewhere.

I walked back to her machine, opened the F12 Developer Tools, and watched the network traffic. The GET for that dynamically-populated list was resulting in a 304 status, and IE was using a cached version of the endpoint.  Of course, I facepalmed as this was something I've not normally had to deal with as I usually use a cachebuster.  However, for this new application, my team has been trying to do things as cleanly as possible.  Wanting to keep our hack count down, I went back to my machine to see what our web framework could do.

Coming from a four-year stint in ASP.NET land, I was used to being able to set caching per endpoint via the :code:`[OutputCache(NoStore = true, Duration = 0, VaryByParam = "None")]` ActionAttribute. Sadly, there didn't appear to be something similar in Flask. Thankfully, it turned out to be fairly trivial to write one.

I found `this post from Flask creator Armin Ronacher <http://flask.pocoo.org/mailinglist/archive/2011/8/8/add-no-cache-to-response/#952cc027cf22800312168250e59bade4>`_ that set me on the right track, but the snippet provided didn't work for all the Internet Explorer versions we were targeting. However, I was able to whip together a decorator that did:

.. gist:: 9434953

To invoke it, all you need to do is import the function and apply it to your endpoint.

.. code:: python

  from nocache import nocache
  
  @app.route('/my_endpoint')
  @nocache
  def my_endpoint():
      return render_template(...)
      
I took this back to QA and was met with success.  The downside of this manual implementation, however, is one needs to be religious in applying it. To this day we still stumble upon cases where a developer forgot to add the decorator to a JSON endpoint.  Thankfully, code review processes a perfect for catching that sort of omission.