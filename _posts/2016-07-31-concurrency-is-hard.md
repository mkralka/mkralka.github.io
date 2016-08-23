---
layout: post
title:  Concurrency is hard
date:   2016-07-31
tags:   [Software Engineering, Programming, Concurrency, Java]
---

No, really. Concurrency *is* hard.

Of all the technical challenges that an engineer faces, concurrency is
high up on the list of the most difficult. It is such a challenge that
most of the companies for which I have worked have taken steps to reduce
the damage concurrency can wreak.

One company that I worked for,
[Proofpoint](https://www.proofpoint.com/), took the educational route.
As part of the normal on-boarding process, we held a 5-day bootcamp for
engineers; all were expected to attend, regardless of experience level.
During this bootcamp, a handful of senior engineers (including me) would
coach the new engineers as they worked through the process of building a
new REST service from the ground up. The attendees would learn about
[the stack](https://github.com/proofpoint/platform), perform code
reviews (both as the reviewer and reviewee), learn about the common
pitfalls of the libraries we used, and a lot more. Concurrency was such
a large concern that an entire day was devoted to its perils. We'd begin
the session with a talk (titled *Concurrency is Hard*) discussing common
concurrency errors, including very specific examples. Even after such
instruction, engineers still fell trap to the very mistakes of which
they were warned. This was so common that the session's closing talk
(titled *No, really. Concurrency **is** hard!*) was dedicated to
covering the concurrency mistakes the atendees made during the session,
in an attempt to drive one point home: *Even when special attention is
paid to writing correct, thread-safe code, engineers still make
mistakes*.  Perhaps we were just bad teachers. Perhaps. I'd like to
believe that concurrency truly is *hard* and us instructors had the deck
stacked against us.

Another company that I worked for,
[*Initech*](https://en.wikipedia.org/wiki/Office_Space)[^initech], took
a different tack: *abolish in-process concurrency*. Concurrency is *so*
hard that even the best engineers will make terrible mistakes.
*Initech*'s main product was written in C and was running on modest
hardware (usually with no more than eight cores). Concurrency was
achieved through multiple single-threaded, isolated processes that did
not share state. Since processes needed to coordinate, interaction was
isolated to standard RPC mechanisms, forcing clean APIs between
processes. Avoiding in-process concurrency, it was hoped, would result
in simpler software, reducing complexity and simplifying the job of the
engineer. Problem Solved. *Problem solved?* If only it were that simple.
Software decisions, especially hard ones, are not about choosing between
a superior and an inferior design or solution; it's about deciding which
trade-offs can be justified. Did the abolition of in-process concurrency
make things easier? Absolutely. But it also came with its own
challenges. Despite each process being single-threaded, processes still
needed to handle multiple simultaneous streams of data. This meant that
the state of each stream needed to be explicitly stored (rather than
being implicitly stored in a thread's stack), allowing the main thread
to resume processing a stream once it was able to transition to a new
state (e.g., because more data became available). A bigger problem that
abolishing in-process concurrency exposed was resource utilization.
Consider the case where the application is running on an 8-core machine
and only one process has any real work to do. In such cases, 7 cores may
be completely idle because the busy process has no threads to run on the
other cores. Don't get me wrong, the architecture served *Initech* well
for many years and allowed us to move quickly when implementing new
features. At the same time, we had use cases where it bit us, requiring
a lot of design work to address certain things that were significantly
harder because of the design choices made.

Let's assume that you've decided that in-process concurrency (i.e.,
writing a multi-threaded application) is the fork you are going to take
in this road. How do you handle concurrency issues in an effective
manner in your software?

---

## Attacking Thread-safety head on

So, you're looking to protect your data structures from multiple
threads, but you're not too sure what data structure to use.

In languages like Java, it's all too easy to slap the `synchronized`
keyword on your method and be done with it. This may be OK, but it's
important to understand what's going on under the covers and how that's
going to impact the performance of your application.

Let's build a consumer registry. This registry allows a producer to
produce to several consumer without the producer having to worry about
which consumers have registered. This registry implements the
[`Consumer`](https://docs.oracle.com/javase/8/docs/api/java/util/function/Consumer.html)
interface and provides a method allowing consumers to register. Data is
produced frequently and consumers registration is relatively rare. How
should we go about building such a solution?

### Mutex

A mutex (Mutual Exclusion lock) is a fundamental locking mechanism that
I assume you are familiar with, so I will not bore you with the details.
(If you are unfamiliar, [wikipedia](https://wikipedia.org) has a good
[article](https://en.wikipedia.org/wiki/Lock_%28computer_science%29).)

How might we implement our consumer registry using a mutex?

~~~ java
import java.util.ArrayList;
import java.util.List;
import java.util.function.Consumer;

public class ConsumerRegistry<T> implements Consumer<T> {

  private final List<Consumer<T>> registrants = new ArrayList<>();

  synchronized public void register(Consumer<T> registrant) {
    if (!registrants.contains(registrant)) {
      registrants.add(registrant);
    }
  }

  @Override
  synchronized public void accept(T t) {
    registrants.forEach(registrant -> registrant.accept(t));
  }
}
~~~

In Java, `synchronized` is syntactic sugar for acquiring and releasing
an object's
[intrinsic lock](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html).
(which is nothing more than a mutex). In fact, Java doesn't even have an
explicit `Mutex` class. To create a mutex, any `Object` will do:

~~~ java
Object mutex = new Object();

synchronized(mutex) {
  // Magic
}
~~~

Mutexes are easy to use, but are often the big-hammer solution to a
concurrency problem. By ensuring only one thread accesses a mutable
object at a time, consistency can be maintained.

One of the significant limitations of a mutex is that it serializes all
access to a data structure, even access that doesn't necessarily
conflict. It's likely that your use case can benefit from a less
heavy-handed solution. Consider one of the solutions below.

### Read/Write Lock

One of the problems with our mutex solution is that when a thread locks
the mutex for the purpose of notifying the registered consumers of data
that was just produced, other threads needing to notify registered
consumers will be blocked until the first thread has finished
(effectively serializing all access to the registrants).  In our use
case, data is produced frequently and consumers don't register very
often. Provided no one is updating the list, multiple threads can access
the list at the same time.

This is exactly the use case for a
[Read/Write lock](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock),
which allows access to data by multiple simultaneous readers while
restricting access to only one writer.

How might our implementation change if we used a read/write lock?

~~~ java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;
import java.util.function.Consumer;

public class ConsumerRegistry<T> implements Consumer<T> {

  private final List<Consumer<T>> registrants = new ArrayList<>();
  private final Lock readerLock;
  private final Lock writerLock;

  public ConsumerRegistry() {
    ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    this.readerLock = readWriteLock.readLock();
    this.writerLock = readWriteLock.writeLock();
  }

  public void register(Consumer<T> registrant) {
    writerLock.lock();
    try {
      if (!registrants.contains(registrant)) {
        registrants.add(registrant);
      }
    } finally {
      writerLock.unlock();
    }
  }

  @Override
  public void accept(T t) {
    readerLock.lock();
    try {
      registrants.forEach(registrant -> registrant.accept(t));
    } finally {
      readerLock.unlock();
    }
  }
}
~~~

This is **much** better. Our readers no longer interfere with one
another and are only blocked by writers. Did we hit a home-run with this
solution? Not quite. At first glance, everything seems OK, but
read/write locks have their own idiosyncrasies.

The astute reader may be asking themselves: "What happens if there is a
never-ending stream of readers?" In the mutex case, this is less
important. Readers and writers are effectively treated equally and,
depending on the implementation, will likely be woken up "randomly" or
in arrival order. How does this work in the case of a read/write lock?
Well, it depends.

In the case of Java's
[`ReentrantReadWriteLock`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/ReentrantReadWriteLock.html),
the default behavior is *unfair*. Readers are allowed to acquire any
lock not held by a writer, allowing a steady stream of readers to starve
a writer. A
[`ReentrantReadWriteLock`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/ReentrantReadWriteLock.html),
can be constructed to behave *fairly*, by trying to favor arrival order,
preventing writers from being starved. This comes at the expense of
lower reader throughput, as readers that arrive after a writer must wait
for the writer to get access, even if the reader could be finished
before the writer gains access.

We can do better, still.

### Lock-Free Readers

In the mutex and read/write lock implementations, readers have to block
the writers from updating the list while at least one reader is
traversing it. If the writers updated the registry by creating a whole
new list (instead of updating the existing list) then the readers would
not have to block.

> Immutable Objects are the concurrent programmer's best friend!

By replacing `registrants` with an immutable list, we can remove all
synchronization from the read path. A reader will essentially take a
snapshot of the current registrants and iterate over them, ignoring any
new registrants that a writer may add. Unfortunately, we can't just do
the same for the write path; to avoid lost registrations, we need to
handle the case where two or more writers attempt to update the
registrants at the same time. Since we need to serialize all updates,
it's easiest to revert to using the `ConsumerRegistry`'s intrinsic lock.
Our updated implementation might look similar to the following:

~~~ java
import com.google.common.collect.ImmutableList;

import java.util.List;
import java.util.function.Consumer;

public class ConsumerRegistry<T> implements Consumer<T> {

  private volatile List<Consumer<T>> registrants = ImmutableList.of();

  synchronized public void register(Consumer<T> registrant) {
    if (!registrants.contains(registrant)) {
      registrants = ImmutableList.<Consumer<T>>builder()
          .addAll(registrants)
          .add(registrant)
          .build();
    }
  }

  @Override
  public void accept(T t) {
    registrants.forEach(registrant -> registrant.accept(t));
  }
}
~~~

Notice how `registrants` is `volatile`? This ensures that threads that
call `accept()` will see any changes made by other threads.  Without it,
changes will only be seen after synchronization points (which may not
exist for the calling thread). If a thread calls `accept()` while
another thread is calling `register()`, one of two things will happen:
either `registrants` will evaluate to the old value (before `register()`
was called) or the new value (after `register()` was called). Either
way, it's the same as the previous mutex and read/write lock examples
with the caller acquiring the lock before or after (respectively) the
thread that performed the update.

If we remove `synchronized` from `register()`, we can lose
registrations. Consider the following scenario:

1. `registrants` contains registrants `{A, B, C}`.
2. Thread 1 grabs a copy of `registrants` (containing `{A, B, C}`) to
   add `D`
3. Thread 2 grabs a copy of `registrants` (containing `{A, B, C}`) to
   add `E`
4. Thread 1 creates a new list containing `{A, B, C, D}` and updates
   `registrants`.
5. Thread 2 creates a new list containing `{A, B, C, E}` and updates
   `registrants`, inadvertently deleting `D`.

Making `register()` `synchronized` is one way to prevent that from
happening.

In our lock-free readers solution, readers no longer participate in
`ConsumerRegistry` locking; they come and go as they please.  Behavior
is no different than if locks were in place but we're now only blocking
writers. Surely we've landed on the perfect solution! Right?

### Lock-Free

In 1991,
[Maurice Herlihy](https://en.wikipedia.org/wiki/Maurice_Herlihy)
published his seminal paper
[Wait-Free Synchronization](https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf),
which (among other things) proved that simple atomic operations, such as
[Compare and Swap](https://en.wikipedia.org/wiki/Compare-and-swap)
(CAS), can be used to build complicated lock-free data structures.

Java provides an implementation of these atomic operations in the
[`java.util.concurrent.atomic` package](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/package-summary.html),
with the
[`AtomicReference`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html)
being of note. `AtomicReference` extends a `volatile` reference with
higher-level lock-free operations, taking advantage of hardware support
for an atomic CAS operation available on contemporary hardware. If
implemented in software, the CAS operation of an `AtomicReference`
behaves as if it had the following implementation:

~~~ java
class AtomicReference<T> {
  private volatile T value;

  synchronized boolean compareAndSet(T current, T update) {
    if (value != current) {
      return false;
    }
    value = update;
    return true;
  }
}
~~~

(In reality, it uses a native method that exploits hardware support for
CAS, making it blazing fast.)

*Note how `compareAndSet()` uses `==` and not `.equals()`, comparing
identical objects not those that are logical equivalent. This is why CAS
can be performed in hardware.*

With this one operation, we can build higher-level atomic update
operations.  For example,
[`getAndUpdate()`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html#getAndUpdate-java.util.function.UnaryOperator-)
(which gets the current value, transforms it using an operator, stores
the transformed value, then returns the original value) may have an
implementation similar to the following:

~~~ java
T getAndUpdate(UnaryOperator<T> updateFunction) {
  T current, update;
  do {
    current = value;
    update = updateFunction(current);
  } while (!compareAndSet(current, update));
  return current;
}
~~~

This solves the last of our problems, removing the synchronization from
the writers. Even though the writes happen infrequently, synchronization
can be an expensive operation, even when there is little contention for
a lock[^futex]. Instead of blocking the writers from updating the list
at the same time, we can let as many writers in at the same time, using
`compareAndSet()` to allow only one writer to update the registry; other
writers will have to try again.

Our completely lock-free implementation may looks similar to the
following:

~~~ java
import com.google.common.collect.ImmutableList;

import java.util.List;
import java.util.concurrent.atomic.AtomicReference;
import java.util.function.Consumer;

public class ConsumerRegistry<T> implements Consumer<T> {

  private final AtomicReference<List<Consumer<T>>> registrants =
      new AtomicReference<>(ImmutableList.of());

  public void register(Consumer<T> registrant) {
    List<Consumer<T>> current, update;
    do {
      current = registrants.get();
      if (current.contains(registrant)) {
        break;
      }
      update = ImmutableList.<Consumer<T>>builder()
          .addAll(current)
          .add(registrant)
          .build();

    } while (!registrants.compareAndSet(current, update));
  }

  @Override
  public void accept(T t) {
    registrants.get().forEach(registrant -> registrant.accept(t));
  }
}
~~~

Let's revisit the scenario from before to see how it handles two
simultaneous writers:

1. `registrants` contains registrants `{A, B, C}`.
2. Thread 1 grabs a copy of `registrants` (containing `{A, B, C}`) to
   add `D`.
3. Thread 2 grabs a copy of `registrants` (containing `{A, B, C}`) to
   add `E`.
4. Thread 1 creates a new list containing `{A, B, C, D}` and updates
   `registrants`.
5. Thread 2 creates a new list containing `{A, B, C, E}` but fails to
   update `registrants` because it refers to a `{A, B, C, D}` and not
   the expected `{A, B, C}`.
6. Thread 2 grabs a copy of `registrants` (containing `{A, B, C, D}`) to
   add `E`.
4. Thread 2 creates a new list containing `{A, B, C, D, E}` and updates
   `registrants`.

One last thing worth noting. Garbage collected languages make lock-free
structures incredibly easy to implement. In non-garbage collected
languages (like C and C++), the updating thread would be expected to
free the memory associated with the old list. This is surprisingly
difficult, as there is no easy way for the writer to know that all
references have been released.
[Andrei Alexandrescu](https://en.wikipedia.org/wiki/Andrei_Alexandrescu)
has written an excellent article,
[Lock-Free Data Structures](http://www.drdobbs.com/lock-free-data-structures/184401865?queryText=%2522%2BLock-Free%2BData%2BStructures%2522),
outlining how this is done in C++.

### Sharded Locks

As awesome as lock-free structures are, it's not always possible to
provide a satisfactory implementation. Sometimes, locking is required.

One strategy to minimize contention on an object is to provide sharded
locks. At construction time, the object is split up into several fixed
shards. Contention occurs only when there is concurrent access to one of
the shards. This applies to any type of lock (e.g., mutex or read/write
lock). A perfect example for this is a hash-based map. The hash buckets
can be split into several shards, with each shard protected by a
read/write lock. When a key is accessed, the shard associated with the
key will be locked for access, allowing other threads to access other
shards.

Without getting into too much detail about how this would be
implemented, this type of locking is provided by Guava's implementation
of a
[`ConcurrentMap`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentMap.html)
and is one reasons you should always use
[`MapMaker`](https://google.github.io/guava/releases/18.0/api/docs/com/google/common/collect/MapMaker.html)
in favor of creating a
[`ConcurrentHashMap`](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html).

---

## Common Pitfalls

Even when making a conscious effort to make code thread safe, mistakes
can slip through. Some of these mistakes tend to be more common than
others.

### Thread-unsafe Interfaces

You're working on a monitor that keeps track of the state of something.
Clients use this to get the state and make decisions based on what they
have read. The interface you've defined looks similar to the following:

~~~ java
import java.time.Instant;

public interface StateMonitor {

  Instant getLastUpdate();

  FooState getFooState();

  BarState getBarState();

  final class FooState {
    // Foo State
  }

  final class BarState {
    // Foo State
  }
}
~~~

You've completed an implementation of this interface and you've managed
to convinced yourself that your implementation is thread safe. Truth be
told, your implementation ***is*** thread safe. Things still aren't
working. But why?

The problem you've failed to consider is the thread safety in the
interface itself! What you've done is designed an interface that cannot
easily be used in a thread-safe manner (if it's even possible to use it
in a thread-safe manner).

To keep things simple, lets assume that you used the brute-force
`synchronized` keyword on all your methods. What that means is that
while a call to `getFooState()` is in progress, nothing can update the
state so the returned `FooState` will be consistent.  The problem
happens when a client needs to atomically acquire two or more components
of the state. Although no updates can occur while `getFooState()` is
being called, updates can happen between calls to `getFooState()` and
`getLastUpdate()`.  Unfortunately, users of this API have little
recourse to ensure atomicity.

How can we address this problem? We have to redesign the interface.

~~~ java
import java.time.Instant;

public interface StateMonitor {

  Snapshot getSnapshot();

  default Instant getLastUpdate() {
    return getSnapshot().getLastUpdate();
  }

  default FooState getFooState() {
    return getSnapshot().getFooState();
  }

  default BarState getBarState() {
    return getSnapshot().getBarState();
  }

  final class Snapshot {
    private final Instant lastUpdate;
    private final FooState fooState;
    private final BarState barState;

    public Snapshot(Instant lastUpdate, FooState fooState, BarState barState) {
      this.lastUpdate = lastUpdate;
      this.fooState = fooState;
      this.barState = barState;
    }

    public Instant getLastUpdate() {
      return lastUpdate;
    }

    public FooState getFooState() {
      return fooState;
    }

    public BarState getBarState() {
      return barState;
    }
  }

  final class FooState {
    // Foo State
  }

  final class BarState {
    // Foo State
  }
}
~~~

Instead of giving access to each field independently, all fields are
bundled in a snapshot, to which clients have access. If a client needs a
consistent view of multiple fields, they can acquire a snapshot and then
pull out the desired fields from it.

For those clients that do not care about consistency between fields,
helper methods for accessing the individual fields can be provided.

### Leakage from Concurrent Collections

Consider a mutable object:

~~~ java
static class Thingy {
  private final Set<String> names = Sets.newSetFromMap(new ConcurrentHashMap<>());
  private final List<Integer> values = Collections.synchronizedList(new ArrayList<>());

  void addName(String name) {
    names.add(name);
  }

  void addValue(Integer value) {
    values.add(value);
  }
}
~~~

We have to maintain a collection of named `Thingy` objects that will be
updated through the life of the application. Since the `Thingy` objects
are already thread safe and independent from everything else, it seems
wasteful to synchronize our update methods. Why not use a
`ConcurrentMap`?

~~~ java
private final Map<String, Thingy> thingies = new ConcurrentHashMap<>();

void addNameToThingy(String thingyId, String name) {
  Thingy thingy = thingies.get(thingyId);
  if (thingy == null) {
    thingy = new Thingy();
    thingies.put(thingyId, thingy);
  }

  thingy.addName(name);
}
~~~

Since we're using concurrent collections, we've inadvertently given
ourselves a false sense of security. The problem happens if two threads
attempt to add a new thingy name for a non-existent thingy.

1. Thread 1 attempts to add `"thread1"` to thingy `"A"`.
2. Thread 1 checks for a current `Thingy` in `thingies` and finds none.
3. Thread 2 attempts to add `"thread2"` to thingy `"A"`.
4. Thread 2 checks for a current `Thingy` in `thingies` and finds none.
5. Thread 2 creates a new `Thingy` for thingy `"A"` and adds it to the
   `thingies` map.
6. Thread 1 creates a new `Thingy` for thingy `"A"` and adds it to the
   `thingies` map.
7. Thread 1 updates its `Thingy` (the one in the `thingies` map), adding
   `"thread1"` to thingy `"A"`.
8. Thread 2 update its `Thingy` (the one no longer in the `thingies`
   map), adding `"thread2"` to a stray thingy.

In the end, thingy `"A"` only has the name `"thread1"` but not
`"thread2"`.

In order to close this hole, we need to leverage some of the
functionality of `ConcurrentMap`. The features to leverage depends on
the version of Java.

In Java 7,
[`ConcurrentMap.putIfAbsent()`](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ConcurrentMap.html#putIfAbsent%28K,%20V%29)
is needed. `addNameToThingy()` may look similar to the following:

~~~ java
void addNameToThingy(String thingyId, String name) {
  Thingy thingy = thingies.get(thingyId);
  if (thingy == null) {
    thingy = new Thingy();
    Thingy oldThingy = thingies.putIfAbsent(thingyId, thingy);
    if (oldThingy != null) {
      thingy = oldThingy;
    }
  }

  thingy.addName(name);
}
~~~

The code is a bit awkward here because `putIfAbsent()` returns the
previously stored value. If it returns `null`, then our new `Thingy`
made it into the map and should be used. If it returns a non-`null`
value, some other thread managed to squeeze in another `Thingy` and we
should use that.

This is substantially easier in Java 8, which introduced
[`ConcurrentMap.computeIfAbsent()`](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentMap.html#computeIfAbsent-K-java.util.function.Function-).
Using it, `addNameToThingy()` may look similar to the following:

~~~ java
void addNameToThingy(String thingyId, String name) {
  thingies.computeIfAbsent(thingyId, key -> new Thingy())
      .addName(name);
}
~~~

---

## Immutability

No talk about concurrency is completely without a nod to immutability.
One of the easiest ways to avoid having to deal with concurrency issues
is to make as many things immutable as possible.

***I can't stress this enough.***

Immutability is an important enough topic to devote an entire post to
it.

[^initech]:
    This company shall remain nameless, to protect the innocent.

[^futex]:
    How expensive really depends on the platform. For example, Linux has
    the concept of a [*futex*](https://en.wikipedia.org/wiki/Futex) (or
    Fast Userspace muTEX), which only require transitioning to kernel
    space (which is expensive) when there is contention for a futex
    (e.g., two threads attempt to acquire the lock at the same time).
