---
layout: post
title:  Awkwardnessful
date:   2016-08-07
tags:   [Software Engineering, Programming]
---

Several years ago, my wife and I picked up and moved across the country. We
bought a new house with a beautiful view in a quiet neighborhood. It was
fantastic. It was perfect. Well, almost. The previous owners had installed an
over-the-toilet cabinet. Poorly. The cabinet wasn't level and would lurch
forward when attempting to open its doors. This instability bothered me
considerably. Not being one to shy away from a small DIY project, I took on the
task of leveling the cabinet and correctly fastening it to the wall. Armed with
a 2-foot level, a bundle of shims, and my wits, I dove in.

I wiggled the cabinet to see which way it was leaning. Forward and to the right,
it seemed. I stabilized the cabinet by placing a few shims under the front,
right corner.  That was simple. Happy with myself, I double-checked my work with
the level, only to be disappointed to find the cabinet was now tilting back and
to the left. It obviously needed more shims on the left, in the back. Out came
shim #2. After leveling out the cabinet, disappointment returned following a
stability test; it was worse than when I had started. The front, left corner was
now off the floor. Time for more shims. This continued for the better part of an
hour, adding and removing shims to balance the cabinet. Eventually, the cabinet
was *level enough* and was happy with myself. Taking a step back to admire my
accomplishment, my pride quickly waned as I saw the mess that had become of the
shims. All four corners were raised up at least 1 cm (⅜" for my American
friends) by the shims. Although I had solved the problem, it was messy and not
something I could be proud of. I am sure that any guests would not have notice
the poor workmanship. And if they did, I am sure they wouldn't care. But for me,
I knew I could do better. I knew there was a cleaner solution. I just lacked the
knowledge to do any better.

Several months after the great cabinet leveling disaster of 2011, I was chatting
with my father about various things and my struggles with leveling the cabinet
came up. You see, my father has always been very handy around the house. He had
singlehandedly finished the basement in our family home, rebuilt the back deck
and fence, built cabinets, and installed hardwood flooring. He really knew his
stuff! After hearing about my troubles he chuckled and went on to explain how to
easily level a cabinet on an uneven surface:

1. Use a level to find the highest corner of the cabinet; the starting corner.
2. Pick one of the adjacent corners (either will do) and raise it until it is
   level with the starting corner.
3. Raise the other adjacent corner until it is level with the starting corner
   (and by transitivity, the first adjacent corner).
4. Raise the remaining corner (diagonal to the starting corner) until it is
   level.

Wow! So simple! And it can even work if you have oddly shaped furniture. Simply
start with the highest vertex and walk your way around the object. Leveling
first on the left, first on the right, second on the left, second on the right,
and so on. Now that I have shared this tip with you, my apologies if you now
have some shelves in your garage to straighten!

---

I was reminded of this story a few weeks back while reviewing some code. The
author had written some functionally-correct code that was more awkward than it
needed to be. The requirements were simple:

> Given a list of `Foo` objects, convert them to `Bar` objects and group by
> type.

The resulting code was similar to the following:

~~~ java
Map<String, List<Bar>> extractValuesByType(List<Foo> foos) {
  Map<String, List<Bar>> valuesByType = new HashMap<>();
  for (Foo foo: foos) {
    List<Bar> valuesForType = valuesByType.get(foo.getType());
    if (valuesForType == null) {
      valuesForType = new ArrayList<>();
      valuesByType.put(foo.getType(), valuesForType);
    }

    valuesForType.add(barFromFoo(foo));
  }
  return valuesByType;
}
~~~

There's nothing ***wrong*** with this code. It does exactly what it needs to do.
No more. No less. Several years ago, I probably would have written this function
exactly as it was presented to me. But not any longer. If asked to implement the
same requirements, I would probably produce something similar to the following:

~~~ java
ListMultimap<String, Bar> extractValuesByType(List<Foo> foos) {
  ListMultimap<String, Bar> valuesByType = ArrayListMultimap.create();
  for (Foo foo: foos) {
    valuesByType.put(foo.getType(), barFromFoo(foo));
  }
  return valuesByType;
}
~~~

