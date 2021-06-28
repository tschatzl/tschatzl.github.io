---
layout: post
title:  "Evacuation Failure and Object Pinning"
date:   2021-06-28 14:05:00 +0200
#categories: gc g1 evacuation failure object pinning performance
tags: [GC, G1, Parallel, Evacuation Failure, Object Pinning, Performance]
---

At the end of the post about [JDK 16 improvements](2021/03/12/jdk16-g1-parallel-gc-changes.html) to the Hotspot VM I mentioned that it added preparations for **object pinning**. I would like to elaborate on that a bit more as I got some questions about it, particularly talking about the problems, potential solutions and where to start working.

This post will give a short overview about what object pinning is about, the state of object pinning in G1 and suggestions how to improve the evacuation failure mechanism which the current suggested implementation is based on. This could lead to enabling object pinning with G1 upstream in the future.

## JNI Critical Sections and Why the GCLocker Is Bad

A lot of the information presented in this section is a summary from the writeup from A. Shipilev [here](https://shipilev.net/jvm/anatomy-quarks/9-jni-critical-gclocker/). Please read it for more background.

In short, there is the family of JNI API functions `Get<PrimitiveType>Array*` which allow you to get a pointer to an object on the Java heap in your C code. The garbage collector should obviously not move these objects while C code is accessing it. There are a few options on how to deal with these objects: currently all Hotspot collectors that move objects except Shenandoah deal with this situation by **disabling the garbage collector** while there is any object of that kind.

Quoting the writeup:

> If GC was attempted, JVM should see if anybody holds that lock. If anybody does, then at least for Parallel, CMS, and G1, we cannot continue with GC. When the last critical JNI operation ends with "release", then VM checks if there are pending GC blocked by GCLocker, and if there are, then it triggers GC. This yields "GCLocker Initiated GC" collection.

Is this a problem? Both no and yes :)

**No**, because
 * the JNI specification suggests to not have your native code run for extended periods of time before calling `Release<PrimitiveType>Array*`, and the implementation assumes that this is. So this situation should not happen too often (hopefully).
 * implementations also assume that the amount of locked objects is small.
 * particularly G1 tries to avoid this situation by actually allowing additional allocation exceeding young gen size - that is I believe the reason for the different behavior between G1 and other collectors in the non-compliant code tested there - to alleviate this problem a bit, i.e. keep other threads running for a just bit more in the hope that the native code completes during that time.
The option `-XX:GCLockerEdenExpansionPercent` controls this amount. Of course this is a tradeoff as well and may cause other issues.

**Yes**, because there may still be stalls of the entire application waiting for the GCLocker.

Since the specification only requires memory management to keep this objects in place (if not copying or preventing garbage collection), the options are about keeping the Java objects that need to stay in place in place, and freeing the space surrounding them for subsequent allocation in different granularities.

Ideally, the garbage collector would just keep only these objects in place, and allow evacuation of everything else surrounding them. The problem is that this complicates subsequent allocation significantly.

The regionalized collectors (G1, Shenandoah, ZGC) already support evacuation of parts of the heap - their regions. So for these collectors, the straightforward solution is to not evacuate the regions that contain such objects **together with all other live objects**. This changes the problem from pinning an object to pinning a region, which is much easier, at the cost of additional unallocatable memory in the Java heap.

There is no good solution for the non-regionalized collectors like Serial and Parallel GC. At least I do not know any, so they will most likely keep on using the GCLocker forever.

Besides, this technique is not without risks: while in most cases better than stalling the application, it does carry the **risk of pinning the entire heap** and causing a VM failure. How big is that risk? There is too little data on that but Shenandoah did not add some GCLocker mechanism so far...

## G1 and Object Pinning

G1 already **fully supports region pinning** in principle. Actually it has been supporting it since its inception for humongous objects. Humongous objects never move. This mechanism can be extended for any regions in the old generation, as actually, the concept of a region being "humongous" and "pinned" has been separate for a long time. G1 currently also uses region pinning for regions that contain object graphs from Class Data Sharing archives.

All this has been about regions located in G1's old generation, but what about pinned regions in the **young generation**?

G1 **Evacuation Failure handling** already provides a mechanism to handle objects that stay in place because they could not be copied. These (live) objects are marked specially during collection, and at the end of the garbage collection pause there is a special phase to fix up these regions.

