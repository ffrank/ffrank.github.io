---
layout: post
category: articles
tags: conference log foss openstack
---

A quick review of the first annual OpenStack DACH Day Berlin.

### Some history

OpenStack is the sort of open source behemoth that is constantly buzzing these days.
Apparently, there are two major international conferences taking place each year,
as well as frequent smaller events around the globe. This year's OpenStack October
summit will be in Tokyo, and next year will have one in Austin, TX.

Berlin has had an annual conference for quite a while, but this year shook things up.
Historically, the OpenStack Day used to be organized in conjunction with Linux Tag,
the largest Linux tech conference in Germany. This event did not materialize this year,
however. (Personally I found 2014's proceedings vastly disappointing. I have no
information concerning whether this was a public opinion or if there is a connection
to this year's break in continuity.)

The OpenStack DACH Day is organized by an international nonprofit that includes members
from local players such as [heinlein](https://www.heinlein-support.de/) and
[SysEleven](http://www.syseleven.de/). In the absence of the Linux Tag, they
set out to create an OpenStack conference in its own right. And did they ever.
Here's my thoroughly biased report.

### The venue

I must have passed the [Urania](http://www.urania.de/die-urania/) (German link) hundreds
of times, but never actually visited it. It must have been quite futuristic back in its day.
The building still feels quite modern, even with its age showing here and there.
The common space is nice and open, bright with large windows and convenient snug seating
for networking during lunch.

It's situated in the part of town that I like to think of as the old core, near the
Zoo and Memorial Church. A good place for a conference, alive
and buzzing, a touristic hotspot.

The main track was in a large room that felt like a mixture of a cinema and a lecture hall.
The folding seats have thick padding and movable tables. The other talks were in a much
smaller room with some sixty simple chairs.
Catering and service were more than adequate for this type of event.

### The proceedings

Mark Collier's keynote is impressive and he presents it well. I found it interesting
how it contrasts components that presumably work well and are fairly mature with
more experimental pieces that are available for choosing as well. This is probably
just me - KISS is my highest principle as far as software is concerned - but a product that
consists of more that a dozen movable parts with varying levels of quality and
capabilities suffers a handicap to its appeal in my eyes. The community
and cooperative aspect on the other hand spoke to me quite clearly.

The second keynote was held by some SUSE developers and was meant to showcase some
scenarios without wallowing in too much technical detail. For its good intention,
the presentation didn't really speak to me. A couple of stories going "a customer
was doing X and faced problem Y so we went there and now they are doing it with
OpenStack" didn't really tell me anything. The video presentation by a Telekom
rep, on the other hand, was perhaps a little *too* easy to follow for most in the audience.
Just saying.

I then sat in on a technical presentation about [Rally](https://wiki.openstack.org/wiki/Rally)
and related topics, which I rather enjoyed. It had the kind of "high-level with 
sprinkles of depth" that I had missed in the keynotes. The Ceph introductory talk
was very well done. I then opted to listen to Erkan Yanar sharing his love for
containers again. I had listened to him at Linux Tag and FOSDEM before and I like
the tongue-in-cheek commentary he peppers into his talks. This one had much more
background on Docker and its emerging ecosystem than the on at Linux Tag, giving
a nice summary. Still, in retrospect, I guess I would have gotten more out of
[Monty Taylor's](https://twitter.com/e_monty) OpenStack developer talk.

After using the lunch break to get some background information from organizer
[Robert Sander](inside://twitter.com/gurubert), I went on to listen to community
ambassador Marton Kiss about contributing and what's new with Gerrit (including
delight when I spotted [Spencer's](https://twitter.com/nibalizer) name in some
patch reviews). I heard an Arista rep explaining their take on Software Defined
Networks and enjoyed [Florian's](https://www.hastexo.com/who/florian) overview
of OpenStack deployment scenarios, along with his usual dose of cynical commentary.

My final presentations were a demo of openATTIC, with a neat glimpse into the
darker depths of Ceph operation. Speaking of the abyss - the final talk was a
highlight. Fellow systems architect Kristian Köhntopp told the tale of SysEleven's
long, winding and rough road to OpenStack adoption at large scale. For both
comedic and dramatic effect, he interspersed his report with rants about
especially nasty feats of "engineering" he encountered in various subsystems.

> We disabled MemCached to see what would happen, and performance *rose* by
> one and a half orders of magnitude.

I really liked how the day was framed by an optimistic, igniting keynote
and a long and borderline bitter rant on technical shortcomings and open issues.

Once presentations were over, we finally managed to get some networking done
and I got a hold of Monty and his fellow developers from HP. We were joined
by a SUSE Tech and Florian for drinks and some Indian food. By the virtue
of Conferences In Your Hometown, I had to leave the party early, but not before
discussing robot cars, dogs with multiple butts and all other things that
matter little in the scope of our daily business.

### So long

As far as conferences go, this one was pretty great. I had a good time in a
nice and convenient venue, heard a couple great talks and no bad ones whatsoever.
The crowd was enjoyable and in good spirits, and the catering was well organized
and very palatable. A big "thank you" to the organizers, who (as far as I am
concerned) did everything right with this first standalone conference.

Technically speaking, I approached the larger topic quite skeptically, and
my doubts were confirmed at a few instances (or that's how I chose to interpret
the facts). To me, OpenStack was always this behemoth with great purpose, but
way too much luggage to ever reach it within the bounds of acceptable effort.
And I never really thought that the architectural flexibility and theoretical
scalability were worth dealing with the complexity.

I didn't learn much that made me reconsider these views, although one of the
most impressive endorsements came, surprisingly, from Kristian himself, in
conversation after his presentation: For all the horrors he had encountered
when trying to deploy at scale, a "normally" sized deployment of a about
a rack full of quality hardware apparently Just Works.

What really makes OpenStack potentially worthwhile, though, is its spread.
It's backed by a couple of major players and it sure is not going anywhere.
Having a common API that is identical across providers is valuable in its
own right. So yeah - for better or worse, we're probably looking at a future
full of OpenStack. Guess it's time to stop struggling and embrace our
cloud-based overlords.
