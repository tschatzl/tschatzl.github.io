---
layout: post
title:  "The Case of Ljdk.vm.internal.FillerArray;"
date:   2022-09-26 10:00:00 +0200
tags: [GC, JDK 19]
---

While writing the [JDK 19 update blog post](/2022/09/16/jdk19-g1-parallel-gc-changes.html) I strongly considered writing about the "new" objects (`jdk.vm.internal.FillerObject` and `jdk.vm.internal.FillerArray`) objects that may appear in heap dumps. I thought they were not that interesting, but it looks like people quickly [noticed them](https://old.reddit.com/r/java/comments/xlnw9q/the_mysterious_ljavainternalvmfillerarray/). This post describes their purpose after all.

## Background

In Hotspot there is a fairly interesting property of Java heaps we call **parsability**. It means that one can parse (linearly walk) the parts of the heap (whether these are called regions, spaces, or generations depending on the collector) from their respective bottom to their top addresses. The garbage collector guarantees that after an object in the Java heap another valid Java object of some kind always follows.

Every object header contains information about the type of the object (i.e. class) which allows inference of the size of that object.

This property is exploited in various ways. This is a non-exhaustive list:
  * heap statistics (`jmap -histo`): just start at the bottom of all parts of the heap, walk to its top, and collect statistics.
  * heap dumps
  * some collectors require heap parsability for at least parts of the heap when they partially collect the heap in young collections: they use an inexact remembered set to record approximate locations where there are interesting references. These recorded areas (remembered set entries) need to walked quickly during garbage collection. [This post](2022/02/15/card-table-card-size.html) explains how this works in a bit more detail in the introduction.

So how could the Java heap get unparsable? After all objects are allocated in a linear fashion already?

There are two (maybe more) causes:
  * the first is LAB allocation: threads do not synchronize with others for all memory allocations, but they carve out larger buffers that they use for local allocation without synchronization to others. These LABs are fixed size - if a thread can't fit an allocation into its current LAB any more, the remainder area needs to be formatted properly before getting a new LAB.
  * the second is class unloading. Class unloading makes objects which classes were unloaded unparsable as their class information will be discarded. Some collectors avoid this issue by compacting the heap after class unloading to remove all these invalid objects from the parsable Java heap area.

Before JDK 19 the Hotspot VM used instances of `java.lang.Object` and integer arrays (`[I`) to reformat ("fill") these holes in the heap. The former are used only for the tiniest holes to fill, while everything else uses integer arrays.

I.e. in a complete heap histogram or heap dump that included non-live data you may have noticed an abundance of `java.lang.Object` and `[I` instances that were not referenced from anywhere in your program.

Here is an example `jmap -histo <pid>` run of some program with an older JVM:

```
 num     #instances         #bytes  class name (module)
-------------------------------------------------------
   1:         16015      350913960  [B (java.base@11.0.16)
   2:        467918       33690096  java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask (java.base@11.0.16)
   3:          4559       18664488  [I (java.base@11.0.16)
   4:             1        2396432  [Ljava.util.concurrent.RunnableScheduledFuture; (java.base@11.0.16)
   5:         10715         342880  java.util.concurrent.SynchronousQueue$TransferStack$SNode (java.base@11.0.16)
   6:         10528         336896  java.util.concurrent.locks.AbstractQueuedSynchronizer$Node (java.base@11.0.16)
   7:         10149         243576  java.lang.String (java.base@11.0.16)
[...]
```

A large part of the `[I` instances are likely filler objects.

## The Change

[JDK-8284435](https://bugs.openjdk.org/browse/JDK-8284435) added special, explicit filler objects with the names `jdk.vm.internal.FillerObject` and `Ljdk.vm.internal.FillerArray;` in Java heap histograms (Actually, in the original change there has been a typo in the filler array type name, it has errorneously been called `Ljava.vm.internal.FillerArray;`, fixed in [JDK-8294000](https://bugs.openjdk.org/browse/JDK-8294000)).

These classes are created up front at VM startup similar to most VM internal classes.

These serve the same purpose as `java.lang.Object` and `[I` instances for filling holes, but have the added advantage to us Hotspot VM developers that if we see them referenced in crash logs, or references into these kinds of objects, there is a high likelihood of some bug related to dangling references to garbage objects. It's not fool-proof, but makes crash investigation a bit easier as we can now more easily distinguish between dangling references and legitimate references to instances of the previous kinds of filler objects. Heap verification can now also better distinguish between filler and live objects independent of the garbage collector.

Here is another run of `jmap -histo` of the same program as above with a current VM:

```
 num     #instances         #bytes  class name (module)
-------------------------------------------------------
   1:         10740      152377136  [B (java.base@20-internal)
   2:        250078       18005616  java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask (java.base@20-internal)
   3:          1699       14286176  Ljdk.internal.vm.FillerArray; (java.base@20-internal)
   4:             1        1065104  [Ljava.util.concurrent.RunnableScheduledFuture; (java.base@20-internal)
[...]
  23:           361          11552  java.lang.module.ModuleDescriptor$Exports (java.base@20-internal)
  24:           127          11264  [I (java.base@20-internal)
  25:           274          10960  java.lang.invoke.MethodType (java.base@20-internal)
[...]
  76:            11           1056  java.lang.reflect.Method (java.base@20-internal)
  77:            66           1056  jdk.internal.vm.FillerObject (java.base@20-internal)
  78:             1           1048  [Ljava.lang.Integer; (java.base@20-internal)
[...]
```

Notice that in this histogram, there are quite a few `Ljdk.internal.vm.FillerArray;` instances and much less `[I` instances (these histograms were taken randomly, so they are not directly comparable). `jdk.internal.vm.FillerObject` instances are rather rare because holes with minimal size are rare.

(On 64 bit machines they only show up when not using compressed class pointers via `-XX:-UseCompressedClassPointers` as otherwise filler arrays also fit minimum instance size).

Obviously the names and everything related are Hotspot VM internal details and can change at any notice.

## Impact Discussion

For the end user there should be no difference apart from instances of these objects showing up in complete heap dumps.

That's all for today, mystery solved :),

*Thomas*

