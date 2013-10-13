.. tags: code, python, webdev, flask, draft
.. date: 2013-10-09 15:31:00
.. slug: flask-nocache
.. title: Disabling caching for specific views in Flask
.. description: In which I share a way to disable browser caching of specific endpoints in Flask.

It was your run of the mill work day: I was trying to close out stories from a jam-packed sprint, while QA was doing their best to break things.  Our test webserver was temporarily out of commission, :doc:`so we had to run a development server in multithreaded mode <flask-multithreading>`.

One bug came across my inbox, flagging a feature that I had developed. Specifically a dropdown that displayed a list of saved user content wasn't updating.  I tested the use case on my machine, no-luck.  Fired up my IE VM to test and, yet again, no luck.  Weird.

I walked over to the tester's desk and asked her to walk me through the bug.  Sure enough, upon saving an item, the contents of the dropdown did not update.  Stumped, I returned to my machine and spoofed her account, wondering if there was an issue affecting her account within our mocked database backend (read: flat file).  I was able to see the "missing" saved content. Now I was getting somewhere.

I walked back to her machine, opened the F12 Developer Tools, and watched the network traffic. The GET for that dynamically-populated list was resulting in a 304 status, and IE was using a cached version of the endpoint.  Of course, I facepalmed as this was something I've not normally had to deal with as I usually use a cachebuster.  However, for this new application, my team has been trying to do things as cleanly as possible.  Wanting to keep our hack count down, I went back to my machine to see what our web framework could do.

Coming from a four-year stint in ASP.NET land, I'm used to being able to set