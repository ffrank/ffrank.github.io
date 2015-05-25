---
layout: post
category: misc
tags: puppet exec parameter dependencies
---

An overview of the (ab-)uses for the `refreshonly` parameter to Puppet's `exec` type.

#### Manifest imperfection

The goal of any Puppet manifest is to describe a desired state.
Puppet's abstraction model breaks this down into an arbitrary set
of discrete *resources*. Each resource describes a specific piece
of system state.

{% highlight puppet %}
file {
	'/etc/shadow':
		owner => 'root',
		mode  => '0600',
}
{% endhighlight %}

This approach is oft proven to yield very maintainable and readable
code. But of course, setting up a piece of software on a machine
is often impossible without resorting to some OS commands that must
be run in a scripted fashion. Such commands will have an effect that
is required before some resources can be synchronized at all.

Enter the `exec` type, one of the less typical resource types in Puppet.

{% highlight puppet %}
exec {
	'apt-get update':
		logoutput   => 'on_failure',
		before      => Package['from-new-repo'],
}
{% endhighlight %}

It does not so much represent a state that Puppet can figure out
how to reach. Instead, it gives the user the opportunity to tell
Puppet exactly *how* to proceed. This is less powerful than Puppets
usual way of expressing things in a platform agnostic fashion.
That's why `exec` should only be used if all else fails.

As a consequence, a resource like the above cannot easily be found
to be in sync. Puppet will synchronize it during each agent transaction.
There is a reasonable way around that, but let's focus on the central
issue first.

#### Make it extra quick and dirty

Since software setup instructions tend to read like "install package X,
then run command Y, *then* command Z and finally start the service",
it is tempting to translate this into a manifest verbatim.


{% highlight puppet %}
class software_x {
	package { 'X': ensure => installed }
	->
	exec { 'Y': }
	->
	exec { 'Z': }
	->
	service { 'X': ensure => 'running', enable => true }
}
{% endhighlight %}

The commands should run *only* during initial setup. The easy
way to achieve this is to

 * make the `package` *notify* the `exec`s and
 * add the `refreshonly` parameter

In simple manifests like this one, the *notify* can be expressed
by the modified dependency arrow `~>`.

{% highlight puppet %}
class software_x {
	package { 'X': ensure => installed }
	~>
	exec { 'Y': refreshonly => true }
	~>
	exec { 'Z': refreshonly => true }
	->
	service { 'X': ensure => 'running', enable => true }
}
{% endhighlight %}

In a perfect world, this will do just what you expect, but
this structure is quite fragile. Consider the following
resource that may be part of the same manifest.

{% highlight puppet %}
class sys_setup {
	package {
		'site-definitions':
			ensure => 'installed',
			before => Exec['X'],
	}
}
{% endhighlight %}

This is contrived, but a similar structure can just emerge
in real life, especially when there are a couple of relationships
between whole classes.

#### Danger, Will Robinson

The `Package['site-definitions']` resource has now become
a dependency of `Exec['X']`. If for any reason, the `site-definitions`
package cannot be successfully installed (think failing post-installation
hooks) during the Puppet agent transaction that should also set up *software X*,
the system under management ends up in a bit of a pickle.

Package *X* will be installed, because it has no relation to the `sys_setup`
class and its contents whatsoever. Puppet should then run two commands
only once. But because a dependency failed, `Exec['X']` will be skipped during
this transaction. This is where `refreshonly` backfires rather terribly.

The `exec` resources will only refresh themselves when they receive
a notification. In many cases, such commands get only one chance to fire.
The subscribed resource is already in sync (package *X* is installed).
During all follow-up agent runs, Puppet will detect this and not touch
the `package` resource anymore. There will be no new notifications
for `Exec['X']` and the system will remain in an inconsistent state.

Even worse, it's quite difficult for the user to detect this and recover
from this state. They would have to uninstall the *X* package again
and let the agent run, so that a new notification is generated.
If such a class is part of a Forge module, there is no way for them
to figure this out, short of analyzing the manifest and deducing which
`exec` resources were missed.

#### You're Doing It Wrong

With the `exec` resource type considered the last ditch, its `refreshonly`
parameter should be seen as especially outrageous. To make an `exec`
resource fit into Puppet's model better, you should use one of the
following parameters instead.

 1. `creates` whenever there is a file that will only exist after
    the command completed successfully or
 2. `onlyif` or `unless` with a more generic query

There is almost always the way to figure out whether a command has
been run successfully. I tend to prefer the `creates` attribute
for its simplicity. Some commands will not create files, though
(think database CLI manipulation). Their results have to be checked
through other commands that are run first, through `unless` or
`onlyif`. They are equivalent except for the logical value of the
query result.

Make sure to read up on the details of these parameters in the [official
documentation](https://docs.puppetlabs.com/references/latest/type.html#exec)
and please - avoid `refreshonly` whenever you can.
