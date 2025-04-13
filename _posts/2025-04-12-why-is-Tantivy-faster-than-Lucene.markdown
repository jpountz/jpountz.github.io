---
layout: post
title:  "Why is Tantivy faster than Lucene?"
date:   2025-04-12
---

In 2023, the [Tantivy](https://github.com/quickwit-oss/tantivy) folks started publishing [comparative benchmarks](https://tantivy-search.github.io/bench/) between Apache Lucene, Tantivy and [PISA](https://github.com/pisa-engine/pisa). As Lucene enjoys a good reputation as a search library, many folks published benchmarks claiming that their software was faster than Lucene over the years, often using disputable methodology. But this one was interesting for the Lucene community for a few reasons:
 - You could sense an effort for fairness rather than tuning the methodology to be able to make the bolder possible claims, something that they [called out](https://github.com/quickwit-oss/search-benchmark-game/blob/3c124e2823e27e2c425769ffac24c71085065502/README.md?plain=1#L19-L21).
 - The setup of the benchmark is very similar to [Lucene's own nightly benchmarks](https://benchmarks.mikemccandless.com/): it uses the Wikipedia dataset and runs various queries against the content of Wikipedia articles.
 - While PISA is hard to compare with Lucene due to very different trade-offs, Tantivy is built on the same architecture as Lucene, so performance should be comparable.

Except that performance was not comparable back in 2023. Tantivy was up to several times faster than Lucene on some queries, especially with the `COUNT` collection type. Given that architectures are comparable, this had to boil down to better algorithms, better implementation details or better compiler/runtime. When these results were first published, I believe that many folks read this benchmark as Rust vs. Java/JVM vs. C++ (PISA is written in C++). We knew that it was more complicated than that, on the other hand none of the differences between Lucene and Tantivy that we could think of could explain such a big performance difference.

Not understanding where the performance difference came from was especially frustrating as Lucene has a good culture around performance. It has benchmarks running nightly with almost 15 years of history, many PRs include benchmark results using the same tooling as nightly benchmarks to assess their performance impact, Lucene has [micro benchmarks](https://github.com/apache/lucene/tree/main/lucene/benchmark-jmh) checked in in the same repository as its source code to help us understand how the JVM compiles some of Lucene's performance-critical code. How could we have been missing something big enough that it could yield a multiple-fold performance improvement?

After some good amount of time trying to understand this performance difference, I believe that the main factors are the ones described below.

### Tantivy's count optimization

In both Lucene and Tantivy, disjunctive queries evaluate by loading windows of matches into a bit set, and then iterating set bit from this bit set into the collector. In the case when the collector is only interested in the number of matches of the query, Tantivy has an optimization that skips iterating this bit set, and instead just counts the number of set bit using `popcnt` instructions.

When a similar [change](https://github.com/apache/lucene/pull/12415) was merged to Lucene, it improved the performance of the `CountOrHighHigh` task - which counts the number of matches of a disjunctive query on two high-frequency terms - by 60% (annotation [FN](https://benchmarks.  mikemccandless.com/CountOrHighHigh.html)). 

### Better auto-vectorization

Let's say you have the following (pseudo, Java-like) code, which is similar to Lucene's code:

```java
class BufferedDocIdSetIterator extends DocIdSetIterator {

  int[] buffer = new int[128];
  int i = buffer.length - 1;

  int docID() {
    return buffer[i];
  }

  int nextDoc() {
    if (++i == buffer.length) {
      refill();
      i = 0;
    }
    return docID();
  }

  void refill() { /* TODO */ }
}

class BitSet {
  long[] bits;

  void set(int index) {
    bits[index / Long.SIZE] |= 1L << index;
  }

  int length() {
    return bits.length * Long.SIZE;
  }
}

void loadIteratorIntoBitSet(DocIdSetIterator iterator, BitSet bitSet, int offset) {
  for (int doc = iterator.docID(); doc - offset < bitSet.length(); doc = iterator.nextDoc()) {
    bitSet.set(doc - offset);
  }
}
```

Both Tantivy and Lucene evaluate non-scoring disjunctive queries by loading matching docs into a bit set, so they use logic that looks like the above. A difference is that LLVM (used by the Rust compiler) seems to be able to auto-vectorize such code, while the JVM's JIT compiler cannot (even if `DocIdSetIterator#nextDoc()` and `BitSet#set` get inlined). Lucene worked around this by refactoring the code to help the JVM compiler a bit. More specically, the code now looks more like that:

```java
class BufferedDocIdSetIterator extends DocIdSetIterator {

  int[] buffer = new int[128];
  int i = buffer.length - 1;

  int docID() {
    return buffer[i];
  }

  int nextDoc() {
    if (++i == buffer.length) {
      refill();
      i = 0;
    }
    return docID();
  }

  void refill() { /* TODO */ }

  void loadIntoBitSet(BitSet bitSet, int offset) {
    while (buffer[buffer.length - 1] - offset < bitSet.length()) {
      // In the below loop, BitSet#set gets inlined and the various arithmetic operations auto-vectorize.
      for (int j = i; j < buffer.length; ++j) {
        int doc = buffer[j];
        bitSet.set(doc - offset);
      }
      refill();
      i = 0;
    }
    for (int doc = docID(); doc - offset < bitSet.length(); doc = iterator.nextDoc()) {
      bitSet.set(doc - offset);
    }
  }
}

class BitSet {
  long[] bits;

  void set(int index) {
    bits[index / Long.SIZE] |= 1L << index;
  }

  int length() {
    return bits.length * Long.SIZE;
  }
}

void loadIteratorIntoBitSet(DocIdSetIterator iterator, BitSet bitSet, int offset) {
  iterator.loadIntoBitSet(bitSet, offset);
}
```

This [refactoring](https://github.com/apache/lucene/pull/14069) yielded a 35% speedup on `CountOrHighHigh`.

### Simpler skip data

Lucene had been [discussing inlining skip data in postings lists for a long time](https://github.com/apache/lucene/issues/4036), which had been identified as a way to improve performance as well as the memory/disk access pattern. The fact that Tantivy performed so well with a single level of skip data encouraged us to also explore using only a couple of levels, vs. the 10 levels that Lucene was previously maintaining. After some benchmarks, we landed on 2 levels of skip data, recorded every block (128 docs) and 32 blocks (4,096 docs).

This [change](https://github.com/apache/lucene/pull/13585) yielded a 33% speedup on Lucene's `CountAndHighMed` task (annotation [GS](https://benchmarks.mikemccandless.com/CountAndHighMed.html)), which counts the number of matches of a conjunctive query over a high-frequency term and a medium-frequency term. It turns out that the speedup was more due to skipping now being significantly simpler than to skip data being inlined in postings. Simpler code often performs faster.

### Lucene has megamorphic method calls in hot code paths

If you spent time looking into performance of Java programs, you may be familiar with the fact that [megamorphic call sites are not eligible to inlining](https://shipilev.net/blog/2015/black-magic-method-dispatch/). And inlining is important because it gives more context to the compiler, which can then apply even more optimizations, including auto-vectorization and other optimizations that can make a big difference.

Because Lucene allows adding any `Query` as a clause of a `BooleanQuery`, call sites of `DocIdSetIterator#nextDoc` and `DocIdSetIterator#advance` are often polymorphic. One solution would be to specialize the query evaluation logic more. For now, Lucene focused on:
 - [reducing polymorphism](https://github.com/apache/lucene/pull/14017) so that term queries would always create a single type of `DocIdSetIterator` for a given `BooleanQuery`,
 - [ensuring that call sites of `DocIdSetIterator#nextDoc` and `DocIdSetIterator#advance` would be bimorphic at most](https://github.com/apache/lucene/pull/14023), so that they are eligible to inlining. If the provided iterator is not the iterator of a term query, it gets wrapped. So the call site always sees either an iterator of a `TermQuery` or this generic wrapper.

These two changes combined contributed a 25% speedup to `OrHighHigh` and a 30% speedup to `AndHighHigh`, which respectively compute the top-100 hits of a disjunctive and conjunctive query on two high-frequency terms.

### Fewer abstraction layers

This is especially noticeable on phrase queries. While Lucene evaluates phrase queries through its `PostingsEnum` abstraction that gives one position at a time and maintains buffers that are periodically refilled from the `Directory`, Tantivy simply loads all positions in memory once per document. Evaluating phrase queries then boils down to intersecting sorted arrays, something that the compiler likely heavily optimizes.

## Conclusion

When working on performance, it's often hard to know whether you have room for improvement and how much. Tantivy has been very helpful from this perspective by giving targets to Lucene that should be achievable given how much Lucene and Tantivy share architecture-wise.

While recent versions of Lucene (10.2 at the time of writing this post) have recovered a good share of the performance difference on many queries and collection types, it is still noticeably slower on some of them, notably phrase queries and exhaustive evaluation with scores (`TOP_100_COUNT`). There is a lot of work left to do, and hopefully we can have even more queries and collection types when Lucene is on par or faster in the future.
