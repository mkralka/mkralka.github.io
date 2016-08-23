---
layout:   post
title:    Exceptionally Safe
subtitle: Writing code that is robust in the face of exceptions
date:     2016-07-10
tags:     [Software Engineering, Programming, Java, C++, Software Design]
---

Exceptions are often overlooked by software engineers. But who can blame us?
They are exceptional, after all. (See what I did there?) When they are not
overlooked, attention is often paid to only one side of the exception coin:
throwing exceptions. We tend to put effort into detecting error cases (e.g., a
method being called when the object is in the wrong state, invalid parameters,
etc.) while overlooking the implications of a thrown exception. When an
exception is thrown from a method, what effect does it have on the code
surrounding the caller? Are any mutable objects in an inconsistent state? Or
worse, an undefined state? The answers to these questions will help you
understand what level of exception safety your code provides to its clients.

{% include post_figure.html
    url="/assets/img/exceptionally_safe_caution_1.png"
    alt="Warning sign of man falling off a cliff. Warning sign of man getting run over by a bulldozer." %}

## What Is Exception Safety?
There are different levels of exception safety guarantees that your code can
provide to its clients. Since this is such an important topic in C++, the four
levels of safety (as described by
[Bjarne Stroustrup in Appendix E of *The C++ Programming Language*](http://www.stroustrup.com/3rd_safe.pdf))
seem like a good place to start. From weakest to strongest guarantee:

No exception safety
: No exception safety guarantees are made. If an exception is thrown from a
  method, the state of the object is effectively undefined. This is usually an
  unacceptable level of exception safety. It basically means that if an
  exception is thrown, the application must terminate immediately.

Basic exception safety
: If an exception is thrown from a method, the state of the object may have been
  changed but will remain internally consistent. No resources (such as memory or
  locks) are leaked. For example, in the implementation of a concurrent map, if
  an exception is throws in a *put* operation, any locks acquired during the put
  will be released. This is often the most practical level to achieve.

Strong exception safety
: If an exception is thrown from a method, the state of the application will be
  as if the method was not called. This is often hard to achieve because it can
  involve complicated roll back operations that may themselves fail.

No-throw guarantee
: No exceptions will be thrown from a method. Cleanup methods often fall into
  this category.

Exception safety is a huge part of C++. A lot of this stems from the fact that
memory must be managed (there is no garbage collector) and a poorly timed
exception can cause memory leaks if care is not taken. Templates and operator
overloading can exacerbate the problem, hiding exceptions in seeming innocuous
code like:

~~~ cpp
a = b + c
~~~

If `a`, `b`, and `c` are basic types (e.g., `int`), then this cannot possibly
throw an exception. However, what if `a`, `b`, and `c` are defined as follows?

~~~ cpp
complex a;
complex &b;
double c;
~~~

In this case, exceptions may be lurking in `a`'s overloaded assignment operator,
when dereferencing `b` (if it is a dangling/null reference), or when invoking
the addition operator for adding a `double` to a `complex`.

~~~ cpp
complex operator+(const complex &amp;lhs, double rhs)
{
    complex result = lhs;
    return result += rhs;
}
~~~

{% include post_figure.html
    url="/assets/img/exceptionally_safe_caution_2.png"
    alt="Warning sign of man injuring back. Warning sign of man about to get hit on head by falling toolbox." %}

## Exception Safety Starts at the Interface

Ever wondered why some C++ collection classes in the STL have such an awkward
interface? For example, why doesn't
[`queue::pop()`](http://www.cplusplus.com/reference/queue/queue/pop/)
return a value? Why must one call
[`queue::front()`](http://www.cplusplus.com/reference/queue/queue/front/)
to get the head of the queue and
[`queue::pop()`](http://www.cplusplus.com/reference/queue/queue/pop/)
to remove it? The answer is, not surprisingly, exception safety. Consider what
would happen if `pop()` also returned the head of the queue, using an interface
similar to the following:

~~~ cpp
template<typename T>
class queue {
public:
    virtual ~queue() =0;

    virtual bool empty() const =0;

    virtual void push(const T &amp;item) =0;
    virtual T pop() =0;

protected:
    queue(){}

private:
    // Prevent default creation
    queue(const queue &amp;other);
    queue &amp; operator=(const queue &amp;other);
};
~~~

A naïve implementation of `pop()` may look similar to the following:

~~~ cpp
T pop()
{
    // TODO: Make sure there's an element
    auto T result = items[read_pos__];
    read_pos = (read_pos + 1) % capacity;
    size -= 1;
    return result;
}
~~~

Looks good, right? Well, sadly, the `return` statement is not as innocuous as it
seems. This will invoke `T`'s copy constructor, which may throw an exception.

~~~ cpp
T(const T &other);
~~~

If an exception is thrown, the element that was the head of the queue is no
longer referenced by the queue and was not returned (an exception was thrown);
it has been lost forever. In fact, there is no safe way to implement `pop()`
without the possibility of elements being lost in this way. By changing the API
to split out the two operations, both can be implemented safely:

~~~ cpp
template<typename T>
class queue {
public:
    virtual ~queue() =0;

    virtual bool empty() const =0;
    virtual const T &front() const =0;
    virtual T &front() =0;

    virtual void push(const T &item) =0;
    virtual void pop() =0;

protected:
    queue(){}

private:
    // Prevent default creation
    queue(const queue &other);
    queue & operator=(const queue &other);
};
~~~

The implementation of `pop()` no longer needs to return a value, so it consists
of only arithmetic operations on the internal structures (operations that cannot
throw exceptions).

~~~ cpp
void pop()
{
    // TODO: Make sure there's an element
    read_pos = (read_pos + 1) % capacity;
    size -= 1;
}
~~~

The implementation of `front()` is also exception safe, as it consists of
nothing more than returning the address of the head of the queue (an operation
that cannot throw an exception).

~~~ cpp
const T &front() const
{
    // TODO: Make sure there's an element
    return items[read_pos__];
}
~~~

{% include post_figure.html
    url="/assets/img/exceptionally_safe_caution_3.png"
    alt="Warning sign of man tripping over a rock. Warning sign of man falling off of a ladder." %}

## MultiRegistry
The concept of exception safety is not limited to C++, although providing
exception-safe code is much less of a concern with languages like Java. Consider
a simple interface that allows you to register *things* with a registry:

~~~ java
interface Registry<T> {
    Registration register(T item);

    interface Registration extends AutoCloseable {
        @Override
        void close();
    }
}
~~~

Registered items can be removed from the registry by closing the returned
`Registration` object. For example, consider an implementation of `Registry` for
registering `Runnable`'s that are called when an application exits:

~~~ java
final class OnExitRegistry implements Registry<Runnable> {
    // Implementation details ...
}
~~~

It may get used similar to the following:

~~~ java
Registry<Runnable> onExitRegistry = new OnExitRegistry();

// ...

// Must clean all the temporary files (created above) if we exit.
Registration registration = onExitRegistry.register(this::cleanup);

// temporary files no longer exist, cleanup no longer needed.
registration.close();
~~~

Our task is to create a registry that manages registrations to multiple
registries. That is, clients register with our registry and we pass that
registration along to multiple downstream registries.

Our first cut at an implementation might look similar to the following:

~~~ java
final class MultiRegistry<T> implements Registry<T> {
    private final List<Registry<? super T>> registries;

    MultiRegistry(List<Registry<? super T>> registries) {
        this.registries = ImmutableList.copyOf(registries);
    }

    @Override
    public Registration register(T item) {
        ImmutableList.Builder<Registration> registrationsBuilder =
                ImmutableList.builder();
        registries.forEach(
                registry -> registrationsBuilder.add(registry.register(item)));
        return new MultiRegistration(registrationsBuilder.build());
    }

    private final static class MultiRegistration
    implements Registration {
        private final List<Registration> registrations;

        private MultiRegistration(List<Registration> registrations) {
            this.registrations = registrations;
        }

        @Override
        public void close() {
            registrations.forEach(Registration::close);
        }
    }
}
~~~

Looks good, right?

Sadly, no. This naïve implementation suffers from several exception safety
issues that can leave the state of the registry in an undesirable state. Let's
put on our *Exception Safety Hat* and go through this implementation.

Focusing in on `register()`, what's the desired behavior? It could be argued
that a *strong exception safety* guarantee is warranted, so that `item` should
not be registered with *any* downstream registry unless it is registered with
*all* downstream registries and a `Registration` is successfully returned. In
this method, several failures can leave the registry in a bad state:

* A call to `registry.register(item)` fails
* A call to `registrationsBuilder.add()` fails
* Creation of the `MultiRegistration` fails

In all of these cases, `item` may be registered with some (or all) of the
downstream registries with no way of closing the registrations. To address this
issue, any new registrations should be closed unless we can return a
`Registration` object. Our second attempt at the `register()` method may look
similar to the following:

~~~ java
public Registration register(T item) {
    List<Registration> registrations = new ArrayList<>(registries.size());

    try {
        for (Registry<? super T> registry: registries) {
            // Save the registration in case it can't be added to the
            // list and needs to be closed
            Registration registration = registry.register(item);
            try {
                registrations.add(registration);
                // The registration has been accepted by the list. If a
                // failure occurs, it will be closed through the reference
                // in the list.
                registration = null;
            } finally {
                if (registration != null) {
                    // An exception was thrown before registration was added
                    // to the list, so it must be closed explicitly.
                    registration.close();
                }
            }
        }

        Registration result =
                new MultiRegistration(ImmutableList.copyOf(registrations));       

        // Now that the result Registration has been created, clear out
        // the unaccounted for registrations so they won't be closed in
        // in the finally block.
        registrations = ImmutableList.of();
        return result;
    } finally {
        // Close any registrations that were not added to a
        // successfully returned Registration object. This will be empty
        // in non-failure cases.
        registrations.forEach(Registration::close);
    }
}
~~~

In this new version, registrations are collected in a mutable list so that they
can be closed if an exception is thrown. The list is cleared out only once there
is no chance of an exception getting thrown.

If `registry.register(item)` fails, then the `egistrations` list will contain
all of the successful registrations so far. These registrations will be closed
in the `finally` block of the outer `try` before the exception is propagated
outwards. Although the state of the downstream registries may have changed
throughout the execution of this method, the closing of the registrations will
undo that work.

If `registrations.add()` fails (which replaced `registrationBuilder.add()`), the
`finally` block of the inner `try` will close the registration (leaving the
registrations already added to the `registrations` list) to be handled by the
`finally` block of the outer `try`. If no such failure occurs, the `finally`
block of the inner `try` will see that the `registration` was set to `null`
(after the successful addition to the `registration` list) and will do nothing.
(This example is somewhat forced since `List.add()` is unlikely to throw an
exception — other than for `OutOfMemory`. Keep in mind that similar operations
may throw exceptions, requiring similar exception handling logic.)

If the creation of `MultiRegistration` fails, then all registrations will be
closed in the `finally` block of the outer `try` because they were added to the
`registrations` list but not removed from it. If the creation succeeds, the
`registrations` list is emptied so that the `finally` block of the outer `try`
will not find any orphaned `Registration` objects to close. Notice how the list
was replaced with `ImmutableList.of()` rather than calling
`registrations.clear()`. This is because the former simply returns a
pre-constructed singleton (and will not thrown an exception) whereas the latter
could, in theory, fail.

The astute reader should have noticed that it looks like we're not done yet. In
the error handling paths, `Registration.close()` is called, which could, itself,
throw an exception. Should these calls be wrapped in a `try`/`catch` block that
will suppress the new exception? Arguably, yes. Unfortunately, this `catch` is
not enough. A thrown exception indicates that closing of the registration failed
and the item may still be registered. This is akin to throwing an exception in a
destructor in C++, which is legal, but
[dangerous](http://bin-login.name/ftp/pub/docs/programming_languages/cpp/cffective_cpp/MEC/MI11_FR.HTM)
In cases like this, destructor-like behavior should not throw exceptions.

{% include post_figure.html
    url="/assets/img/exceptionally_safe_caution_4.png"
    alt="Warning sign of man slipping on oil. Warning sign of man getting electrocuted by a live wire." %}

## Try-with-resources

Since my primary language has shifted to Java, one of the things that I miss
most from C++ is (the poorly named)
[Resource Acquisition Is Initialization](https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)
(RAII) and its use of stack unwinding to perform rollback operations. The
textbook example of RAII involves resource locks (e.g., a mutex) where all calls
to *acquire* (or lock) must be matched with a call to *release* (or unlock).
Code can quickly get messy if all calls to acquire a lock also require a
`try`/`catch` block to unlock the mutex. Instead, C++ takes advantage of
automatic calls to destructors of stack variables during stack unwinding.

~~~ cpp
class scoped_lock {
public:
    scoped_lock(lock &lock) :
        lock__(lock)
    {
        lock__.acquire();
    }

    ~scoped_lock() {
        lock__.release();
    }

private:
    lock &lock__;

    // Prevent default creation
    scoped_lock(const scoped_lock &);
    scoped_lock &operator=(const scoped_lock &);
};
~~~

To execute a block of code with the lock acquired, simply create a `scoped_lock`
and let the compiler do all the work:

~~~ cpp
scoped_lock scoped_lock(mutex);
int result = 0; 
for (size_t i = 0; i < items_count; ++i) {
    result += items[i].count();
}
return result;
~~~

`mutex` is automatically released when `scoped_lock` is destroyed, whether this
happens because the function returned the result or an exception was thrown in
the body of the method. This pattern can be extended to support all-or-nothing
transaction-like rollbacks.

~~~ cpp
Transaction transaction;

// compute stuff_params ...
transaction.do_stuff(stuff_params);

// computer othter_stuff_params ...
transaction.do_other_stuff(other_stuff_params);

transaction.commit();
~~~

If an exception is thrown before `transaction.commit()` is called, the
destructor of `transaction` will perform whatever operations are required to
undo the operations that have already completed.

Java flirts with RAII with its support for
[`try-with-resources`](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html).
Although slightly more verbose than RAII in C++, try-with-resource will
automatically call `close()` on one or more
[`AutoCloseable`](https://docs.oracle.com/javase/8/docs/api/java/lang/AutoCloseable.html)
objects.

Consider a simple address book with a method for finding all addresses that
match a given predicate. The first-cut implementation might look similar to the
following:

~~~ java
class AddressBook {
    private final ReadWriteLock lock;
    private final List<Address> addresses;

    AddressBook() {
        this.lock = new ReentrantReadWriteLock();
        this.addresses = new LinkedList<>();
    }


    Collection<Address> findAll(Predicate<Address> predicate) {
        ImmutableList.Builder<Address> result = ImmutableList.builder();
        Lock readLock = lock.readLock();
        readLock.lock();
        for (Address address : addresses) {
            if (predicate.test(address)) {
                result.add(address);
            }
        }
        readLock.unlock();
        return result.build();
    }
}
~~~

Because the address book is updated infrequently but read very often, we use a
[`ReadWriteLock`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/ReadWriteLock.html)[^concurrency_is_hard]
to allow multiple concurrent readers but only a single writer. Unfortunately,
this means that we can't take advantage of Java's `synchronized` keyword (which
takes care of unlocking for us).
Like the `MultiRegistry` example above, if an exception is thrown after the read
lock has been acquired but before the read lock is released, the read lock will
be leaked and all subsequent writers will be blocked. We can fix this by adding
a `try`/`finally` block that will always unlock the read lock:

~~~ java
Collection<Address> findAll(Predicate<Address> predicate) {
    ImmutableList.Builder<Address> result = ImmutableList.builder();
    Lock readLock = lock.readLock();
    readLock.lock();
    try {
        for (Address address : addresses) {
            if (predicate.test(address)) {
                result.add(address);
            }
        }
    } finally {
        readLock.unlock();
    }
    return result.build();
}
~~~

This is now correct, but is verbose. We have to remember to add a
`try`/`finally` whenever we need to acquire/release a lock. This might be OK if
there are only a few instances, but if there are many instances, we can do
better by defining a few helpers:

~~~ java
private ScopedLock readLock() {
    return new ScopedLock(lock.readLock());
}

private ScopedLock writeLock() {
    return new ScopedLock(lock.writeLock());
}

private static class ScopedLock implements AutoCloseable {
    private final Lock lock;

    ScopedLock(Lock lock) {
        this.lock = lock;
        lock.lock();
    }

    @Override
    public void close() {
        lock.unlock();
    }
}
~~~

Much like the C++ `scoped_lock` example above, `ScopedLock` automatically locks
the lock in its constructor and releases the lock in the close method. To
acquire a read lock and automatically release it when done, simply use it in a
try-with-resources block.

~~~ java
Collection<Address> findAll(Predicate<Address> predicate) {
    ImmutableList.Builder<Address> result = ImmutableList.builder();

    try (ScopedLock ignored = readLock()) {
        for (Address address : addresses) {
            if (predicate.test(address)) {
                result.add(address);
            }
        }
    }
    return result.build();
}
~~~

> Pro-Tip: When implementing `AutoClosable`, remove the `throws Exception` from
> `close()`, otherwise you'll have to add a `catch` clause to your
> try-with-resources.

{% include post_figure.html
    url="/assets/img/exceptionally_safe_caution_5.png"
    alt="Warning sign of man who lost hand in power saw. Warning sign of man avoiding toxic waste." %}

## ScheduledExecutorService

Java's
[`ScheduledExecutorService`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledExecutorService.html)
runs submitted tasks asynchronously, with
[`scheduleAtFixedRate()`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledExecutorService.html#scheduleAtFixedRate-java.lang.Runnable-long-long-java.util.concurrent.TimeUnit-)
and
[`scheduleWithFixedDelay()`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledExecutorService.html#scheduleWithFixedDelay-java.lang.Runnable-long-long-java.util.concurrent.TimeUnit-)
scheduling the same tasks periodically. There's one catch: the executor service
will stop scheduling a periodic task if the task should ever throw an exception,
requiring your tasks to provide the `no-throw guarantee`.

~~~ java
executorService.scheduleAtFixedRate(
        () -> {
            try {
                checkFile(url, onChangeHandler);
            } catch (Exception ex) {
                logger.log(WARNING, "Error while checking file " + url, ex);
            }
        },
        0,
        pollingPeriod,
        pollingPeriodTimeUnit);
~~~

---

Exception safety begins at the interface. When strong exception safety is
needed, mind the interfaces that you're defining. As we discussed earlier, an
interface may not be implementable in an environment where strong exception
safety is required.

When making your code exception safe, whether *strong* or *basic* exception
safety is required, keep in mind that any method call can, in theory, throw an
exception. Always Be Careful. Exceptions can be lurking almost everywhere. This
is especially true in languages like C++ where what looks like a simple variable
assignment or the addition of two numbers may actually invoke any number of
overloaded operators, copy constructors, implicit constructors, and cast
operators.

Although not covered in this post, our good friend immutability can be a huge
help in achieving exception safety. Since immutable objects have no mutable
state, their methods do not need to consider exceptions; any thrown exception
will have no effect.

{% include post_figure.html
    url="/assets/img/exceptionally_safe_caution_6.png"
    alt="Warning sign of man falling backward off his chair. Warning sign of man slipping on ice." %}

[^concurrency_is_hard]:
    There are actually better ways to handle this scenario as explained in
    [Concurrency is hard](concurrency-is-hard).
