---
layout: post
category: hacks
tags: [ puppet, mgmt, catalog, graph ]
summary: Some early musings on possible future migration paths from Puppet to mgmt.
---

### Abstract

We have already implemented a way to replace the Puppet Agent with the mgmt
engine. It can run a graph on a machine that is derived from the machine's
Puppet catalog. The user is limited to mgmt features that can be expressed
in Puppet's DSL in this scenario. In order to unlock mgmt's full potential,
the user will have to rewrite all their Puppet code in mgmt's language.

This presents a significant challenge for maintainers of larger Puppet code
bases that are actively used and evolving. That's why it would be useful to
allow a step-by-step migration, during which the user selectively removes
pieces of the Puppet manifest and replaces them with mgmt code.

We present a theoretical scheme that allows the creation of a graph
from two distinct source code inputs. It involves merging of certain
graph nodes based on a name matching scheme.

### Introduction

You may or may not be aware, but
we just finished another instance of ConfigManagementCamp. The 2018 edition
featured amazing keynotes, great talks, a legendary hallway and dinner track,
and of course the annual fringe event day, full of hack days
and workshops. I spent this last day in the mgmt room, as I did last year.
I'm happy that this year, for the first time, we formed an actual crowd
of more than 10 people at the hack day, solving onboarding issues,
discussing details and ideas, and generally having a good time.

One of the discussions revolved around a scenario that I hope will become
quite popular once mgmt stabilizes enough to become generally available:
Given an existing Puppet code base, a team wants to migrate to mgmt with
as little friction as possible. There were several immediate questions.

* How well will the Puppet integration work?
* Will I have to rewrite the manifests eventually?
* Can I mix native mgmt code with translated Puppet code in the interim?

You may have seen my [earlier posts](/features/2016/08/19/translating-all-the-things/)
about the Puppet language translator.
(It's been a while. This is me trying to make 2018 the year I actively
work on Open Source software again.)
At the time of writing this, it works fairly well. It will translate
what resources it can into mgmt's counterparts, and it will emit
`exec` resources whenever a translation is not possible.
The `exec` resource will call back to Puppet to do the legwork.

This is all good and fine, but it confines the user to a world where
there is still a Puppet master running the show, and everyone will keep
writing Puppet code. This is not a bad thing per se, but it makes
many of the more powerful and innovative features of mgmt inaccessible.
For example, the complete range of reactive code behavior that mgmt
introduces cannot be expressed in a Puppet manifest.

(Caveat: mgmt *will* behave reactively on resources and so forth,
even when the graph is derived from a Puppet manifest. However, the more
advanced features will elude the user.)

This is why most if not all users will sooner or later want to switch to using
the native mgmt language instead of Puppet's. Of those who choose the
Puppet language support to help the transition, many will want to replace
the Puppet code step by step, probably one or few modules at a time.
There is one big problem, however: The Puppet support, as it is implemented
at the moment, is a complete alternative to the native language. As James
put it at one point, there are several distinct sources for the resource
graph. One is the mgmt language. One is the Puppet translator. (It will
likely be possible to also enter a YAML graph directly, although that
seems quite cumbersome in comparison.)

The bottom line is, the user will have to make a clean cut and replace
the full Puppet code base with one in mgmt's language. It should definitely
be our goal to make this quite straight-forward. But no matter the absolute
difficulty of translating one language to the other, such a project could
pose quite a hurdle to some users. Rewriting takes more time the larger
the amount of Puppet code, of course. What's more, some Puppet users will
see frequent changes in their code base, depending on the style of operations,
and how the tools are customarily used.
Freezing the Puppet code for an extended period in order
to calmly perform a rewrite may or may not be feasible.

