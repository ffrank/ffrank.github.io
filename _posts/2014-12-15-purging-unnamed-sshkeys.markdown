---
layout: post
category: bugs
tags: puppet bugs ssh keys purging
---

Puppet got in trouble when users manually created resources that had no description.
Unnamed resources are difficult for Puppet to handle. To allow successful
purging, it is necessary to generate internal names for such resources.

### Background

In this post, I want to describe a problem that cropped up in a feature that I
had implemented for one of the late 3.x releases. At the time, it was *the*
most requested item, with dozens of votes on Redmine (of course it was old),
and later on Jira.

The wanted functionality was simple enough: Allow Puppet to *purge* SSH keys
that have been authorized for SSH login, but are not being actively managed.
This is useful for security, because Puppet will make sure that users don't
authorize additional keys that might grant them access even after their
original key has been removed from the machine(s) in question.

The same thing has long been possible for other types such as `/etc/hosts` entries,
files in managed directory trees, mounted partitions, and so forth. It usually
goes like this:

    resources {
        'mount':
            purge => true
    }

A satisfactory implementation remained elusive for a long time. The fundamental
issue is that Puppet cannot reliably tell where authorized keys could be located.
I pondered a design that would discern the viable key locations from the list
of users on a system and the SSH server configuration.

Upon asking for opinions on the developers' mailing list, it soon became clear
that such an approach would likely not be comprehensive either. Enumerating
all systems users is tricky to get right, seeing as agent systems can use LDAP or similar
registries alongside the native user database.
The `ssh_authorized_keys` provider would need to support them all
to allow for a `resources` based purging strategy.

