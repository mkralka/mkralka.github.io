---
layout:   post
title:    Astonishing, isn't it?
subtitle: The principle of least astonishment
date:     2016-06-13
tags:     [Software Engineering, Software Design, Programming, API Design]
---
When designing *things* to be used by humans, it is very important to remember
that users are part of the system and when your *thing* doesn't behave as the
user expects, they are astonished. Astonished users have a tendency to blame
themselves for misusing your *thing* and may end up feeling stupid for
misunderstanding how your *thing* actually works. Unless you are a
*schadenfreudesüchtig*[^schadenfreudesuchtig], you probably want to avoid making
your users feel stupid.

Lots of things can astonish users of your *thing*, especially users who are
unfamiliar with it. When I first started programming in Java, I was unfamiliar
with Java's memory model and made a ton of mistakes. Luckily for me, I was
surrounded by exceptional engineers who were patient with me and taught me about
some of Java's dark corners. Over time, the astonishment faded as I became more
familiar with Java. It's not that Java's memory model was "obviously designed
by morons!", it was different enough from my previous experience with C/C++ that
I had to build a new mental model for writing Java code. This is not the kind of
astonishment I'm talking about. New, innovative *things* are, by definition,
different than existing *things*, leading to unfamiliarity. This is much
different than the astonishment from using something familiar and it does not
behave as expected.

---

Keeping users from being astonished leads to an important principle in usability
design:

> The principle of least astonishment
>
> People are part of the system. The design should match the user's experience,
> expectations, and mental models.

(From *Principles of Computer System Design: An Introduction* by *Jerome H.
Saltzer* and *M. Frans Kaashoek*.)

