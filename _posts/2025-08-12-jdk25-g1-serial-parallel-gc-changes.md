---
layout: post
title:  "JDK 25 G1/Parallel/Serial GC changes"
date:   2025-08-12 11:00:00 +0200
tags: [GC, G1, Parallel, Serial, JDK 25, Performance]
---

JDK 25 RC 1 [is out](https://mail.openjdk.org/pipermail/jdk-dev/2025-August/010296.html), and as is custom by now, here is a brief overview of the changes to the stop-the-world collectors in OpenJDK for that release.

The full list of changes for the entire Hotspot GC subcomponent for JDK 25 is [here](https://bugs.openjdk.org/issues/?jql=project%20%3D%20JDK%20AND%20status%20%3D%20Resolved%20AND%20fixVersion%20%3D%20%2225%22%20AND%20component%20%3D%20hotspot%20AND%20Subcomponent%20%3D%20gc%20ORDER%20BY%20summary%20ASC%2C%20status%20DESC), listing around 200 changes in total that were resolved or closed. This matches recent trends.

Let's start with the interesting change breakdown for JDK 25:

## Parallel GC

  * The most important change for Parallel (and Serial) GC has probably been [JDK-8192647](https://bugs.openjdk.org/browse/JDK-8192647).

    If your application used [JNI](https://docs.oracle.com/en/java/javase/24/docs/specs/jni/) extensively, you might have encountered premature VM shutdowns after `Retried Waiting for GCLocker Too Often Allocating XXX words` warnings. These can not occur any more with either collector beginning with this release.

    The fix makes sure that a thread that is blocked to perform a garbage collection due to JNI is guaranteed to have triggered its own garbage collection before giving up.

## Serial GC

  * Fixing the same GCLocker issue as described for Parallel GC.
  * Serial GC fixes an issue to avoid full GCs in some circumstances in [JDK-8346920](https://bugs.openjdk.org/browse/JDK-8346920).

## G1 GC

  * [JDK-8343782](https://bugs.openjdk.org/browse/JDK-8343782) enables G1 to merge any old generation region's [remembered set](https://docs.oracle.com/en/java/javase/24/gctuning/garbage-first-g1-garbage-collector1.html#GUID-1CDEB6B6-9463-4998-815D-05E095BFBD0F) with others for additional memory savings compared to last release.

  * G1 now uses an additional measure to avoid pause time spikes at the end of a old generation [Space-Reclamation phase](https://docs.oracle.com/en/java/javase/24/gctuning/garbage-first-g1-garbage-collector1.html#JSGCT-GUID-F1BE86FA-3EDC-4D4F-BDB4-4B044AD83180). They were caused by wrong selection of candidate regions due to lack of information.

    The current process is roughly like this: in the `Remark` pause, after whole-heap liveness analysis, G1 determines a set of candidate regions that it is going to collect later. It tries to choose the most efficient regions, i.e. where the gain in memory is large, and the effort to reclaim is low. Then, for these candidate regions, from the `Remark` to the `Cleanup` pause, G1 recalculates the [remembered sets](https://docs.oracle.com/en/java/javase/24/gctuning/garbage-first-g1-garbage-collector1.html#GUID-99526C47-2C71-408C-9DBE-4F38ED839FF0), the data structure containing all references to the live objects in these regions. Remembered sets are required to evacuate objects in the following garbage collection pauses because they contain the locations of references to the evacuation objects. These locations must, after moving, point to the new object locations after all.

    Determining the candidate regions is where the cause of the high pause times lies: efficiency is a weighted sum of amount of live data in the region and the size of the remembered set for that region, i.e. its connectedness; G1 prefers to evacuate regions with low liveness and small remembered sets as they are most efficient. However, at the time these regions are selected, G1 does not have information about the connectedness, actually G1 is about to recreate this information just after this selection. Until now, G1 solely relied on liveness, assuming that the remembered set size is proportional to that.
  
    It isn't in many cases. In the majority of cases this does not matter: evacuation is fast enough anyway, but you may have noticed that later mixed collections are typically slower than earlier ones. However this behavior can cause large pause time spikes typically in the last collection of the Space-Reclamation phase as only the most expensive-to-evacuate regions are left to evacuate.

    One option for this problem is to just skip evacuating these regions: but the problem is, the initial candidate selection was based on freeing enough space to continue without a full gc. So continuously leaving them out from space-reclamation can lead to full gcs.
  
    The change instead improves the selection mechanism: we do not have the exact metric for connectedness at that point, but is there something cheap and fast to calculate that could be a substitute? Actually, yes there is: during marking, for every region G1 now counts the number of incoming references (just counting, not storing them somewhere). This is actually a very good substitute.
    
    G1 uses this much more accurate efficiency information during candidate selection in the `Remark` pause.

  * The [write-barrier improvements](https://bugs.openjdk.org/browse/JDK-8340827) explained in [this post](/2025/02/21/new-write-barriers.html) originally scheduled for JDK 25 did not make it in time due to process issues. Hopefully it can make it for JDK 26.

  * There were some ([here](https://bugs.openjdk.org/browse/JDK-8355756) and [here](https://bugs.openjdk.org/browse/JDK-8355681)) fixes to avoid some superfluous garbage collection pauses in some edge cases. Another [change](https://bugs.openjdk.org/browse/JDK-8271871) improves performance of certain garbage collections.

## What's next

* Currently a lot of effort in the G1 area is spent on implementing [Automatic Heap Sizing for G1](https://bugs.openjdk.org/browse/JDK-8359211): an effort where G1 automatically determines a maximum Java heap size that is efficient and does not blow the available memory budget by tracking free memory in the environment, and adjusting heap size accordingly.

    A few changes necessary for automatic heap sizing (like above fixes that avoid superfluous garbage collection pauses, but also [here](https://bugs.openjdk.org/browse/JDK-8247843) or [there](https://bugs.openjdk.org/browse/JDK-8247843)) already landed in JDK 26, and a few stepping stones are out for review (e.g. [this](https://bugs.openjdk.org/browse/JDK-8359348]) and [that](https://bugs.openjdk.org/browse/JDK-8357445)).

* Another JEP we would like to complete in this release time frame is to make [ergonomics always select G1 as default collector](https://bugs.openjdk.org/browse/JDK-8359802) to avoid any more confusion about this particular topic :)

## Thanks go to…

... as usual to everyone that contributed to another great JDK release. Looking forward to see you next (even greater) release.

*Thomas*