The [final design proposal](https://groups.google.com/forum/#!msg/puppet-users/AgvUwA9RMLM/QWGeYMP_9xoJ)
was made by the illustrious [John C. Bollinger](https://www.linkedin.com/in/johncbollinger).
It might warrant a future post in its own right. In a nutshell, the purging
works through `user` resources rather than the `resources` type now.

    user {
        'felix':
            home           => '/home/felix',
            purge_ssh_keys => '~/.ssh/authorized_keys',
    }

Or more simple for this default location:

    user {
        'felix':
            home           => '/home/felix',
            purge_ssh_keys => true,
    }

### The bug

When building the feature, I made a classic mistake by assuming rather clean
environments for Puppet to work in. What I was totally unaware of was the fact
that authorized keys are not required to take the usual form:

    ssh-rsa AAAAB3NzaC1yc2EAAAAD...jmFDamawfU2Dpm2MK90Jk/ felix@comp.example.net
    ssh-rsa AAAAB3NzaC1yc2EAAAAD...9Z1NDbgI+Gfmj5Ovc62ULT rcmd-key

Specifically, the comment at the end of the line is allowed to be empty.

    ssh-rsa AAAAB3NzaC1yc2EAAAAD...jmFDamawfU2Dpm2MK90Jk/

This poses yet another fundamental issue, because Puppet uses the comment field
for the unique name that each resource must have. When encountering the above
line in an authorized keys file, Puppet will register the key as a resource
with an empty name.

    Ssh_authorized_key['']

In and of itself, this does not pose an issue, but it becomes significant
when unmanaged entities are to be purged from the system.

### Technical details of the issue

Puppet purges unmanaged entities by generating resources for them.
All agent work revolves around the catalog, which mainly consists
of a collection of resources and relationships among them.
Most if not all of those resources are added to the catalog during its
creation, by the compiler; these resources are defined right in the manifest.

But the agent can also add resources on its own. It does this by strict rules,
and the catalog controls its behavior. For purging, the agent will query
the providers for the names of all entities that can be found on the system.
Each of these names is used to spawn a resource that is marked for purging.
Resources that are already part of the catalog (they are managed) are always
silently discarded.

This usually works without a hitch, because Puppet identifies every system
entity by a name that is guaranteed to be unique. Take

    User['felix']

for example. Since the operating system does not allow duplicate user names, it is
safe to assume that this reference is unambiguous.

However, authorized SSH keys are not managed in such a fashion by the operating
system or any service. Each is just a line of text in an appropriate file. SSH
does not even care about any duplication. Therefor, Puppet resources of this type
are prone for resource duplication.

With generated resources, this can easily happen if the user opts to use
the same comment on multiple keys (perhaps in multiple `authorized_keys` files).
However, with uncommented keys, the clash is especially likely. After all,
it will not often make sense to deliberately annotate multiple keys with
the same information. But where users customarily skip the comment, Puppet
is bound to generate duplicate resources and fail the catalog.

### Solving the problem

A similar issue was solved for the `cron` type
[in the past](http://projects.reductivelabs.com/issues/3220).
Just like authorized SSH keys, `cron` jobs are not inherently
named anything - they too are basically just lines in some files.
When managing `crontab` content, Puppet will add comments of a special
format to hold each job's name.

    # Puppet Name: do-this-sometimes
    1 1 2 * * find / -name malware\* -delete

For most users, the purging option is mostly attractive for such
entries that have not been created by Puppet in the first place, and hence
have not been given a Puppet name. To allow Puppet to reference such entries
and mark them for purging, a name has to be generated along with the respective
resource.

However, only few catalogs will opt to remove these unmanaged jobs.
Generally, users will want to ignore them instead - Puppet will happily manage
only parts of a system, and ignore whatever is not mentioned in the manifest.
With the provider for `cron` (and now also `ssh_authorized_key`) making up
resource names, this will lead to unwanted comments with `Puppet Name` information
in all `crontabs`.

    # Puppet Name: find_~/Downloads_-mtime_+30_-delete
    @reboot find ~/Downloads -mtime +30 -delete

Therefor, the agent should take special care to *forget* these names before
writing its data back to disk, so to speak.

For `ssh_authorized_key`, there is an additional challenge: The actual provider
for this type cannot be used for the purpose of generating resources at all,
because it will only act on key files that are referenced by `ssh_authorized_key`
resources from the manifest. The purging needs to be based off of `user` type
resources - the user adds a purge option to `user` resources in order to clean
keys that have been authorized for the respective user.

Special care must be taken when dynamically generating resource names under these
circumstances. The `user` type can generate resources that use artificial
names with no problem. However, these names cannot be communicated to the
`ssh_authorized_key` provider, who is in charge of performing the actual
key removal, later during the agent transaction.

The provider must generate the artificial names on its own. For the purging
to work, the names that the provider chooses must be identical to the ones
that have been given to the generated resources by the `user` type. Otherwise,
Puppet will consider the generated resources to be unrelated to the resources
that the provider finds on disk.

After some brainstorming with
[Rob and Henrik at the triage meetings](https://github.com/puppet-community/community-triage/blob/master/core/notes/2014-10-15.md)
I came up with the following strategy to make sure that each uncommented key
gets a unique resource name:

 1. Keep a counter that is increased for each uncommented key that is encountered.
 2. Each artificial resource name is composed of the full path to the file
 it resides in, and a suffix that is derived from the counter value.

Let's take a look at an example of how this works.

### Resulting behavior

Puppet does not actually care for the structure of the key data. A line needs not
contain an actual public key to be viable for management through the
`ssh_authorized_key` type. The overall format must be correct, but the key itself
can be an arbitrary coherent string. We can there for use the following
`authorized_keys` file to demonstrate.

    ssh-rsa ACTUAL-KEY felix@remote
    ssh-rsa KEY1
    ssh-rsa KEY2
    ssh-rsa KEY1
    ssh-rsa ANOTHER-KEY rcmd-key

Of the three unnamed keys, two are identical. They can still be purged without
problems. You can use the following manifest to start purging:

    puppet apply -e 'ssh_authorized_key { "felix@remote": user => "ffrank" }
        user { "ffrank": purge_ssh_keys => "/home/ffrank/.ssh/authorized_keys" }'

Puppet will generate distinct names for the uncommented keys, so that
the resulting actions look like this:

    Notice: /Stage[main]/Main/Ssh_authorized_key[/home/ffrank/.ssh/authorized_keys:unnamed-3]/ensure: removed
    Notice: /Stage[main]/Main/Ssh_authorized_key[/home/ffrank/.ssh/authorized_keys:unnamed-1]/ensure: removed
    Notice: /Stage[main]/Main/Ssh_authorized_key[/home/ffrank/.ssh/authorized_keys:unnamed-2]/ensure: removed
    Notice: /Stage[main]/Main/Ssh_authorized_key[rcmd-key]/ensure: removed

This way, Puppet can purge unnamed SSH keys from arbitrary numbers of files,
without danger of duplicate resources.
