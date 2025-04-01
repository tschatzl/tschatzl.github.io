---
layout: post
title:  "JDK 24 G1/Parallel/Serial GC changes"
date:   2025-04-01 11:00:00 +0200
tags: [GC, G1, Parallel, Serial, JDK 24, Performance]
---

JDK 24 has been [released](https://openjdk.java.net/projects/jdk/24/) a few weeks ago. This post is going to provide the usual brief look on changes to the stop-the-world collectors in OpenJDK in that release. Originally I thought there was not much to talk about, hence the delay, but in hindsight I have been wrong.

Similar to the [previous release](/2024/07/22/jdk23-g1-serial-parallel-gc-changes.html) JDK 24 is a fairly muted one in the GC area, but there are good things on the horizon particularly for JDK 25 that I will touch on at the end of this post :)

The full list of changes for the entire Hotspot GC subcomponent for JDK 24 is [here](https://bugs.openjdk.org/issues/?jql=project%20%3D%20JDK%20AND%20status%20%3D%20Resolved%20AND%20fixVersion%20%3D%20%2224%22%20AND%20component%20%3D%20hotspot%20AND%20Subcomponent%20%3D%20gc%20ORDER%20BY%20summary%20ASC%2C%20status%20DESC), showing around 190 changes in total having been resolved or closed. Nothing particular unusual here.

Besides, there has been a [JavaOne](https://dev.java/community/javaone-2025/) conference again.

## Parallel GC

  * Some unnecessary synchronization in the (typically very hot) evacuation loop has been removed with [JDK-8269870](https://bugs.openjdk.org/browse/JDK-8269870). This may reduce pause times in some situations.

## Serial GC

  * Cleanup and refactoring of Serial GC code continued.

## G1 GC

  * G1 uses predictions of various aspects of garbage collection that are dependent on both the application and the environment the VM to meet pause time goals. Examples for these predictions are how much does memory copying actually cost, how many references typically need to be updated. G1 retrains related predictors every time the VM is run. At startup, since there are no samples for the predictors yet, G1 uses some pre-baked values for these predictions. Ideally, these would be updated and maintained regularly, and appropriate for all environments. As you can imagine, this is an impossible task: these pre-baked values were actually determined a long time ago on some "large" SPARC machine, and then... forgotten.

    The inadequacy of these values has actual drawbacks: nowadays almost any machine the JVM runs on is faster than that reference machine, the values are very conservative, meaning that e.g. expected costs are much higher than actual so G1 GC pauses tend to be much shorter than necessary until newly measured values eventually effectively overwrite these pre-baked values. In some measurements, it takes around 30 or so meaningful garbage collections until G1 is primed for the current environment.

    The change in [JDK-8343189](https://bugs.openjdk.org/browse/JDK-8343189) modifies the way new, actual values are used in the predictors: instead of adding to the existing pre-baked predictor values, the first actual values directly overwrite these pre-baked predictor values. The effect is that apart from the first garbage collection, actual samples for these predictions are used. This speeds up adaptation of G1 to the application and environment tremendously.

    There is one small snag here: The original, very conservative estimates did have the advantage that you were almost guaranteed that G1 does not exceed the pause time goal in the first few garbage collections. They dampened the predictor outputs quite a bit. So with the new system, there may be more initial overshoots in pause times in the first few garbage collections.

  * Additionally we have been working on decreasing native memory usage, targeting the [remembered sets](https://docs.oracle.com/en/java/javase/24/gctuning/garbage-first-g1-garbage-collector1.html#GUID-99526C47-2C71-408C-9DBE-4F38ED839FF0) once again. As written in the respective section of the garbage collection tuning guide, G1 [now](https://bugs.openjdk.org/browse/JDK-8336086) manages remembered sets for the young generation as a whole, single unit, removing duplicates and so saving memory. Less remembered set entries for the young generation also decreases garbage collection pauses (very) slightly.
  
    The JDK 23 [blog post](/2024/07/22/jdk23-g1-serial-parallel-gc-changes.html#single-remsets-multiple-regions) describes the idea with some additional graphs at the end.

## What's next

Last time this section contained a lot of information about how the idea of using single remembered sets for multiple regions applies to any set of regions and its effects. Extending merging young generation remembered sets, [JDK-8343782](https://bugs.openjdk.org/browse/JDK-8343782), which has already been integrated in JDK 25, implements this idea for old generation regions for even larger native memory savings. I.e. with that change, merges sets of old generation regions by default.

Scheduled for JDK 25 there is probably one of the largest changes to the interaction between the application and the G1 garbage collector ever: the write-barriers, small code that is executed for every reference write in an application in the VM to synchronize with the garbage collector, which have a large impact on application throughput, [especially so in G1](https://bugs.openjdk.org/browse/JDK-8253230) were completely overhauled. By reducing this write barrier significantly, large improvements are possible.

We are in the process of preparing [a JEP](https://bugs.openjdk.org/browse/JDK-8340827) to get this change integrated in the next months. For more information please also look at the [implementation CR](https://bugs.openjdk.org/browse/JDK-8342382), the respective [PR](https://github.com/openjdk/jdk/pull/23739) and the extensive [blog post](/2025/02/21/new-write-barriers.html) about this topic on this blog.

Overall, one can expect free throughput improvements up to 10%, shorter pauses, better code generation (and code size savings) from this change. There is some minimal regression in native memory usage, but we think it is tolerable, let me explain with the following graph:

![Native Memory Usage BigRAMTester 20GB](/assets/20250401-native-memory-usage.png){:style="display:block; margin-left:auto; margin-right:auto"}

The graph above shows native memory usage of the G1 collector for the BigRAMTester benchmark, on a 20GB Java heap, for recent'ish JDKs (Fwiw, JDK8 uses 8GB native memory). The drop of the base memory usage at the start from JDK 17 to JDK 21 from 1.2GB to around 800MB is because of the change described [here](/2022/08/04/concurrent-marking.html). The JDK 24 line (orange) shows the impact of the mentioned use of a single remembered set for the entire young generation (and [JDK-8225409](https://bugs.openjdk.org/browse/JDK-8225409)).

Finally, JDK 25 (shown as dashed red "latest" line) adds [JDK-8343782](https://bugs.openjdk.org/browse/JDK-8343782), remembered set merging for the old generation, and the changes in the write barrier that enabled further memory reductions. So where is the regression - that line for JDK 25 certainly looks best? The truth is, these changes need one more, larger static memory block than before (of fixed ~0.2% Java heap size). Its impact is shown at the very beginning of the graph. While in this benchmark, the savings from the remembered sets outweigh the addition of this new data structure, in general this is not the case.

While this milestone is complete now and apart from bringing this change along the JEP process, to ultimately integrate it into the release, more work in this area is ongoing as we were somewhat conservative in our changes. There is reason to believe that throughput will improve even more in the future.

Other than that, there are some potentially interesting topics currently being discussed on the hotspot-gc-dev mailing list:

* Longer term, there is a project underway to bring Automatic Heap Sizing similar to [ZGC](https://openjdk.org/jeps/8329758) to the stop-the-world collectors too. First contributions are coming, thanks for Microsoft and Google contributing to this effort!

* There is [some](https://mail.openjdk.org/pipermail/hotspot-gc-dev/2025-February/051121.html) [public](https://mail.openjdk.org/pipermail/hotspot-gc-dev/2025-March/051557.html) [discussion](https://mail.openjdk.org/pipermail/hotspot-gc-dev/2025-April/051778.html) on making G1 the "true" default collector: right now VM ergonomics may select the Serial garbage collector in environments with small heaps and little CPU resources instead of G1 if you do not specify any garbage collector on the command line. More and more, measurements show that G1 is very close to or superseedes the Serial collector in most or all metrics when run out-of-box, i.e. without additional tuning.

If you happen to have some interesting data points for the latter, feel free to chime in.

## Thanks go to…

... as usual to everyone that contributed to another great JDK release. Looking forward to see you next (even greater) release.

*Thomas*
