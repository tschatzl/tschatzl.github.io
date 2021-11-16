---
layout: post
title:  "Heap Regions X-Large"
date:   2021-11-15 12:00:00 +0200
tags: [GC, G1, JDK 18, Performance, Memory Usage, Heap Regions]
---

Until now G1 heap region size has been limited to 32MB due to previous limitations in the remembered set data structures. With [JDK-8275056](https://bugs.openjdk.java.net/browse/JDK-8275056) JDK 18 will **bump that limit to 512MB**. Read on about how this works, how you can use it and a brief discussion about the possible impact of this change for you.

## Introduction and Background ##

G1 splits the Java heap into [evenly sized regions](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-g1-garbage-collector1.html#GUID-15921907-B297-43A4-8C48-DC88035BC7CF) to facilitate incremental evacuation: during garbage collection it selects arbitrary regions it thinks contain lots of empty space from dead objects, and moves and compacts their live objects into another (much smaller) set of regions. The difference is what G1 freed in this collection for further application use.

Objects **must not span regions** in the common case: typical Java objects range in the few tens of bytes, so the range of available region sizes (from 1MB to 32MB) is usually not a problem. For the uncommon objects larger than half a region, G1 reserves sets of contiguous regions just for that object in the old generation.

![Allocation destination based on region size](/assets/20211115-xlarge-regions-allocation.png)

The figure above shows the Java heap as set of regions at the bottom, and several kinds of objects with different sizes that are about to be allocated into these free regions (white areas): small objects like `A` may be allocated at any place in a region, more than half a region sized (humongous) objects like `B` are allocated as separate, single objects in a region. Similarly, humongous objects of type `C` that are larger than a region, can only be allocated in contiguous sets of regions.

This implementation choice for humongous object allocation can cause problems if your application uses lots of large objects:
  * fragmentation/waste at the end of regions. Unfortunately in practice it is common that the payload data of these large objects are 2^n sized. Adding the Hotspot object header causes total object size to just go over a region size, wasting almost completely empty regions in many cases. E.g. even if a complete Java object is one byte larger than a single region, it takes up **two** regions on the heap.
  
    The figure below gives an example of an object `D` which payload has `2^n` bytes: in this case the waste at the end of humongous object is proportionally very large.
  ![Problem with header of 2^n aligned objects](/assets/20211115-xlarge-regions-headerproblem.png)
  
  * humongous objects require contiguous sets of regions for placement. This can be a problem particularly if the heap is already littered with other humongous objects - G1 never moves them for efficiency reasons, so G1 might not find any space for a new allocation at all.

     An example for such a situation could look as depicted below: the regions in the Java heap are filled at various places, with very little contiguous free regions.
  ![Problem with region fragmentation](/assets/20211115-xlarge-regions-outerfragmentation.png)


If you notice full gcs or in general many garbage collections when there is still lots of free space in the Java heap and use a large amount of humongous objects, these fragmentation issues might be the reason. Humongous objects on the heap generally do not have impact on pause times, but the frequency of collections can be higher.

Apart from changes to the application (e.g. start your initial array size with 15 elements instead of 16 when your resizing heuristics expands by doubling that value), techniques like [eager reclamation](https://bugs.openjdk.java.net/browse/JDK-8048179) of humongous objects improve the situation by reclaiming some humongous objects without a full heap liveness analysis (i.e. concurrent marking or full gc).

The [tuning guide](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-garbage-collector-tuning.html#GUID-2428DA90-B93D-48E6-B336-A849ADF1C552) suggests to either increase the Java heap size or increase region size. This will either increase the number of regions available, or decrease the the amount of humongous objects that can cause these issues.

The size of these regions has until now been limited to 32MB due to some implementation choices and limitations of the remembered set that have only been lifted with [JDK-8017163](https://bugs.openjdk.java.net/browse/JDK-8017163) recently.

As a reminder, the remembered set of a region `X` stores approximate locations of references from all other regions into that region `X`. These locations are required during garbage collection to update the references of objects moved during garbage collections. This [FOSDEM 2018 presentation](https://archive.fosdem.org/2018/schedule/event/g1/) in section "Faster Card Scanning" explains the principle in detail.

The mentioned implementation choice is related to the possible range of values that can be stored in a particular *remembered set card container*. Remembered set containers store, for a given region the cards that contain the necessary references.

![Remembered Set Containers](/assets/20211115-xlarge-regions-containers.png)

The figure above depicts a region `A` with it's remembered set `RS(A)`. For every region that has a reference into `A` the remembered set stores the cards (dashed boxes in the regions) in such a remembered set container. There are different containers that differ on various properties, depicted as differently colored boxes. For one of these containers, **G1 uses a 16 bit integer to store card indexes** to save space. However this limits the maximum size of regions:

A card has a fixed size of `512 (=2^9)` bytes currently, this results in `2^16 * 2^9 = 2^25 = 32MB` maximum region size.

Further, before JDK-8017163, the remembered set implementation required a 1:1 mapping of Java heap region and remembered set container. I.e. **there can be only one container per region**.

## The Change ##

So what does [JDK-8275056](https://bugs.openjdk.java.net/browse/JDK-8275056) do to overcome the heap region size limit? The new remembered set implementation simply does not require a 1:1 mapping of Java heap region and remembered set any more. The change implements a 1:n mapping of Java heap regions to remembered set containers: a single heap region may now be covered by multiple **card regions**, i.e. areas a given card set container covers.

![Card regions](/assets/20211115-xlarge-regions-cardregion.png)

The figure above shows an example setup assuming that a single region is larger than 32MB. Look how the heap region to the left of `A` is split into two card regions, each having its own remembered set container in the remembered set.

The change unlocks configurable heap region sizes of up to 512MB - which is an arbitrary limit - that should allow sufficient **flexibility for avoiding the mentioned fragmentation issues** even with the largest humongous objects and Java heaps. The "optimal" amount of regions that G1 strives for, `2048`, would require a Java heap to exceed 1 TB with this change.

## Impact Discussion ##

Ergonomics will still limit the maximum region size selected to 32MB. The end user, i.e. you, will need to override the heap region size on larger heaps manually. The reason is not that it does not work, but heap region size affects several other heuristics ("magic numbers") that are automatically extended and interpolated. While we have a lot of experience and performance data with G1 using up to 32MB regions, we do not have a lot data for larger heap region sizes.

One problem we can think of is the mentioned threshold of when G1 allocates an object in extra regions as humongous compared to allocating it in the young generation as a regular object. During evacuation, a single thread moves the data of an object alone, so with every increase in heap region size there is risk that that single-threaded operation of copying the object will be a bottleneck for an entire garbage collection pause.

On the other hand, TLABs (thread local allocation buffers) may be larger now, reducing potential contention when allocating objects in the Java application, speeding up the application.
Also some internal data structures that are managed on a per-region basis will take less memory overall, potentially allowing faster access as more data can be kept in caches. Remembered sets are likely to be smaller as given usual locality assumption, there will be less cross-region references to record, less memory used and less time spent on looking for references into the evacuated area, decreasing pause time.

Initial results from us and [others](https://bugs.openjdk.java.net/browse/JDK-8272773) with larger heap region sizes on appropriately sized heaps show some promise, but we also noticed diminishing returns in our testing for heap region sizes equal to or larger than 128MB.

This limitation of the ergonomics heuristics is intentional - we wanted users to intentionally try out larger heap region sizes if they need it only, and assume that people changing this test the impact of this change on their application thoroughly. We would be happy to hear your experience tuning heap region size though, please send comments to me, [hotspot-gc-use@openjdk.java.net](mailto:hotspot-gc-use@openjdk.java.net) or [hotspot-gc-dev@openjdk.java.net](mailto:hotspot-gc-dev@openjdk.java.net).


That's all folks,

*Thomas*

