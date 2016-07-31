---
layout: post
title:  It's About Time
date:   2016-07-05
tags: [Time, Software Engineering, Programming]
---

Time. Measuring it seems effortless. By taking a peek at a *clock*, you can
quickly see the current time and get a good sense of when you need to feed the
dog, tuck the kids into bed, or catch the train to work. There are probably
several *clocks* surrounding you (a smart phone, tablet, laptop, or even that
bulky clock on the kitchen counter that is occasionally used to cook food),
making it much easier to keep track of time. For such a seemingly simple
concept, there are many opportunities to make subtle mistakes when attempting
to measure time in the code you write. Why is time so complicated? As with many
things in life, the answer is: *It's complicated*.

Let's peel back the layers of time and explore how your computer keeps track of
time. With this knowledge, you will understand what makes a reliable time
library and what can befall your code when using a poorly designed library.

**Disclaimer: This post is based mainly on how GNU/Linux and other UNIX-like operating systems work on Intel PCs. YMMV.**

* * *

Measuring time with both
[*accuracy*](https://en.wikipedia.org/wiki/Accuracy_and_precision) and
[*precision*](https://en.wikipedia.org/wiki/Accuracy_and_precision) is
surprisingly difficult, even when ignoring complications such as
[time dilation](https://en.wikipedia.org/wiki/Time_dilation) and
[special relativity](https://youtu.be/ERgwVm9qWKA). To keep this simple, let's
assume that every measurer can be found in the same inertial frame (e.g.,
located anywhere on the surface of the earth, be it in a data center or on your
desk), requiring no compensation for special relativity.

* * *

## Achieving an Accurate Wall Clock

The *Wall Clock* is the clock with which you are likely most familiar. It
represents a shared time upon which everyone agrees. It's the time you see when
you look at your phone, tablet, or notebook. Without everyone in agreement about
what the current time is and how to measure it, you will likely arrive late to
your appointments or pay your taxes too early. This clock is very important when
interacting with users or when referring to specific instants in time (be it in
the past, present, or the future) between components of your system.

When a computer first boots, the operating system uses the hardware clock, also
known as the [Real-Time clock](https://en.wikipedia.org/wiki/Real-time_clock)
(or RTC) to initialize its wall clock. This piece of hardware is battery-powered
so it keeps time even when the computer is powered off. Reading the RTC is an
awkward and expensive operation (in terms of the number of CPU cycles), so
modern kernels maintain a system clock for measuring the passage of time while
the system is running.

This system clock is tied to the heartbeat of the kernel: the timer interrupt.
The
[Programmable Interval Timer](https://en.wikipedia.org/wiki/Programmable_interval_timer)
(or PIT) is configured to raise a timer interrupt at regular intervals,
typically in the 100–1000 Hz range, to wake up the kernel to see if any
scheduling must be performed (e.g., switching away from the current process that
ran out of its time slice, running delayed work, etc.). Since the kernel knows
the frequency of the timer interrupt, it can advance the system clock by one
timer tick every interrupt. For example, if the timer is set at 100 Hz, the
kernel will advance the timer by 10 ms. To provide the current time with more
granularity than the period of the timer interrupt, the kernel will interpolate
using various reliable, hardware sources.[^sources].

The astute reader may have noticed that the RTC and system clock are driven by
different oscillators: one in the RTC and the other in PIT. Although these two
oscillators are extremely precise, they are not necessarily accurate and will
tend to drift apart. Modern operating systems will periodically
[jam sync](https://en.wikipedia.org/wiki/Jam_sync) (which we will discuss more
below) the RTC to match the current system time. If the jam sync happens
frequently enough, the amount of drift that can accumulate will likely go
unnoticed by the casual user (the Linux kernel, for example, does this every 11
minutes). Unfortunately, this is insufficient as it doesn't account for the
computer being turned off for long periods of time. For example, if the RTC and
PIT drift by 5 ppm (5 seconds every 1 million seconds), the RTC and PIT will be
off by about 13 seconds after 1 month. To compensate for this, modern operating
systems will measure and store the rate of drift between the RTC and system
clock along with the time of the last jam sync. When initializing the system
clock, it will take this drift into consideration and compensate for the lost or
gained time.

Great, we now have a system that can measure wall clock time precisely, but is
it accurate? Sadly, no. If the RTC and PIT can drift from each other, which is
correct? Not surprisingly, both are probably inaccurate. The oscillators in the
RTC and PIT are inexpensive, resulting in precise but not necessarily accurate
oscillations. In order to keep accurate time, a significantly more accurate
clock is needed. Unfortunately, more accurate clocks are expensive, making them
an impractical clock source.

Your computer, like many other computational devices, get around this problem
using layers of indirection via the
[Network Time Protocol](https://en.wikipedia.org/wiki/Network_Time_Protocol)
(NTP). NTP consists of three components: extremely accurate clocks, the accurate
transmission of time between nodes across jittery networks, and the algorithms
for using the acquired time to accommodate for clock skew. The source of time
typically comes from an
[atomic clock](https://en.wikipedia.org/wiki/Atomic_clock) that is so accurate
that it will drift by less than 1 second in over 211 million years! These
devices are known as Stratum 0 devices. A Stratum 1 server connects directly to
a Stratum 0 device to keep its time synchronized within a few microseconds.
Stratum 2 devices synchronize their time with one or more Stratum 1 devices,
and so on. [^ntp]

[![Network Time Protocol Strata](/assets/img/ntp_clients_and_servers.svg){:height="420px" width="470px"}](https://en.wikipedia.org/wiki/Network_Time_Protocol#Clock_strata)

*Image courtesy of [Benjamin D.  Esham](https://en.wikipedia.org/wiki/Network_Time_Protocol#/media/File:Network_Time_Protocol_servers_and_clients.svg)*

Unless you have your own atomic clock, your server, notebook, or desktop
computer will usually fetch time from one or more Stratum 3 (or above) servers
(usually only one, but can sometimes be an odd number of at least three). It
will periodically fetch time and will then synchronize the system clock in one
of two ways: jam syncing and slewing. In a jam sync, the system clock is
abruptly forced to match the time received by NTP. This corrects the clock as
quickly as possibly, but can wreak the most havoc. A jam sync does not care
which direction the clock is moving and may adjust the time backwards,
potentially breaking many applications (or even your build if it relies on
timestamps to know what needs to be built). A slew adjusts the clock very
slowly, usually no more than 500 ppm (or 500 microseconds per second). Although
much more courteous than a jam sync, corrections to the system clock will take a
long time (e.g., it will take about 14 days to correct a 10 minute drift). A
slew is achieved by telling the kernel to adjust the system clock by an
additional, but small, amount every time tick. For example, if the system clock,
which has fallen behind, will be slewed at a rate of +500 ppm with an interrupt
timer frequency of 1 kHz, the kernel would adjust the system clock by 1,000,500
ns each timer tick (instead of 1,000,000 ns).

Once the system clock has been corrected, NTP may take action to prevent further
drift. By measuring the drift the unadjusted system clock has with the NTP
source, the kernel will be instructed to accommodate for this drift. This is
very similar to the slew adjustment, but is intended to keep the system clock
from drifting away from the correct time as opposed to bringing the system clock
closer to the correct time. For example, if NTP measures the drift to be +14 ppm
with an interrupt timer frequency of 100 Hz, the kernel will adjust the system
clock by 9,999,860 ns each timer tick (instead of 10,000,000 ns).

The system clock is great for determining the most probable wall clock time,
though it is potentially imprecise when it comes to measuring elapsed time.
Consider the case where you want to measure the duration of an operation. If,
between the captured start and end times, the system clock is jam synced to the
NTP time source by advancing the time 127 seconds, your measurement will be off
by more than 2 minutes! If you’re measuring an operation that takes several
days, this may not matter as much as it would if the operation takes a few
seconds. How can we measure the passage of time more accurately
?
## Precise Measurement of the Passage of Time
In addition to the system clock, modern operating systems also have a monotonic
clock. This clock, as its name suggests, monotonically increases as time
progresses. It is unaffected by leap seconds, jam syncing or slewing by
NTP[^monotonic], or system time updates by an administrator. It is a perfect
choice for measuring durations between two events.

Unlike the system clock, the time reported by the monotonic clock is not based
on a globally recognised reference. Instead, time is usually reported as the
number of units (usually microseconds or nanoseconds) since some arbitrary
instant. Since each machine may use a different instant as a reference, times
from a monotonic clock from one machine cannot be used with times from another.
The time reported by this clock is intended to be used when calculating the
duration between two instants within the same application. Like the system
clock, the monotonic clock advances in real time (i.e., the clock will advance
one second every second).

Some operating systems provide other clocks that are not based on real time. For
example, POSIX-compliant systems may provide a clock that measures the amount of
CPU time used by a process or the calling thread, pausing whenever the current
process or thread is suspended (respectively). This makes it possible to measure
the amount of time used by the process (which should be unaffected by other
processes within the system). Since this is essentially measuring CPU cycles,
this type of clock can be useful in measuring the efficiency of an
implementation.

---

## Leap Seconds and Time Zones and Calendars (Oh My!)

Measuring the passage of time for the wall clock is one thing; representing and
displaying wall clock time is a whole other kettle of fish. Although the
Gregorian calendar is predominant in the Western world, there are other
calendars that are still in common use (such as the
[Hijri Calendar](https://en.wikipedia.org/wiki/Islamic_calendar) or

[Ha-Luah ha-Ivri](https://en.wikipedia.org/wiki/Hebrew_calendar)). Even with a
single calendar, there is no escape from the complications caused by time zones
and leap seconds. Politics has a heavy influence on the choice of Calendar and
time zone(s). For example, in 2007 the United States (and Canada)
[changed the time of year Daylight Saving Time was observed](https://en.wikipedia.org/wiki/Energy_Policy_Act_of_2005#Change_to_daylight_saving_time).

Modern operating systems address these issues by explicitly ignoring them and
using a calendar-agnostic representation of time, usually by measuring the
number of units (seconds, milliseconds, microseconds, etc.) since some arbitrary
instant in time (the epoch). In such systems, instants prior to the epoch are
represented by negative numbers and instants following the epoch are represented
by positive numbers. The most common implementation is the UNIX timestamp, which
is measured in seconds with an epoch of 1970–01–01T00:00:00Z in the
[Gregorian Calendar](https://en.wikipedia.org/wiki/Gregorian_calendar). Another
prevalent, but less known, implementation is NTP, which counts 1/2<sup>32</sup>
seconds with an epoch of 1900–01–01T00:00:00Z in the
[Gregorian Calendar](https://en.wikipedia.org/wiki/Gregorian_calendar). For
example, the lunar module touched down on the moon during the
[Apollo 11 mission](https://en.wikipedia.org/wiki/Apollo_11) at precisely
-14,182,916 UNIX time (2,194,805,884 NTP time).

Calendars help solve a labelling problem. It's much easier for us humans to say
“Meet me at the *Cafe 80s* at 1PM next Wednesday” than it is to say “Meet me at
the *Cafe 80s* at 1,445,457,600.” Since our labels were chosen long ago, it's
often hard to separate the two. Time operations involving calendar units can
only be made in context of a calendar. When adding a time unit of variable size,
we often expect it to behave sensibly. Adding a month usually means adding the
appropriate number of days so that the day of the month doesn't change. How many
days worth of seconds need to be added? It's likely somewhere between 28 and 31
(inclusive), but which is it? Even if the number of days is known, the number of
seconds is still context dependent. What if the day's worth of seconds being
added crosses the transition to or from the observance of Daylight Saving Time?
Do we need to add 82,800 s, 86,400 s, or 90,000 s? Calendar-based timestamps can
even be ambiguous. Does “2016–11–06 01:30” in the US Pacific timezone refer to
1,478,421,000 (the first 01:30) or 1,478,424,600 (the second 01:30)? These sort
of complications are enough to make your head spin.

Up until now, we've been able to completely avoid the controversial topic of
[leap seconds](https://en.wikipedia.org/wiki/Leap_second). These pesky critters
are inserted into (or removed from) seemingly random places to account for the
drift between the definition of a day (86,400 s) and the length of time it takes
the earth to make one revolution about its axis
([mean Solar Day](https://en.wikipedia.org/wiki/Solar_time) — as opposed to the
[Sidereal Day](https://en.wikipedia.org/wiki/Sidereal_time)). Without going into
detail about deciding when to add (or remove) leap seconds, it's interesting to
note how they affect our clocks. When a leap second is added (or removed), the
addition (or removal) always happens at the end of the day (UTC), with the label
23:59:60 being added (or 23:59:59 being removed). In UNIX time, leap seconds are
treated as if they didn't occur. The last leap second occurred on June 30, 2015.
Both 2015–06–30T23:59:59Z and 2015–06–30T23:59:60 had a timestamp of
1,435,708,799. Some systems will stretch one second out, others will slowly slew
the clock during the day. If a leap second were ever removed, UNIX timestamp
would simply skip over the leap second.

At this point, you might be thinking that UNIX time is heavily dependent on the
second being the base unit of time. If we were ever to switch to
[Decimal Time](https://en.wikipedia.org/wiki/Decimal_time), with its shorter
seconds, would we be doomed? Not necessarily. Time could still be kept using
UNIX timestamps, with conversions between standard seconds and decimal seconds
being made much like conversions between °C, °F, and K for temperature.

---

## Stop! Java Time!
Java 8 introduced a new time subsystem in the ``java.time`` package. This fixed
many of the issues with the existing time subsystem (such as mutable objects, a
hard-to-use API, limited calendar support, slow updates to handle calendar
changes, etc.) that drove users to third party packages such as the
[Joda time library](http://www.joda.org/joda-time/). Although Joda was a huge
improvement over the earlier Java time subsystem, it still suffered from some
poor design choices. Most notably, its timestamps had an associated timezone. If
two timestamps referred to the same instant in time but where in two different
time zones, they would not be considered equal. (Things were made worse by
adding an
[``isEqual()``](http://www.joda.org/joda-time/apidocs/org/joda/time/ReadableInstant.html#isEqual-org.joda.time.ReadableInstant-)
method that ignored timezone. Talk about
[astonishing](https:./medium.com/@kralka/astonishing-isnt-it-f5256d501025)!)

Java 8's time package addressed the *an-instant-has-a-timezone problem* by
making its
[``Instant``](https://docs.oracle.com/javase/8/docs/api/java/time/Instant.html)
timezone free. However, it still conflates calendars with measuring time by
attaching a timezone to its
[``Clock``](https://docs.oracle.com/javase/8/docs/api/java/time/Clock.html)
implementation. Clocks should not have time zones. Time zones are a
labelling/presentation construct.

---

Java 8's ``Clock`` class can be used to measure time using the wall (or system)
clock.

~~~ java
Clock clock = Clock.systemUTC();
Instant instant = clock.instant();
~~~

The system clock is effectively a wrapper around ``System.currentTimeMillis()``,
but can easily be stubbed out during unit tests.

Java 8 provides a few other clocks that can be useful. The
"[fixed](https://docs.oracle.com/javase/8/docs/api/java/time/Clock.html#fixed-java.time.Instant-java.time.ZoneId-)"
clock will always report the same time, which can be useful for repeatable unit
tests.

~~~ java
Instant moonLanding = Instant.ofEpochSecond(-14_182_916);
Clock clockFixedAtMoonLanding = Clock.fixed(moonLanding);
~~~~

The "[tick](https://docs.oracle.com/javase/8/docs/api/java/time/Clock.html#tick-java.time.Clock-java.time.Duration-)"
clock will truncate a reference clock to an arbitrary resolution. This can be
used to report the current time, rounded down to the nearest minute, hour, day,
year, etc. (and is one of the reasons why a clock needs an associated timezone).

~~~ java
Clock minuteTicker = Clock.tick(Clock.systemUTC(), Duration.ofMinutes(1));
~~~~

or more simply:

~~~ java
Clock minuteTicker = Clock.tickMinutes(ZoneOffset.UTC);
~~~

---

Java's [``System.nanoTime()``](https://docs.oracle.com/javase/8/docs/api/java/lang/System.html#nanoTime--)
is available for reading the system's monotonic clock. This is perfect for
measuring the passage of time (e.g., how long it took for a call to execute).
This being a static function, it is harder to stub out for unit tests.
Fortunately, Guava provides a
[``Ticker``](https://google.github.io/guava/releases/19.0/api/docs/com/google/common/base/Ticker.html)
for accessing monotonic clocks, along with other utility classes (such as
[Stopwatch](https://google.github.io/guava/releases/19.0/api/docs/com/google/common/base/Stopwatch.html))
for easier use.

~~~ java
Stopwatch stopwatch = Stopwatch.createUnstarted(ticker);

doSetup();
stopwatch.start();
doSomething();
long micros = stopwatch.elapsed(TimeUnit.MICROSECONDS);

System.out.println(format(&quot;doSomething() took %s µs&quot;, micros));
~~~

---

[^sources]:
    This has been greatly simplified. It does not take into consideration things
    such as ["tickless" kernels](https://www.quora.com/What-is-a-tickless-kernel),
    missing interrupts, or the various timer sources that are available (such as
    [HPET](https://en.wikipedia.org/wiki/High_Precision_Event_Timer) or
    [TSC](https://en.wikipedia.org/wiki/Time_Stamp_Counter)).

[^ntp]:
    Again, this has been greatly simplified. Stratum n servers will often query
    multiple Stratum n-1 servers to separate the *false tickers* (those with
    inaccurate time) from the *true chimers* (those with accurate time). The
    algorithms for doing it are complicated and well beyond the scope of this
    post.

[^monotonic]:
    On some OSes, the monotonic clock may be affected by NTP slewing.  This
    makes sense when the adjustment is to keep the clock from drifting, but less
    sense when the adjustment is intended to correct an already drifted system
    clock.
