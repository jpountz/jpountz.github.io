---
layout: post
title:  "Should your search engine use segments for vector search?"
date:   2025-05-27
---

Inverted indexes, as a data structure, are hard to update. This is why many search engines, including Lucene, don't update them in-place and instead organize their data into segments that store an immutable inverted index and a mutable set of live docs. Adding more documents requires adding a new segment to the index, deleting documents requires marking documents as deleted, and background merging helps keep the total number of segments under control. You're probably already familiar with this.

To combine vector search and keyword search capabilities, you then have two choices: you can store your vector search data structures within segments, just like inverted indexes, or you can store your vector search data structures at the top level, independently of the segmented structure of the inverted index. In the case of Lucene, vector search data structures are stored within segments.

This design has raised questions, both internally and externally, as segments are not friendly to [HNSW](https://en.wikipedia.org/wiki/Hierarchical_navigable_small_world), which is the basis for some of the fastest vector search data structures, and the one currently implemented in Lucene's default codec. On the other hand, HNSW graphs support in-place modifications. I still believe that storing vector search data structures in segments is the right design for Lucene. I already [argumented in favor of it a while ago](https://www.elastic.co/search-labs/blog/vector-search-elasticsearch-rationale) but I see the question come back from time to time and I've had more time to think of the trade-off at a higher level, so I'm giving it another try, this time focusing more on the implications for the direction of Lucene than for users.

A few paragraphs ago, I wrote that live docs are mutable. This is actually a lie in the case of Lucene. When the live docs of a segment need to be updated, a new set of live docs is created, and a new segment is produced that points to the existing index and this new set of live docs. This is how Lucene was originally designed: it is built on top of a write-once filesystem abstraction that doesn't support updating files in-place or even appending to an existing file, so updates need to perform some form of [copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write). Everything that Lucene introduced since then follows the same pattern: points (KD trees) are part of segments, doc-value updates work the same was as live docs, etc. This doesn't mean that Lucene shouldn't have introduced support for in-place updates of files in order to have better support for vector search, but it's important to understand that these limitations also introduce interesting properties:
 - There is zero contention between updates and search by design, since search always operates on immutable data-structures.
 - It is easy and relatively cheap to keep referring to the same point-in-time view of the index throughout a potentially long search session, it will not be affected by concurrent updates.
 - Changes to the index can be replicated by copying files (segment-based replication) rather than by replicating operations.

Segment-based replication is especially interesting because it enables a very different architecture at a higher level. Let's consider the following architectures:

Architecture 1:
 - Embrace segments for all the data, including vectors.
 - Segment-based replication.

Architecture 2:
 - Vector data structures stored on their own.
 - In-place updates for columnar storage or vectors.
 - Operation-based replication.

These architectures has very different trade-offs, and it's worth noting that it doesn't make much sense to do something in-between or you're more likely to get the downsides of both than the benefits of both. For instance, you can't do segment-based replication and have realtime search.

The big advantage of architecture 1 is how it makes search easier to scale out/in. Need more search capacity? Just add more nodes, replicate segments to them and they can start serving queries. Over-provisioned? Just claim back nodes that you don't need. It comes with a few other interesting properties:
 - The indexing cost doesn't scale with the number of search replicas.
 - Indexing and search can be decoupled and run on different hardware. This allows running heavy work on the indexing tier without harming the search tier. Think of things like aggressive merging to have a low number of segments, or sophisticated doc ID reordering like recursive graph bisection.

But architecture 2 also comes with compelling properties. Most notably, in-place updates can be much cheaper than with architecture 1. But also:
 - It can support true realtime search by searching directly into data that is still in memory, while architecture 1 supports near-realtime search at best.
 - Supporting realtime search further reduces the cost of updates, since you no longer need to publish segments just to make changes visible. So you save on merging.
 - Vector indexing and search can be more efficient since it's not tied to segments.

Whichever one is the best will depend on your use-case. My intuition is that the vast majority of use-cases could work just fine with either architecture, so the choice will likely be made on other factors. However, I think that there are interesting takeaways for Lucene and Lucene-based search engines like Solr and Elasticsearch.

## Takeaways for Lucene and Lucene-based search engines

Given how Lucene embraces segments for everything, Lucene-based search engines should prefer implementing architecture 1 and perform segment-based replication. Otherwise you're getting an efficiency loss for vector indexing/search at the Lucene index level without getting the replication efficiency benefits of segment-based replication.

In my opinion, there is much more value to create for Lucene by enabling applications to implement architecture 1 right, than by giving them the option to do 1 or 2. This means that we shouldn't spend too much energy trying to support a vector datastore at the top level, do in-place doc-value updates, or support [realtime search](https://github.com/apache/lucene/issues/3388) (though realtime search may still be useful for embedded use-cases, e.g. desktop applications that want search capabilities)?

Decoupling indexing and search makes sense once you do segment-based replication. So maybe Lucene doesn't need to care as much about merges disturbing searches, e.g. is merge throttling actually important? And maybe defaults should be geared more towards faster search, e.g. more aggressive merging defaults?

Finally, we should document these architecture best practices in Lucene: segment-based replication, decoupled indexing and search, etc.
