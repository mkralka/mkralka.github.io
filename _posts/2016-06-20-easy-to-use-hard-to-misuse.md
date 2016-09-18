---
layout:   post
title:    Easy to Use, Hard to Misuse
subtitle: No matter how smart or careful your users are, if there is a way to misuse your API, they will find it!
date:     2016-06-20
tags:     [Software Engineering, Software Design, Programming, Java, API Design]
---
When thinking about what makes a *thing* a pleasure to use, we often think of
how *easy* it is to use. Was I able to accomplish my task quickly, without
effort? Was I able to figure out how to use the *thing* in the first place? Did
I spend a significant amount of time reading through documentation to understand
how the *thing* worked before I could use it?

> No matter how smart or careful your users are, if there is a way to misuse
> your API, they will find it!

When designing an API, we tend to put more thought into making it easy to use at
the expense of making it easy to misuse. I suspect this is because it's much
easier to think about *what* the API must do (and by extension, how to make
*that* easy for users) than it is to think about *how* the API could be
misused/abused (and by extension, how users might break the API). Besides,
making things easy to use will attract users. (And we all want more users!) But
don't forget that making things hard to misuse will keep them safe. No matter
how smart or careful your users are, if there is a way to misuse your API, they
will find it!

Many of the strategies for maximizing *ease of use* are broadly covered by
adhering to the *Principle of Least Astonishment*, which I covered in greater
detail [last week](astonishing-isnt-it). If you haven't read it already, you
should do so now. (Don't worry, I'll be here when you get back.  You're done?
Great!) At a high level, ensure that the obvious way to use your API is also the
correct way.

---

Keeping users safe can be a thankless job; it often goes unnoticed. I would be
hard-pressed to recall an anecdote where an API saved me from harm. I cannot say
the same for an API that put me in harms way. APIs that are easy to misuse will
eventually get misused. When you do, you'll acquire a battle scar or two.

A few weeks ago, my team was working on a new feature that would allow our
customers (other engineers) to test certain hard-to-reproduce scenarios in
production. All of our functional testing went smoothly; everything looked
great. We rolled the new build out to canary instances in production, replacing
about 10% of the existing instances with the the new build. After running for
about 30 minutes, the canary instances showed no signs of failure or performance
degradation. We rolled out the new build to the entire cluster. Shortly after
the last instance had been updated, things started to go awry. Latency shot
through the roof; our p99 latency shot up from about 40 ms to well over 150 ms.
After redeploying the new build with the new feature disabled, we began the
postmortem process, digging through our metrics trying to figure out what went
wrong and, more importantly, why the problem didn't appear on the canary
instances.

After a lot of digging, we eventually tracked down the problem to a misuse of
our metrics collection API. By default, all generated metrics are kept in
memory, using a collector that is now mostly used by unit tests. All percentile
metrics are stored as raw values, which we were generating, resulting in an
explosion of memory use. This explosion caused GC thrashing in an attempt to
reclaim memory, triggering a massive CPU overload, causing our latency to
suffer. The in-memory metrics collector has no business being used in
production, but our API made this easy. We have since fixed the API, making it
much harder to store stats in-memory on production instances. We were put at
risk by our API, we paid for it, and we'll likely remember it (at least for the
next few years).

---

### Difficulty of Misuse

When maximizing the *difficulty of misuse* of an API, the goal is to maximize
the probability that users will detect misuse, while minimizing the effort
needed to do so.

#### Let your Compiler be your Guide

The most effective way to maximize the difficulty in misusing your API is to let
the compiler do the work for you. By catching API misuse with the compiler, only
correct use of the API will compile. A perfect example of this was the
introduction of Generics in Java 1.5. Prior to the introduction of Generics,
generic APIs (such as List and Map) were forced to use the most inclusive type
in its API, forcing awkward casts by the caller.

~~~ java
public final class Math {
    public static int sum(List values) {
        int result = 0;
        for (int i = 0; i < values.size(); ++i) {
            int value = (Integer) values.get(i);
        }
        return result;
    }
}
~~~~

