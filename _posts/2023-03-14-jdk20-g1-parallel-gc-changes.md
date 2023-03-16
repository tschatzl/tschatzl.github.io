---
layout: post
title:  "JDK 20 G1/Parallel/Serial GC changes"
date:   2023-03-14 12:00:00 +0200
tags: [GC, G1, Parallel, Serial, JDK 20, Performance]
---

Yet another JDK release that is on track - JDK 20 GA is [almost here](https://openjdk.java.net/projects/jdk/20/). Another opportunity to summarize changes and improvements in Hotspot´s stop-the-world garbage collectors for the JDK 20 release.

This release does not contain any JEP for the garbage collection area, but the JEP for [Generational ZGC](https://mail.openjdk.org/pipermail/jdk-dev/2023-March/007457.html) reached Candidate status just recently, so maybe it will be ready for JDK 21. :)

Other than that, the full list of changes for the entire Hotspot GC subcomponent for JDK 20 is [here](https://bugs.openjdk.org/browse/JDK-8298968?jql=project%20%3D%20JDK%20AND%20issuetype%20in%20standardIssueTypes()%20AND%20status%20in%20(Resolved%2C%20Closed)%20AND%20fixVersion%20%3D%20%2220%22%20AND%20component%20%3D%20hotspot%20AND%20Subcomponent%20in%20(gc%2C%20gc%2C%20gc%2C%20gc%2C%20gc)), showing around 220 changes in total being resolved or closed.

## Parallel GC

  * The only notable improvement to Parallel GC is parallelization of the handling of objects that cross compaction regions [JDK-8292296](https://bugs.openjdk.org/browse/JDK-8292296) during Full GC.

    Instead of having a single-threaded phase at the end that iterates and fixes up these objects, worker threads collect objects crossing their local compaction regions and handle them on their own. M. Gasson, the contributor, [reports](https://github.com/openjdk/jdk/pull/10313) 20% reduction of full gc pause times in select cases.

## Serial GC

  * There were no notable changes for Serial GC. There have been quite a few changes cleaning up the Serial GC code.

## G1 GC

Broadly speaking JDK 20 provides all items on that long "What's next" list in the previous report - and more :)

  * [JDK-8210708](https://bugs.openjdk.java.net/browse/JDK-8210708) reduces G1 native memory footprint by ~1.5% of Java heap size by removing one of the mark bitmaps spanning the entire Java heap. The [Concurrent Marking in G1](/2022/08/04/concurrent-marking.html) blog post includes a thorough discussion of the change.
 
    That posting also obsoletes the information about concurrent marking in the original [G1 paper](http://cs.williams.edu/~dbarowy/cs334s18/assets/p37-detlefs.pdf). There is not much left in the paper that is accurate about current G1.

  * Actually, in some way that blog post is out-of-date already too: [JDK-8295118](https://bugs.openjdk.org/browse/JDK-8295118) moves a sometimes lengthy part of preparation for concurrent marking called `Clear Claimed Marks` out of the concurrent start garbage collection pause. A new phase called `Concurrent Clear Claimed Marks` will show up in the logs with `gc+marking=debug` logging.

  * Preparing for future [region pinning support](https://openjdk.org/jeps/423) in G1, [JDK-8256265](https://bugs.openjdk.org/browse/JDK-8256265) decreases the parallelization granularity when handling regions that were pinned by the user (or could not be evacuated because there is no space left in the Java heap): instead of handing out entire regions on a per-thread basis the task granularity is parts of regions now.

    This makes threads share work better, significantly reducing unproductive waiting for completion.

  * [JDK-8137022](https://bugs.openjdk.org/browse/JDK-8137022) makes refinement thread control more adaptive: previously, during a garbage collection pause G1 calculated discrete thresholds at which a particular refinement thread got activated (and deactivated) at mutator time to help with refinement. This calculation has been based on the allowed time the user wanted to spend on refining cards (the option `-XX:G1RSetUpdatingPauseTimePercent`) in the garbage collection pause, the most recent interval to the next pause and lots of magic.

    Due to not observing the application directly, like accounting for recent incoming and refinement thread processing rate of work, and the expected time left until the next garbage collection, refinement thread control was strongly biased to wake up and do more work than necessary in a bursty fashion to avoid too much work left for the collection pause. This behavior not only wastes CPU cycles in the refinement threads, but has another drawback: few cards were left unrefined on the Java heap - that sounds good to have, but below a certain level this can be somewhat counterproductive: the write barrier needs to perform more work whenever there is a new, unvisited card as described [here](/2022/02/15/card-table-card-size.html#refinement) than if the card remains in the queue for later processing.

    This means that not only the additional activity of refinement threads takes away cpu resources, but G1 also repeatedly refined the same cards without changing the work left for the pause much.

    Taking mutator activity into account with the change, G1 better distributes refinement thread activity between pauses, leaves more cards on the refinement task queues for longer, which reduces the rate of generation of new cards and achieves the user's intent (`-XX:G1RSetUpdatingPauseTimePercent`) more deterministically. In the end this often takes less cpu cycles away from the application, providing better throughput.

  * G1 uses thread-local allocation buffers (PLABs) to reduce synchronization overheads during the garbage collection pause. PLABs are resized based on recent allocation patterns to reduce unused space in these buffers at the end of the garbage collection pause. If there is little need for allocation, they shrink, otherwise they grow. This per-collection pause adaptation works well for applications that have fairly constant allocation between garbage collections; however this does not work well if allocation is extremely bursty. The PLABs will be too large, wasting space, or too small, wasting cpu cycles in the next garbage collection.

    On some platforms we noticed very long pause spikes in the range of seconds due to that. The change in [JDK-8288966](https://bugs.openjdk.org/browse/JDK-8288966) adds some reasonably aggressive resizing of PLABs _during_ garbage collection to counter these situations.

  * A significant amount of effort has been spent in JDK 20 to improve predictions that are used to size the young generation which is ultimately responsible how long garbage collection takes ([this](https://bugs.openjdk.org/browse/JDK-8296419?jql=labels%20%3D%20gc-g1-prediction) bug tracker query gives an overview).

    Better prediction makes G1 observe the pause time goal better (as specified by `-XX:MaxGCPauseMillis`), which reduces pause time overshoots and increases usage of the available pause time goal by using more young generation regions per garbage collection. This increases pause time within the allowed goal, but can decrease the number of garbage collection significantly. We have measured applications to spend 10-15% less time in garbage collection pauses with these changes.

  * JDK 20 disables preventive garbage collections by default with [JDK-8293861](https://bugs.openjdk.org/browse/JDK-8293861). Preventive garbage collections were introduced in JDK 19 to avoid G1 not having enough Java heap memory for evacuating objects (also called "evacuation failure") during garbage collection. Handling of regions that experienced an evacuation failure has traditionally been fairly slow, so the argument for that feature has been that it is better to do a garbage collection that does not have such an evacuation failure preemptively in the hope that it frees up enough memory to avoid these evacuation failures completely.

    The problem is how to anticipate this situation correctly: the predictions G1 used to determine the need for a preventive collection proved to be suboptimal, often starting them too early, introducing extra garbage collections early or not at all making the application experience evacuation failures anyway. Further, these kind of garbage collections made garbage collections more irregular, which if they occurred, made predictions harder.

In summary, I believe there are significant additions to garbage collection worth upgrading, even if only later with JDK 21.

## What's next

Work for JDK 21 is already full steam ahead. Here is a short list of interesting changes that are already integrated or currently in development. As usual, there are no guarantees that they will make the next release - although it is highly likely that the ones already integrated will stay ;).

  * After improving refinement thread control, [JDK-8225409](https://bugs.openjdk.org/browse/JDK-8225409) removes the Hot Card Cache. This data structure has been in some way a workaround for the problem described above that refinement has been too aggressive, and it is advantageous to keep cards unrefined for longer. In this mechanism, for every card G1 kept a counter about how often it had been refined in this mutator cycle, and if that count exceeded a threshold, the card was not refined and kept in a small fixed size ring buffer until it would either be evicted from there because of overflow or garbage collection occurred.

    At the latest after the JDK 20 refinement thread control changes above, we could not measure any impact of the Hot Card Cache any more. The current refinement control seems more than lazy enough with refinement to subsume Hot Card Cache functionality.

    This frees up 0.2% of Java heap size of native memory for other use.

  * Current ongoing work on improving G1 behavior in presence of high region-level fragmentation. Currently, if G1 can not find a contiguous range of memory for a humongous object allocation even after a Full GC, the VM exits with an `OutOfMemoryException` although in aggregate there would be enough memory available. This [PR](https://github.com/openjdk/jdk/pull/12830) changes the behavior of the "last-ditch" G1 Full collection to also move humongous objects to create more contiguous memory. This should in many cases avoid the `OutOfMemoryException` at the cost of a longer full collections.

    Since "last-ditch" G1 Full collections occur only right after there had been a regular G1 Full Collection, the lengthening of that collection seems an acceptable trade-off to avoid VM failure.

  * To improve the resilience of G1 against regions that failed evacuation (or future pinned regions) swamping old generation, there is [work](https://bugs.openjdk.org/browse/JDK-8140326) progressing on allowing G1 to evacuate these Old regions at _any_ garbage collection as soon as possible after they are generated.

    The current policy for regions that failed evacuation is to make them Old regions, which means G1 can not allocate into them any more although most often they only contain a few live objects. Currently the only way to reclaim space from them is waiting for the next concurrent marking to complete. If many regions are failing evacuation potentially across multiple garbage collections, the Java heap fills up quickly with these typically very lightly occupied regions. This can quickly lead to a Full GC.

    This completely removes the previous assumption that young-only collections would never collect old generation regions. Although technically, G1 has already been trying to collect some humongous objects which are Old generation regions since JDK 8u65, so that assumption has not strictly held for a long time.

  * Some unsuccessful attempts to improve the predictions for preventive garbage collections lead to the decision to remove this functionality completely with [JDK-8297639](https://bugs.openjdk.org/browse/JDK-8297639) given that handling of regions that failed evacuation has little overhead, and will be even less with the work suggested above. The additional garbage collections cause many problems without providing benefits.

More to come in the next months :)

## Thanks go to…

Everyone that contributed to another great JDK release. See you in the next (LTS) release :)

*Thomas*
