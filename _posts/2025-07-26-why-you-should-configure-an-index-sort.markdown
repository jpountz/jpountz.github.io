---
layout: post
title:  "Why you should configure an index sort on your Lucene indexes"
date:   2025-07-26
---

Some time ago, I [wrote](https://x.com/jpountz/status/1907454969877291035) that "if you do not configure an index sort on your Lucene indexes, you are missing search-time efficiency benefits that are almost certainly worth the (low) index-time overhead".

For clarity, index sorting refers to the ability of configuring the way data is ordered on disk within segments.

One application of index sorting is early termination. For instance, it would make sense to sort a web index by descending `page_rank` and terminate searches after collecting X matches. Since the index is sorted by a-priori relevance, results collected so far would likely be quite good. Another application of index sorting - which I find more interesting since it's more generally applicable - is clustering similar documents together in the doc ID space.

How can you do this? Most datasets have natural dimensions that usually make good candidates for the index sort. If you have several of them, try to order them hierarchically (e.g. `country` before `city`), by importance (how likely they are used for filtering and faceting) and decending cardinality. For instance,
 - An e-commerce catalog could be sorted by `category`, then `brand` (within products of the same category).
 - A web index could be sorted by reverse domain name (e.g. `io.github.jpountz` for this blog) then URL path.
 - An logging dataset could be sorted by container ID, then file path, then descending timestamp.
 - A catalog of cars for sale could be sorted by  `type` (station wagon, SUV, ...), then `brand`, then `model`, then `fuel_type`, then `registration_year`, then `mileage` (just for the sake of giving an example with a longer sort).

If you are familiar with analytical databases/workloads, this likely sounds familiar to you. Making the data sorted on disk by some fields makes operations on these fields (including filtering, grouping, etc.) _much_ more efficient and is key to working with very large datasets.

Why does it help? There are several reasons, I'll give a few ones. First, if similar documents are clustered in the doc ID space then postings lists will also have clusters of doc IDs that are close to one another, with large gaps between these clusters. These gaps are important, because processing conjunctive queries (including filtered queries) in search engines boils down to advancing iterators in a leap-frog fashion. And if one iterator has a large gap between consecutive doc IDs, then the conjunctive query will naturally skip evaluating other clauses over this large gap. It's much better to have few very large gaps than many small gaps efficiency-wise.

Disjunctive queries also benefit from clustered postings lists. Disjunctive queries typically merge multiple iterators in a streaming fashion by using a heap. If doc IDs are clustered in postings lists, then the next minimum doc ID after the current one is more likely to belong to the iterator at the top of the heap than to another iterator. This results in less heap reordering overall. 

Another reason is that clustered postings lists compress better. Better decompression speed. Lower space requirements as well. And these lower space requirements also help better take advantage of the memory bandwidth.

It's worth noting that it will not only help queries on your index sort fields, but also queries on fields that correlate with your index sort fields. For instance, the terms that appear in the description of products of an e-commerce catalog are highly correlated with the category that the product belongs to. So postings lists of the `description` field will also be clustered. 

How much does it help? As always with this sort of thing, mileage will vary. One data point is this [benchmark](https://tantivy-search.github.io/bench/), which has two Lucene candidates: `lucene-10.2.0` and `lucene-10.2.0-bp`. The latter uses recursive graph bisection, a sophisticated doc ID reordering technique that effectively clusters similar documents together in the doc ID space, in a similar way as what you could do with index sorting, except that it works directly on postings lists. You can check out the various collection types, `lucene-10.2.0-bp` consistently performs faster than `lucene-10.2.0`, just by using a different doc ID order. I would expect even bigger speedups on more structured datasets.

Unfortunately, Lucene cannot configure an index sort on behalf of the user. Elastic has started initiatives to automatically configure an index sort (among other things) when indexing [logs](https://www.elastic.co/search-labs/blog/elasticsearch-logsdb-index-mode) or [metrics](https://www.elastic.co/docs/manage-data/data-store/data-streams/time-series-data-stream-tsds), but there are still many cases when input from the user is needed. Recursive graph bisection helps bring similar
benefits as index sorting and doesn't require input from the user, but it comes at a higher index-time overhead, so it's unclear to me if it could become the default at some point. In the meantime, consider configuring an index sort on your Lucene (and Solr, Elasticsearch) indexes!
