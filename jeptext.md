Summary
-------

Reduce the size of object headers in the HotSpot JVM from between 96 and 128 bits down to 64 bits on 64-bit architectures. This will reduce heap size, improve deployment density, and increase data locality.


Goals
-----

When enabled, this feature

* Must reduce the object header size to 64 bits (8 bytes) on the target 64-bit platforms (x64 and AArch64),
* Should reduce object sizes and memory footprint on realistic workloads,
* Should not introduce more than 5% throughput or latency overheads on the target 64-bit platforms, and only in infrequent cases, and
* Should not introduce measurable throughput or latency overheads on non-target 64-bit platforms.

When disabled, this feature

* Must retain the original object header layout and object sizes on all platforms, and
* Should not introduce measurable throughput or latency overheads on any platform.

This experimental feature will have a broad impact on real-world applications. The code might have inefficiencies, bugs, and unanticipated non-bug behaviors. This feature must therefore be disabled by default, and enabled only by explicit user request. We intend to enable it by default in later releases and eventually remove the code for legacy object headers altogether.


## Non-Goals

It is not a goal to

* Reduce the object header size below 64 bits on 64-bit platforms,
* Reduce the object header size on non-target 64-bit platforms,
* Change the object header size on 32-bit platforms, since they are already 64 bits, or
* Change the encoding of object content (i.e., fields and array elements) or array metadata (i.e., array length).


## Motivation

A Java object stored in the heap has metadata, which the HotSpot JVM stores in the object's header. The size of the header is constant; it is independent of object type, array shape, and content. In the 64-bit HotSpot JVM object headers are between 96 bits (12 bytes) and 128 bits (16 bytes), depending on how the JVM is configured.

