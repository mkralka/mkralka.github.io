---
layout:   post
title:    Rows by any other name...
subtitle: Naming things is hard.
date:     2016-06-06
tags:     [Software Engineering, Software Design, Programming, API Design]
---

There are two consumers your code: other humans and compilers. Humans are a
finicky bunch. We have a hard time keeping more than about [4 things in our head
at any one time](http://www.livescience.com/2493-mind-limit-4.html), we use our
personal experiences to influence our decisions, we take shortcuts in our
thinking using heuristics we have learned (for better or worse), and we make
tons of mistakes. Compilers are finicky, too; they are **very** unforgiving of
us humans when we violate syntax.

Writing for compilers is easy. Your compiler doesn't care how you've split your
logic across your class hierarchy, how complex your method is, or if your
parameters have meaningful names; as long as your code is syntactically correct,
your compiler will be happy. By comparison, humans care very much for these
things.

Writing for humans is hard. Really hard. Truly hard.

Since humans spend a disproportionate amount of energy with code and are the
ones tasked with making changes it, we should put almost all of our effort into
making our code easily read by our fellow humans.

There are several things that can be done to improve the readability of code. Of
the many books, blog posts, and Wikipedia articles that have been written on the
subject, many focus on the structure of your code. *Use these design patterns.*
*Use this coding style.* *Refactor.  Refactor. Refactor.* One concept that often
goes overlooked, perhaps because it **is** so hard, is the importance of finding
a good name for the things we create, be it a class, method, variable, field, or
parameters.

> “There are only two hard things in Computer Science: cache invalidation and
> naming things.”
>
> — Phil Karlton

---

## It truly is hard to name things

Outside of software engineering, people struggle to name things. It's such a
challenge that there is lots of help out there for naming your
[business](https://www.entrepreneur.com/article/76958),
[blog](http://www.successfulblogging.com/how-to-come-up-with-a-blog-name/),
or arbitrary
"[thing](http://www.alexandrafranzen.com/2014/04/21/the-ultimate-guide-to-naming-your-thing/)".
Naming things poses a unique challenge for software engineers. We are constantly
creating new packages, classes, fields, methods, parameters, and variables. On a
given day, we may name 100 new things and every name counts.

One of the reasons why I think it is so hard to come up with good names for
things is that there isn't usually an obvious analogy between a real-world
concept and the abstractions that we are implementing. If an analogy exists, it
may be hard to realize or have a tenuous relationship with the abstraction. When
an analogy does exists, there are common terms that we can use when naming the
nouns and verbs in our system. If you're creating a website for an online
retailer, the names for many of your nouns and verbs will come from those used
when describing brick and mortar retailers. Why would you name the object that
holds the things you want to purchase anything other than a *cart* or *basket*?
Doing so will only confuse the readers of your code.

Well named classes and methods can have a surprisingly positive impact on the
readability of not only your code but also that of your users.

---

## Naming things, like a champ!

When trying to form a suitable name, there are some guidelines that can steer
you in the right direction.

### Be Self Explanatory

The purpose of your class or function should be obvious from its name. Your user
should not have to read the documentation to get a good idea of what your class
does and how it should be used. That's said, you can take things too far in the
other direction; creating awkwardly long names.

Consider some code from the database implementation. You're working on
persisting the data to disk and you come across the following code:

~~~ java
private void init() {
    for (File file: files) {
        if (!file.check()) {
            initialize(file);
        }
    }
}
~~~

What is `check()` checking? What about the file is `initialize()` initializing?
I could guess, but I suspect that my confidence in my guess would be low enough
that I would end up looking at the documentation. Contrast that experience to
the one of encountering the following code:

~~~ java
private void generateMissingTables() {
    for (TableFile table: tables) {
        if (!table.exists()) {
            initializeEmptyTable(table);
        }
    }
}
~~~

Without reading any documentation, you can easily see that this method will
create empty table files for any table whose file doesn't exist.

### Be Consistent

Use the same names for the same "things" and don't use them for other "things".

When choosing a name to represent an abstraction there are often many choices.
It doesn't often matter which name is chosen, provided that once a choice is
made, it is used consistently to represent that abstraction. For example,
consider an interface to a remote storage system. There are many verbs that can
be used to represent the retrieval an item from the storage system, such as
`fetch()`, `get()`, or `retrieve()`. If you chose `fetch()`, you should use that
verb whenever retrieving an item from remote `storage()`.

~~~ java
interface RemoteStorage<K, V> {
    Optional<V> fetch(K key);
    Conditional<V> fetchIfNewerThan(K key, Timestamp instant);
    Conditional<V> fetchIfDifferentThan(K key, V value);
}
~~~

Using multiple verbs for the same action will increase the surface area of your
API, requiring your users to learn more to use your package.

When a name has been chosen for an abstraction, you should avoid using that name
for other, different abstractions. When encountering similarly names classes or
methods, users tend to assume they have the same behaviors. Having to reuse the
same name for different abstraction isn't the end of the world, but it may add
confusion. For example, consider a cluster manager for a distributed
application. If the term *Node* was chosen to refer to the *physical* host on
which the applications runs, it would be unwise to reuse *Node* to also refer to
an application instance. Instead, choose a different term (such as *Instance*).

Be wary of using synonyms for slightly different behavior. For example, if you
have `remove()` and `delete()` methods that have different behavior, your users
are going to be confused and may unintentionally choose the wrong one.

~~~ java
/**
 * Remove the first instance of {@code value} from this collection.
 *
 * @param value The value to remove.
 * @return {@code true} if the collection was modified.
 */
 boolean remove(V value);

/**
 * Remove the all instances of {@code value} from this collection.
 *
 * @param value The value to remove.
 * @return {@code true} if the collection was modified.
 */
 boolean delete(V value);
~~~

Instead, use a name derived from the original term with appropriate color to
disambiguate.

~~~ java
/**
 * Remove the first instance of {@code value} from this collection.
 *
 * @param value The value to remove.
 * @return {@code true} if the collection was modified.
 */
 boolean remove(V value);

/**
 * Remove the all instances of {@code value} from this collection.
 *
 * @param value The value to remove.
 * @return {@code true} if the collection was modified.
 */
 boolean removeAll(V value);
~~~

### Find an analogy
When the right analogy can be made between your abstraction and some real-world
object, universally agreed upon terms can be used. Anyone familiar with the
analogy can easily understand what is possible with your abstraction and what
the terms in your abstraction mean.

For example, you're working on a read-only store and decide to use a library
analogy to describe your abstraction.

* *librarian* may refer to the entity that manages what data is available in the
  data store.
* *title* may refer to the smallest unit of data that the librarian can add or
  remove from the data store.
* *edition* may refer to the chronological version of a title.

Be careful when choosing an analogy. Avoid anything but the most common
analogies or those used in the domain of your application. An analogy within an
esoteric field will likely add to the confusion.

### Trust the Warning Signs
f you are finding it hard to come up with a succinct name for your class or
function, consider refactoring. Your implementation may be doing too much or too
little.

---

Remember not to get hung up trying to come up with the perfect name. A great, or
even good, name will provide huge gains to the usability and understandability
of your code. Your goal should be to allow client code to read less like
computer code and more like prose.

~~~ java
if (engine.currentTemperature() > MAXIMUM_SAFE_ENGINE_TEMPERATURE) {
    dashboard.illuminateIndicator(CHECK_ENGINE);
}
~~~
