---
layout: post
title:  "JDK 23 G1/Parallel/Serial GC changes"
date:   2024-07-22 11:00:00 +0200
tags: [GC, G1, Parallel, Serial, JDK 23, Performance]
---

Given that JDK 23 is in [rampdown phase 2](https://openjdk.java.net/projects/jdk/23/) this post is going to present you the usual brief look on changes to the stop-the-world collectors in OpenJDK.

Compared to the [previous release](/2024/02/06/jdk22-g1-parallel-gc-changes.html) JDK 23 is a fairly muted one in the GC area, but there are good things on the horizon that I will touch on at the end of this post :)

The full list of changes for the entire Hotspot GC subcomponent for JDK 23 is [here](https://bugs.openjdk.org/issues/?jql=project%20%3D%20JDK%20AND%20issuetype%20in%20standardIssueTypes()%20AND%20status%20in%20(Resolved%2C%20Closed)%20AND%20fixVersion%20%3D%20%2222%22%20AND%20component%20%3D%20hotspot%20AND%20Subcomponent%20in%20(gc)), showing around 230 changes in total being resolved or closed at the time of writing. Nothing particular unusual here.

## Parallel GC

  * The probably largest change to Parallel GC for some time has been the replacement of the existing Parallel GC Full GC algorithm with a more traditional parallel Mark-Sweep-Compact algorithm.

    The original algorithm overwrites object headers during compaction, i.e. object movement, but still needs object sizes to (re-)calculate the final position of moved objects to update references. To enable that even in the presence of overwritten objects, Parallel GC Full GC stored the location where live objects *end* in a bitmap of the same size as the one that records live object starts during its first phase. Then, when needing an object size for a given live object, the algorithm scans forward from that object's start in the end bitmap, looking for the next set bit and subtracts the former from the latter.
    This can be slow, and needs to be done every time as part of looking up the final position of moved objects. That is actually quadratic in the number of live objects as rediscovered in [JDK-8320165](https://bugs.openjdk.org/browse/JDK-8320165), and although there were several related ([JDK-8145318](https://bugs.openjdk.org/browse/JDK-8145318)) and less related improvements targeted to this issue, this could not completely mitigate the problem as JDK-8320165 shows.

    With [JDK-8329203](https://bugs.openjdk.org/browse/JDK-8329203) we replaced the somewhat unique algorithm of Parallel GC with (basically) G1's parallel Full GC which does not suffer from these hiccups. At the same time that second end bitmap (taking 1.5% of Java heap size) could be removed as well, while our measurements showed that overall the performance stayed the same (and improved a lot in these problematic cases).

  * Another performance bump in Parallel Full GC could be achieved by decreasing contention on the counters recording the number of live bytes per region: instead of every thread updating global counters immediately, with [JDK-8325553](https://bugs.openjdk.org/browse/JDK-8325553) Parallel GC has every thread locally record per-region liveness as long as that thread only encountered live objects within the same (small set of) regions.

## Serial GC

  * Cleanup and refactoring of Serial GC code continued.

## G1 GC

  * One of the more long-standing issues that got resolved in JDK 23 has been [JDK-8280087](https://bugs.openjdk.org/browse/JDK-8280087) where G1 did not expand some internal buffer used during reference processing as it should. Instead it prematurely exited with an unhelpful error message.

  * Performance improvements like [JDK-8327452](https://bugs.openjdk.org/browse/JDK-8327452) and [JDK-8331048](https://bugs.openjdk.org/browse/JDK-8331048) improve pause times and reduce native memory overhead of the G1 collector.

## All (STW) GCs

Some time ago we introduced dedicated (internal) filler objects which I wrote about [here](/2022/09/26/jdk-vm-internal-fillerarray.html). The community pointed out that `jdk.vm.internal.FillerArray` is not a valid array class name, and some tools somewhat rightfully choke on such "variable sized regular" objects.

With [JDK-8319548](https://bugs.openjdk.org/browse/JDK-8319548) the class name of filler arrays has been changed to `[Ljdk/internal/vm/FillerElement;` to conform to that, and backported to JDK 21 and above.

## What's next

{: #single-remsets-multiple-regions }

Reduction of G1 remembered set storage requirements is still a very important topic to us: one [old idea](https://bugs.openjdk.org/browse/JDK-8058803) is to use a single rememebered set for multiple regions, removing the need to store remembered set entries within the group of regions with the same remembered set.

The figures below show the principle: Currently regions (the big rounded boxes at the top, with "Y" for "Young region" and "O" for "Old region") may have remembered sets associated to them (the vertical small boxes below the regions). The entries refer to approximate locations (areas) outside of the respective region, where there may be a reference into that region.

Young regions always have remembered sets associated with them, as they are required to find references into these regions that need to be fixed up when moving a live object within them, and they are evacuated/moved at every garbage collection. You might notice that remembered set entries of different regions refer to the same locations in the old generation regions.

![Per-region remembered set](/assets/20240722-remset-initial.png){:style="display:block; margin-left:auto; margin-right:auto"}

Now, as mentioned, particularly young generation regions are always evacuated at the same time, together, which means that all these remembered set entries that refer to the same locations are redundant. This is what the [initial change](https://bugs.openjdk.org/browse/JDK-8336086) to implement above idea does: use a single remembered set for all young generation regions, as depicted below, automatically removing the redundant remembered set entries. This not only saves memory, but also time during garbage collection to filter out these entries.

(Technically, combining remembered sets also removes the need for storing remembered set entries that refer to locations between regions in a group; however, for young generation regions no such remembered set entries will ever be generated so they are not shown and do not add to savings).

![Combined young remembered set](/assets/20240722-remset-merged.png){:style="display:block; margin-left:auto; margin-right:auto"}

The gain for applying this technique for the young generation remembered sets only can already be substantial as shown below in some measurements of some larger application. The graph depicts remembered set memory usage before (blue line) and after (pink line) applying the change. Of particular note is the halving of memory usage for the remembered set in the first 40 seconds where only young generation regions have remembered sets.

As old generation regions get remembered sets assigned to them, the improvement decreases in that prototype due to lack of support for merging old generation regions, but the impact of just young generation region rememembered set savings are still noticeable.

![Memory savings combined young remembered set](/assets/20240722-remset-memory-improvements.png){:style="display:block; margin-left:auto; margin-right:auto"}

This initial change for combining young generation region remembered sets is actually [out for review](https://github.com/openjdk/jdk/pull/20134), but a more generic variant to also merge old generation regions into groups using a single remembered set is in preparation.

In this release cycle another fairly large focus for improvements to the G1 collector have been changes to the write barriers: we spent a lot of effort to investigate how to best close the throughput gap of the G1 collector to Parallel GC in some cases (e.g. [here](https://bugs.openjdk.org/browse/JDK-8253230) and the differences reported [here](https://timefold.ai/blog/java-21-performance)), and we believe we found very good solutions without compromising the latency aspect of the G1 collector.

One step in that direction is [JEP 475: Late Barrier Expansion for G1](https://openjdk.org/jeps/475) which seems to be [on track](https://github.com/openjdk/jdk/pull/19746) for JDK 24 inclusion. It enables some of the optimizations we are planning.

There are still some details and kinks to work out, and it will take some time to do so, and even more time to have everything in place, but more to come in the next months...

## Thanks go to…

... as usual to everyone that contributed to another great JDK release. Looking forward to see you next release.

*Thomas*
