---
layout: post
title: "MongoDB queries don't always return all matching documents"
# date: XXX
---

## Stumbling onto strange Mongo behavior

These days my main job is building the backend of
[the Meteor Galaxy hosting service][galaxy]. We store a lot of data our
[MongoDB][mongo] databases, including the state of all of the containers we've
run.  Containers have a variety of states, like `starting`, `healthy`,
`unhealthy`, and `stopped`.

One of our services periodically polls the database and reads the list of
running containers with the query `containers.find({state: {$in:
['healthy', 'unhealthy']}})`. Running containers can flap back and forth between
`healthy` and `unhealthy`, but once they get changed to some other state like
`stopped`, they should never return to `healthy` or `unhealthy`.  So if a
container that was returned from one iteration of this query later disappears
from the query's results, we should never see it re-appear again.

While investigating a bug in the service, I realized that occasionally (a few
times a day), the service saw a container appear in the query's results,
disappear from the results when the query was run again, and then re-appear on a
third run. This was really surprising!  I figured maybe there was a bug in the
code that wrote the states that broke my assumptions about allow state
transitions.

One nice thing about MongoDB is that you can actually see the history of your
database, by running a query on the oplog. I looked for changes to this document
in the oplog and I saw only reasonable changes: the container went from
`starting` to `healthy` and then occasionally flipped back and forth between
`unhealthy` and `healthy`. So at every point after this document started
matching the query, it should have continued to match the query!  But... my
logging showed that it did not.  Occasionally, around the time that the
container flipped from `unhealthy` to `healthy`, it would fail to match the
`{state: {$in: ['healthy', 'unhealthy']}}` query.

But why?

## MongoDB: neither fish nor fowl

MongoDB occupies an interesting middle ground between systems like SQL databases
and sytems like key-value stores or Bigtable.

SQL databases offer powerful transactional guarantees and offer a query planner
that can run queries against various user-defined indexes, but you tend to lose
these guarantees when you shard data in order to scale.

In the pursuit of scalability, key-value stores and Bigtable don't let you
change arbitrary data in a single transaction. This generally means that they
don't have built-in indexes and query planning, and it's the your job to
structure data in a way that's efficient for the queries you need to make.

MongoDB is somewhere in the middle. On the one hand, the basic unit of atomicity
is the single document: you can make transactional writes to a document, but not
across documents. On the other hand, MongoDB does support indexes and has a query
planner that knows how to use them.

MongoDB has a
[long document describing its concurrency properties][concurrency-faq]. The
basic gist of it is that you should only expect consistency at the single
document level.  So it's not surprising that it provides
["Non-point-in-time read operations"][concurrency-faq-isolation]: if you modify
a few documents while executing a slow query on their collection, you might see
some of them in the state before the modification and some of them as already
modified.

What's a little more surprising is this caveat: "Reads may miss matching
documents that are updated during the course of the read operation".  Well, that
seemed to be exactly what I saw.  So what's going on?

## How MongoDB queries actually work

The MongoDB query planner is relatively straightforward.  (I'm going to ignore
things like geospatial and full-text indexes.)  Most queries are handled by a
single scan either over an entire collection or over a subset of an index.
There's no big lock taken out for the scan; it's possible that during the scan,
writes occur in the collection.  Writes won't happen while looking at a single
document, though.

If we're scanning over the whole collection, writes may change a document before
we get to it, or they may not, but they're not going to re-order the documents
of the collection.

But scanning over an index works differently!  The index is essentially a list
of document IDs, sorted first by the actual index keys and then by the ID
itself.  If a document is updated in a way that affects an index key, it
actually moves around in the index --- that's the whole point!

# XXX oplog for debugging

[concurrency-faq]: https://docs.mongodb.org/manual/faq/concurrency/
[concurrency-faq-isolation]: https://docs.mongodb.org/manual/faq/concurrency/#what-isolation-guarantees-does-mongodb-provide
[galaxy]: https://www.meteor.com/galaxy/
[mongo]: https://www.mongodb.org/
