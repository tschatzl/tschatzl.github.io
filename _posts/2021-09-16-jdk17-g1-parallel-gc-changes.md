---
layout: post
title:  "JDK 17 G1/Parallel GC changes"
date:   2021-09-16 12:00:00 +0200
#categories: gc g1 parallel JDK-17 performance
tags: [GC, G1, Parallel, JDK 17, Performance]
---

A few days ago JDK 17 [went GA](https://mail.openjdk.java.net/pipermail/jdk-dev/2021-September/006037.html). For this reason it is time for another post that  attempts to summarize most significant changes in Hotspot´s stop-the-world garbage collectors in that release - G1 and Parallel GC.

Before getting into detail about what changed in G1 and Parallel GC, a short overview with statistics about the whole GC subcomponent: there has been [no JEP in the garbage collection area](https://openjdk.java.net/projects/jdk/17/).

The full list of changes for the entire Hotspot GC subcomponent is [here](https://bugs.openjdk.java.net/browse/JDK-8271064?jql=project%20%3D%20JDK%20AND%20issuetype%20in%20standardIssueTypes()%20AND%20status%20in%20(Resolved%2C%20Closed)%20AND%20fixVersion%20%3D%20%2217%22%20AND%20component%20%3D%20hotspot%20AND%20Subcomponent%20%3D%20gc), clocking in at 312 changes in total. This is in line with recent releases.

A brief look over to [ZGC](https://wiki.openjdk.java.net/display/zgc/Main): this release improved usability by dynamically adjusting concurrent GC threads to match the application to on the one hand optimize throughput and on the other hand avoid allocation stalls ([JDK-8268372](https://bugs.openjdk.java.net/browse/JDK-8268372)). Another notable change, [JDK-8260267](https://bugs.openjdk.java.net/browse/JDK-8260267) reduces mark stack memory usage significantly. Per is likely going to share more details in [his blog](https://malloc.se/) soon.

## Generic improvements

  * The VM can now use different large page sizes for different memory reservations since [JDK-8256155](https://bugs.openjdk.java.net/browse/JDK-8256155): i.e. the code cache may use differently sized large pages than the Java heap and other large reservations. This allows better usage of configured large pages in the system.
  
    My colleague Stefan has some write-up on why and how to use large pages [here](https://kstefanj.github.io/2021/05/19/large-pages-and-java.html).

## Parallel GC

Parallel GC pauses have been sped up a bit by making formerly serial phases in the pauses to be executed in parallel more than before. This includes

  * [JDK-8204686](https://bugs.openjdk.java.net/browse/JDK-8204686) that implements **dynamic parallel reference processing** like G1 does for some time now. Previous work in the last few releases allowed easy implementation of this feature. It works just like the G1 implementation:
    * based on the amount of `java.lang.ref.Reference` instances that need reference processing for a given type (Soft, Weak, Final and Phantom) during a given garbage collection, Parallel GC now starts different amounts of threads for a particular phase of reference processing. Roughly, the implementation divides the observed number of `j.l.ref.References` for a given phase by the value of `ReferencesPerThread` (default `1000`) to determine the amount of threads Parallel GC is going to use for that particular phase.
    * the option `ParallelRefProcEnabled` is enabled by default now, enabling this mechanism. Since the introduction of this feature in G1 [in JDK 11](https://bugs.openjdk.java.net/browse/JDK-8043575) we have not heard complaints, so this seems appropriate.

    Please also check the [Release Notes](https://jdk.java.net/17/release-notes#JDK-8204686).

  * Similarly, processing of **all internal weak references** has been changed to automatically exploit parallelism in [JDK-8268443](https://bugs.openjdk.java.net/browse/JDK-8268443).

  * Finally, [JDK-8248314](https://bugs.openjdk.java.net/browse/JDK-8248314) shaves off **a few milliseconds of Full GC pauses** for the same reason.

We also noticed small single-digit percent improvements in throughput in some applications compared to JDK 16, which, are however more likely related to compiler improvements in JDK 17 than GC ones. That is, unless above changes are exactly solving your application's issue.

## G1 GC

  * G1 now schedules **preventive garbage collections** with [JDK-8257774](https://bugs.openjdk.java.net/browse/JDK-8257774). This contribution by Microsoft introduces a special kind of young collection with the purpose to avoid typically long pauses with [evacuation failures](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-g1-garbage-collector1.html#GUID-BE157AF6-29E7-461A-82CF-50C1978785DA). This situation, where there is not enough space to copy objects to, often occurs because of a high rate of short-living humongous object allocation - they may fill up the heap before G1 would normally schedule a garbage collection.
  
    So instead of waiting for this situation to happen, G1 starts an out-of-schedule garbage collection while it can still be confident to have enough space to copy surviving objects to, assuming that [eager reclaim](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-g1-garbage-collector1.html#GUID-D74F3CC7-CC9F-45B5-B03D-510AEEAC2DAC) will free up lots of heap space and regular operation can continue.
  Preventive collections will be tagged as `G1 Preventive Collection` in the logs, i.e. the corresponding log entry could look like the following:
  
    ```
    [...]
    [2.574s][info][gc] GC(121) Pause Young (Normal) (G1 Evacuation Pause) 86M->83M(90M) 5.781ms
    [2.582s][info][gc] GC(122) Pause Young (Normal) (G1 Evacuation Pause) 86M->83M(90M) 4.936ms
    [2.596s][info][gc] GC(123) Pause Young (Normal) (G1 Preventive Collection) 86M->84M(90M) 9.997ms
    [...]
    ```
      
    Preventive garbage collections are enabled by default. They may be disabled by using the diagnostic flag `G1UsePreventiveGC` in case they cause regressions.

  * A significant bug with large page handling on Windows has been fixed: [JDK-8266489](https://bugs.openjdk.java.net/browse/JDK-8266489) enables G1 to use large pages when the region size is larger than 2 MB, increasing performance significantly in some cases on larger Java heaps.

  * With [JDK-8262068](https://bugs.openjdk.java.net/browse/JDK-8262068) Hamlin Li added support for the `MarkSweepDeadRatio` option in G1 Full GC in addition to Serial and Parallel GC. This option controls how much waste is tolerated in regions scheduled for compaction. Regions that have a higher live occupancy than this ratio (default 95%), are not compacted because compacting them would not return an appreciably amount of memory, and take a long time to compact only.

    In some situations this may be undesirable. If you want maximum heap compaction for some reason, manually setting this flag's value to `100` disables the feature (like with the other collectors).

  * Significant memory savings may be gained by **pruning collection sets early** ([JDK-8262185](https://bugs.openjdk.java.net/browse/JDK-8262185)): with that change, G1 tries to keep the remembered sets only for a range of old generation regions that it will almost surely evacuate, not all possible useful candidates. There is a [posting](https://tschatzl.github.io/2021/02/26/early-prune.html) on my blog that highlights the problem and shows potential gains.

Some additional parallelization of parts of the GC phases (e.g. [JDK-8214237](https://bugs.openjdk.java.net/browse/JDK-8214237)) may result in overall improved performance as e.g. reported [here](https://www.optaplanner.org/blog/2021/09/15/HowMuchFasterIsJava17.html).

## Other Noteworthy Changes

In addition, there are JDK 17 changes that are important but less or not visible at all for end users.

  * This mostly concerns the start of aggressive refactoring of the G1 collector code. Particularly we are in the process of moving out code from the catch-all class `G1CollectedHeap`, separating concerns and so slice it into more understandable components. This already improves maintainability and hopefully speeds up further work.

## What's next

Of course the GC team and other contributors is already actively working on JDK 18. Here is a short list of interesting changes that are currently in development and you may want to look out for. Without guarantees, as usual, they are going to be integrated when they are done ;-)

  * First, the actually already integrated change [JDK-8017163](https://bugs.openjdk.java.net/browse/JDK-8017163) massively reduces G1 memory consumption at no cost. This rewrite of [remembered set](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-garbage-collector-tuning.html#GUID-A0343B53-A690-4DDE-98F9-9877096DBF0F) data storage reduces its footprint by around 75% from JDK 17 to JDK 18. The following figure shows memory consumption as reported by NMT for the GC component for some database-like application as a teaser for various recent JDKs.
  
    ![here](/assets/20210920-bigramtester-memoryusage.png)

    Particularly note the yellow line showing current (18b11) total memory usage compared to 17b35 (pink) and 16.0.2 (blue). You can calculate remembered set size by subtracting the "floor" (cyan) from a given run of the curve.

    There will be a more thorough evaluation and explanation of the change in the future in this blog. At least remembered set size tuning should to a large degree a thing of the past. More changes building on this change to improve performance and further reduce remembered set memory size are planned.

  * Serial GC, Parallel GC and ZGC support string deduplication like G1 and Shenandoah in JDK 18. [JEP 192](http://openjdk.java.net/jeps/192) gives details about this technique, now applicable to all Hotspot collectors.

  * Support for archived heap objects for Serial GC is in development in [JDK-8273508](https://bugs.openjdk.java.net/browse/JDK-8273508).

  * Hamlin Li is currently doing great work on improving evacuation failure handling with the apparent goal to enable object pinning in G1 in the future; I wrote a short post on the problems and possible approaches [earlier](https://tschatzl.github.io/2021/06/28/evacuation-failure.html).

More to come :)

## Thanks go to…

Everyone that contributed to another great JDK release. See you in the next release :)

*Thomas*
