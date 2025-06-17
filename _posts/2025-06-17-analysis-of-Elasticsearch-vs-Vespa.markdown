---
layout: post
title:  "A look at the Vespa vs. Elasticsearch benchmark"
date:   2025-06-17
---

I was attending [Berlin Buzzwords](https://2025.berlinbuzzwords.de/) today and someone asked me about the [Elasticsearch vs. Vespa comparison](https://blog.vespa.ai/elasticsearch-vs-vespa-performance-comparison/) produced by the Vespa people, so I thought I'd publish my thoughts.

## Disclaimers

I haven't run the benchmarks myself to reproduce its results, but the data looks plausible to me so I'm assuming that it has been run in good faith. Also I don't know much about Vespa so I'm very happy to be corrected if I'm making wrong assumptions about it. Even though the benchmark targets Elasticsearch rather than Lucene, most of the comparison relates to decisions that have been made at the Lucene level, which is why you'll see more mentions of Lucene than Elasticsearch, the comparison would likely be similar with any Lucene-based search engine. 

I'll naturally be spending more time on some questionable aspects of this benchmark, but in general it looks like great care has been taken about making it an apples-to-apples comparison. Thanks to the Vespa team for that, most benchmarks don't have this level of care.

Finally, after so many years working on Lucene and Elasticsearch, I likely have biases!

## Introduction

As a Lucene developer, Vespa is interesting to look at because it has taken a different route even though it tries to solve the same problem. It's a bit like studying how evolution led to very different plant and animal species on different continents despite similar living conditions. Lucene embraces its write-once `Directory` abstraction that prevents in-place updates of files. So deletes and doc-value updates are effectively implemented through copy-on-write, and document updates are implemented through atomic delete-then-add operations. On the other hand, Vespa happily performs in-place modifications of files. Both engines then  capitalized on these initial design decisions, Vespa by supporting realtime search, and Lucene by enabling efficient incremental replication of data at the file level.

## Write and update performance

The benchmark highlights a downside of Lucene's near-realtime search compared with Vespa's realtime search: publishing changes requires segments to be flushed, and these segments then need to be merged in order to keep the total number of segments under control. These flushes and merges add up to a non-negligible overhead. This downside is further amplified by the fact that vector merging is quite slow. This makes Vespa perform better when changes need to be published frequently.

On the other hand, it looks like Lucene's ability to better batch indexing operations makes it perform better on the append-only workload.

I wasn't sure if there is a reason why the full reindex runs in-place. It would be more natural to me to index into a brand new index, then update the alias that is used for searching to point to the new index, and finally to remove the old index. It should also perform significantly faster.

## Query performance

Many queries are filtered in the real world, whether it's by tenant ID, category, ... yet filtering is the dead angle of many search performance benchmarks, so I like that this benchmark focuses on filtering.

I also like that the benchmark searches across multiple fields, another dead angle of many search benchmarks. But taking the maximum similarity across both fields doesn't feel like the best approach, something like BM25F would be more robust? This is a problem with Elasticsearch as well: the more robust approach for querying multiple fields, the [`combined_fields` query](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-combined-fields-query) - which does BM25F under the hood - isn't clearly documented as more robust than the `multi_match` query - which takes the max score across fields by default.

Finally, the benchmark says it didn't use RRF because this feature was still marked as experimental in the Elasticsearch docs, I hope that the next run of the benchmark will use RRF, which should again be a more robust way of combining lexical and semantic hits. I would also expect performance to improve for Elasticsearch this way, as the way that the vector query is put as a SHOULD clause of the `bool` query likely disables dynamic pruning in practice (an issue that the [linear retriever](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/retrievers#linear-retriever) doesn't have).

I am curious how Vespa manages to be ~2x faster (both latency-wise and throughput-wise) than Elasticsearch in the single-client force-merged unfiltered semantic-search case. In practice, both should be using a single HNSW graph with similar construction parameters, so I was expecting closer performance numbers. Maybe Lucene has some missing optimizations there?

## Conclusion

Elasticsearch and Lucene are fast-moving targets these days. It will be interesting to see what numbers now look like after [improvements to pre-filtering](https://www.elastic.co/search-labs/blog/filtered-hnsw-knn-search), [faster vector merging](https://www.elastic.co/search-labs/blog/hnsw-graphs-speed-up-merging) , or optimizations to `DisjunctionMaxQuery` (the query that takes the max similarity across several fields, see annotation [HP](https://benchmarks.mikemccandless.com/DismaxOrHighHigh.html)).

Hopefully the benchmark can also be improved a bit to run more robust queries, e.g. BM25F and RRF instead of max-similarity and a linear combination of vector and lexical scores.

Yet Lucene and Elasticsearch still have things to learn from this benchmark. The most important one is probably that we should better benchmark the "searching while indexing" case, with a low refresh interval. There are likely some relatively low hanging fruits there, such as enabling [merge-on-refresh](https://blog.mikemccandless.com/2021/03/open-source-collaboration-or-how-we.html) in Elasticsearch or [bypassing HNSW graph building for tiny segments](https://github.com/apache/lucene/issues/13447) in Lucene. Doc-value updates are worth evaluating as well, given that they only incur write amplification on the updated field, and not on other fields such as vector fields that are expensive to merge. Doc-value updates are already exposed in Solr and nrtSearch, they would be a good addition to the Elasticsearch toolbox as well.

In general, I expect Vespa to keep an advantage for update and realtime search scenarios in this benchmark, though it will be good for Lucene/Elasticsearch to see how much of the gap can be closed. Even though Vespa gets better results, I still feel good about Lucene, which gets other benefits not highlighted in this benchmark: thanks to immutable segments, you can easily scale out the search load of an index by replicating segment files to N search nodes. This is a very powerful feature that more and more Lucene-based search engines are trying to fully take advantage of: amazon.com, Yelp nrtSearch, Elasticsearch Serverless, OpenSearch, ... Then the Vespa vs. Elasticsearch performance trade-off boils down to how much value you can get from Lucene's efficient scalability through segment replication vs. Vespa's realtime update and search support.
