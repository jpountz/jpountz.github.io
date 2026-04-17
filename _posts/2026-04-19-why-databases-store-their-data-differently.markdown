---
layout: post
title:  "Why databases store their data differently"
date:   2026-04-19
---

I was recently chatting with a friend about database architecture, and why, for
instance, OLTP databases and search engines are implemented very differently.
This friend is familiar with computer science concepts, but not with database
internals, so I tried to make the explanation as simple as I could. This blog
is an expanded version of the discussion we had.

I focused on the storage layer, which is arguably the most distinctive
implementation detail of a database, and shared my mental model of how
databases store their data.

The storage layer needs to lay out the data with two major constraints:
 - data must be updatable,
 - data must be searchable.

Let's review a few options.

## A: Search tree

A natural strategy for making data searchable is to store it in a sorted data
structure. A tree such as a [binary search
tree](https://en.wikipedia.org/wiki/Binary_search_tree) is a good starting
point: it can answer point and range queries efficiently, and also support
updates.

Here is an example tree that maps a few numbers to their remainder when divided
by 2. 

```
                  4 (0)
                 /     \
               /         \
            1 (1)       7 (1)
               \        /    \
              3 (1)   5 (1)  9 (1)
```

## B: Sorted array

You can also store the same data in a sorted array and use binary search to
perform point or range queries:

```
[1 (1), 3 (1), 4 (0), 5 (1), 7 (1), 9 (1)]
```

Since arrays can't efficiently remove or insert entries in the middle, updates
are implemented by writing new arrays, and searches must span the union of all
arrays. For instance, let's insert an entry for 0:

```
[1 (1), 3 (1), 4 (0), 5 (1), 7 (1), 9 (1)]

[0 (0)]
```

Now, how do we handle updates or deletions? There are two approaches:

### B1: Sorted arrays, layered

One approach is to stack arrays on top of each other: newer layers shadow older
ones. To support deletions, we introduce the ability to map keys to a
tombstone. Starting from our original array, we insert an entry for 0, remove
the entry for 7 and decide to map numbers to the remainder of their division by
4 rather than 2, so 3 now needs to map to 3 rather than 1:

```
[1 (1), 3 (1), 4 (0), 5 (1), 7 (1), 9 (1)]

[0 (0), 3 (3), 7 (∅)]
```

Keys 3 and 7 exist in both arrays, so the second array is the source of truth
for them since it's newer.

You can keep making updates by adding more layers, and compact layers to keep
their total number at bay.

### B2: Sorted arrays, partitioned

Another approach is to ensure that each array stores a disjoint subset of the
data, so no precedence rules are needed. To do this,
updates and deletes attach a deletion marker to the array where the entry was
previously stored so that each entry is only active in a single array at a
time.

```
Data:    [1 (1), 3 (1), 4 (0), 5 (1), 7 (1), 9 (1)] 
Deletes: [       XXXXX                XXXXX       ]
 
Data:    [0 (0), 3 (3)]
Deletes: [            ]
```

## Analysis

There are major databases in each category, for instance MySQL (InnoDB) stores
its data in a B-tree, so it's part of the A category. B1 is essentially the
description of a [log-structured
merge-tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree), which is
the foundation of wide-column stores like Cassandra and ScyllaDB, or key-value
stores like RocksDB. Key-value stores can in turn serve as the storage layer of
OLTP databases like MyRocks or CockroachDB. Finally, Lucene is B2 and calls
these immutable arrays "segments".

OLAP databases like to think of their data as append-only, which is convenient
since B1 and B2 are effectively the same in this case, and you get the benefits
of both at the same time (discussed further below). That said, some OLAP
databases have added support for updates/deletes to support a broader range of
use cases. For instance, ClickHouse is in the B category, and can be seen as B1
when you query it with the [`FINAL`
keyword](https://clickhouse.com/docs/sql-reference/statements/select/from#final-modifier)
to force it to deduplicate entries by primary key, or as B2 if you delete data
using [lightweight
deletes](https://clickhouse.com/docs/guides/developer/lightweight-delete#how-lightweight-deletes-work-internally-in-clickhouse)
- which is then very similar to Lucene.

Trees are simple conceptually, but a bit difficult to store on disk.
Traditional filesystems don't support inserting data in the middle of a file
efficiently, which makes updating these trees challenging. This has led to the
development of [B-trees](https://en.wikipedia.org/wiki/B-tree) and their many
variants. A benefit of trees is that their updates are localized to specific
nodes, which makes it easier to support transactions by locking the regions of
the tree that need updating. This explains why trees are still popular for OLTP
databases.

Category B avoids the problem of inserting data in the middle of a file by
writing files once and never modifying them until they become unused
(e.g. after compaction): each array is stored in its own immutable file. One
interesting side-effect of treating files as immutable once written is that it
makes it possible to store data in object stores.

B1 is the only approach that can publish updates without having to read
existing data first. A (trees) must find the node to update, and B2 must locate
which array contains the key to attach a deletion marker to. This makes B1
great at write throughput.

With B2, each array stores a disjoint subset of the data with no precedence
rules, which makes it convenient for read workloads.
For instance, imagine that you want to count the number of entries in a range.
You can count the number of non-deleted entries in each array independently
and then sum up the per-array counts. This is why B2 makes sense for OLAP
databases or search engines, which optimize for read efficiency.

However, one read workload in particular performs equally well with B1 and B2:
simple key-value lookups. In both cases, you need to iterate over arrays until
you find the one that has the key of interest. The only benefit of B2 over B1
is that you can loop in arbitrary order rather than precedence order, but this
doesn't make a meaningful difference. This is why B1 is such a sweet spot for
key-value stores: great write efficiency and read efficiency combined.

## Conclusion

This is a high-level look at how databases store their data. In
practice, many other storage-level decisions are highly important, such as what
the keys and values in these index structures represent, row-oriented vs.
column-oriented storage, how secondary indexing is performed, ...

Still, I've found this useful as a mental model for connecting how a database
stores its data to its high-level category (OLTP vs. OLAP vs.  key-value store
vs. search engine). I hope it'll be useful for you too.