Objects in Java programs tend to be small. [Experiments conducted as part of Project Lilliput](https://wiki.openjdk.org/display/lilliput/Lilliput+Experiment+Results) show that many workloads have average object sizes of 256 to 512 bits (32 to 64 bytes). This implies that more than 20% of live data can be taken by object headers alone.  Thus even a small improvement in object header size could bring a large improvement in footprint. Cutting down the header of each object from 96 to 64 bits means improving overall heap usage by more than 10%, since the header is a fixed cost for every object. A smaller average object size leads to improvement in memory usage, GC pressure, and data locality.

Early adopters of Project Lilliput who have tried it with real-world applications confirm that live data is typically reduced by 10%–20%.


## Description

In the HotSpot JVM, object headers support many different features:

* *Garbage collection* — Storing forwarding pointers and tracking object ages;
* *Type system* — Identifying an object's class, which is used for method invocation, reflection, type checks, etc.;
* *Locking* — Storing information about associated light-weight and heavy-weight locks; and
* *Hash codes* — Storing an object's stable identity hash code, once computed.


### Current object headers

The current object header layout is split into a *mark word* and a *class word*.

The mark word comes first, has the size of a machine address, and contains:

    Mark Word (normal):
     64                     39                              8    3  0
      [.......................HHHHHHHHHHHHHHHHHHHHHHHHHHHHHHH.AAAA.TT]
             (Unused)                      (Hash Code)     (GC Age)(Tag)

In some situations the mark word is overwritten with a tagged pointer to a separate data structure:

    Mark Word (overwritten):
     64                                                           2 0
      [ppppppppppppppppppppppppppppppppppppppppppppppppppppppppppppTT]
                                (Native Pointer)                   (Tag)

When this is done, the tag bits describe the type of pointer stored in the header. If necessary, the original mark word is preserved (*displaced*) in the data structure to which this pointer refers, and the fields of the original header (e.g., the age bits and hash code) are accessed by dereferencing the pointer to get to the displaced header.

The class word comes after the mark word. It takes one of two shapes, depending on whether compressed class pointers are enabled:

    Class Word (uncompressed):
    64                                                               0
     [cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc]
                              (Class Pointer)

    Class Word (compressed):
    32                               0
     [CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC]
         (Compressed Class Pointer)

The class word is never overwritten, which means that an object's type information is always available, so no additional steps are required to check a type or invoke a method. Most importantly, the parts of the runtime that need that type information do not need to cooperate with the locking, hashing, and GC subsystems that can change the mark word.


### Compact object headers

For compact object headers we remove the division between the mark and class words by subsuming the class pointer into the mark word:

    Header (compact):
    64                    42                             11   7   3  0
     [CCCCCCCCCCCCCCCCCCCCCCHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHVVVVAAAASTT]
     (Compressed Class Pointer)       (Hash Code)         /(GC Age)^(Tag)
                                  (Valhalla-reserved bits) (Self Forwarded Tag)

Locking no longer overwrites the mark word with a tagged pointer, and thus preserves the compressed class pointer. GC forwarding operations become more complex to avoid losing direct access to the compressed class pointer, as discussed below.

Compact object headers are guarded by a new experimental runtime option, `-XX:(+/-)UseCompactObjectHeaders`. This option is disabled by default.


### Compressed class pointers

Today's compressed class pointers encode a 64-bit pointer into 32 bits.  JDK&nbsp;8 introduced compressed class pointers, which first depended on compressed object pointers; JDK&nbsp;15 [removed this dependency](https://bugs.openjdk.org/browse/JDK-8241825). Today, there are only two scenarios in which one would disable compressed class pointers:

- An application uses JVMCI, which on some platforms does not support compressed class pointers, or
- An application loads more than roughly four million classes.

The theoretical limit of four million classes is imposed by the way that class space is implemented today. An application that hits this limit would use approximately six to 40 GB of [metaspace](https://openjdk.org/jeps/387) alone; we have yet to see an application that does this.

Compact object headers reduce the bits available to identify a class to 22. They therefore require compressed class pointers to be enabled unconditionally. The encoding will be modified so that the JVM can address a comparable number of classes with fewer bits.The two scenarios mentioned above must be addressed before eventually removing support for legacy object headers; in the mean time, applications can always disable compact headers.

### Stack locking

Stack locking is the first light-weight step in HotSpot's object-locking subsystem. It works in cases where the object's monitor is uncontended, no thread control methods (`wait()`, `notify()`, etc.) are called, and no JNI locking is used. In such cases HotSpot displaces the original mark word to the stack of the locking thread and overwrites the word with a pointer to that stack location. This associates the locking thread with the object, since we know the thread stack's address range, while retaining the original header.

With compact object headers, stack locking is inherently racy during unlocking from the perspective of non-owner threads. Suppose that thread A tries to access the compressed class pointer in an object's displaced header, and thread B stack-unlocks the object and removes the displaced header from the stack. Thread A is then exposed to a dangling stack reference, which likely contains garbage.

The same problem exists in the current stack locking code when, e.g., accessing the identity hash code stored in the displaced header. In that situation the race condition is avoided by inflating the lock into a full-blown monitor lock (see next section). This approach would not work well with compact object headers because accesses to class pointers are orders of magnitude more frequent than accesses to identity hash codes. Inflating the majority of stack locks would result in unacceptable performance overheads.

To address the race condition for compact object headers, a prerequisite improvement for this work is an alternative light-weight locking protocol ([8291555](https://bugs.openjdk.org/browse/JDK-8291555)) that stores locking data in a separate thread-local area rather than in the object header, preserving the original header and class information.


### Monitor locking

Monitor locking is the final heavy-weight step in HotSpot's object-locking subsystem. It is invoked when the monitor is contended or thread control methods are used, or any light-weight locking step fails. Monitor locking creates a separate `ObjectMonitor` structure, stores the original header there, and overwrites the header with the pointer to that `ObjectMonitor`.

Compared to stack locking, monitor locking is not as racy. Once a monitor has been created and installed, i.e., *inflated*, it remains until it is disposed of, i.e., *deflated*. Java threads coordinate with monitor deflation so that, when a Java thread loads a header that carries a monitor pointer, it can safely access that monitor without risk of accessing a stale monitor. Once a lock has been inflated to a full `ObjectMonitor` it is safe to load the displaced header from the `ObjectMonitor` and decode the class from there.

However, not all GC threads coordinate with monitor deflation, and in some cases can not be made to coordinate without introducing a lot of complexity. Therefore, a new way to map heap objects to ObjectMonitors is introduced, which, together with the new lightweight locking, will eliminate the concept of 'displaced headers' altogether. This also removes the need for threads which needs to access object headers to coordinate with monitor deflation.

### GC forwarding

Moving GCs perform object relocations in two steps: Move the object and record the mapping between its old and new locations (i.e., *forwarding*), and then use this mapping to update the references in the entire heap or in just a particular generation.

Of the current HotSpot GCs, only ZGC uses a separate forwarding table to record forwardings. All other GCs record forwarding information by overwriting the header of the old object with the location of the new object. There are two distinct scenarios that involve headers.

- *Copying phases* — Objects are copied to an empty space, and the forwarding pointer is stored in the header of the old copy. The header is preserved in the new copy, so all data from the original header is available. GC code that requires the class pointer to determine the object's size for heap iteration, for example, can reach the new copy and the original header there.

  There is one exception in this scenario: When copying the object to its new location fails then the GCs install a forwarding pointer to the object itself, thus making the object _self-forwarded_. With compact object headers, this would overwrite the type information. To address this we indicate that an object is self-forwarded by setting the third bit of the object header rather than by overwriting the entire header.

* *Sliding phases* — Objects are relocated, i.e., slid to lower addresses, within the same space. This is typically invoked in situations when heap memory is exhausted and not enough space is left for copying objects. When that happens, a last-ditch effort is made to do a full collection using a sliding GC.

  Sliding GC works in four phases:

  1. *Mark* — Determine the set of live objects.

  2. *Compute addresses* — Walk over all live objects and compute their new locations, i.e., where they would be placed one after another. Record those  locations as forwardings in the object headers.

  3. *Update references* — Walk over all live objects and update all object references to point to the new locations.

  4. *Copy* — Actually copy all live objects to their new locations.

  Step 2 destroys the original headers. This is also a problem for the current implementation: If the header is *interesting*, that is, it has an installed identity hash code, or locking information, etc, then we need to preserve it. The current GCs do that by storing these headers in a side table, and restoring them after a GC. This works well because there are usually only a few objects with interesting headers. With compact object headers, every object comes with an interesting header, because now that header contains the crucial class information. Storing a large number of preserved headers would consume a significant amount of native heap.

  To overcome this problem, a prerequisite for this work is an alternative implementation that uses a compact native forwarding table to store forwardings, leaving the original headers intact ([8305896](https://bugs.openjdk.org/browse/JDK-8305896)). We know from experience with ZGC that it is possible to implement such a forwarding table with reasonable footprint.


### GC walking

Garbage collectors frequently walk the heap by scanning objects linearly. This requires determining the size of each object, which requires access to each object's class pointer.

When the class pointer is encoded in the header, some simple arithmetic is required to decode it. The cost of doing this is low compared to the cost of the memory accesses involved in a GC walk. No additional implementation work is needed here, since the GCs already access class pointers via a common VM interface.

### Identity hash codes

The current implementation of identity hash codes stores only 31 bits on 64-bit platforms and only 25 bits on 32-bit platforms. With compact object headers, this is not going to change.


## Alternatives

* *Continue to maintain 32-bit platforms* — The mark and class words in object headers are sized as machine pointers, so headers on 32-bit platforms are already 64 bits. However, the difficulty of maintaining the 32-bit ports, coupled with the industry move from 32-bit environments, make this alternative impractical in the long term.

* *Implement 32-bit object headers* — With more effort, we could implement 32-bit headers. This would likely involve implementing an on-demand storage for identity hash codes, and so forth. That is our ultimate goal, but initial explorations show that it will require much more work. This proposal captures an important milestone that brings substantial improvements which we can deliver with low risk as we work further toward 32-bit headers.


## Testing

Changing the header layout of Java heap objects touches many HotSpot JVM subsystems: the runtime, all garbage collectors, all just-in-time compilers, the interpreters, the serviceability agent, and the architecture-specific code for all supported platforms. Such massive changes warrant massive testing.

Compact object headers will be tested by:

* Tier 1–4 tests, and possibly more testing tiers by vendors which have them;
* The SPECjvm, SPECjbb, DaCapo, and Renaissance benchmark suites, to test both correctness and performance;
* JCStress, to test the new locking implementation; and
* Some real-world workloads.

All of these tests will be executed with the feature turned on and off, with multiple combinations of GCs and JIT compilers, and on several hardware targets.

We will also deliver a new set of tests which measure the size of a variety of objects, e.g., plain objects, primitive type arrays, reference arrays, and their headers.

The ultimate test for performance and correctness will be real-world workloads once this experimental feature is delivered.


## Risks and Assumptions

* *Future runtime features need object header bits* — This proposal leaves no spare bits in the header for future features that might need such bits. We mitigate this risk organizationally by discussing object header needs with other major JDK projects. We mitigate this risk technically by assuming that identity hash codes and compressed class pointers can be shrunk even further in order to make bits available should future runtime features need them.

* *Implementation bugs in feature code* — The usual risk for an intrusive feature such as this is bugs in the implementation. While issues in the header layout might be visible immediately with most tests, subtleties in the new locking and GC forwarding protocols may expose bugs only rarely. We mitigate this risk with careful reviews by component owners and by running many tests with the feature enabled. This risk does not affect the product, so long as the feature remains experimental and off by default.

* *Implementation bugs in legacy code* — We try to avoid changing legacy code paths, but some refactorings necessarily touch shared code. This exposes the risk of bugs even when the feature is disabled. In addition to careful reviews and testing, we mitigate this risk by coding defensively and trying to avoid modifying shared code paths, even if it requires more work in feature code paths.

* *Performance issues in feature code* — The more complex protocols for compact object headers may introduce performance issues when the feature is enabled. We mitigate this risk by running major benchmarks and understanding the feature's impact on their performance. There are performance costs for accessing the class pointer indirectly, for using the alternative stack locking scheme, for employing the alternative GC sliding forwarding machinery, and for having fewer identity hash code bits. This risk does not affect the product, so long as the feature remains experimental and off by default.

* *Performance issues in legacy code* — There is a minor risk that refactoring the legacy code paths will affect performance in unexpected ways. We mitigate this risk by minimizing the changes to the legacy code paths and showing that the performance of major workloads is not substantially affected.

* *Compressed class pointers support* — Compressed class pointers are not supported by JVMCI on x64. We mitigate the immediate risk by disabling compact object headers when JVMCI is enabled. The long-term risk is that compact headers are never implemented in JVMCI, which would forever block removing the legacy header implementation. We assign only a minor probability to this risk, since other JIT compilers support compact object headers without intrusive changes.

* *Compressed class pointers encoding* — As stated above, the current implementation of compressed class pointers is limited to about three million classes. Presently, users can work around this limitation by disabling compressed class pointers, but if we remove the legacy header implementation then that will no longer be possible. We mitigate the immediate risk by providing compact object headers as an experimental feature; in the long term we intend to work toward more efficient compressed class pointer encoding schemes.

* *Changing low-level interfaces* — Some components that manipulate object headers directly, notably the Graal compiler as the major user of JVMCI, will have to implement the new header layout. We mitigate the current risk by identifying these components and disabling the feature when those components are in use. Before the feature graduates from experimental status, those components will need to be upgraded.

* *Soft project failure* — There is a minor risk that the feature has irreconcilable functional regressions compared to the legacy implementation, e.g., limiting the number of representable classes. A related risk is that while the feature provides significant performance improvements on its own, it comes with significant functional limitations which might lead to an argument for keeping both the new and legacy header implementations forever. Given that the goal of this work is to replace the legacy header implementation eventually, we consider this a soft project failure. We mitigate this risk by carefully examining current limitations, planning future work to eliminate them, and looking to early adopters reports to identify other risks before we invest too much effort.

* *Hard project failure* — While very unlikely, it may turn out that compact object headers do not yield tangible real-world improvements, or that the achievable improvements do not justify their additional complexity. We mitigate this minor risk by gating the new code paths as experimental, thus keeping a path open to removing the feature in a future release should the need arise.


## Dependencies

* Project Valhalla requires 4 bits in the object header. We reserved those 4 bits in the compact object header layout.