OK, that's not **too** bad. But what happens if the caller accidentally calls
with the wrong list?

~~~ java
public void doStuff(List names, List values) {
    for (int i = 0; i < names.size(); ++i) {
        System.out.println(names.get(i) + "=" + values.get(i));
    }
    System.out.println("Total=" + Math.sum(names));
}
~~~

Unfortunately, the compiler would dutifully compile this method and the
unsuspecting user would only find out at runtime that they had made a mistake
when a `ClassCastException` was thrown because of the attempt to cast a `String`
to a `Integer`. Oops!

With the introduction of Generics, this whole class of problem goes away.
Instead, the `sum()` method would not perform a cast and would only accept a
`List` of `Integers`:

~~~ java
public final class Math {
    public static int sum(List<Integer> values) {
        int result = 0;
        for (int i = 0; i < values.size(); ++i) {
            int value = values.get(i);
        }
        return result;
    }
}
~~~

Since the caller would also restrict the type of List objects it accepted, the
misuse of calling `Math.sum()` with anything other than a `List` of `Integer`
objects would fail to compile:

~~~ java
public void doStuff(List<String> names, List<Integer> values) {
    for (int i = 0; i < names.size(); ++i) {
        System.out.println(names.get(i) + "=" + values.get(i));
    }
    // This will not compile. names is a List<String>
    System.out.println("Total=" + Math.sum(names));
}
~~~

#### Break at Runtime

For those cases where using the compiler to catch your users' mistakes is
impractical, detect misuse as soon as possible and break at runtime. This is not
going to catch as many misuses as using the compiler to catch errors since it
requires the offending code path to be executed for misuse to be detected.
However, if you die violently as soon as the misuse occurs and your users have
good test coverage, they will be aware of their misuse and will be able to
correct it.

#### Have Good Documentation

Good documentation is always better than no documentation, which is always
better than bad documentation. If detecting misuse at compile time or runtime is
impractical, as a last resort produce good documentation that describes how not
to misuse your API. Good documentation will hopefully keep your users from
looking at the implementation (which can result in your users depending on bugs
that are present in your implementation).

*Your users like to code; they don't like reading documentation.* Users tend to
read the documentation only after they've gotten into a bind and are desperate
to get out of it. *If* your users read the documentation, they will likely skim
through it, searching for a specific keyword; they will almost certainly miss
the warnings urging them to [steer clear of the
dragons](https://en.wikipedia.org/wiki/Here_be_dragons#Computer_Programming).

---

### What to do when Ease of Use conflicts with Difficulty of Misuse?

It's not uncommon to be faced with a design choice that the goal of *Difficulty
of Misuse* conflicts with the goal of *Ease of Use*. Resist the urge to favor
*Ease of Use* over *Difficulty of Misuse*. Favoring *Difficulty of Misuse* is
often the better choice.

{% include post_figure.html
    url="/assets/img/shift_pattern.svg"
    height="285px"
    width="287px"
    alt="Dogleg Manual Shift Pattern" %}

A concrete example of this from the real world is the [inhibitor or interlock
that is present on some manual transmission
vehicles](https://www.quora.com/How-do-manual-transmissions-prevent-shifting-into-reverse-at-forward-speeds-greater-than-5-mph)
In the most common [shift
pattern](https://en.wikipedia.org/wiki/Gear_stick#Shift_pattern), reverse is
below 5th gear. When in 5th gear, you're likely travelling at highway speeds,
making a drop down into reverse an expensive mistake. Some vehicles have an
inhibitor, which forces you to enter reverse from neutral, making it impossible
to pull straight down from 5th into reverse. Other vehicles have an interlock,
requiring the gear stick to be pushed downward (towards the floor) or a ring
around the stick to be pulled upward before the shifter will slide into reverse.
This interlock makes the gearbox harder to use since it is harder to shift into
reverse. This complication is worth the extra effort because it eliminates a
whole class of errors from accidental (and dangerous) shifts into reverse.
