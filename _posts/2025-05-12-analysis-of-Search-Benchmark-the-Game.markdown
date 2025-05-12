---
layout: post
title:  "An analysis of Search Benchmark, the Game"
date:   2025-05-12
---

"Search Benchmark, the Game" is maintained at [https://github.com/quickwit-oss/search-benchmark-game](https://github.com/quickwit-oss/search-benchmark-game) by the Tantivy folks and published at [https://tantivy-search.github.io/bench/](https://tantivy-search.github.io/bench/). I don't know the full history behind this benchmark, GitHub says that this repository was started by Jason Wolfe in 2018 as a comparison between Lucene and (early versions of!) Tantivy before getting forked and more actively maintained as of 2023.

It's a simple benchmark. It runs queries from the [AOL query dataset](https://en.wikipedia.org/wiki/AOL_search_data_leak) against an export of English Wikipedia. Additional queries have been added over time that have interesting performance characteristics, queries that only contain stop words for instance. Adding an engine is simple too, you just need to create an executable that creates an index from content provided on the standard input, and another executable that takes a command and a query on the standard input, and runs it against this previously generated index. The following engines are checked in at the moment:
 - [Apache Lucene](https://lucene.apache.org/), a popular search library in Java. It comes in two flavors, `lucene-10.2.0`, which indexes docs in no specific order, and `lucene-10.2.0-bp` which reorders doc IDs with recursive graph bisection and is discussed a bit more further below,
 - [Tantivy](https://github.com/quickwit-oss/tantivy), a search library in Rust that shares a lot of its design with Lucene,
 - [PISA](https://github.com/pisa-engine/pisa), a C++ search library that is designed at testing out new techniques to improve search performance,
 - [Bleve](https://github.com/blevesearch/bleve), a Go search library managed by Couchbase,
 - [Rucene](https://github.com/zhihu/rucene), a Rust port of Lucene,
 - [Bluge](https://github.com/blugelabs/bluge), a Go search library.

Two libraries are specially interesting to me beyond Lucene:
 - PISA, because it happily trades anything in favor of faster search, so it sets a good north star in terms of search performance.
 - Tantivy, because it is actively maintained and shares so much of its design with Lucene that seeing much better performance with Tantivy than Lucene indicates that Lucene likely has opportunity for doing better.

## Benchmark trade-offs

Benchmarks naturally have biases, as a function of what they run and what metrics they collect. In my opinion, this specific benchmark is pretty reasonable, but I'll call out some trade-offs that this benchmark makes for reference:
 - Data is read-only. This makes the problem quite simpler compared with searching read-write data. That said, read-only datasets - or datasets with rare updates - exist in the real world, it is quite common for datasets to be read-only in search benchmarks, and it makes the benchmark easier to reproduce, so it's not unreasonable.
 - There are no deletions in the index. This one is worth noting because deletions on an index make some optimizations a bit harder to apply or a bit less efficient. Specifically, vectorizing query evaluation becomes a bit harder once you have docs that need to be filtered out in postings lists.
 - Publicly available results were produced on an AWS c7i instance. This instance type supports AVX-512 instructions, which gives an advantage to engines that are capable of taking advantage of vectorization vs. engines that are not.
 - All queries run against a single field. One could argue that queries more often target multiple fields (e.g. `title` and `body`) in the real world. That said, it's quite standard for search benchmarks to run against a single field, so again nothing too questionable here. That said, it's worth noting that searching across multiple fields is a very different problem depending on how you want to combine scores (e.g. BM25F).
 - The benchmark collects an average latency, which is a good proxy for throughput, as well as 50th, 90th and 99th percentiles, which are good indicators for tail latencies.
 - The benchmark does not report index size or indexing time. So you could beat current top engines by creating one that trades index size and/or indexing time in favor of faster search and it would look better, even though it may not be practical. For instance, Recursive Graph Bisection, a doc ID reordering technique used both by the `lucene-10.2-bp` and the `pisa-0.8.2` engines is a heavy contributor to the indexing time.
 - All data fits in the page cache. Again, this is rather standard for search benchmarks, but some real-world use-cases need to deal with data residing on slower storage like NVMe drives (or even object storage, something that Quickwit tries to optimize for).
 - The dataset contains about 6M documents. This is not small, but not big either. One could imagine that not all engines would scale as well if you increased the size of the dataset by 10x or 100x.
 - The `COUNT` collection type is not that useful in practice. I imagine that it was initially introduced as a way to benchmark exhaustive query evaluation when scores are disabled, something that happens in the real world when computing facets for a query, e.g. counts per category on an e-commerce catalog. But some engines now have `COUNT`-specific optimizations that would not apply to more general faceting.

## Interesting queries and optimizations

### WAND and MAXSCORE

To have a chance of performing well with the `TOP_*` collection types, engines have no choice but to support dynamic pruning and skip evaluating documents whose score would have no chance of making it to the final top hits. Tantivy and PISA implement block-max WAND while Lucene implements block-max MAXSCORE, two state-of-the-art for dynamic pruning.

### Term query on a stop word (`the`) and `TOP_*` collection types

Traditional dynamic pruning algorithms like WAND and MAXSCORE focus on disjunctive and conjunctive queries. Handling queries with a single term is easy, but you can tell from the results of the `TOP_10` collection type that PISA doesn't specialize this case.

### Query with only stop words (`to be or not to be`)

This query exists in its disjunctive, conjunctive and phrasal forms in the benchmark. It's a hard query because all terms exist in many if not most documents. So not only is exhaustive evaluation hard because the query matches many terms, but dynamic pruning is hard as well because there is no term whose contribution to the score significantly dominates others. 

One thing that makes it obvious that this query is hard is that both Tantivy and PISA are slower at evaluating this query with the `TOP_100` collection type than with the `TOP_100_COUNT` collection type. This suggests that the overhead of dynamic pruning is higher than the savings from evaluating fewer hits.

One optimization that Lucene applies to this specific query that other engines don't apply to my knowledge is term deduplication. It effectively rewrites `to be or not to be` as `to^2 be^2 or not`, a query that gives that same hits and scores, but only needs to process 4 postings lists instead of 6.

### Query with a stop word and a rare term (`the incredibles`) and `TOP_*` collection types

The idea behind dynamic pruning is to skip documents that only contain low-scoring terms. But you can only do this safely if your best k-th score so far is greater than the contribution of these low-scoring terms. This typically starts happening after visiting `k` documents that contain high-scoring terms. With queries such as `the incredibles`, this happens pretty late during query evaluation because `incredibles` is a rare term, so it takes a long time before k documents containing this rare term are found. It may get even worse if recursive graph bisection clustered documents containing these rare terms towards the end of the doc ID space.

One well-documented optimization to avoid this worst-case scenario is to evaluate the top-k-th score up-front, but none of the engines use this optimization to my knowledge.

### Queries mixing stop words and standard terms (`lord of the ring`, `battle of the bulge`, `jesus as a child`) and `TOP_*` collection types

These queries are good at measuring the ability of engines to drive query evaluation by terms with a higher contribution to the score, especially as they have 4 terms (the more terms, the harder dynamic pruning gets).

### Filtered phrase query (`+"the who" +uk`)

This query was specifically added to highlight Lucene's support for two-phase evaluation, which breaks down phrase queries into an approximation which matches all phrase terms, and a verification step that checks if these phrase terms occur at consecutive positions. This allows first intersecting the approximation with other required clauses (`uk` here) before checking positions of phrase terms (`"the who"` here).

### Query on a whole paragraph

This query `a search engine is an information retrieval software system designed to help find information stored on one or more computer systems` is bigger than your typical query string, but not unseen, e.g. users may copy-paste quotes to find who said it first, or error messages to find how to resolve the error. Furthermore, such long queries are becoming more common with RAG use-cases to retrieve relevant content for a given prompt. In practice, applications may take shortcuts to avoid too high latencies for such long queries, but it's still interesting to see how well such queries are evaluated as-is.

This query challenges how well the various collection types scale with the number of terms. Something that is interesting with the `TOP_*` collection types is that Tantivy and PISA use the block-max WAND algorithm for dynamic pruning while Lucene uses block-max MAXSCORE. These algorithms are based on the same idea of skipping evaluating documents that only contain low-scoring terms, but WAND re-evaluates which terms are optional on every document while MAXSCORE only does this on a per-block basis. In practice, this means that WAND can skip more documents, but comes with a higher overhead to query evaluation. So WAND is faster than MAXSCORE when the amount of work saved from evaluating fewer documents is greater than the overhead of finding documents that may be skipped in the firt place. As dynamic pruning becomes harder and harder as the number of query terms grows, the fact that Lucene is faster at this query than Tantivy and PISA likely has to do with its use of MAXSCORE instead of WAND.

### PISA's recursive graph bisection

If you wonder how PISA can be so fast, there are multiple factors that contribute to it but the main one is recursive graph bisection. Recursive graph bisection is an index-time algorithm that clusters similar documents together. It works by recursively splitting the doc ID space in halves and swapping documents across both sides when the terms that they contain occur more frequently on the other side than on the side they're already in.

This clusters similar documents together, which is something that query evaluation naturally takes advantage of, without needing any modification. Postings lists have longer gaps, which helps conjunctive queries. Blocks of documents have more uniform score impacts, which helps dynamic pruning.

Lucene later [adopted recursive graph bisection](https://github.com/apache/lucene/pull/12489), you can tell the search-time benefits of recursive graph bisection by comparing `lucene-10.2` with `lucene-10.2-bp` (`bp` stands for Bipartite graph Partitioning, another name of recursive graph bisection).

### Tantivy's COUNT optimization

Like Lucene, Tantivy evaluates disjunctive queries by OR-ing windows of matches of sub queries into a bit set, and then iterates set bits of this bit set and feeds them into the collector. This is an efficient way of combining multiple sorted streams of doc IDs into a sorted stream that matches the union of the disjunctive clauses, especially when one or more clauses have dense matches.

For the `COUNT` collection type, Tantivy goes one step further by doing a [population count](https://en.wikipedia.org/wiki/Hamming_weight) on the bit set instead of linearly iterating set bits. Lucene later [adopted this optimization in version 9.8](https://github.com/apache/lucene/pull/12415).

### Lucene's encoding of dense postings blocks as bit sets

This is a recent optimization at the time of writing this blog. Given that disjunctive queries are evaluated by loading matches into a bit set, we could store dense blocks of postings as bit sets directly in the index as well so that loading matches into a bit set would just be a matter of OR-ing bit sets. Combined with Tantivy's COUNT optimization, this allows counting hits of some disjunctive queries with less an one instruction per match, hence why Lucene performs faster than Tantivy and PISA on some queries and the `COUNT` collection type.

This worked so well for disjunctive queries that Lucene does something similar for conjunctive queries when all clauses are dense (e.g. `+new +york +population` in this benchmark), by loading windows of matches of each clause into bit sets and AND-ing these bit sets together. 

It is likely possible to better take advantage of this optimization for the `TOP_*` and `TOP_100_COUNT` collection types, something that Lucene doesn't do at the moment. Also, while this optimization gives Lucene an advantage at the moment, I would expect Tantivy to do something similar and catch up in the future, similarly to how Lucene borrowed Tantivy's `COUNT` optimization in the past.

One thing that is fascinating to me is how this sort of optimization is only coming up now when traditional databases have had [bitmap indexes](https://en.wikipedia.org/wiki/Bitmap_index) for decades. I suspect that there are many such cases when search engines could learn more from traditional databases, and possibly vice-versa.

## Closing thoughts

This benchmark does a good job at highlighting how some design choices affect query evaluation efficiency. It's been interesting to see where PISA or Tantivy performed better than Lucene, and find what could be learned from them. Two engines I wish were added are PostgreSQL's full-text search capabilities and Vespa, as I'm sure that there are things to learn from those as well.

It would also be interesting to extend this benchmark to see how performance evolves when there are deletions in the index, or when queries are filtered. Two cases that are very frequent in the real world, but don't receive as much attention from Information Retrieval researchers.
