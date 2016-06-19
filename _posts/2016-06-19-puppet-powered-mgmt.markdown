---
layout: post
category: features
tags: [ puppet, catalog, compiler, mgmt, graph ]
summary: An quick overview of what mgmt can do with Puppet by now.
---

[Recently](/features/2016/06/12/puppet,-meet-mgmt.html), I wrote a veritable deep-dive
on `mgmt`'s new Puppet integration code, but didn't include a good overview of how
the new features look in practice. Here we go.

### The original interface

If you recall, `mgmt` has no configuration language of its own. With its highly dynamic
and distributed nature, it's also not a good fit for the languages that established tools
such as `puppet` and `chef` use. (Please note that Salt has no language either, and Ansible
relies on a weird YAML hybrid).

You can use `mgmt`'s prereleases by supplying actual resource *graphs*, consisting of
vertices (resources) and edges (relations). The designated notation format is YAML.

> Speaking for myself, this is a good choice: It's a pure data notation with little
ambiguity, good support across languages, and great readability and maintainability
(unlike, say, JSON).

One of the implications is that it's mostly impractical to pass the graph directly
on the command line, the way you can do with Puppet manifests using `puppet apply -e`.
Technically it would work, seeing as YAML can be inlined to resemble JSON, but the
encumbrance from boilerplate and structure elements would make this a tedious exercise.

So as of May 2016, writing YAML files was the only way to feed graph data to `mgmt` for
testing and debugging. And debugging in particular is where I really like my one-shot
shell invocations. Enter the Puppet integration.

### From manifest to resource graph

For the record, this is the first manifest that saw intensive testing with `mgmt` as
the agent backend:

```puppet
package { 'cowsay': ensure => installed } -> exec { '/usr/games/cowsay boo': }
```

This was how we first ran it:

```
bundle exec puppet mgmtgraph print --manifest /tmp/cowsay.pp >/tmp/cowsay.yaml
mgmt run --file /tmp/cowsay.yaml
```

This was before `mgmt` was capable of invoking the `puppet mgmtgraph` command on its own.
This is marginally better than hand-crafting the YAML, because the Puppet manifest
is way more succinct than the resulting YAML. You can even commit the one-line manifest
to your shell history

```
puppet mgmtgraph print --code 'package { ... } -> exec { ... }' >/tmp/cowsay.yaml
```

By now, `mgmt` in its current `master` revision will even do the legwork of invoking
`puppet mgmtgraph`, using the following call syntax:

```
mgmt run --puppet /tmp/cowsay.pp
# or
mgmt run --puppet "file { ... } -> exec { ... }"
```

Finally, `mgmt` will also use `puppet`'s agent aspect to receive and translate a catalog
from a Puppet master:

```
mgmt run --puppet agent
```

### Customizing the puppet

Internally, `mgmt` will just invoke `puppet mgmtgraph`. The `puppet` binary must be
in `mgmt`'s search path for this to work.
It's up to the user to set up Puppet to their liking. Personally, I've
found myself appreciating the Ruby gem more and more, for day-to-day debugging and test
invocations. This article will close with a quick-setup guide for that.

Puppet has a well-defined way of initializing itself with default values and overrides.
The default values are hard-coded and can depend on the environment.
The most important way to customize Puppet is the `puppet.conf` file. It is loaded from
a default location that depends on your user ID and Puppet version.

For **root** and **puppet**:

| Puppet 3.x                | Puppet 4.x                           |
|---------------------------|--------------------------------------|
| `/etc/puppet/puppet.conf` | `/etc/puppetlabs/puppet/puppet.conf` |

For all other users:

| Puppet 3.x                | Puppet 4.x                             |
|---------------------------|----------------------------------------|
| `~/.puppet/puppet.conf`   | `~/.puppetlabs/etc/puppet/puppet.conf` |

It can be perfectly adequate to customize the puppet translator through the default config
file, especially if you run `mgmt` from your regular account and don't use `~/.puppetlabs`
for anything else.

In general, however, the user should have the freedom to customize the way Puppet behaves
in the `mgmt` context, without implications for other Puppet activities. This is why
`mgmt` accepts the `--puppet-conf` parameter. Here's what it could look like in action:

```
# in ~/.mgmt/puppet.conf`
[main]
server=localhost
masterport=8150
runinterval=600
vardir=~/.mgmt/puppet/var
```

Then `mgmt` gets invoked as follows. It will connect to the Puppet master process on the
local machine via port `8150`, with repeats every 10 minutes:

```
mgmt run --puppet agent --puppet-conf ~/.mgmt/puppet.conf
```

### Quickstart guide

Here's how I like to whip up quick and easy environments for running Puppet. The best way
to get a "private" copy of Puppet on a *NIX system is the Rubygem. Install it in your user's
home directory to be sure it doesn't interfere with the rest of your system, and to enable
simple clean-up.

1. Configure your Rubygems to live in a specific location in your home. Add this to your
`.bashrc` for permanence.

        export GEM_HOME=$HOME/gems_for_puppet

2. Make sure to have binaries from your local gems early in your search path.

        PATH=$GEM_HOME/bin:$PATH

3. Get the Puppet gem.

        gem install puppet --no-ri --no-rdoc

4. If the default settings with Puppet's `environmentpath` in your `~/.puppetlabs` is adequate
for you, just go ahead and get the translator module.

        puppet module install ffrank-mgmtgraph

Go ahead and test the translation.

```yaml
$ puppet mgmtgraph print --code 'file { "/tmp/foo": content => "bar" }'
---
graph: fflaptop.local
comment: generated from puppet catalog for fflaptop.local
resources:
  file:
  - name: "/tmp/foo"
    path: "/tmp/foo"
    state: exists
    content: bar
edges: []
```

Now you can just go ahead and play with `mgmt` using Puppet manifests. Get it from
[GitHub](https://github.com/purpleidea/mgmt) and follow the build instructions.
If you face any issues getting this to
run, please let us know on [Twitter](https://twitter.com/mgmtconfig) and IRC.
If it *does* work for you, please let us know as well :-)
