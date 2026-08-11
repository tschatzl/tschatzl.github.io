---
layout: post
title:  "JDK 27 G1/Parallel/Serial GC changes"
date:   2026-08-10 11:00:00 +0200
tags: [GC, G1, Parallel, Serial, JDK 27, Performance]
---

OpenJDK 28 release is on the horizon I thought it would be time to discuss interesting changes in OpenJDK's stop-the-world collectors in the HotSpot VM once more.

The complete list of resolved/closed changes for the GC subcomponent for JDK 27 is [here](https://bugs.openjdk.org/issues/?jql=project%20%3D%20JDK%20AND%20status%20%3D%20Resolved%20AND%20fixVersion%20%3D%20%2227%22%20AND%20component%20%3D%20hotspot%20AND%20Subcomponent%20%3D%20gc%20ORDER%20BY%20summary%20ASC%2C%20status%20DESC), containing around 350 changes in total.

This raw count aligns with last release; similar to then, these numbers may be a bit inflated because of a fairly unusual number of refactorings and cleanups in the GC code. One particular area has been replacing our idiom to indicate cross-thread shared variables, from using `volatile` and appropriate `AtomicAccess` methods on them to using `Atomic<T>` introduced in [JDK-8367013](https://bugs.openjdk.org/browse/JDK-8367013) to make their use more explicit. Other notable refactorings include G1 state machine cleanups (including some [very very old](https://bugs.openjdk.org/browse/JDK-8080226) ones) and other code improvements. These make up around half of the changes.

Roughly another 35% of changes deal with bug fixes and general correctness and robustness improvements. The remainder is about new or materially revised behavior and features for HotSpot's STW collectors - and this is the part this post will dig down further now :)

Mixed in with these changes was infrastructure work for [JEP 401: Value Objects (Preview)](https://openjdk.org/jeps/401), like adapting object iterators and eager reclaim for value objects.

## G1 GC

The most impactful change in this release has probably been [JEP 523: Make G1 the Default Garbage Collector in All Environments](https://openjdk.org/jeps/523). G1 is now the default collector if one does not specify any garbage collection algorithm explicitly on the command line. No exceptions, no Serial GC in any case by default (that is, unless G1 is not included in your distribution of OpenJDK). We thought that G1 fit the bill of a default collector nicely, and selecting something else depending on arcane environmental conditions was more of a burden than an advantage. Throughput, footprint and latency profile were step-by-step closing in on Serial GC particularly in the past few releases, so we felt it was about time to flip the switch.

One may still select Serial GC with the `-XX:+UseSerialGC` option if you observe notable differences to get pure, unchanged Serial GC. There obviously are some cases where for your application profile Serial GC is the better choice - G1 is not the best everywhere, but we think it is certainly the best to start off with.

Next to that change, there are a few others:

* G1 no longer resizes the heap after Full GC based on the (arbitrary) `-XX:MinHeapFreeRatio` and `-XX:MaxHeapFreeRatio` percentages, that is, their default values have been set so that they do not impact regular, other heuristics based sizing (e.g. garbage collector CPU usage relative to the application's CPU usage). [JDK-8238686](https://bugs.openjdk.org/browse/JDK-8238686) set their defaults to `0` and `100` percent respectively, from `40` and `70` respectively.

  The original defaults caused this heuristic and others like the CPU usage based one to fight each other, resulting in unnecessary Java heap size modifications, causing performance issues when a full heap garbage collection shrunk the heap and then the other heuristics almost immediately undid these changes.

  Other than that, these flags' functionality is preserved.

* Adaptive start of concurrent marking has been improved to be more resistant to adverse conditions that previously caused unnecessary continuous concurrent work ([JDK-8379846](https://bugs.openjdk.org/browse/JDK-8379846), [JDK-8381006](https://bugs.openjdk.org/browse/JDK-8381006)).

* Previously, humongous objects that actually could be reclaimed were inadvertently held live by weak references (e.g. `java.lang.ref.Reference` instances), fixed with [JDK-8378331](https://bugs.openjdk.org/browse/JDK-8378331) and [JDK-8378336](https://bugs.openjdk.org/browse/JDK-8378336).

* Minor observable changes include that the G1 Cleanup pauses do not update `MemoryPoolMXBean.getCollectionUsage()` as they do not change the Java heap in [JDK-8386332](https://bugs.openjdk.org/browse/JDK-8386332). [JDK-8373894](https://bugs.openjdk.org/browse/JDK-8373894) where previously garbage collections that could not find enough space to copy to were not counted in garbage collector CPU usage. Including them improves heap-sizing decisions.

## Parallel GC

Parallel GC had some notable bug fixes that may improve performance:

* The adaptive tenuring threshold determines how many young collections an object may stay within the young generation. Previously, the threshold tended to only increase, but very rarely decrease. This means such a persistently high threshold could leave older objects taking much of the survivor space, forcing younger objects to be promoted prematurely when it filled up. With [JDK-8380590](https://bugs.openjdk.org/browse/JDK-8380590), the threshold can decrease as well, promoting longer-lived objects sooner and preserving younger objects that are likely to die shortly in the young generation.

* Parallel GC now correctly expands the Java heap when repeated large allocations cause thousands of Full GCs while the Java heap did not grow even if there was enough headroom [JDK-8377561](https://bugs.openjdk.org/browse/JDK-8377561).

## Serial GC

No notable Serial GC specific changes - except for not being the default collector anymore.

## All Collectors

There were a few interesting changes to all collectors:

* TLAB sizing now better handles applications that create many short-lived, lightly allocating threads. This avoids oversizing TLABs that result in a large amount of waste, increasing the number of collections [JDK-8381834](https://bugs.openjdk.org/browse/JDK-8381834).

* [JDK-8372348](https://bugs.openjdk.org/browse/JDK-8372348) improves string deduplication logging and JFR events. Logging adds “new unknown” strings and gives more meaningful aggregate byte counts.

## What's next

Our focus remains on Automatic Heap Sizing, improving control of various aspects of memory management so that G1 can respond more intelligently to external conditions and user intent.

## Thanks go to…

... as usual to everyone that contributed to another JDK release. Finally, with some major refactorings (`Atomic<T>`) complete, more time is available for other items. Looking forward seeing you in the next - maybe even bigger - release :)

*Thomas*

