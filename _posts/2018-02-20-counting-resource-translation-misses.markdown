---
layout: post
category: hacks
tags: [ puppet, mgmt, catalog, graph ]
summary: Our roadmap to get mgmt into a state to more properly substitute parts of Puppet.
---

This is another post in the wake of CfgMgmtCamp 2018, where the mgmt hack room
saw lively discussion that inspired
[some new ideas](/features/2018-02-13-thinking-about-migration-from-puppet-to-mgmt/)
as well as reminding me of some ideas I had last year, but didn't manage to implement
or describe yet. This post is about one of those.

### Recap

Even though we now have a slick [Puppet module](https://forge.puppet.com/ffrank/mgmtgraph)
that allows early adopters and testers to run mgmt from Puppet manifest code, our
ability to run existing, complex manifests is somewhat limited. We *will* run pretty
much any code, but many resources will not be compatible with mgmt's current limited
feature set. I wrote about this [earlier](/features/2016-08-19-translating-all-the-things/).

Puppet supports a large number of resource types, many of them implemented in third party
modules. The set in mgmt is limited, and will likely not match the Puppet ecosystem
any time soon. That's why we added a default rule to the code translator, which will
transform unsupported resources into `exec` type resources. These `exec`s call Puppet
and ask it to perform the task of synchronizing the untranslatable resource.

This trick will now also be used for resources that can be translated in principle,
but not with respect to all their specific parameters. For example, early mgmt releases
did not support managing a file's permissions. A Puppet manifest that uses a `file`
resource with a value for its `mode` property would not cleanly translate.

```
file { "/root/secrets.txt": owner => "root", mode => "0600" }
```

mgmt would ignore the mode, so the semantics were actually different.

```
file:
- name: /root/secrets.txt
  owner: root
```

In what I dubbed
"conservative mode", the Puppet translator module will now reject such resources.
They get replaced with `exec` nodes, just like unsupported types.

```
exec:
- name: File /root/secrets.txt
  cmd: puppet resource file /root/secrets.txt owner=root mode=0600
```

(This is pseudo-code...the actual default translation is a good deal more complex, in order
to also support resources that are not this straight forward.)

### The problem

This approach works for the most part. We expect that it synchronizes all managed resources
into the correct state. However, the trick has serious performance implications. Each single
resource that is run through Puppet requires its own, fresh Puppet process. Without any
tuning, this typically takes between 1 and 3-4 seconds, depending on the speed of the system.
That's because each Puppet invocation launches in a fresh Ruby interpreter.

The bottom line is that in the presence of many untranslatable resources, mgmt will run
quite slowly, much slower than a native Puppet run even. As such, it's important to reduce
the number of these pseudo-translations.

### The solution

We need to collect data about how well the translator works. I touched on this in the
[earlier post](/features/2016-08-19-translating-all-the-things/) I mentioned above.
There are two numbers that are of interest: First, what are the resource types that
most frequently run into the `puppet resource` workaround because there is no
counterpart in mgmt. In essence, this should give us a prioritized list of resources
that we should add to mgmt, in order to improve its utility as a drop-in replacement
for Puppet.

The other value of interest is the relative frequency of each resource parameter that
currently causes a failed translation (for types that mgmt actually does support).
By collecting this from a large number of existing Puppet manifests, we should get
a to-do-list for parameters that we need to add to mgmt's resources.

Both strategies should present us with a number of low-hanging fruit that, once
picked, will drastically improve the performance of mgmt's integrated Puppet support.

### The way forward

I'd like to collect statistics from two sources. First and foremost, we should ask
our users (or right now, early adopters) to tell us the numbers as reported from
their own Puppet code. The other approach is more pragmatic, and should report
numbers that are gleaned from translating modules from
[the Forge](https://forge.puppet.com) (especially the more popular modules).

Mining Forge code will be more challenging than it may seem. Installing a module
from a script is simple enough, but Puppet modules are not really functional
with some code that actually *invokes* them. For many modules, this is as simple
as including the module's main class, like in the following examples.

```
include puppetdb
include ntp
```

However, for some modules, it will be a little more complex. Some will only tap
into much of their code when certain parameter values are passed. Others will
encapsulate the most important code in some defined types (the notorious
`apache::vhost` comes to mind). Some modules will not even *have* a main class
with the module's name.

So, in addition to mining the module code itself from the Forge, we will need
to figure out a way to come up with adequate manifests to make proper use of
each of them.

To properly facilitate all of this, I will add a new function to the
[translator module](https://forge.puppet.com/ffrank/mgmtgraph). It should
work kind of like this:

```
$ puppet mgmtgraph stats /path/to/manifest.pp
Statistics:
  31 translated resources
  17 successful
   4 not supported
  10 with parameter failures
Error messages:
  6x File[...] uses a puppet fileserver URL source - this will not be translated
  3x Augeas[...] uses the unsupported 'onlyif' parameter
  1x Exec[...] expects return code(s) other than 0, which mgmt does not support.
  ...
```

For the input manifest, instead of printing the graph for mgmt, it breaks down
the errors that happen along the way. In the mock above, just over half of the
resources from the manifest translated cleanly. 14 of them had to fall back to
the `puppet resource` workaround, for various reasons.
The reasons are consolidated into groups of identical messages, and these groups
are presented in order, from the largest to the smallest.

This feature will soon be available in the translator Puppet module. It will be
supplemented with a script to run this through various Puppet Forge modules. This
will allow us to gather the statistics I talked about in this article. However,
this latter science project does not have a very high priority right now.

In fact, what's much more useful to us is real feedback from real people. So if
you read this, and you use Puppet in any capacity, *and* you would like to help
the mgmt effort, please consider getting the `ffrank-mgmtgraph` Puppet module
and gathering the numbers (that is, as soon as the `stats` feature is added).
Reach out to [me](https://twitter.com/felis_rex) or
[James](https://twitter.com/purpleidea) via Twitter, or open an issue in the
[upstream mgmt repo](https://github.com/purpleidea/mgmt) and report your findings.

We are quite eager to hear from you; your contribution to our Open Source effort
is most appreciated!
