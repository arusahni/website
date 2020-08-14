.. title: Elasticsearch Frustration: The Curious Query
.. slug: elasticsearch-frustration
.. date: 2017-04-23 16:23:28 UTC-04:00
.. tags: tech, elasticsearch
.. link:
.. description: In which I describe a frustrating Elasticsearch issue and my solution.
.. type: text

Last year I was poking at an Elasticsearch cluster to review the indexed data
and verify that things were healthy. It was all good until I stumbled upon this
weird document:

.. code:: javascript

  {
    "_version": 1,
    "_index": "events",
    "_type": "event",
    "_id": "_query",
    "_score": 1,
    "_source": {
      "query": {
        "bool": {
          "must": [
            {
              "range": {
                "date_created": {
                  "gte": "2016-01-01"
                }
              }
            }
          ]
        }
      }
    }
  }



It may not be immediately obvious what's going on in the above snippet.
Instead of a valid :code:`event` document, there's a document with a query as
the contents. Additionally, the document ID appears to be :code:`_query`
instead of the expected GUID. The combination of these two irregularities makes
it seem as if someone accidentally posted a query to the wrong endpoint. No
problem, just delete the document, right?

.. code::

  DELETE /events/event/_query
  ActionRequestValidationException[Validation Failed: 1: source is missing;]

Wat.

.. TEASER_END

I reached out to some of my coworkers to see if they could point me in the
right direction, but all that I received was an (unhelpful) "I've seen this
error before, and we solved it, but no one seems to remember how it was done."
Great.

After much head-scratching, it turns out that, since the ID is :code:`_query`,
Elasticsearch's URL router thinks that I'm trying to issue a query and
validates the HTTP action as such. Part of that validation is the requirement
that queries have a body. Oops.

While passing an empty object should conceivably have worked, I wanted to play
things extra safe in case ES was executing the query (this *was* production,
after all (why are you looking at me like that?)), so I passed in a query
object that constrained the results to only the problematic document.

.. code::

  DELETE /compositeevents/compositeevent/_query
  {
      "query": {
          "match": {
              "_id": "_query"
          }
      }
  }

... and the document was deleted successfully! Hopefully putting this to blog
form will help others who encounter it in the future (including me).


