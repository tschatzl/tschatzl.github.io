---
layout: post
title:  "JDK 16 G1/Parallel GC changes"
date:   2021-03-12 16:39:24 +0200
#categories: gc g1 parallel JDK-16 performance
tags: [GC, G1, Parallel, JDK 16, Performance]
---

Now that JDK 16 release date is very close (2021/03/16 - next Tuesday), there is some time to recap significant changes in Hotspot's stop-the-world garbage collectors - G1 and Parallel GC.

First, a short overview about the whole GC subcomponent: The only JEP in the garbage collection area this time around is for **ZGC** with [JEP 376](https://openjdk.java.net/jeps/376) where thread-stack processing has been moved out from its stop-the-world pauses, making these pauses really small now. **G1 and Parallel GC** got some interesting smaller updates that I will highlight below. Of course there have been the usual micro-optimizations here and there as usual in JDK 16 too.

The full list of changes for the entire Hotspot GC subcomponent is [here](https://bugs.openjdk.java.net/issues/?jql=project%20%3D%20JDK%20AND%20issuetype%20in%20standardIssueTypes()%20AND%20status%20in%20(Resolved%2C%20Closed)%20AND%20resolution%20%3D%20Fixed%20AND%20fixVersion%20%3D%20%2216%22%20AND%20component%20%3D%20hotspot%20AND%20Subcomponent%20in%20(gc%2C%20gc%2C%20gc%2C%20gc%2C%20gc)%20ORDER%20BY%20key%20ASC), clocking in at 325 changes in total.

## Parallel GC

  * The change [JDK-8252221](https://bugs.openjdk.java.net/browse/JDK-8252221) **parallelizes pre-touching memory** after it has been committed. This helps with startup on huge heaps with `-XX:+AlwaysPreTouch` enabled. There were some improvements in [JDK-8254699](https://bugs.openjdk.java.net/browse/JDK-8254699) and follow-ups (that also apply to G1 as it shares the same code) where the size of the work packets has been optimized on x86 and ARM64 platforms.

  * **Processing of some more internal references** to the Java heap has been **parallelized** in [JDK-8247820](https://bugs.openjdk.java.net/browse/JDK-8247820), in some cases decreasing pause times a little in both Parallel and the G1 collector ([JDK-8247819](https://bugs.openjdk.java.net/browse/JDK-8247819)).

## G1 GC

  * Until now, when G1 noticed that it owned too much Java heap memory during a GC pause, and the user allowed it (having different minimum and maximum heap sizes), it gave back that memory right at the time when it noticed that surplus. Depending on the amount of memory this was, which could easily be a few GB of memory, the GC pause could be lengthened quite a bit (some tests showed 50-100ms+). [JDK-8236926](https://bugs.openjdk.java.net/browse/JDK-8236926) changes this behavior. During the pause, G1 now only determines how much and what memory areas should be uncommitted, and **defers the actual uncommit work to the concurrent phase** in a background task. This background task incrementally processes these areas. The increments are sized such that each step takes only a few milliseconds at most to avoid long waits to start GC pauses.

    You can track its work by enabling `gc+heap` logging at debug level. Messages containing `Concurrent Commit` allow you to get more information about what it did, like the size of these chunks and the durations of these steps. The release note is [here](https://bugs.openjdk.java.net/browse/JDK-8256691).

  * [JDK-8240556](https://bugs.openjdk.java.net/browse/JDK-8240556) addresses a **shortcoming of the calculation for when to start concurrent marking** in case G1 can eagerly reclaim many humongous objects (a feature introduced in JDK 9 [JDK-8048179](https://bugs.openjdk.java.net/browse/JDK-8048179)): at the start of a garbage collection, G1 determines whether this garbage collection should prepare for a concurrent marking or not depending on whether current heap occupancy is larger than an internal threshold. If so, G1 performs some additional steps during the garbage collection to prepare for the concurrent marking afterwards. This heuristic did not take into account that during this particular garbage collection the heap occupancy might actually go below that internal threshold again! So in this case, **G1 now cancels that marking**, preparing to undo some work done earlier (concurrently of course). This is much faster and much less cpu consuming than doing the concurrent marking and a few mixed collections when it is not required.

  * Similarly there has been an improvement to **determine the heap occupancy threshold when to start this marking** in [JDK-8245511](https://bugs.openjdk.java.net/browse/JDK-8245511). This heap occupancy threshold is computed using recent allocation rate in the old generation and how long it typically takes to start the first old generation garbage collection from that start of the marking. The larger the allocation rate and the longer the time from the start of marking to the first collection in the old generation, the lower this threshold is, i.e. the earlier G1 starts the marking.

    The problem are humongous objects that are always directly allocated in the old generation, so they typically factor heavily into this allocation rate. However, since JDK 9, with the eager reclamation mentioned above, these humongous objects can reclaimed very quickly and the overall allocation rate in the old generation is actually very low. This led to many unnecessary concurrent markings. The mentioned change for JDK-8245511 solves this problem by taking eager reclamation into account better. There are some nice graphs showing the impact in the [bug report](https://bugs.openjdk.java.net/browse/JDK-8245511).

## Other noteworthy changes

This section contains some changes that are important but less or not visible at all for end users.

  * JDK 16 adds some preparations to **support object pinning** (tracked in [JDK-8236594](https://bugs.openjdk.java.net/browse/JDK-8236594)) where in the situation that a single thread requires that an object does not move (mainly for interaction with non GC-ed languages via JNI), instead of blocking garbage collection and all Java application threads that want to start a garbage collection because they need memory, G1 will just go ahead and move all objects but the ones that need to stay in place (and a few more surrounding it).

    A. Shipilev has a write-up of this problem and the options on how to handle this [here](https://shipilev.net/jvm/anatomy-quarks/9-jni-critical-gclocker/); following this write-up, this change would move G1 behavior from option 1 (disabling gc completely) to option 3 (pin the subspace(s) containing the object(s)).

    Examples for such preparatory changes are [JDK-8253600](https://bugs.openjdk.java.net/browse/JDK-8253600) and [JDK-8253081](https://bugs.openjdk.java.net/browse/JDK-8253081). **The missing part** is not enabling this functionality, that's just [a few lines of code](https://github.com/openjdk/jdk/compare/master...tschatzl:full-pin-support) at this point, but the as of this writing its behavioral impact has not been fully explored. Some initial testing showed some potential issues with unkown impact, after which we brainstormed for [potential solutions](https://bugs.openjdk.java.net/issues/?jql=labels%20%3D%20gc-g1-pinned-regions). If anyone has interest to investigate this a bit more and then try one of these (or other) suggestions, feel free to contact me.

## What's next

We are already busily working on JDK 17, which is an LTS release for Oracle and most other vendors. Here is a short list of changes you might want to look out for. As usual, no guarantees of any kind that anything noted here will actually show up. When it's done, it's done :)

  * JDK 17 might give some **huge improvements** in G1 non-Java heap memory usage with some particular focus on optimizations to decrease **remembered set memory usage**. This [recent post](https://tschatzl.github.io/2021/02/26/early-prune.html) demonstrates the problem. The change explained there ([JDK-8262185](https://bugs.openjdk.java.net/browse/JDK-8262185)) has already been pushed to mainline by now with the explained impact. There are some additional changes that we have been working on for a long time may reduce remembered set memory consumption much more so far. Stay tuned.

  * Microsoft is doing some very interesting work on **scheduling proactive garbage collections** in G1 in [JDK-8257774](https://bugs.openjdk.java.net/browse/JDK-8257774) to **avoid very long garbage collections** having evacuation failures.

  * In [JDK-8262068](https://bugs.openjdk.java.net/browse/JDK-8262068) Hamlin Li **improves G1 Full collections** using a simple heuristic: if a region is almost full, then do not bother compacting it - with all the corresponding performance improvement. Other collectors already implement this with the `-XX:MarkSweepDeadRatio` option, but it seems nice to allow this too in G1. The change is already out for review.

## Thanks go to…

Everyone that contributed to another great JDK release. See you next release :)