POLA applies to all areas of interface design, from the tangible (e.g., a door)
to the abstract (the UI of your favorite app). As software engineers, we tend to
associate POLA with UI design, forgetting that all of the software we write has
users. Your users may be limited to your team (in the case of an internal
interface realizing an implementation detail of the project for which your team
is responsible), other teams within your company (in the case of an internal
library for which your team is responsible), or other engineers around the world
(in the case of an open source library that your team maintains). Don't be
fooled; not all users are alike. Users of [Twitter's REST
API](https://dev.twitter.com/rest/public) are very different than the users of
[Twitter's UI](https://twitter.com/), but the need to minimize astonishment in
both groups remains. Users of the REST API come with different experience,
expectations and mental models, so the REST API should cater to these
expectations.

---

Serendipitously, a few days ago [Let's Encrypt](https://letsencrypt.org/)
experienced the business-end of some [astonishing
behavior](https://bugs.python.org/issue10839) resulting in them apparently
[leaking 7618 email addresses to the owners of the same 7618 email
addresses](https://community.letsencrypt.org/t/email-address-disclosures-preliminary-report-june-11-2016/16867).
[Tom van der Woerdt](https://twitter.com/TvdW) quickly [pointed
out](https://twitter.com/TvdW/status/741481798014664704) the "feature", with
some example python code, that resulted in the leak. What he posted was similar
to the following:

~~~ python
from email.MIMEText import MIMEText
m = MIMEText("Hello World!")
m['To'] = 'address1@example.com'
m['To'] = 'address2@example.com'
print m
~~~

generated a message with two `To` headers:

~~~
From nobody Sun Jun 12 15:18:28 2016
Content-Type: text/plain; charset="us-ascii"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
To: address1@example.com
To: address2@example.com

Hello World!
~~~

Why is this a problem? Consider the context under which the email package was
likely used by Let's Encrypt: sending a newsletter to all 383,000 of their
users, with each user receiving an identical email. It's not unreasonable to
create a template *message*, then, for each recipient, change the `To` header
and deliver the message:

~~~ python
from email.MIMEText import MIMEText

addrs = ['address1@example.com', 'address2@example.com']

m = MIMEText('Hello list subscriber!')
for addr in addrs:
    m['To'] = addr
    print m
~~~

Any expert python hacker, unfamiliar with the email package, would likely expect
this to work as intended. They would (rightfully) expect the 'To' header to be
replaced during each iteration of the loop. In python, as with many other
languages, indexing into a collection uses brackets to denote the index. As an
[lvalue](https://en.wikipedia.org/wiki/Value_%28computer_science%29#lrvalue),
the collection is updated by replacing the value stored at the index (if any)
with a new value. As an
[rvalue](https://en.wikipedia.org/wiki/Value_%28computer_science%29#lrvalue),
it evaluates to the value stored at the index.

Armed with these expectations, the expert python hack would expect the following
emails:

~~~
From nobody Sun Jun 12 15:22:54 2016
Content-Type: text/plain; charset="us-ascii"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
To: address1@example.com

Hello list subscriber!
~~~

and

~~~
From nobody Sun Jun 12 15:22:54 2016
Content-Type: text/plain; charset="us-ascii"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
To: address2@example.com

Hello list subscriber!
~~~

Much to their surprise, the second email message would include both email addresses:

~~~
From nobody Sun Jun 12 15:22:54 2016
Content-Type: text/plain; charset="us-ascii"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
To: address1@example.com
To: address2@example.com

Hello list subscriber!
~~~

Is this a bug? What's going on here? Sadly, this is not a bug. The email package
is designed this way. On purpose. The behavior is
[documented](https://docs.python.org/2.7/library/email.message.html#email.message.Message.__setitem__).

If you're familiar with [RFC-822](https://www.ietf.org/rfc/rfc0822.txt)
and [RFC-2822](https://www.ietf.org/rfc/rfc2822.txt), the behavior almost makes sense. Since the
standard allows for a header to appear multiple times, it's expected that the
email package supports this feature. By coopting syntactic sugar provided by
python, this API turned a "set" into an "append", unsuspectingly. Instead, an
API similar to the following would be much clearer:

~~~
class MIMEHeaders(object):
    # Other stuff

    def set(self, name, value):
        # Replace all existing headers of type "name" with one with
        # value "value".

    def add(self, name, value):
        # Add a new header of type "name" with value "value", while
        # retaining existing headers of type "name".

class MIMEText(object):
    def __init__(self):
    self._headers = MIMEHeaders()

    @property
    def headers(self):
        return self._headers
~~~

---

*How can I minimize the astonishment I inflict on my users?* Glad you asked. A
lot of this boils down to one easy to understand (but hard to implement)
principle:

> Do every reasonable thing in your power to meet the expectations of your
> users.

Here are just a few reasonable things you can do.

Ensure behavior is obvious, consistent, and predictable
: "Obvious" tends to be subjective; something that is obvious to a python hacker
  may not be to a Java guru or C++ magician. Be consistent with the platform for
  which you are writing and the domain in which you are working. Consider the
  other classes/methods of your package.

Avoid unexpected side effects
: Separate out queries from state changes. A mutator that says it does one thing
  should do only that one thing and not some other, unrelated thing.

[Naming is important](rows-by-any-other-name)
: Names should clearly communicate what a class/method does. Be consistent with
  similar classes/methods so your new class/method feels familiar to your users.

Stuff happens
: It's not an anomaly to fail when an irrecoverable error condition is met.
  However, it is surprising when something doesn't fail when it should. Consider
  a deserializer that returns half deserialized objects when a deserialization
  error occurs. It may seem like this is being nice to the user because they do
  not have to deal with the error, but when they treat partial data as if it
  were complete, debugging will likely be a nightmare.

Handle ambiguity sensibly
: When behavior is ambiguous, choose the option that is least likely to surprise
  the user and not the one that is most intuitive based on the implementation.
  (Implementation details are subject to change; what's intuitive now may not be
  intuitive after your next refactoring.)

---

Remember, do every reasonable thing in your power to meet the expectations of
your users. They will be happier, more productive.

[^schadenfreudesuchtig]:
    *schadenfreudesüchtig*: (noun) A schadenfreude-seeky. One who has an
    inclination to seek pleasure from another person's misfortune.
