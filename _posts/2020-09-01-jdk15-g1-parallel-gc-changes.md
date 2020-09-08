---
layout: post
title:  "JDK 15 G1/Parallel GC changes"
date:   2020-09-01 11:58:24 +0200
#categories: gc g1 parallel JDK-15 performance
tags: [GC, G1, Parallel, JDK 15, Performance]
---

JDK 15 just moved into the release candidate phase and I thought it is a good time for another entry in my recaps of significant changes in the OpenJDK stop-the-world garbage collectors G1 and Parallel GC - this time in blog format.

The big change in JDK15 in the garbage collection area is certainly **ZGC** becoming a **production ready** feature with [JEP 377](https://openjdk.java.net/jeps/377). For **G1 and Parallel GC** JDK 15 is more about **maintenance** than a big feature release regarding these collectors as the [JEP list](https://openjdk.java.net/projects/jdk/15/) shows. Next to the usual micro-optimizations here and there that in total account for a few percent of pause time improvements, there are a few interesting changes that I would like to particularly highlight though.

## Parallel GC

Yes, Parallel GC isn't dead, we are still happily improving it :)

* [JDK-8246718](https://bugs.openjdk.java.net/browse/JDK-8246718) brought **improved** garbage collection **pause times**: while comparing the inner loops copying surviving objects, we found a major difference between G1 and Parallel GC:

  When encountering a new reference from a recently copied object, G1 prefetches that reference and pushes it immediately onto the task queue for further processing, while Parallel GC tried to examine the object first to see whether it had already been copied. This caused lots of cache misses on that access, and reduced performance.

  On [SPECjbb2015](https://www.spec.org/jbb2015/) this improves its throughput-under-latency (critical-jOPS) score that roughly corresponds to pause times on internal benchmarks by ~5%. A few more numbers are provided in the referenced CR.

* After changing Parallel GC to use the same parallel worker thread management mechanism in [JDK-8204951](https://bugs.openjdk.java.net/browse/JDK-8204951) in JDK 14, the community started using it to **parallelize** a few previously **serial phases**, for example [JDK-8240440](https://bugs.openjdk.java.net/browse/JDK-8240440).

These were the more interesting, performance focused changes for JDK 15 for Parallel GC. I would like to point out one more user facing change:

* The `-XX:UseAdaptiveGCBoundary` product option **has been obsoleted**, effectively removing this functionality. The reason is that this functionality has been buggy since its initial version, causing random crashes. See the related [CSR](https://bugs.openjdk.java.net/browse/JDK-8242164) for more information.

## G1 GC

* After the release of JDK 14, [Phoronix](https://www.phoronix.com/scan.php?page=article&item=openjdk-14-benchmark) posted a comparison between JDK 8 to JDK 14 SPECjbb2015 out-of-box performance. The **results** in there **look pretty bad** for G1. The usual explanation is that, in JDK 9, the default garbage collector changed to G1 with [JEP 248](https://bugs.openjdk.java.net/browse/JDK-8073273) which he did not consider. However the **difference** between Parallel GC and G1 **seemed too large** at around 40% for us, so we investigated. It turned out that default heap region sizing has been the cause for a large part of the difference, fixed in [JDK-8241670](https://bugs.openjdk.java.net/browse/JDK-8241670).

  My colleague Stefan posted a much more detailed summary on his [blog](https://kstefanj.github.io/2020/04/16/g1-ootb-performance.html).
  
* There were a few **startup improvements** for G1 that initialize some data structures lazily, shaving off a few ms of startup time ([JDK-8242038](https://bugs.openjdk.java.net/browse/JDK-8242038), [JDK-8241920](https://bugs.openjdk.java.net/browse/JDK-8241920), and [JDK-8235551](https://bugs.openjdk.java.net/browse/JDK-8235551)).

## Other noteworthy changes

This section contains some minor changes or changes not directly related to garbage collection that are worth mentioning.

* **Biased locking** functionality has been **disabled** by default and marked as **deprecated** with [JEP 374](https://bugs.openjdk.java.net/browse/JDK-8235256). Biased locking reduces synchronization overhead for the uncontended case, i.e. when monitors are only ever locked by a single thread. In essence it seems to be a kind of workaround for badly programmed multi-threaded applications - why would you want to lock something that is only ever accessed by a single thread? However its implementation complicates Hotspot code a lot, and by deprecating and disabling it by default we are looking for feedback on its impact on applications in the real-world.

  So if you happen to notice significant performance regressions, try enabling this option manually and please report your findings.

* The `UseParallelOldGC` option is **now obsolete** and the functionality has been removed after being deprecated via [JEP 366](https://bugs.openjdk.java.net/browse/JDK-8229492) in JDK 14.

## What's next

We are currently working on changes to make G1 more economical with memory, giving it back to the operating system more frequently, and concurrent marking related improvements. A short overview:

* One of these changes is [JDK-8238687](https://bugs.openjdk.java.net/browse/JDK-8238687) where G1 will **adjust** current **committed heap size** dynamically on current application activity **at every GC** - instead of only during concurrent marking or at Full GC. This makes G1 memory usage much more dynamic, and also allows easy implementation of [`SoftMaxHeapSize`](https://bugs.openjdk.java.net/browse/JDK-8222145). I will probably write about this some more in a different post :)

* Liang Mao contributed another interesting change to allow **undo of concurrent mark cycles** for G1: if at the end of a concurrent mark start pause where concurent marking is set up, G1 may notice that the reason for starting it is gone. The change makes G1 to not continue with the concurrent marking and undo all related setup work performed in the pause earlier.

  This particularly helps with applications that allocate lots of short-lived humongous objects which frequently start concurrent marking, but eager reclaimation actually most of the time reclaim enough space to not need the concurrent cycle. This change just needed a bit more time to bake [JDK-8240556](https://bugs.openjdk.java.net/browse/JDK-8240556).

* An improvement to G1 adaptive **IHOP calculation** contributed by Ziyi Luo changes how **short-lived humongous objects** are **accounted** to determine when to start marking cycles: previously, they were not handled specially, so any application that continuously allocated humongous objects that were almost immediately reclaimed inflated the allocation rate as they were accounted regularly. This decreased the marking start threshold at which marking started with [JDK-8245511](https://bugs.openjdk.java.net/browse/JDK-8245511) significantly, causing a lot of unnecessary concurrent marking activity. This work has already been pushed to JDK 16.

## Thanks go to...

Everyone that contributed to another great JDK release. See you next release :)


