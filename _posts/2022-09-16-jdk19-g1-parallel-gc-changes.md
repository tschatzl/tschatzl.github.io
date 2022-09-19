---
layout: post
title:  "JDK 19 G1/Parallel/Serial GC changes"
date:   2022-09-16 12:00:00 +0200
tags: [GC, G1, Parallel, Serial, JDK 19, Performance]
---

JDK 19 GA is [almost here](https://openjdk.java.net/projects/jdk/19/). Let me summarize changes and in particular improvements in Hotspot´s stop-the-world garbage collectors in that release - G1 and Parallel GC - for yet another time. :)

Overall there has been no JEP specifically for the garbage collection area for this release. There were two other large changes that required assistance in the garbage collection area: First, the [JEP 425: Virtual Threads (Preview)](https://openjdk.org/jeps/425) JEP, second [JDK 8 Maintenance Release 4](https://jdk.java.net/java-se-ri/8-MR4).

Virtual threads introduce on-Java heap objects that contain thread stacks. These new kind of objects need to be handled properly by all garbage collection algorithms. However the main complication with these objects is that unloading stale compiled code referenced by them requires some fairly complex dance between garbage collectors and the code cache sweeper ("GC for compiled code"), at least in JDK 19. Work has started on improvements already, for example by having the garbage collector taking [complete control over method unloading](https://bugs.openjdk.org/browse/JDK-8290025) or [too aggressive sweeping with Loom](https://bugs.openjdk.org/browse/JDK-8284404).

When Loom is not enabled via the preview option, these problems do not occur.

JDK 8 got a maintenance release. The changes detailed [here](https://jcp.org/aboutJava/communityprocess/maintenance/jsr337/jsr337-mr4-changes.html)) describe changes to undefined behavior in `java.lang.ref.Reference` processing that could previously cause crashes. A side effect of this change is that reference processing in the garbage collectors could be improved a bit.

Other than that, the full list of changes for the entire Hotspot GC subcomponent for JDK 19 is [here](https://bugs.openjdk.org/issues/?jql=project%20%3D%20JDK%20AND%20issuetype%20in%20standardIssueTypes()%20AND%20status%20in%20(Resolved%2C%20Closed)%20AND%20fixVersion%20%3D%20%2219%22%20AND%20component%20%3D%20hotspot%20AND%20Subcomponent%20in%20(gc%2C%20gc%2C%20gc%2C%20gc%2C%20gc)), showing around 200 changes in total being resolved or closed. Above two projects left their impact :)

A brief look over to [ZGC](https://wiki.openjdk.java.net/display/zgc/Main) shows that the changes this release were minor: [JDK-8255495](https://bugs.openjdk.org/browse/JDK-8255495) added partial CDS support (class data is archived, but not heap objects), in addition to some bug fixes. For some time now the main focus has been on generational ZGC - you can follow development [here](https://github.com/openjdk/zgc/tree/zgc_generational), please look forward to it.

## Parallel improvements

  * Parallel GC now also supports **archived heap objects**. This feature has been added with [JDK-8274788](https://bugs.openjdk.org/browse/JDK-8274788).

  * [JDK-8280705](https://bugs.openjdk.org/browse/JDK-8280705) improves work distribution in the first phase of full collection, sometimes yielding very good performance improvements.

## Serial GC

 * There were no notable changes for Serial GC.

## G1 GC ##

Except [JDK-8280396](https://bugs.openjdk.org/browse/JDK-8280396) that implements that same optimization mentioned above for Parallel GC for G1 Full GC there were no notable improvements.

JDK 19 introduced a native memory usage [regression](https://bugs.openjdk.org/browse/JDK-8292654) very late in the JDK 19 release process so that the fix could not be integrated any more. The [release note](https://bugs.openjdk.org/browse/JDK-8293707) provides a workaround. The regression has already been addressed for JDK 19.0.1 and later.

## What's next

Of course we have long since begun working on JDK 20. Here is a short list of interesting changes that are currently in development and you may want to look out for. Many of them are actually already integrated - Virtual Threads and the JDK 8 MR4 took away quite a few resources that made changes slip past FC for JDK 19. Apart from those, there are no guarantees that they will make JDK 20. They are going to be integrated when they are done ;-)

  * G1 native memory footprint has been reduced significantly with the integration of [JDK-8210708](https://bugs.openjdk.java.net/browse/JDK-8210708) into JDK 20, removing one of the mark bitmaps spanning the entire Java heap. The [Concurrent Marking in G1](/2022/08/04/concurrent-marking.html) blog post includes a thorough discussion of the change, and its impact.
  
    The change finally realizes the native memory footprint originally forecasted for JDK 19 in the "Prototype (calculated)" line [here](/2022/03/14/jdk18-g1-parallel-gc-changes.html#memory-usage-jdk19-forecast).

  * Another important [change](https://bugs.openjdk.org/browse/JDK-8256265) that improves parallelism (and performance) during evacuation failure has already been integrated into JDK 20, as a further step towards support for region pinning as described in [JEP 423: Region Pinning in G1](https://openjdk.org/jeps/423) in the future.

  * Concurrent refinement control is currently being revised as part of [JDK-8137022](https://bugs.openjdk.org/browse/JDK-8137022). Generally, the rate of processing cards to refine concurrently is now smoother. As a side effect, G1 will then leave more dirty cards in the queue for longer, reducing the rate of generation of new dirty cards, and ultimately reducing the amount of concurrent refinement work overall. This can result in increased throughput in applications.

  * Applications that promote objects in bursts could exhibit very long pause time spikes in the range of a low amount of seconds instead of 1-200ms. Particularly large Aarch64 machines were susceptible to that. This is caused by untrained (or mis-trained) sizes for promotion allocation buffers (PLABs). [JDK-8288966](https://bugs.openjdk.org/browse/JDK-8288966) adds some reasonably aggressive resizing of PLABs during garbage collection to solve the problem.
  
  * There is ongoing investigation to improve predictions for G1 to make it better observe pause time goal (`-XX:MaxGCPauseMillis`). The goal is to have less overshoots (less latency spikes) and less undershoots so that less garbage collections are required while keeping the pause time goal. In some applications this can significantly lessen the total time spent in garbage collections, leaving more time for the application. Parts that work include e.g. [JDK-8231731](https://bugs.openjdk.org/browse/JDK-8231731).

  * Just recently there has been renewed [interest](https://mail.openjdk.org/pipermail/hotspot-dev/2022-September/064190.html) in implementing a few long-standing improvements to make Hotspot garbage collectors more runtime configurable (e.g. for G1: [JDK-8236073](https://bugs.openjdk.org/browse/JDK-8236073), [JDK-8204088](https://bugs.openjdk.org/browse/JDK-8204088)) and nicer to other tenants (e.g. [JDK-8238687](https://bugs.openjdk.org/browse/JDK-8238687)) so that they can be more container friendly.

More to come :)

## Thanks go to…

Everyone that contributed to another great JDK release. See you in the next release :)

*Thomas*
