---
layout: post
title:  "More on Vespa vs. Lucene/Elasticsearch"
date:   2025-07-25
---

In a previous [post](/2025/06/17/analysis-of-Elasticsearch-vs-Vespa.html), I took a look at the Vespa vs. Elasticsearch benchmark that the Vespa people run. The results made me want to dig a little deeper to see how Vespa and Lucene/Elasticsearch differ implemetation-wise. I took advantage of Vespa being open-source under the Apache License 2.0 to take a look at the [source code](https://github.com/vespa-engine/vespa). I found some similarities and differences that I expected, but also some that I did not expect!

Like Lucene/Elasticsearch, Vespa is a large codebase that takes time to get familiar with, so there are chances that I misundertood some things about Vespa. Feel free to let me know if you spot any mistake.

## Vespa

Like Lucene, Vespa builds on HNSW and segmented inverted indexes, evaluates queries in a document-at-a-time fashion, dynamically prunes low-scoring hits to speed up the evaluation of top-k queries, etc. All queries must be able to produce doc-ID-ordered iterators of matches, including boolean queries, so you can flexibly compose queries however you want. Disjunctive queries combine iterators using a heap, conjunctive queries advance iterators in a leap-frog fashion. This makes Lucene and Vespa similar in many regards.

Vespa's code is pretty clean and well organized into reusable abstractions. For instance, it has an internal implementation of a mutable yet slow-ish inverted index, another one of a fast yet immutable segmented inverted index, which are then combined to get the best of both worlds (fast and mutable), similarly to how LSM trees buffer data in memory and later flush it to disk.

Obviously, the way how Vespa prioritizes realtime-ness when Lucene prioritizes incremental replication of indexes is a first big difference. But there are more differences (and similarities) that are worth looking into.

### Slow query handling

Textbook information retrieval works great for full-text search but tends to fall short in the real world when needing to work with more complex queries, such as queries on unindexed fields. Computing the next matching document on such queries may require doing a linear scan. Lucene and Vespa have different solutions to this problem that are based on similar ideas.

Lucene allows queries to be decomposed into a fast approximation and a slower per-document check called `TwoPhaseIterator`. For instance, in the case of a phrase query, the approximation would match documents that contain all phrase terms, and the per-document check would read positions to check if these terms can be found at consecutive positions. This helps conjunctive queries first find agreement between all approximations before running the more expensive per-document checks. For instance, a phrase query on "Lucene in action" filtered by the "Books" category would only evaluate positions of the "Lucene in action" phrase query on documents that not only contain all 3 terms ("Lucene", "in" and "action") in the searched field, but also "Books" in the category field.

Vespa takes a different approach and allows search iterators to be not "strict", which effectively allows iterators to not advance to the next match if the target document doesn't match. Said otherwise, doing `iterator.seek(target)` (`iterator.advance(target)` in Lucene) would set the current doc ID to `target` if it matches, and may otherwise not advance the iterator if `target` doesn't match. Many queries can produce either strict or unstrict iterators, and the decision is made based on how selective required clauses are. This is similar to how Lucene passes a `leadCost` information at `Scorer` creation time to allow queries to optimize the iterators that they produce, which is [how `IndexOrDocValuesQuery` works](https://www.elastic.co/blog/better-query-planning-for-range-queries-in-elasticsearch).

### Dynamic pruning

Vespa does dynamic pruning via the WAND algorithm. The Lucene story is a bit more complicated, it uses block-max WAND in the general case and (a variant of) block-max MAXSCORE in optimized cases (what Lucene calls "bulk scoring"), but there are now so many optimized cases that it's fine to assume that Lucene uses block-max MAXSCORE almost all the time.

Lucene and Vespa's implementations of WAND are quite different from the textbook WAND implementation, yet similar to one another!

First, instead of maintaining a sorted list of iterators, they organize iterators into:
 - a list of iterators positioned on the pivot doc ID (called "lead" in Lucene, "present" in Vespa),
 - a heap of iterators after the pivot doc ID (called "head" in Lucene, "future" in Vespa) sorted by doc ID,
 - a tail of iterators before the pivot doc ID (called "tail" in Lucene, "past" in Vespa) sorted by cost.

Implementing WAND then boils down to advancing iterators while maintaining the WAND invariants, notably the sum of score upperbounds of the tail/past heap must never exceed the minimum competitive score, or that would mean that you could miss matches. I don't remember seeing this approach documented anywhere, and looking at git logs of both projects suggests that they came up with it independently, which is fascinating. If I missed the paper that documents this approach, please share it with me. Vespa stores these 3 logical collections into a single array quite elegantly, while Lucene uses a linked list and two heaps.

Second, both Lucene and Vespa integrate their WAND implementation with their support for slow queries, in order to do as little work as possible on documents that do not match filters. Typically, if the filter is rather selective, the WAND iterator will only check if the provided candidate document may be competitive, and not spend too much effort finding a good next candidate since it's better to let the filter provide the next candidate. Again, I don't remember this being documented anywhere. It shows that both projects have been thinking hard about combining dynamic pruning and filtering, a hard problem that isn't getting much attention.

Finally, both Lucene and Vespa use fixed-point numbers to track score upper bounds, to be able to add and subtract aggregates of score upper bounds without accumulating accuracy losses due to floating-point arithmetic.

My main surprise there was to not see block-max indexes in Vespa. They allow WAND to works with local upper bounds of the score instead of global upper bounds of the score, something that has proved to help query evaluation significantly on several serious benchmarks. Block-max indexes also help perform some skipping with single term queries, which is not anecdotal as queries on a common word are otherwise worst-case scenarios from the perspective of retrieval efficiency.

### Postings lists

Lucene splits postings lists into blocks of 128 doc IDs. Full blocks are encoded with FOR-delta or as a bit set, whichever takes the least space. Tail blocks (less than 128 docs) are encoded using group-varint. Vespa encodes postings lists as delta vInts. Both interleave term frequencies and postings into the same file.

Lucene stores field lengths in a separate file, while Vespa denormalizes field lengths into the inverted index, interleaved with postings and frequencies. This likely gives Vespa better memory locality in exchange for higher storage requirements, since field lengths need to be duplicated for every term of the same field and document. It would be interesting to evaluate how doing the same in Lucene would affect retrieval efficiency.

Lucene has 2 levels of skip lists, every 128 and 4,096 postings. Vespa has 4 levels of skip lists, every 16, 128, 1,024 and 8,192 postings.

### Vector search & filtering

Both Lucene and Vespa use HNSW, support pre-filtering, and implement [ACORN-1](https://www.elastic.co/search-labs/blog/filtered-hnsw-knn-search) to speed up pre-filtering.

In Vespa, various thresholds can be configured to tell Vespa how to apply the filter based on how selective it is. It supports the following options:
  - exact search on matches of the filter (for very selective filters),
  - filter-first HNSW search (for rather selective filters),
  - normal HNSW search (for filters that match many docs),
  - as a post-filter (for filters that match most documents) by scaling the number of candidates to retrieve by the inverse of the selectivity of the filter.

In contrast, Lucene doesn't rely on thresholds. It starts performing an HNSW search. At every step of the HNSW search, it decides whether to explore the extended neighborhood (ACORN-1) depending on how much of the immediate neighborhood got filtered out.  If at some point, the number of vector comparisons exceeds the number of matches of the filter, then it falls back to an exact search on the matches of the filter.

The Lucene approach likely better copes with worst-case scenarios, when the filter matches many docs, yet the graph needs to be deeply explored to find enough nearest neighbors that match the filter - something that cannot be captured by thresholds. However, it may need to run up to 2x more comparisons than necessary, and I wouldn't be surprised if simple heuristics like Vespa's worked well on average in practice.

## Takeaways

I could tell that Vespa makes significant efforts in combining retrieval with filtering, both for lexical search and vector search. Like Lucene. This is interesting because this topic is surprisingly not covered much in the literature. What is even more interesting is how Lucene and Vespa independently converged to similar solutions to the problem of combining dynamic pruning and filtering, without any coordination as far as I can tell.

I had never considered denormalizing length normalization factors into postings lists. I am now curious to evaluate how it affects performance, it should be rather easy to hack in Lucene to get an idea.

On the other hand, I was expecting postings to be encoded in a more modern way (such as PFOR-delta or partitioned Elias-Fano) and support for block-max indexes. Both these changes resulted in significant improvements when they got released in Lucene. I am curious if there is anything that would prevent these improvements, or if the Vespa team simply never got to them.

Another thing I could not find in Vespa is support for index sorting, which I consider one of the most powerful (and under-used) tools of the Lucene toolbox.

I found the different approaches to pre-filtering interesting. In general, I don't like thresholds and prefer more dynamic approaches like Lucene's. That said, Lucene is likely a bit too defensive and could identify cheap heuristics when an exact search would perform faster with extremely high likelihood without running a HNSW search first.
