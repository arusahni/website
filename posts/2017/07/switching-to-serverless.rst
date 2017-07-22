.. title: Switching to Serverless
.. slug: switching-to-serverless
.. date: 2017-06-23 16:23:28 UTC-04:00
.. tags: tech, serverless, cloud, python
.. link:
.. description: In which I describe how I migrated a Python application to a serverless environment.
.. type: text

A few years ago, I wrote a webapplication to address a glaring need - hunger.
Specifically, my hunger.  This webapp took the form of a Slack integration
that, when asked for :code:`/foodtrucks`, would provide a list of foodtrucks to
the Slack channel. (If you'd like to learn more, check out `the talk I gave at
DC Python <https://www.youtube.com/watch?v=qR8e9MopTII>`_).

This application (Slacktrux) was running smoothly on a t2.micro instance on
AWS.  Like the best programs, I was able to set it up and then not think about
it.  This extreme reliability had a very real cost, however: money.
Specifically, I received an AWS bill a few months ago for the monthly cost of
that EC2 instance. (The reason it had suddenly popped up is I had credit for a
reserved instance for another project that I ended up decommissioning a few
months back, and the remaining time had ceded to that instance, which had just
come off the introductory free year of it's life).  I like being able to
provide this service to others, but not $20/month-like.  If I wanted to keep
Slacktrux available to my friends and coworkers, I needed to do *something*.
Enter the latest buzzword in computing: serverless architectures.

Put simply, a serverless architecture is one when you decompose your
application into a series of stateless functions that can be invoked as a
service.  Their ephemeral nature makes them cost efficient (i.e., you only pay
for the cycles you use), as well as very scalable (i.e., the functional
abstraction can scale horizontally to meet increased demand).  If I could
somehow rearchitect Slacktrux as a serverless application, I may well be able
to keep the foodtruck lights on, while keeping my fridge stocked with beer.

Slacktrux 101
-------------

First, however, a quick primer on the application I'm converting.  Slacktrux is
a Python-based WSGI application.  It exposes one REST endpoint that is invoked
via a Slack slash-command. When a POST is received from Slack, it downloads a
HTML page from a known-good foodtruck tracker and scrapes the listed trucks for
the requested neighborhood.  The list of relevant trucks is then sent back to
Slack via an incoming webhook.

The above is implemented in two discrete contexts:

1. Handling the incoming request.
2. Scraping the page and serving the results.

At the end of the first phase, Slacktrux returns a "Please wait" response to
the user, and spins the invocation of the second phase into an out-of-process
task.  This is because the latency associated with downloading a remote
resource and scraping it introduces unwelcome latency into a web endpoint.
While it's doable, it's also undersirable.

The task invocation in my WSGI implementation uses `Celery
<http://www.celeryproject.org/>`_, backed by RabbitMQ.  Overengineered?
Perhaps, but it was a fun learning experience at the time.

The Serverless Landscape
------------------------

Serverless architectures, or BaaS (Backend as a service), are springing up
everywhere. Amazon Web Services has Lambda, Google Cloud Platform has Cloud
Functions, and Microsoft Azure has Azure Functions.  They all support Python,
but, for the sake of convenience and knowledge portability (the organizations I
support at work are all-in on AWS), I chose Lambda.