So all set, flip the switch as suggestion in [this patch](https://github.com/openjdk/jdk/compare/master...tschatzl:full-pin-support) and call it a finished job? Not so fast :)

## Evacuation Failure Handling Analysis

Evacuation failure handling is fairly slow compared to "just" evacuation. The need for an extra phase during garbage collection might have indicated that already. The implementation is not as optimized as other areas as the assumption is that evacuation failure **happens infrequently only**. So relatively little care has been taken about its direct impact on pause time, and indirect impact on application behavior.

Direct impact is pause time, indirect impact is what happens to these regions: currently the heuristics literally converts all young generation regions that have a single object that failed evacuation into an old generation region. Since they do not have a remembered set, they will only be collected in the next marking cycle. This can fill up the Java heap very quickly.

Anyway the additional work for evacuation failure handling during the pause is as follows:

**During evacuation**

  * if G1 notices that it could not evacuate an object because it could not find space for the copy, it marks that object and the containing region as having failed to evacuate. G1 overwrites the object's mark word to indicate that that object failed evacuation with a special value, saving the old value if necessary (in [`G1ParScanThreadState::handle_evacuation_failure_par()`](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/gc/g1/g1ParScanThreadState.cpp#L600)).

**After evacuation**

  * G1 restores these preserved marks again. This task is typically not an issue: only objects that have been locked, or one that have a hash value need to be preserved, and these are typically few compared to the set of all evacuated objects (using the [`RestorePreservedMarksTask`](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/gc/shared/preservedMarks.cpp#L95) work gang).
  * G1 promotes these young regions to old ones. This requires a significant amount of work (in [`RemoveSelfForwardPtrHRClosure::remove_self_forward_ptr_by_walking_hr()`](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/gc/g1/g1EvacFailure.cpp#L223)):
     - dead objects between these surviving objects need to be overwritten so that anyone walking these regions linearly does not find dead, evacuated objects. For that, and since there is not Block Offset Table, a single thread linearly walks all objects found both dead or alive and then filling in dummy objects between live objects.
     - the Block Offset Table, a very important side table over the Java heap that is used for fast location of object starts, needs to be created.
     - set up concurrent remembered set refinement so that these objects will be scanned for references into regions requiring a remembered set.

I.e. basically some optimizations generational collectors like G1 exploit for the young generation need to be undone. Generational concurrent collectors would probably do all this or similar work concurrently, so it will not show up in a pause, but only complicates code.

Maybe this is not an issue after all - only more measurements than already performed can show that it is (not).

## Where Can I Help?

In general from initial testing the amount of pinning required because of this is not very high (few regions), however the amount of objects that need to be handled can vary a lot, from just a few objects to many objects due to single objects located in regions where a lot of unrelated objects also need to stay in place.

Within the Oracle garbage collection team we already brainstormed a bit [how to](https://bugs.openjdk.java.net/issues/?jql=labels%20%3D%20gc-g1-pinned-regions) improve the existing evacuation failure handling.

Here is an annotated list with suggestions:
  * avoid promotion of regions to old: this simply avoids the need for doing most of the fixup work mentioned above, particularly if live object locations can be accessed without walking all objects in the heap. Some aging heuristics needs to be thought of (e.g. region age?) about to avoid keeping a long-living pinned region in the young generation forever.
Changing this will require some consideration in the young gen sizing policy which currently determines young gen size in region granularity. [JDK-8254738](https://bugs.openjdk.java.net/browse/JDK-8254738).
  * an extension to above suggestion is to consider putting regions that failed evacuation in the collection set to evacuate them in the next garbage collection. After all, we just kept a whole region alive only for a few actually pinned objects [JDK-8140326](https://bugs.openjdk.java.net/browse/JDK-8140326). This certainly only applies to old generation regions: if the region is still in young generation, it will automatically be evacuated in the next collection.
  * during full gc rebuild remembered sets for some regions (like very empty pinned regions) to allow the regular collector to immediately collect them asap [JDK-8254896](G1: Rebuild remembered sets for some regions during full gc).
  * optimize evacuation failure handling for regions with just a few live objects: one option could be to at least initially track all live objects within (parts of) a region and then later use this list for quickly finding the live objects. There may still be a prototype for this somewhere. [JDK-8254739](https://bugs.openjdk.java.net/browse/JDK-8254739).
  * increase parallelism within failed regions: currently a single thread takes over evacuation failure handling of a single region, which effectively serializes that phase. One option is to record potential entry points/live object starts within a region to allow multiple threads work on different parts of a region. Note that the Block Offset Table should not be used as there is none. [JDK-8256265](G1: Improve parallelism in regions that failed evacuation).
  * enhancing above could be a change that actually stores which regions are affected by the evacuation failure: currently in G1 all threads search the entire region table for failed regions which can be slow in itself if there are many regions. [JDK-8254167](https://bugs.openjdk.java.net/browse/JDK-8254167).

All of these suggestions improve evacuation failure handling and could be tested by inducing failed evacuations. Then there is finally the matter of actually turn on object pinning by turning on the feature [JDK-8236594](https://bugs.openjdk.java.net/browse/JDK-8236594), with some required refactoring.

More extensive conceptual changes in that area could involve moving work out of the pause: if evacuation did not use the object header for forwarding pointers, these regions could be walked concurrently and all of the work described above done concurrently as well.

## Conclusion

Yes, G1 already supports object pinning. No, it's not ready yet due to performance issues. :) If you asked me where to start with, of these suggestions the last three are probably easiest to get your feet wet with the code.

If you have any particular questions, feel free to ask me directly or preferably on [hotspot-gc-dev](mailto:hotspot-gc-dev@openjdk.java.net).

See you next time,

*Thomas*

