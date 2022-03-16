---
layout: post
title:  "JDK 18 G1/Parallel/Serial GC changes"
date:   2022-03-14 12:00:00 +0200
#categories: gc g1 parallel serial JDK-18 performance
tags: [GC, G1, Parallel, Serial JDK 18, Performance]
---

Time flew by and JDK 18 GA is [around the corner](https://openjdk.java.net/projects/jdk/18/). Let me summarize changes and in particular improvements in Hotspot´s stop-the-world garbage collectors in that release - G1 and Parallel GC - for another time. :)

There is no delivered JEP particular to the garbage collection area for this release, but one that will have big impact on garbage collection in general: [JEP 421: Deprecate Finalization for Removal](https://openjdk.java.net/jeps/421). The presence of finalizers imposes some performance penalty to garbage collectors in addition to significant code complexity. Personally as a GC engineer I am looking forward to be able to shed them. Finalizers also have so many usage drawbacks as pointed out in the JEP that it seems almost impossible to use in a reliable and safe way anyway.

There is no particular timeline associated to the JEP, and the authors "[...] expect a lengthy transition period before removing finalization [...]", so finalization will stay for a while at least.

The full list of changes for the entire Hotspot GC subcomponent is [here](https://bugs.openjdk.java.net/browse/JDK-8269294?jql=project%20%3D%20JDK%20AND%20issuetype%20in%20standardIssueTypes()%20AND%20status%20in%20(Resolved%2C%20Closed)%20AND%20fixVersion%20%3D%20%2218%22%20AND%20component%20%3D%20hotspot%20AND%20Subcomponent%20in%20(gc%2C%20gc%2C%20gc%2C%20gc%2C%20gc)), clocking in at 300 changes in total. No surprises here.

A brief look over to [ZGC](https://wiki.openjdk.java.net/display/zgc/Main) shows that the changes this release were relatively minor: [JDK-8267186](https://bugs.openjdk.java.net/browse/JDK-8267186) added string deduplication, and there is now a PPC64 port ([JDK-8274851](https://bugs.openjdk.java.net/browse/JDK-8274851)). There were also a few usability improvements and some bugfixes. However, generational ZGC is under very heavy development - you can follow development [here](https://github.com/openjdk/zgc/tree/zgc_generational), look forward to that.

## Generic improvements

  * All OpenJDK garbage collectors now support string deduplication as explained in [JEP 192](https://openjdk.java.net/jeps/192). Parallel GC implemented support with [JDK-8267185](https://bugs.openjdk.java.net/browse/JDK-8267185), Serial GC with [JDK-8272609](https://bugs.openjdk.java.net/browse/JDK-8272609) and ZGC with [JDK-8267186](https://bugs.openjdk.java.net/browse/JDK-8267186). Note that the implementation differs from what has been explained in the JEP, but the basic principle still applies. Enable using the `-XX:+UseStringDeduplication` option.
  
  * V. Chand [contributed a change](https://bugs.openjdk.java.net/browse/JDK-8272773) that allows configuration of the card table card size. The `-XX:GCCardSizeInBytes` can be used to change this value, with allowed values of `128`, `256`, `512` and `1024` (the last is 64 bit only).

    Changing this value can have significant impact on pause time length. Since this is an interesting topic to me, I wrote an [extra blog post](/2022/02/15/card-table-card-size.html) containing a more elaborate discussion.

## Serial GC

 * Serial GC adds **support for archived heap objects** in [JDK-8273508](https://bugs.openjdk.java.net/browse/JDK-8273508). This can reduce startup time significantly. Usage is the same as in G1.

## G1 GC

There have been quite a few user-visible changes specific to G1 this release:

  * The [remembered set](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-garbage-collector-tuning.html#GUID-A0343B53-A690-4DDE-98F9-9877096DBF0F) rewrite covered by [JDK-8017163](https://bugs.openjdk.java.net/browse/JDK-8017163) **massively reduces G1 native memory consumption at no cost**. It reduces native memory footprint for remembered sets significantly. The following figure compares native memory consumption as reported by [NMT](https://docs.oracle.com/en/java/javase/17/vm/native-memory-tracking.html) for the GC component for some object cache-like application for JDK 17 and JDK 18.
  
    ![Memory usage BigRAMTester 20GB](/assets/20220215-bigramtester-memoryusage.png)

    Native memory usage, in particular remembered set memory usage, has gone down from around 2GB peak (JDK 17, blue) to 1.3GB (JDK 18, purple) peak in stable state, saving around 35-40% of memory.
    
    (Trivia: JDK 8 uses almost 5.8GB of native memory for the same application, JDK 11 4GB).
    
    The change also adds a new NMT category called `GCCardSet` which counts the native memory usage of the G1 remembered set only; the existing `GC` NMT category contains all other garbage collection native memory usage. In the graph above the `GC` component only native memory usage for JDK 18 is indicated by the yellow "Floor" line.

    Following is an example on how these categories look like in the NMT report:

        -                        GC (reserved=883951KB, committed=883951KB)
                                    (malloc=72291KB #10978)
                                    (mmap: reserved=811660KB, committed=811660KB)
                            
        -                 GCCardSet (reserved=241444KB, committed=241444KB)
                                    (malloc=241444KB #36143)

    This separation helps tuning remembered set configuration if needed.

  * JDK 18 lifts the 32MB region size limit previously imposed by some internal data structures with the change [JDK-8275056](https://bugs.openjdk.java.net/browse/JDK-8275056). For now the maximum heap region size has been set to 512MB, but larger is possible. See my earlier blog [post](/2021/11/15/heap-regions-x-large.html) about this topic for further information.

  * There has been more progress on improving evacuation failure as a building block for region pinning in G1 by Hamlin Li. Recently, the corresponding [JEP-423](https://openjdk.java.net/jeps/423) has been made candidate - it contains lots of information about the purpose of this change.
  
    ![Pause times with induced evacuation failures](/assets/20220217-evac-failure-improvements.png) 

    The graph above shows pause times of a SPECjbb2015 run with fixed injection rate (constant load) in blue on JDK 17 - as expected, pause times are basically flat. The purple line shows pause time with regular induced evacuation failures (every fifth garbage collection or so) on JDK 17 - having significantly increased pause time. JDK 18 improves upon that, shown in the yellow line. In this case, pause times are even below the undisturbed pause time run.

    While much better than JDK 17 behavior, this is of course suboptimal, as it means that G1 does more (although shorter) garbage collections than necessary. Ergonomics do not cope well with the added regions promoted to Old generation as indicated in the list of issues at the end [here](/2021/06/28/evacuation-failure.html).
    
    An example for an improvement to fix this could be [JDK-8254739](https://bugs.openjdk.java.net/browse/JDK-8254739).
  

  * G1 waited on the completion of an active concurrent marking when the Java application is about to exit to actually exit in some cases. This can cause very long delays depending on the complexity of that task. JDK 18 fixes this problem with [JDK-8273605](https://bugs.openjdk.java.net/browse/JDK-8273605), "immediately" exiting the VM.

  * [JDK-8278824](https://bugs.openjdk.java.net/browse/JDK-8278824) fixes an interesting performance regression introduced in [JDK 14](https://bugs.openjdk.java.net/browse/JDK-8213108): when looking for references into the regions to be collected, the evacuation work is not distributed well initially. On larger machines this exposes problems in the work stealing mechanism, resulting in overly long pauses.

    ![Repro pause times](/assets/20220216-repro-pause-times.png)
    
    The graph above shows the problem: with JDK 11 (blue) pause times are fairly stable throughout the measurement interval. In contrast to that, JDK 17 (without the fix, purple) starts off significantly faster, but at around second 330 there is a large spike in pause times. This is where some large `j.l.Object` array is put into the old generation because of its size. At that point garbage collections need to scan this array every time. Due to bad initial work distribution and work stealing not working well enough, on many core processors pauses increase.

    The fix contributed by W. Kemper improves the initial work distribution, getting the pause times back to normal, shown with the JDK 18 (yellow) line. This change has already been backported to latest JDK 17, so make sure you are up to date.

  * We updated the garbage collection tuning guide to reflect the current state of G1 for JDK 18 improving various sections. There were also some minor clarifications in the general sections. I believe the correct link will be [this one](https://docs.oracle.com/en/java/javase/18/gctuning/index.html) when it will be published with GA of JDK 18.

## What's next

Of course we have long since begun working on JDK 19. Here is a short list of interesting changes that are currently in development and you may want to look out for. Without guarantees, as usual, they are going to be integrated when they are done ;-)

  * Some potential starvation issue in work distribution in G1 ([JDK-8280396](https://bugs.openjdk.java.net/browse/JDK-8280396)) and Parallel GC ([JDK-8280705](https://bugs.openjdk.java.net/browse/JDK-8280705)) Full GC causing long pauses with high variance were already integrated.

  * As pointed out in the review for [JDK-8278824](https://github.com/openjdk/jdk/pull/6840#issuecomment-994936159) and above, issues in work distribution during garbage collection have been found. Selection of victims to steal work from and actual stealing are somewhat inefficient particularly with higher thread counts.

    ![Repro pause times](/assets/20220216-repro-pause-times-proto.png)

    The figure above adds prototype results in cyan showing promising results.

  * H. Li continues collaboration on region pinning for G1 ([JEP-423](https://openjdk.java.net/jeps/423)).

  * We are experimenting on reducing footprint even further in [JDK-8210708](https://bugs.openjdk.java.net/browse/JDK-8210708). Hypothetical native memory consumption after that change could look like the cyan line in the graph below if that were integrated:
  
    ![Native memory usage BigRAMTester 20GB](/assets/20220216-bigramtester-memoryusage-projected.png)

    For this application it is a coincidence that the savings by removing a mark bitmap is almost the same as remembered set usage. Typically the remembered sets of Java applications are much smaller than in this case, so the relative gain in native memory footprint would be much higher.

  * The runtime team is working on archive heap objects support for Parallel GC in [JDK-8274788](https://bugs.openjdk.java.net/browse/JDK-8274788).

More to come :)

## Thanks go to…

Everyone that contributed to another great JDK release. See you in the next release :)

*Thomas*
