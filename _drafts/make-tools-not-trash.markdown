---
layout: post
category: features
tags: [ puppet, mgmt, chef, ansible, salt, devops, hugops ]
summary: Some thoughts on the configuration management space
---

One of the most special weeks in Berlin just concluded. It saw
yet another DevOpsDays conference, as well as the first installment
of a CfgMgmtCamp outside of Belgium. Surprising no-one, both conferences
were choke full with a lovely and international crowd of DevOps
practitioners and various tech professionals. Definitely a highlight
of my year.

What stuck with me most were all the inspiring little moments of collaboration,
empathy, and love. It was heart-warming to see Puppet's [Eric](https://twitter.com/ahpook)
hug Chef's [Nathen](https://twitter.com/NathenHarvey) upon entering the stage.
So did [Vox Pupupli's](http://voxpupuli.org) [Igor](https://twitter.com/hirojin) when
he made up with Nathen after inadvertently haggling him during his presentation.
On the hallway track, Puppet contributor par excellance [Spencer](https://twitter.com/nibalizer)
advised me on how specific Ansible features enable unique coding practices.
Speaking of, I also had a friendly little chat with Eric and [Robin](twitter.com/robinbowes)
about idiosyncrasies of Ansible compared to Puppet.
I would even count myself, spreading love for both Puppet and mgmt in the span
of a 5 minute Ignite Talk.

###Not all hearts and rainbows

This is not always the face of the configuration management scene. I was not yet
present when Puppet emerged and took the tooling throne from CFEngine.
Judging from the later tension between Puppet and Chef, however, I assume
that it wasn't always peaceful. (Granted, the history between Puppet and Chef
was...special, but still.) In fact, the cultural gap between the communities
of Puppet and Chef felt very wasteful to me for a long time. I feel much at home
with Puppet, but I know for a fact that there are many amazing people besides Nathen
who identify with Chef. I really wanted for all these folks to feel as one family,
not bickering clans.

More generally speaking, I'm quite certain that talking smack within each of our
communities is quite common. Harping on the tool or tools that are out of favor
with us and our peers not only passes time. It serves as a form of reinforcement as well.
We bond with our friends and colleagues, and we nurture our conviction that we
made a good tooling choice.
In her nicely put together CfgMgmtCamp
[keynote](http://www.slideshare.net/geekygirldawn/config-management-community-awesome-awful-or-apathetic),
[Dawn](https://twitter.com/geekygirldawn) acknowledged this trope as well. However,
participants of her survey had sorted it into the bucket of negative aspects of
config management culture.

Personally, I sure am no stranger to smack talking software and technology.
In my defense, I was quite young when I started dabbling in programming and
the Linux family of operating systems. Being a fanboy of open source and the
C language gave me a sense of pride. What a thrill to rant about the supposedly
inferior "mainstream" tech.
If you reminisce of similar attitudes and get a sense of pettiness, I feel you.
And we're probably on to something.

###Chilling out

The more time I spent around tech topics, the more weary I grew of fighting petty battles
about my favourites. So Windows is a thing that people find useful. What gives?
Sure, it crashed a lot in earlier releases, but guess what? The blame is shared
with vendors who would distribute crappy software that abused APIs. Guess which
OS you could install on pretty much any x86 system throughout the aughts, without
mandatory major updates? Surely not Debian. (Yes, this wasn't by virtue of Windows'
amazing upwards compatibility, but it was a fact nonetheless.) Eventually, I found I had
stopped caring about Windows as the enemy.

It was a minor revelation: life was so much easier without the need to get defensive
when faced with "another" platform. In programming, I made similar experiences.
For a long time, I was searching for the "best" language for solving the widest range
of problems possible. Pascal was so much cleaner than Basic. C so much more flexible.
Perl so much more powerful. Shell more elegant. Object oriented languages offered
such an impressive ability to model systems. Of course, you eventually realize that
it's all about trade-offs, and a complex evolution is ever continuing.

Many Linux users have a favourite distribution. I remember so many exchanges that
consisted of playful riffing on one project or the other. However, ultimately it
also boils down to some trade offs, and lots of love labor. Good work is being done.
Some things are universally terrible but getting there. Doggedly defending your
favourite doesn't really help anyone. Why bother?

Finally, among operations professionals, we're forming lots of opinions about the
recent wave of infrastructure tooling. From configuration management, through testing
and monitoring tools to container runtimes and virtualization engines, there are many
fields of healthy competition. But does opinions have to imply talking smack? Why does
it seem to be en vogue to hate Open Stack?

It's weird how long it has taken me to apply the lessons mentioned above to ops tooling.
At last, challenging my preconceptions really has paid off. It's difficult to come to grips
with a piece of technology if you expect the experience to be terrible. And yet our echo
chambers can prime us to do exactly that. Have you tried reading up on the advantages of
systemd?

Bottom line: It's fine to have opinions and discussions, but be weary of hate. People
built those tools with love. They deserve some in return.

###Going bravely

Let's keep talking about configuration management (surprise! And yes, that's still a thing.
No, it's not going anywhere. You might as well hop on the band-wagon now while it's still
cool.)

James has been showing mgmt around a lot. Especially his full live demo presentation has
proverbially blown lots of minds. Of course, configuration management is not a full time
job to every person in tech. To those who only need to use the tools on occasion, the
growing number of alternatives is a bother more than anything else. Do we need to learn
a new tool every two years now?

The proliferation of tooling alternatives surely isn't helpful in and of itself. However,
it bears acknowledging that the creators of those tools are not just indulging themselves.
So far, each iteration has brought some unique new properties to the table. With mgmt,
I feel that the innovation is especially incisive, not unlike Puppet's conception back in
the day. It's not even a huge step. With a few decisive design decisions, mgmt is gearing up
to fit the needs of highly distributed systems much better than any of the other tools in
this specific problem space.

Some of those who already see it this way have taken jumped to a very different conclusion:
This spells the end of Puppet. (Does this include Chef? Does it extend to Ansible?)
This drastic view never appealed to me. I guess I'm still too much of a Puppet fanboy.
But even objectively, I just don't suppose that mgmt (or any other iteration I can think of)
will make all contenders obsolete anytime soon. Rather, we're moving onto a new and important
branch of the tooling evolution. Puppet, Chef, and their peers run on an impressive range
of platforms that mgmt will likely not rival. Puppet also has a very mature chain of complimentary
tools, with its competition still in the process of of catching up. My personal estimation is
that Puppet won't be driven from the enterprise anytime soon.

###Oh that discussion? Nah, turns out it was a non-issue

In short, I don't think that the work that I or anyone is doing is a direct threat to
Puppet or its competition.
Nor does it really matter which tool might obsolete whatever features from another. As
operations engineers and automation professionals, we need to see a bigger picture. Love and
respect should not only pervade the interactions of tool creators and contributors. In an
effort to extend the DevOps ideal to our whole industry, we all should open ourselves
to the tooling diversity. So you got rid of the silos in your organization? Great!
Tackle the walls between our communities next.

Whenever we or our colleagues engage in technical discussions that focus heavily on the
weaknesses in software and solutions, folks are bound to get defensive. This is rarely if ever
a good basis for a constructive exchange. If you want to get emotional during technical
conversation, let's at least make sure we feel good about it.

This is not to say that we should avoid mentioning perceived disadvantages of certain
tools or approaches. Respectfully discussing them can be quite enlightening. Your bad
experiences could serve as a lead-in for someone else to give you valuable new
information to help you along. Don't get defensive when someone points out a weak spot
of your favourite. Be mindful and understand more about its trade-offs instead. 

Speaking of: have you concluded, for any problem domain, what the best available tool
is? You might be right for the specific scenarios that you typically face. But for any
tool that gained significant community traction, you can rest assured that has strengths
in some areas (that you might not be keenly aware of) where it becomes superior.

There are also impressive examples for combinations of different tools to greater effect.
For example, Ansible has adopted a Puppet module. It allows the user to trigger Puppet's
agent directly from Ansible, giving you the expressive power of Puppet, with the operational
flexibility of Ansible. Conversely, people have been known to invoke Ansible from Puppet's
pre- and post-run hooks, lending Ansible's immediate cluster management capabilities to Puppet.

Taking interoperation even a step further (and excessively tooting my own horn right now),
I went so far as to build support for `mgmt` right *into* Puppet through a
[module]({% post_url 2016-06-19-puppet-powered-mgmt %}). All in the name of a more powerful
ecosystem. And nobody has gotten mad at me. (As of writing. :-)

###Liberating ourselves from tool allegiance

Here's the thing though: The vendors don't build their products in isolation. They don't
lock their software designers in a windowless cellar. Software tooling evolves in an
ecosystem. This feels especially true for open source tools. New contenders will often
inherit the best aspects of established solutions, while trying to avoid their more severe
(supposed) shortcomings. Competition does not happen without a certain amount of rivalry,
but in the end, config management tools are birds of a feather, as are their respective
creators.

It does not necessarily follow that the open source communities around these tools should
form a unit. However, I propose that users of any (combination of) tools would be better off
if they felt like a part of a larger config management community, rather than a specific
crowd that tries to find its own voice to stand out from the others. On the flip side, it
makes little sense for users and contributors of an open source project to adopt the rivalry
the software creator feels towards competing projects.

As for our professional disposition, some folks will actually benefit from specializing in
fewer tools and approaches in order to be most effective. However, if you feel that you can
manage a little more, and aren't daunted by the multitude, I warmly recommend to widen your
horizon and pick up a small bouquet of additional skills. You will most certainly gain more
insight on your favorite tool this way, and who knows...you might even take to a new
instrument of choice. Don't be afraid of branching out. A smart person
[once said](https://twitter.com/patrickdebois/status/603960527912591360) that learning
your first config management tool is the hardest.

###No silver bullet, anyway

I have alluded to this before already: A comparison of the different available config
management solutions boils down to identifying the trade-offs that each of them accepts.

[some examples of tooling trade-offs (mention own [lack of] proficiencies)]
[puppet and chef share powerful model and fully featured languages]
[salt forgoes language in favor of YAML, adds live management capabilities]
[ansible uses YAML as well, adds agent-less operation and register variables]
[mgmt does embrace language, and eliminates some architectural issues, probably no full support on as many platforms]

[each tool brings some unique strengths to the table]
[professional users and contributors can profit from knowing the respective tradeoffs]
[we can all have more fun by interacting with all communities]
[being more accepting of each others might even further broader inclusivity and diversity]

[everyone start mingling]
[- become mgmt contributor]
[- join the community of a tool you don't (yet) use (be positive there!)]
[- acknowledge the other communities in your meetups etc.]
[- say hi to the others at conferences and other places, and listen to their stories]

[notify me if this was helpful]
[share it with your friends]
