---
layout: post
title:  "What is... a Concurrent Undo Cycle"
date:   2023-09-28 12:00:00 +0200
tags: [GC, G1, JDK 16, Performance, Optimization, Marking]
---

Recently I got questions about what **Concurrent Undo Cycle** messages in the logs mean - in this post I would like to explain this optimization and what it means for your application.

## Introduction

The G1 garbage collector performs concurrent whole heap liveness analysis (marking) concurrently to the application. This process starts when G1 notices that old generation Java heap occupancy reaches the *Initiating Heap Occupancy Percent* (IHOP). The IHOP value is dynamically calculated based on typical duration of the concurrent marking and allocation rate into the old generation.

When the garbage collector notices that old generation heap occupancy reaches this IHOP value, the next garbage collection will be a *Concurrent Start* garbage collection pause, followed by concurrent marking. After completion, G1 would start a mixed phase, collecting old generation heap regions.

## Problem

A concurrent marking cycle can take a while and takes CPU resources. For example, below log snippet shows that G1 required around 4,2 seconds to finish the concurrent marking cycle. This process includes traversing the old generation object graph and some setup for the following old generation evacuation (described in detail in [this post](/2022/08/04/concurrent-marking.html)).

```
[15,431s][info   ][gc       ] GC(12) Pause Young (Concurrent Start) (G1 Evacuation Pause) 11263M->10895M(20480M) 153,495ms
[15,431s][info   ][gc       ] GC(13) Concurrent Mark Cycle
[15,431s][info   ][gc,marking] GC(13) Concurrent Scan Root Regions
[15,431s][debug  ][gc,ergo   ] GC(13) Running G1 Root Region Scan using 6 workers for 41 work units.
[15,507s][info   ][gc,marking] GC(13) Concurrent Scan Root Regions 75,589ms
[15,507s][info   ][gc,marking] GC(13) Concurrent Mark
[15,507s][info   ][gc,marking] GC(13) Concurrent Mark From Roots
[18,731s][info   ][gc,marking] GC(13) Concurrent Mark From Roots 3224,174ms
[18,731s][info   ][gc,marking] GC(13) Concurrent Preclean
[18,731s][info   ][gc,marking] GC(13) Concurrent Preclean 0,038ms
[18,734s][info   ][gc        ] GC(13) Pause Remark 11287M->11287M(20480M) 2,292ms
[18,734s][info   ][gc,marking] GC(13) Concurrent Mark 3226,866ms
[18,734s][info   ][gc,marking] GC(13) Concurrent Rebuild Remembered Sets and Scrub Regions
[19,687s][info   ][gc,marking] GC(13) Concurrent Rebuild Remembered Sets and Scrub Regions 953,419ms
[19,688s][info   ][gc        ] GC(13) Pause Cleanup 11447M->11447M(20480M) 0,523ms
[19,688s][info   ][gc,marking] GC(13) Concurrent Clear Claimed Marks
[19,688s][info   ][gc,marking] GC(13) Concurrent Clear Claimed Marks 0,044ms
[19,688s][info   ][gc,marking] GC(13) Concurrent Cleanup for Next Mark
[19,695s][info   ][gc,marking] GC(13) Concurrent Cleanup for Next Mark 6,145ms
[19,695s][info   ][gc        ] GC(13) Concurrent Mark Cycle 4263,187ms
```

So what if after the *Concurrent Start* pause the conditions for starting the concurrent mark cycle do not apply any more, i.e. the old generation heap occupancy went below the start threshold? G1 **does not start** the whole concurrent marking cycle because memory occupancy indicates that it is not necessary.

The problem is that the pause did some global changes in the *Concurrent Start* garbage collection pause. This is the purpose of the *Concurrent Undo Cycle*.

## Background ##

The *Concurrent Start* garbage collection pause is just like a regular young garbage collection pause. In a regular young collection pause, G1 selects the regions to collect (the whole young generation and parts of the collection set candidates), and scans references from root locations for references into these regions. The referenced objects and their followers are evacuated. At the same time, G1 also performs a limited and conservative liveness analysis for humongous objects to try to (eagerly) reclaim dead humongous objects. After some cleanup the application continues.

The differences of the *Concurrent Start* pause from a regular young collection pause are minor:

  * during scanning the VM root locations, G1 marks locations into the old generation on the mark bitmap. For example, G1 marks a reference from a variable on a thread stack that points to an object into the old generation.

  * G1 determines *root regions*. These are memory areas (typically whole regions) that potentially contain references into the old generation (which will not be marked through). Root regions are similar to above root locations, but their referents are not marked in the bitmap during the garbage collection pause to save time. 
    The following *Concurrent Root Region Scan* part of the marking cycle scans the live objects in the root regions for such references into the old generation and marks their referents in the bitmap. Examples for such regions are Survivor regions, but also collection set candidate regions that are going to be collected in the next garbage collection (potentially during marking). Examples for the latter are regions where evacuation failed.

Both evacuation of collection set candidate regions and eager reclaim above means that in some cases, the old generation occupancy can be significantly smaller after the garbage collection pause.

## The Change ##

[JDK-8240556](https://bugs.openjdk.org/browse/JDK-8240556) implements short-cutting the concurrent marking cycle. If old generation heap occupancy falls below the IHOP value during the *Concurrent Start* pause due to large object eager reclamation, instead of executing the whole marking cycle as shown in the state diagram below, G1 will take the shortcut indicated as a blue arrow to the right.

![Concurrent mark cycle with shortcut](/assets/20230915-concurrent-mark-cycle-states.png){:style="display:block; margin-left:auto; margin-right:auto"}

The log messages above show that the skipped states can result in a substantial saving of cpu usage (and time).
The remaining states comprise a *Concurrent Undo Cycle* as shown below:

![Concurrent undo mark cycle](/assets/20230915-concurrent-undo-cycle-states.png){:style="display:block; margin-left:auto; margin-right:auto"}

These are reflected in the corresponding log output. The following example shows such a snippet for a *Concurrent Undo Cycle*:

```
[6278.870s][info][gc        ] GC(54) Concurrent Undo Cycle
[6278.870s][info][gc,marking] GC(54) Concurrent Clear Claimed Marks
[6278.886s][info][gc,marking] GC(54) Concurrent Clear Claimed Marks 0.016ms
[6278.886s][info][gc,marking] GC(54) Concurrent Cleanup for Next Mark
[6278.948s][info][gc,marking] GC(54) Concurrent Cleanup for Next Mark 77.814ms
[6278.948s][info][gc        ] GC(54) Concurrent Undo Cycle 78.038ms
```

G1 needs to execute the *Concurrent Clear Claimed Marks* and *Concurrent Cleanup for Next Mark* phases only: the first one resets internal information about in-use class loaders, and the second resets the mark bitmap that has been used for marking root references from the VM internal data structures. Everything else is skipped. 

Note that the *Concurrent Cleanup for Next Mark* phase can still take a substantial amount of time, but compared to a regular concurrent marking cycle it should finish much more quickly. Some alternatives that could be even faster are recorded in [JDK-8316355](https://bugs.openjdk.org/browse/JDK-8316355), so if it itches your scratch get in contact. :)

## Summary ##

In G1 a *Concurrent Undo Cycles* is the result of an optimization to avoid the most cpu intensive parts of the concurrent marking cycle, and nothing to be worried about. If they occur, G1 simply found that it is unnecessary to do a complete whole concurrent marking. *Concurrent Undo Cycles* were introduced with JDK 16, so if you are running a recent version of the JDK, you might have already come across one or the other.

See you next time,

*Thomas*