Woah! That's is significantly more succinct. The 
[`ListMultimap`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ListMultimap.html)
implementation took care of all the messiness around adding the first `Bar` of a
given type. And since
[`Multimaps.asMap()`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/Multimaps.html#asMap%28com.google.common.collect.ListMultimap%29)
exists, callers can still get a `Map<String, List<Bar>>` view of the returned
map.

*What's changed?* *Have I learned more?* *Do I know more?* *Have I expanded my
toolkit?* *Am I more familiar with available libraries?* *Do I know more about
the classes they provide?* *Am I just more familiar with the APIs?* All of these
are true, but there is something more subtle to consider. Knowing more about
what's available definitely helps, but only in figuring out ***how*** something
might be implemented. *How should I implement this?* *What classes do I use?*
*Which libraries are most appropriate?* These kinds of questions are easy to
answer when you've amassed a large repertoire of tools in your belt. What's
missing is the ***why***. *Why is this implementation awkward?* *Should I look
into whether there is a simpler implementation?*

---

## Knowing how Something can be Better

There is really only one way to improve your answers to the *how* question.
*Experience*. In particular, *breadth of experience*. It's hard to describe
something unfamiliar. You can expand your experience by doing and learning from
your successes and failures. But just knowing that something exists is often
enough. Knowing that Guava has several
[collections](https://github.com/google/guava/wiki/CollectionUtilitiesExplained)
classes is often enough to know where to look if you find yourself building a
complicated collection. (`Map<String, Map<Int, Foo>>` anyone? It looks
`Table<String, Int, Foo>` is what you're looking for.)

### Read Code
I have learned more about good software engineering practices by reading others'
code than any other method.

Most of us write code for projects to which many engineers have contributed,
making it difficult to know about all of the nooks and crannies. Although there
may be a lot of overlap in knowledge (e.g., the core components of a project),
there will always be someone else who knows things you don't (no matter how much
you know). By reading code written by others, you can expand your knowledge to
include something they know. I've used this to learn about everything from
unknown functionality in our project (*Nice! Someone wrote a builder to simplify
the creation of new `Widgets`*) to features of a library I was unaware of
(*zOMG! I never knew you could do that with Guice!*).

Perhaps even more important than expanding your knowledge, reading code written
by others can teach you how to write *comprehensible* code. As engineers, we
spend a disproportionate amount of time ensuring our code is *functionally
correct* at the expense of ensuring our code is easily understood by our fellow
engineers.  *Why is this important?* To answer that question, ask yourself
this question: *For whom am I writing this code?* And by that, I don't mean who
is going to use the feature you're writing. I mean, who is going to *read* it,
*debug* it, *maintain* it? This is your audience. Write for them. Don't
forget, *future you* is in your audience. Be kind to them. Since such a big
part of coding is writing *comprehensible* code, if your code is
incomprehensible, you've mostly failed, even if it is the most efficient,
*functionally correct* implementation possible.

Reading code written by someone else that you find easy to understand is more
likely to be understandable (it is, after all, already understandable by at
least two people). By reading this code, you can easily see what works (the
stuff you can easily grok) and what doesn't (the stuff you have to devote
significant effort to understand how it works when it appears that it could not
possibly).  This has helped me a lot in my career. I have had a tendency to
over-engineer and over-generalize my solutions. I have a tendency to create
clever abstractions that are extensible across many axes. I suspect both are
very painful to review.  I've acknowledged this problem and am working on
improving it.

> Hi, my name is Michael and I'm an engineer.
>
> Hi Michael!
>
> As of today, I haven't written an over-generalized abstraction in 3 months ...

If I had to hazard a guess, I would say between 10-25% of the effort put into
writing high-quality code goes into making the code *functionally correct*. The
remaining 75-90% of the effort goes into making it comprehensible. It is not an
easy task, so don't get discouraged when you struggle with it. I still struggle.
A lot.

There are lots of ways to read more code. The one that I've found the most
successful is to read change listings (code reviews). I make a habit of reading
all code reviews generated by my team. When I read reviews (unless I'm an owner
who needs to approve the review) I focus on reviewing at a high level. I'm
looking to see how the code is structured, what libraries are being used, and
readability.  Most importantly, I read over the review *after* it has been
submitted. This allows me to read over the review comments and learn from my
colleagues. You'd be surprised how much you can learn this way. This method has
the added benefit of keeping up with all the changes being made (what new
functionality is available, what new abstractions are being introduced, what has
been deprecated, etc.).

If reading all reviews is too much, start off by selecting someone who writes
good code. Read all of their reviews. Include reviews of code they have reviewed
and see what types of things they comment about. Don't do this for too long,
though. You'll be limiting yourself to what they know and how they write (or
don't write) comprehensible code.

### Ask an Expert

As engineers, we like a challenge. We thrive on it.  Nothing gives us quite as
much pleasure as figuring something out and getting our code to work. As
discussed above, it's not always enough to get things working. We have to also
think about how comprehensible our code is.  At the same time, we are not social
animals. We often prefer to figure things out on our own. It's how we learn. We
don't like getting disturbed when we're *in the zone*, so we tend not to disturb
others. I think that is at the loss of the entire team.

The [cost of change curve](http://www.agilemodeling.com/essays/costOfChange.htm)
can be very steep. Waiting too long in the process to "figure things out" can
result in expensive mistakes. Over the years, I have seen that engineers tend to
wait until *code review time* to solicit feedback from experts. After figuring
out a solution to a tough problem, they continue on until the code is
functionally correct. This can be a costly mistake. Instead, once you think
you've found a solution, ask the expert. It doesn't need to be formal. Shoot off
an email or an IM:

> Hey Bob, I'm trying to interface with the `Widget` and was planning on
> using the `foobar` to calculate costs. I'm passing in blahty, blah, blah,
> blah. Does this sound reasonable?
>
> Hrm. `foobar` doesn't account for `whizbang`, so it might not give the results
> you're after. It wasn't really designed for that purpose. Have you spoken with
> Alice, she recently had to do the same thing. I think she used the
> `blartyblar`.

In scenarios like these, this quick five minute conversation can save a lot of
redesign work.

---

## Knowing when something can be better

Expanding your experience seems like it's a sure-fire way to learn when
something can be better. Just like my familiarity with leveling a rectangular
cabinet on an uneven floor has allowed me to level any furniture,
[Guava Collections](https://github.com/google/guava/wiki/CollectionUtilitiesExplained)
has allowed me to recognize when code or interfaces would be much less awkward
if reimplemented using Guava collections. Furthermore, I can apply this
knowledge and recognize that when doing something *unusual* with nested
collections, there might already exist an appropriate abstraction. For example,
[Guava Collections](https://github.com/google/guava/wiki/CollectionUtilitiesExplained)
do not appear to support something that behaves like a
`SortedMap<Key, SortedSet<Value>>`. However, the 
[`MultimapBuilder`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/MultimapBuilder.html)
will allow you to create a `SetMultimap` that observes the behavior.

There is something to be said about applying previous knowledge to new
situations. Being able to see parallels with what you already know and a current
situation is an invaluable skill. As your knowledge base grows, the number of
scenarios to which you can apply your knowledge also grows. However, you're
still limited by your experiences. This is how I have spent the majority of my
career. I've used the techniques described above to expand my knowledge base.

This all reminds me of the idea of [code
smell](https://en.wikipedia.org/wiki/Code_smell). Although this idea is a bit
different, I think the analogy still applies. It's not so much knowing that a
certain library feature exists that would improve the quality of the code as it
is realizing that such a feature is *likely* to exist (and if it doesn't, it
*should*). The fact that this feature is not being used *smells fishy*,
warranting further investigation.  For example, before I started writing this
post, I didn't know that you could create a
[`Multimap`](https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/Multimap.html)
with sorted keys and values. I wasn't sure if it existed, but I thought that it
*should* exist (because it's a natural extension of the
[`SortedSetMultimap`](https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/SortedSetMultimap.html)
functionality). When I encounter code that would benefit from such a collection,
I would first look into whether such an implementation existed.

I've been thinking a lot about how to recognize opportunities for improvement
when one lacks directly related experience. At first the concept seems
completely clear in my head, but as I try to articulate these ideas into
coherent sentences, it disappears. It's like the [grid
illusion](https://en.wikipedia.org/wiki/Grid_illusion); no matter how hard you
try to look directly at a black dot, it disappears. So what do you think?  I'd
be interested in hearing your thoughts.