For this reason (and also because we like to make cool things in order
to make
everyone's life easier), we have been considering ways to remove this
requirement for a full rewrite in one go, and allowing actual mixing
of Puppet and mgmt code.

### More seamless ways of transition

What I'd really like to see is a way to freely mix resources that were
defined in Puppet with those from mgmt code. Implementing this might be
difficult; I will outline the specific idea below, but it's unclear
how much code this will require. We did discuss some alternative
ideas as well. These I do not particularly care for, frankly. So let's
review these first.

One approach is to run in two passes. For example, resources from
Puppet code would run first, followed by those from mgmt. (The
engine is always mgmt here.)
This is quite impractical, because it requires the user to be able
to split the existing code in some strictly sequential chunks.
This is actually not as bad as it sounds, because ideally, Puppet code
consists of a series of Forge modules that are stringed together
in a given order. Still, this approach ties the user into a set order
for replacing code. Low hanging fruit cannot be prioritized, and
a particularly difficult to translate module could block progress early
if it is near the start of the evaluation chain.

Another idea was to just run two mgmt processes independently from one
another, one to run legacy Puppet code, and the other for what code
has been translated already. This seems actually dangerous, because
any dependencies and ordering disappear, and neither process can be
trusted to reliably perform its tasks.

This issue also touches on the central problem with my "mixed graph" approach.
I will explain this problem shortly, but first let's review the idea itself.

You have probably seen James run simple graphs through mgmt, the graphs
being defined directly in YAML. In his most recent demos, he also showed
examples that were powered by code in his new DSL. In my own presentations,
I have run various Puppet code through mgmt. What all examples have in common,
however, is that there is always one graph from one particular input.
During a transitional phase, I would like mgmt to offer the ability to
have not one but two sources for the graph. Each works like they do right now:
It emits an arbitrary number of resources and graph edges between them.
The finished graph should be the combination of both sets. Implementing
this part should be easy enough.

There is that one missing detail, however. When mixing
code segments from Puppet and mgmt, you will want to form dependencies between
the different parts. Consider a Puppet manifest that handles four subsystems:
NTP (first), distributed storage and Java (after NTP, in that order), and finally
whatever custom application and some monitoring agent (both after Java).

![Diagram of the Puppet example](https://user-images.githubusercontent.com/436765/36080725-0377fe66-0f95-11e8-9002-2fcc3f1c391b.png)

Now let's assume that the user wants to replace their Java module with some
mgmt code. Java is no longer part of the Puppet manifest. The manifest
can no longer express any of Java's relationships, neither its dependency on
the storage module, nor the fact that application and monitoring need to
wait for it.
Even if the user adds some new edges in the Puppet code, the mgmt engine will likely try
and install the JDK way too early, because of mgmt's parallel execution model.

![Altered example](https://user-images.githubusercontent.com/436765/36123085-e173ff60-104b-11e8-9239-ba1a131bea42.png)

As a workaround to slightly mitigate this effect, you could exploit mgmt's
`--sema` option (or the `sema` metaparameter; see
[James's metaparameter post](https://purpleidea.com/blog/2017/03/01/metaparameters-in-mgmt/)
for more information) in order to suppress any parallelism. However, the
order of the Java related resources will still be random with respect to
the rest of the graph.

(Note: This example is contrived and unspecific. In actual code, this might
happen to actually do the right thing, with or without the `sema` options,
because mgmt will still add "AutoEdges", described in the same post by James
linked earlier. If these happen to form adequate dependencies, the mixed
graph will actually be fine. We cannot rely on this to work in the general
case, however.)

We are not yet sure how hard or easy it will be for mgmt to accept edges
to and from resources that are not part of the same piece of code. We do know
that Puppet will absolutely not accept them. So we need a solution.

### Graph Merging

Here is my initial idea. I have not yet built a proof-of-concept implementation,
so all of this may well go down the drain soon, but I'd like to keep it
alive here for posterity.

Consider the earlier pseudo-code example. Assume that the Java module was
extracted into mgmt code. The graph from both the Puppet and mgmt code is
supposed to run in one mgmt engine.

We need to define some points to hand over control from one sub-graph to the other.
Starting from the leftover Puppet code,
the place of the Java module must be taken by a subgraph from the mgmt code.
So the Puppet code needs to declare relationships to this subgraph's entrypoint,
and from its end.

In our example, the mgmt code that replaces the Java module would receive two
additional nodes: One that serves as the entry point, with outgoing edges to
all resources that have no other direct predecessors. The other new node
becomes its terminus, situated as a successor to all nodes that have none.

![Graphs with new nodes](https://user-images.githubusercontent.com/436765/36128552-4646e33c-1063-11e8-8293-8edf690c7004.png)

The Puppet graph also receives additional nodes, at more arbitrary points.
This scheme would also need a way to declare the direction of the edge
that connects the Puppet side with the mgmt side. Luckily, I have a
simplification in mind.

The approach is to allow the mgmt engine to *merge* certain nodes at the moment
of loading the two graphs from the distinct sources. This should make it easier
to use the *anchor* style graph nodes described above more intuitively.

![Graphs with merged nodes](https://user-images.githubusercontent.com/436765/36129233-69d9b90c-1066-11e8-929a-f9354652bb7e.png)

In order to implement this, both Puppet and mgmt must provide respective ways
to define nodes and relationships in such a way that mgmt can tell which nodes
are to be merged.

The easiest way that comes to mind is a scheme for defining certain resources
that do nothing, but can be matched with counterparts in the other respective
language. In mgmt, the `noop` resource is a nice example. In Puppet, it would
be possible to allow the definition of empty classes with certain names. In an
[earlier article](/features/2016/07/12/edging-it-all-in/), I already described
how Puppet's classes inject implicit resources into the graph, two so-called
`whit`s representing the class's entry and exit point respectively. These are
translated into `noop` resources for mgmt. Merging these should not require
much beyond a clever naming scheme, right?

![Node merging](https://user-images.githubusercontent.com/436765/36129877-b28a0fc8-1069-11e8-9490-3b84fa4ee4b3.png)

Here's the plan: A class named `mgmt_<placeholder>` in Puppet will have its
start and end `whit` resources merged with a `noop` resource from the mgmt code.
On the mgmt side, the merging node will be defined as a `noop` resource
named `puppet_<placeholder>`. The placeholder name must be mutually identical.

![Merging graphs](https://user-images.githubusercontent.com/436765/36123084-e14d8e02-104b-11e8-9018-1cf430ad8963.png)

This scheme will hopefully make it possible for both pieces of code to interact
in a fashion that is intuitive to the user.
