---
layout: post
title:  "JDK 21 G1/Parallel/Serial GC changes"
date:   2023-08-04 12:00:00 +0200
tags: [GC, G1, Parallel, Serial, JDK 21, Performance]
---

JDK 21 GA is [on track](https://openjdk.java.net/projects/jdk/21/) - as well as this update for changes of the stop-the-world garbage collectors for OpenJDK for that release. ;)

This release is going to be designated as an LTS release by most if not all OpenJDK distributions. Please also have a look at previous blog posts for [JDK 18](/2022/03/14/jdk18-g1-parallel-gc-changes.html), [JDK 19](/2022/09/16/jdk19-g1-parallel-gc-changes.html), and [JDK 20](/2023/03/14/jdk20-g1-parallel-gc-changes.html) to get more details about what the upgrade from the last LTS, JDK 17, offers you in the garbage collection area.

Back to this release, before looking at stop-the-world garbage collectors, probably the most impactful change to garbage collection in this release is the introduction of [Generational ZGC](https://openjdk.org/jeps/439). It is an improvement over the Z Garbage Collector that maintains generations for young and old objects like the stop-the-world collectors, focusing garbage collection work on where most garbage is created typically. It significantly decreases required Java heap and CPU overhead to keep up with applications. To enable generational ZGC, along the `-XX:+UseZGC` flag pass the flag `-XX:+ZGenerational`.

Other than that, the full list of changes for the entire Hotspot GC subcomponent for JDK 21 is [here](https://bugs.openjdk.org/issues/?jql=project%20%3D%20JDK%20AND%20issuetype%20in%20standardIssueTypes()%20AND%20status%20in%20(Resolved%2C%20Closed)%20AND%20fixVersion%20%3D%20%2221%22%20AND%20component%20%3D%20hotspot%20AND%20Subcomponent%20in%20(gc)), showing around 250 changes in total being resolved or closed at the time of writing. This is a bit more than the previous releases.

Here is a run-down of the Hotspot stop-the-world collectors and the most important changes to them in JDK 21:

## Parallel GC

  * There were no notable changes for Parallel GC.

## Serial GC

  * There were no notable changes for Serial GC.

## G1 GC

Here are the JDK 21 user facing changes for G1:

  * During Full GC, G1 will now move humongous objects to decrase the possibility of OOME due to region level fragmentation. While JDK 20 and below strictly never moved humongous (large) objects, this restriction has been lifted for last-ditch full collections. A last-ditch full collection occurs after a regular full collection did not yield enough (contiguous) space to satisfy the current allocation ([JDK-8191565](https://bugs.openjdk.org/browse/JDK-8191565)). 
  
    This improved last-ditch full collection can avoid some previously occurring Out-of-Memory situations.

  * Another somewhat related improvement for full collections ([JDK-8302215](https://bugs.openjdk.org/browse/JDK-8302215)) makes sure that the Java heap is more compact after full collection, reducing fragmentation a little.

  * The so-called Hot Card Cache has been found to be without impact (at least) after concurrent refinement changes in JDK 20, and removed. The intent of this data structure has been to collect locations where the application very frequently modifies references. These locations were put into this Hot Card Cache to be processed once in the next garbage collection, instead of constantly concurrently refine them. The reason why this removal can be interesting is that its data structures used a fairly large amount of native memory, that is 0.2% of the Java heap size, to operate. This native memory is now available for use for other purposes.

    The corresponding option `-XX:+/-UseHotCardCache` to toggle its use is now obsolete.

  * On applications with thousands of threads, in previous releases a significant amount of time of the pause could be spent in tearing down and setting up the (per-thread) TLABs. These two phases have been parallelized with [JDK-8302122](https://bugs.openjdk.org/browse/JDK-8302122) and [JDK-8301116](https://bugs.openjdk.org/browse/JDK-8301116) respectively.

  * The preventive garbage collection feature has been removed completely with [JDK-8297639](https://bugs.openjdk.org/browse/JDK-8297639) for the reasons stated [previously](/2023/03/14/jdk20-g1-parallel-gc-changes.html#preventive).

## What's next

Work for JDK 22 has started, continuing right away with features that did not make the cut. Here is a short list of interesting changes that are already integrated or currently in development. As usual, there are no guarantees that they will make the next release ;).

  * The change to improve the resilience of G1 against regions that failed evacuation swamping the old generation has been out for review for some time now ([JDK-8140326](https://bugs.openjdk.org/browse/JDK-8140326) [PR](https://github.com/openjdk/jdk/pull/14220)).

    This change removes the previous limitation in G1 that only certain [types of young collections](https://docs.oracle.com/en/java/javase/20/gctuning/garbage-first-g1-garbage-collector1.html#GUID-F1BE86FA-3EDC-4D4F-BDB4-4B044AD83180) could reclaim space in old generation regions. With this change any young collection may evacuate old generation regions if they meet the usual requirements.
    
    Currently this helps with reclaiming space after an [evacuation failure](https://docs.oracle.com/en/java/javase/20/gctuning/garbage-first-g1-garbage-collector1.html#GUID-BE157AF6-29E7-461A-82CF-50C1978785DA) quickly, but among other interesting ideas, this change provides the necessary infrastructure to allow efficient and timely evacuation of [pinned regions](https://openjdk.org/jeps/423) right after they become un-pinned.

  * The fix to the long standing problem of the GCLocker starving threads from progress and causing unnecessary Out-Of-Memory exceptions is in development [JDK-8308507](https://bugs.openjdk.org/browse/JDK-8308507) and actually [out for review](https://github.com/openjdk/jdk/pull/14077).

More to come in the next months :)

## Thanks go to…

Everyone that contributed to another great JDK release. See you in the next release :)

*Thomas*
