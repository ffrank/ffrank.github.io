---
layout: post
category: features
tags: [ mgmt, puppet, resource, pippet ]
summary: Explaining the improved puppet resource workaround in mgmt
---

## Abstract

When running a graph that was created from a Puppet manifest, mgmt has so far
used a workaround for each resource that does not translate. This workaround
involved an `exec` type resource, shelling out to a variant of `puppet
resource`. Using the new `puppet yamlresource receive` subcommand, mgmt can now
send resources to a persistent Puppet process instead. The native support for
this is implemented through the new pseudo-resource `pippet` in mgmt.
It's not meant to be used in mgmt code, but the Puppet code translator will
rely on it when required.

## Ensuring semantics

I have blogged quite extensively about the [Puppet compatibility
layers]({% post_url 2018-02-13-thinking-about-migration-from-puppet-to-mgmt %})
in the past. They work because the most important core resource types from
Puppet (such as `file` and `package`) can be found in mgmt as well. We expect
that much Puppet code that is out there could be run through native mgmt
resources.

The Puppet ecosystem of custom resource types in module is quite large,
however. Even the full set of "core adjacent" types is nothing to scoff at.
Any resources that mgmt does not support need to get synchronized in
another way.

Additionally, any actual resources even of mgmt supported types
cannot be modeled correctly by mgmt if they use properties that are
exclusive to Puppet. For example, mgmt in version 0.0.21 does not (yet?)
have a counterpart to Puppet's `refresh_only` parameter in the `exec` type
(which is [an
anti-pattern]({ post_url 2015-05-26-friends-don't-let-friends-use-refreshonly %})
anyway, but that's not the point here). Imagine such an `exec` resource:
If we were to just transform it to `mgmt`, it would fire its command
indiscriminately, while the Puppet code was designed to run it only under
very specific circumstances.

To provide for such resources, we implemented [a
workaround]({% post_url 2016-08-19-translating-all-the-things %}) that would
substitute such unsupported resources by a stand-in. The idea was to let
Puppet do the work of synchronizing these resources. The basic approach was
to generate `exec` resources that roughly do the following:

```
exec "substitute Cron[refresh-my-stuff]" {
    cmd => "puppet resource cron refresh-my-stuff command=/path/to/script minute=...
```

The actual implementation is a little more involved (see linked article), and
had me introduce the `puppet yamlresource` face. We have recently boosted its
performance using [a new
subcommand]({% post_url 2020-02-09-puppet-scripting-host %}) `puppet
yamlresource receive`.

In order to allow mgmt to take advantage of this latest addition, we needed
to add three properties to mgmt with respect to its Puppet integration:

 * mgmt needs to launch and track the Puppet receiver process
 * individual resources in mgmt's graph must be able to use this receiver
 * the Puppet manifest translator must emit such resources for mgmt

Let's look into each of these steps in turn.

## Running a persistent Puppet

When using `puppet yamlresource receive` in order to connect Puppet to
another piece of software (in this case, mgmt), the processes communicate
via I/O pipes (see following diagram).

![mgmt with pipe to Puppet](http://ffrank.github.io/presentations/2020-02-puppet-from-mgmt-on-overdrive/pippet.png)

This design does not really follow a pattern that I have encountered in any
of the projects I have worked on before. Implementing it is not super hard,
fortunately. Essentially the process is just kicked off in the background,
and mgmt just needs to be careful to hold on to its stdin and stdout I/O
channels.

For easy management inside mgmt, I borrowed the Singleton pattern from
object oriented programming. There is a global struct with a synchronized
function that will retrieve it, and run its initialization if needed. This
initialization procedure mainly consists of the start of the Puppet process.
Users of this data object are typically graph resources that rely on
Puppet. They call yet another function that implements the protocol of
`puppet yamlresource receive`, passing the resource details, and a reference
to the mentioned global struct. The reference uses a minimalistic interface
that this structure implements.

An aside on golang code design: The synchronization function was originally
implemented as a method on the Puppet process structure itself. That's because
I was in the mind trap of thinking in terms of object oriented programming.
I had intended to build tests for all this code by generating a mock of the
(extensive) general interface 
[aside: was originally method of the runner struct]
[makes testing hard]
[see below about testing]

## Graph nodes for Puppet

[sending strings to puppet cannot go into an exec]
[also not desirable]
[dedicated resource instead]

[structure is simple, meant for easy marshalling]
[actually cleaner than exec workaround]
[parameters are not structured, but held as YAML instead]
[note: passed to Puppet as YAML wrapped in JSON]
[JSON is there to make sure titles can be arbitrary]

## New graphs from Puppet

[replace exec workaround]
[actually added new default, old one still available]
[further savings: skip noop query, should not be required]

## A word about tests

[implementation started with Ruby state of mind]
[intended to mock pippetrunner, stub sync method]
[found that this makes testing brittle if not impossible]

[looked into dependency injection instead]
[found great blog post]
[incidentally understood Go interface ideas much better]

[sync function explicitly designed for dependency injection]
[mock implementation for pippetrunner interface]

## Summary

[both mgmt and the translator were enhanced for faster puppet integration]
[as a side-effect, Puppet received a scripting host]
[closing in on ability to run arbitrary Puppet code]
