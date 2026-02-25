---
layout: post
title:  "JDK 26 G1/Parallel/Serial GC changes"
date:   2026-02-26 11:00:00 +0200
tags: [GC, G1, Parallel, Serial, JDK 26, Performance]
---

The OpenJDK project is about to release JDK 26 in a few days. This is a good opportunity to add information about notable changes to the stop-the-world collectors in the Hotspot VM.

The complete list of resolved/closed changes for the GC subcomponent for JDK 26 is [here](https://bugs.openjdk.org/issues/?jql=project%20%3D%20JDK%20AND%20status%20%3D%20Resolved%20AND%20fixVersion%20%3D%20%2226%22%20AND%20component%20%3D%20hotspot%20AND%20Subcomponent%20%3D%20gc%20ORDER%20BY%20summary%20ASC%2C%20status%20DESC), containing around 380 changes that were resolved or closed in total.

Close to double the amount of issues were closed in JDK 26 compared to the last release. Some brief analysis indicates that the composition of changes is different: there were many small changes both end-user visible or just code quality improvements rather than a few single, larger changes. Most larger changes were integrated early, meaning most development time was spent in earlier releases. Additionally a few new contributors to the GC component had meaningful impact.

Let's start with the breakdown of in my opinion changes worth mentioning categorized by stop-the-world garbage collector for JDK 26.

## All Collectors

This release there have been a significant amount of changes affecting all garbage collectors (including non-stop-the-world collectors):

  * garbage collection CPU usage accounting improved significantly. The VM now tracks GC CPU usage in a more complete manner, including both time spent inside garbage collection pauses as well as CPU time spent in concurrent work (e.g. concurrent marking) or adjacent tasks like string deduplication.

    In earlier releases the information was fragmented or required  manual aggregation, and were difficult or impossible to obtain reliably for users.
    
    Now there are multiple ways of obtaining that information - one way to do so is enabling `gc+cpu=info` logging that prints something like the following at VM exit:
  ```
[info ][cpu] === CPU time Statistics =========================================
[info ][cpu]                                                          CPUs
[info ][cpu]                                              s       %  utilized
[info ][cpu]    Process
[info ][cpu]      Total                            443.4045  100.00       9.0
[info ][cpu]      Garbage Collection               365.2198   82.37       7.4
[info ][cpu]        GC Threads                     365.1982   82.36       7.4
[info ][cpu]        VM Thread                        0.0216    0.00       0.0
[info ][cpu] =================================================================
  ```
    The output shows Total Process and Garbage Collection CPU usage in absolute and relative terms ([JDK-8359110](https://bugs.openjdk.java.net/browse/JDK-8359110)). Hsperf counters were added or updated, see [JDK-8315149](https://bugs.openjdk.org/browse/JDK-8315149) for the exact counter names. There is also a programmatic way of obtaining this information in your own application.

    This [article](https://norlinder.nu/posts/GC-Cost-CPU-vs-Memory/) from one of my colleagues about analysis of the CPU-memory relationship in garbage collection shows and uses this new information.
  * [JEP 516: Ahead-of-Time Object Caching with Any GC](https://openjdk.org/jeps/516) introduces a new mechanism for loading [ahead-of-time](https://openjdk.org/jeps/483) (AOT) linked and loaded objects, with the explicit goal of working well with all Hotspot garbage collectors. The reason is garbage collection algorithm independent support of Project [Leyden](https://openjdk.org/projects/leyden/).
  
    The previous AOT cache disk format was GC algorithm and VM option specific, which made extending it, particularly towards ZGC, difficult. The new format is GC-independent and independent of VM options.
    
    The drawback is that the new algorithm requires more work at startup. The implementation of JEP 516 mitigates this startup impact by populating the objects concurrently to the application during startup.
    
    To enable the new format, use the new `-XX:+AOTStreamableObjects` command line option.
  * Robustness of the shutdown process has been improved. Prior to JDK 26 it was possible for a JVMTI agent to allocate memory after the garbage collection subsystem had already shut down which could lead to unexpected behavior like hangs, out-of-memory errors or crashes.

    Relevant issues include [JDK-8367902](https://bugs.openjdk.org/browse/JDK-8367902) and [JDK-8366865](https://bugs.openjdk.org/browse/JDK-8366865).
  * For a significant amount of command line options the defaults changed, or they have been deprecated for removal.
  
    `-XX:InitialRAMPercentage` which determines the amount of the Java heap that is committed at startup based on a percentage of installed memory has been defaulted to 0. I.e. the amount of total memory installed is not relevant to initial Java heap size any more. ([JDK-8371987](https://bugs.openjdk.org/browse/JDK-8371987)).
  
    In practice, the amount of memory is committed by the VM at startup (impacting startup time) now only follows the minimum heap size derived by other mechanisms (`-XX:MinHeapSize`/`-Xms`/`-XX:InitialHeapSize` and others), instead of arbitrarily initializing a percentage of installed RAM of your machine for the Java heap during startup.
  
    The [CSR](https://bugs.openjdk.org/browse/JDK-8371987) for this change further details the reasoning for this change.
  
    `-XX:AggressiveHeap`, which collects a set of optimizations mainly for "long-running memory-intensive benchmarking scenarios", has also been deprecated. Multiple reasons including the generic name, its focus on SPECjbb, the complexity of finding a similar set of tunings for nowadays' very diverse landscape of these applications led to its impending removal. The [CSR](https://bugs.openjdk.org/browse/JDK-8370814) lists the options it enables if one wants to recreate its effects manually.

    The set of little known and typically only for internal testing used options that were deprecated for removal in the next releases are `-XX:MaxRAM` [JDK-8369346](https://bugs.openjdk.org/browse/JDK-8369346), `-XX:AlwaysActAsServerClassMachine`, and `-XX:AlwaysActAsServerClassMachine` [JDK-8370844](https://bugs.openjdk.org/browse/JDK-8370844). Their effect can similarly be replaced by other options.

  * Finally, there is a new JFR event that details string deduplication results ([JDK-8360540](https://bugs.openjdk.org/browse/JDK-8360540), reducing the need to rely on log interpretation.

## Parallel GC

Parallel GC only saw a fair amount of cleanup changes in JDK 26. The most important for end-users is probably the deprecation and obsoletion of several VM options.

Obsoletion of the `-XX:ParallelRefProcEnabled` option is the most likely to affect end users. It controls parallelization of the *java.lang.ref.Reference* processing phase. That option has been enabled by default for a long time without issues, so there was no reason to keep it. Particularly, if one ever needs to reduce the parallelism of this phase, `-XX:ReferencesPerThread` can be used to modify work per thread to achieve the same effect.

G1 garbage collector also used this option, so it is affected the same by this change.

Next to that, several rather unknown Parallel GC-specific used options were deprecated, obsoleted or removed - `-XX:PSChunkLargeArrays`, `-XX:HeapMaximumCompactionInterval`, and the debug-build only `GCExpandToAllocateDelayMillis`.

## Serial GC

There were some significant internal changes (e.g. Eden and Survivor spaces were swapped in address space [JDK-8368740](Jttps://bugs.openjdk.org/browse/DK-8368740)) and refactoring, there were no major user-visible changes in this release.

## G1 GC

G1 received the largest number of changes among the stop-the-world collectors in addition to the generic ones.

  * The by far largest and probably most impactful single change for G1 for some time is integration of [JEP 522: G1: Improve Throughput By Improving Synchronization](https://openjdk.org/jeps/522).

    An earlier [post](/2025/02/21/new-write-barriers.html) describes the change in detail: in essence, memory synchronization between garbage collector and application in G1 has been reduced to almost the same level as Serial and Parallel GC (there is opportunity to improve here). This improves throughput and makes G1 more competitive with Parallel GC, with mininmal compromises on other algorithm attributes that make G1 a solid default choice: low, predictable pauses, scalability and reasonable native memory usage. G1 should now achieve almost the raw throughput of Parallel GC without long full-heap collections.

    This change is a significant milestone to make G1 [the only true default collector](https://openjdk.org/jeps/523), although more work is necessary.

  * Throughout this release cycle, most of our attention has been on [Automatic Heap Sizing](https://bugs.openjdk.org/browse/JDK-8359211) work for G1. The goal in this effort is to have G1 size the Java heap based on environmental conditions: if there is abundant free memory, use up more of the available resources if it helps performance, but scale down total VM memory usage as neighbouring processes or other system conditions reduce available memory. One result is that there should be little need for manual tuning of maximum heap size (`-Xmx`/`-XX:MaxHeapSize`) in the future in many deployments.

    Steps towards this goal include re-evaluating Java heap memory usage more frequently ([JDK-8238687](https://bugs.openjdk.org/browse/JDK-8238687)) based on CPU usage. This ties in with more appropriate GC cpu usage monitoring ([JDK-8359348](https://bugs.openjdk.org/browse/JDK-8359348)) - now also considering concurrent work for GC CPU usage calculations.
    
    This aligns with a long overdue reduction of default G1 GC CPU usage goal ([JDK-8247843](https://bugs.openjdk.org/browse/JDK-8247843)). The change reduces its default value from 8% to 4%. The original value dates back to G1's initial release, but over the years G1 got much faster and performs much more consistently across a larger variety of workloads, and can often maintain the same performance on a tighter heap and lower GC CPU usage than before.

  * One of the most often asked for functionality from the Parallel collector is support for `-XX:UseGCOverheadLimit`. When active (which is the default), the VM throws an Out-Of-Memory exception if GC CPU usage stays higher than a threshold (`-XX:GCTimeLimit`, default 98%) and free Java heap space below another threshold (`-XX:GCHeapFreeLimit`, default 2%) for a long time. The intent is to avoid situation where a severely misconfigured VM spends almost all of its time in GC without the application actually being able to progress.

    Log messages at `gc+info` level and below give some additional context about how the trigger conditions had been reached.

  * As always, there also were a set of general performance improvements: some parallelization efforts (e.g. [JDK-8363932](https://bugs.openjdk.org/browse/JDK-8363932)) and an improvement to the eager reclaim mechanism: the behavior described [here](https://docs.oracle.com/en/java/javase/25/gctuning/garbage-first-g1-garbage-collector1.html#GUID-D74F3CC7-CC9F-45B5-B03D-510AEEAC2DAC) now applies to all types of humongous objects ([JDK-8048180](https://bugs.openjdk.org/browse/JDK-8048180)). The CR contains a performance graph showing how this can reduce unexpectedly frequent or long collections in headroom-constrained scenarios.
  
    Finally, startup improvements (e.g. [JDK-8371019](https://bugs.openjdk.org/browse/JDK-8371019)) help with Project [Leyden](https://openjdk.org/projects/leyden).

## What's next

Our focus remains on Automatic Heap Sizing, improving control of various aspects of memory management so that G1 - and in general all the stop-the-world collectors - can respond more intelligently to external conditions and user intent. In parallel, there is also ongoing work for [JEP 401](https://openjdk.org/jeps/401) in the garbage collection area.

## Thanks go to…

... as usual to everyone that contributed to another strong JDK release. To me, this one has been bigger and more broadly impactful than I expected when starting to write this overview. Looking forward to see you next (even bigger) release :)

*Thomas*

