---
layout: post
title:  "An ode to self-optimizing query plans"
date:   2026-06-04
---

Common wisdom is that databases should make decisions at the query planning stage and then hand over the plan to a dumb executor. This has several nice properties. Separation of concerns helps each tool focus on one thing and do it right: computing the cheapest possible plan in the planner, and squeezing the best efficiency out of the hardware in the executor. It's easy to introspect how the database evaluates a query by simply looking at the produced plan. It's easier to decompose complex queries into reusable building blocks. And more.

But making the right decision up front is hard. Cost models help us compare plans, but they rely on simplifications: they may evaluate I/O costs and ignore CPU costs, they may assume that filter selectivity is uniform across the dataset, or that the various conjuncts of an And filter are not correlated. Sometimes we use thresholds and weights tuned on a representative set of queries. There are many reasons why a planner may not always make the optimal decision.

Often, such up-front planning is the best approach, but there are also cases when you can do better. I'll share a few examples I'm familiar with from the Apache Lucene codebase.

### Vector search filtering with ACORN-1

It's possible to apply a filter with standard HNSW search, by browsing the HNSW graph as usual, and only inserting nodes into the top-k heap if they pass the filter. This approach works well with unselective filters. However, if the filter is selective, it becomes very hard to find good vectors and HNSW search either gets slow or sacrifices recall. ACORN-1 makes things better by not only looking at the immediate neighborhood of a node to generate candidates, but also neighbors of its neighbors. This disappointingly simple idea works very well in practice.

One way of integrating ACORN-1 would consist of letting the planner look at the selectivity of a filter. For instance, if a filter matches less than X% of the vectors then use ACORN-1, otherwise use standard HNSW search. Lucene took a different approach: instead of making a global decision based on the selectivity of the filter, it makes a [local decision](https://github.com/apache/lucene/blob/d1f2aced7ff0136c872359713e9d17ce7bca3b09/lucene/core/src/java/org/apache/lucene/util/hnsw/FilteredHnswGraphSearcher.java#L164-L189) based on the selectivity of the filter for the specific neighborhood it's currently exploring.

I like this approach because it's more robust against cases when there is correlation between the query and the filter. For instance, say your query is "computers" filtered by "category: books" because you're looking for a book on the history of computers. If there are lots of books in your catalog, this filter may be considered unselective, and a planner would suggest using standard HNSW search. However, most matches for computers may be in "category: electronics", so HNSW search may be looking at neighborhoods that have lots of nodes that don't match the filter. When this happens, the Lucene approach falls back to ACORN-1 for exactly those neighborhoods.

### When WAND fails

[Block-max WAND](https://www.elastic.co/blog/faster-retrieval-of-top-hits-in-elasticsearch-with-block-max-wand) speeds up BM25 retrieval by using per-block score upper bounds to skip over documents that cannot make it into the top-k. It's great on short, high-impact queries, but it's also easy to make block-max WAND perform slower than exhaustive search, e.g. on long queries. In such cases, WAND spends lots of cycles recomputing which documents can be skipped only to realize that it can't skip much. Plus several optimizations for exhaustive evaluation cannot be applied with WAND, making it noticeably slower.

One approach to solve this would consist of having a planner that categorizes the query based on whether it's expected to perform better via block-max WAND or exhaustive evaluation. Lucene took a different approach and engineered [its BM25 retrieval algorithm](/2025/10/11/vectorized-evaluation-of-disjunctive-queries.html) so that it can skip nearly as effectively as block-max WAND while evaluating hits nearly as efficiently as exhaustive evaluation.

### Top-k hits by field via dynamic filtering

To compute the top-k hits by a field, a traditional database may consider two plans:
 - iterate over the data in order of the field used for sorting (thanks to the index of this field), stopping after finding k hits that match the filter,
 - iterate over the data in storage order, collecting hits that pass the filter in a heap.

The first approach is faster on unselective filters, the second one is faster on selective filters, so the planner is responsible for guessing which approach to use based on filter selectivity estimates.

Lucene always evaluates such queries by iterating over the data in doc ID order, but it adds a dynamic filter that matches hits that compare better than the current top-k-th hit. Such a range filter with a dynamic bound is not trivial to implement, but it results in a single query plan that naturally optimizes itself based on the selectivity of the filter:
 - if the filter is selective, then few hits will need to be collected in total anyway and the query is efficient,
 - if the filter is unselective, then the top-k-th hit likely converges quickly, which in turn means that the dynamic filter quickly becomes very selective, which helps the query evaluate efficiently.

### Conclusion

This isn't meant to be a critique of traditional query planning, [Lucene does a bit of this as well](https://www.elastic.co/blog/better-query-planning-for-range-queries-in-elasticsearch). But I like these examples because each of them could have been solved more easily by leaning on the planner, and Lucene chose the harder path of building a single plan that self-optimizes from actual data. The payoff is graceful degradation: when statistics are wrong, when filters and queries are correlated, when the corpus drifts, these self-optimizing plans react to what they see rather than to what the planner guessed up front.
