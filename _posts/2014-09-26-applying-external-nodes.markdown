---
layout: post
category: bugs
tags: puppet bug apply environments
---

Learning environment basics, with a bold venture into the brave new world of directory environments.

### Sudden breakage of puppet apply for some users

Things started harmlessly, with [yet another mailing list post](https://groups.google.com/d/msgid/puppet-users/9323195d-0c03-4137-94f4-de671c06f3c2%40googlegroups.com?utm_medium=email&utm_source=footer).
A user was reporting strange behavior of `puppet apply`.
The resource from the applied manifest was being ignored, apparently.

Misunderstandings about Puppet behaving in an unexpected fashion are commonplace.
But after exchanging some debugging instructions and further issues, it soon became clear that something was going seriously
wrong.
Will Partain then notified me about [PUP-3258](https://tickets.puppetlabs.com/browse/PUP-3258) off-list.
My curiosity was piqued.

### Dissecting the report

Will had left a great bug report! It was a little thin on the description, but came with a complete lab environment for
reproducing it. This was especially valuable to me, having never used an ENC of any kind, much less an ENC script.
The included README file did a good job of describing what was happening, and what should have been happening instead.

My preference for trying buggy manifests is to place them somewhere in `/tmp` and using `puppet apply` on them directly.
Things that are directly involved with master operations, I put in `~/.puppet`, so that I can just launch a `WEBrick` master
without becoming root.  PUP-3258 used some of both.
The issue caused `puppet apply` to make inappropriate use of the values from `puppet.conf` for
some reason. Having a simple Puppet environment prepared in `~/.puppet` comes in handy for this kind of occasion as well.

Will's scenario included

 * a `site.pp` file
 * a module
 * an ENC script

all reasonably stripped down to some very basic content that demonstrated the issue. He triggered it with a rather complex
invocation.

    /usr/bin/sudo /usr/bin/puppet apply \
        --node_terminus exec --external_nodes /home/partain/t/puppet-vapply-bug/enc \
        --modulepath /home/partain/t/puppet-vapply-bug/modules:/etc/puppet/modules \
        --parser future --test --diff_args=-U1 --noop \
        /home/partain/t/puppet-vapply-bug/site.pp

I would ignore `sudo` of course, but what interested me most was whether I could get rid of the module and `--modulepath` flag.
It was simple - the example ENC returned a classes value of `[ foo ]`, so by changing the YAML to

    classes: []

I could just avoid including the module, and shed the need for a special `modulepath`. The bug remained reproducible.

My first hope was that `--parser future` was causing trouble. That would have narrowed down the possible causes nicely.
But removing it didn't change the faulty behavior. I was also reasonably certain that `--test` (not sensible with `apply`
anyway), `--diff_args` and `--noop` had nothing to do with the problem. And sure enough, removing those options had no
effect. What *was* necessary to keep reproducing the issue were the options for external nodes.

    bundle exec puppet apply /tmp/site.pp --node_terminus exec --external_nodes /tmp/enc

That's manageable - now the search could begin.

### Locating the regression

The issue was known to have occurred at some point in between the `3.6.1` and `3.7` releases.
First point of order: Verify that current `master` is still affected. It was.

Since I had no clear idea of where to start, and had a very direct way of reproducing, things
called for a `git` bisection. The best way to run the code from your `git` checkout is via `bundler`,
but there was a slight complication here. The `3.6` and `3.7` tags locked the `rspec` gem
into different versions. So when the bundle is in shape to run one version, it will refuse to run the other.
This is especially annoying, seeing as `rspec` is all about testing, and not about running the application at all.

But no big deal - it's just important to make sure that `bundler` does not skew the result of the
bisection test. This can be achieved by simply calling `bundle update` on each step. I used the following scriptlet.

    #!/bin/sh
    bundle update
    bundle exec puppet apply /tmp/site.pp \
        --node_terminus exec --external_nodes /tmp/enc \
        | grep -q '/tmp/site.pp is in use'

The `grep` was effective because the correct manifest was prepared with the output to `grep` for (using a
`notify` resource). [Happy bisecting!](http://nethackwiki.com/wiki/Tsurugi_of_Muramasa)

    git bisect good 3.6.1
    git bisect bad 3.7.0
    git bisect run /tmp/bisect-test

Yes, this took a while, what with phoning home to [rubygems.org](http://rubygems.org) on each run (and mostly
just accepting the present bundle anyway). Still, as expected, it unearthed the apparently innocuous change
that broke this scenario.

    (PUP-2519) Create fallback environment only for default production

To verify that the identified patch was indeed all that was wrong with `3.7` and above, I like to try and
revert it on `master`. This proved quite difficult. The suspect patch concerned directory environments,
and those have seen quite some stitching prior to the `3.7.0` release. Reverting my change resulted in
conflicts in almost all affected files. Without an understanding of how any of that code worked, it
would have been way too much work to resolve all of that.

So I scrapped that idea and checkout out the offending revision again. Time for some sharp looking at
what was happening there.

### Finding a remedy

Full disclosure, I have no idea how legacy environments worked code-wise, much less the shiny new
directory environments. What's more, I had never traced the code that makes `puppet apply` work.
As such, I had a hard time figuring out the intent of the code changes in question.

Next easiest step: Revert each patch in the offending commit individually, and see if one of them is solely
responsible for the problem.  It turned out to be
[a single line](https://github.com/puppetlabs/puppet/commit/53d391b37f0a5f8f5937fcd7584fb6aae6db424b#diff-3acace79a768858f1f8964ecdb582af2L190) in `lib/application/apply.rb`.

    -    Puppet.override(:environments => Puppet::Environments::Static.new(apply_environment)) do
    +    Puppet.override(:current_environment => apply_environment) do

Still no idea what was going on there. So I first took a look at the `Environments::Static` class definition.
It is [relatively simple](https://github.com/puppetlabs/puppet/blob/53d391b37f0a5f8f5937fcd7584fb6aae6db424b/lib/puppet/environments.rb#L55-91) and allows the creation of a set of environments.
I then got curious what this `apply_environment` was all about, anyway.
The answer is right above its use in the code.

The `apply_environment` is a pragmatic approach to making `puppet apply` work the way it does:
It is basically the [current environment with the site manifest overridden](https://github.com/puppetlabs/puppet/blob/53d391b37f0a5f8f5937fcd7584fb6aae6db424b/lib/puppet/application/apply.rb#L185-188)
to the manifest that was specified on the command line. The next thing I wanted to know was
how the whole environment lookup process worked. I'm not big on the Ruby debugger (even though
[pry](http://pryrepl.org/) is pretty neat), and my first try is often good old `printf` debugging. Sorry, *puts* debugging.

Since the issue revolved around `nodes` somehow (the `exec` node terminus caused the problem, after all),
I wanted to find out how the `Node` class interacted with environments. I added output to `Node#environment=`
to see what values it was receiving. While at it, I also pulled a respective stacktrace using `Kernel#caller`.
This is almost always enlightening, since it helps understanding the way several code pieces work together, especially
when you're not familiar with any or most of them. Comparing outputs, this revealed two things:

 1. The caller is `lib/puppet/indirector/node/exec.rb` when using the ENC script, as opposed to `plain.rb` otherwise.
 2. In the former case, it passes a value of `:production` instead of `production`.

I was still unsure what I was looking at, but the pieces were almost complete. The `exec` variant of the
`node` indirector terminus (Masterzen has a [great article](http://www.masterzen.fr/2011/12/11/the-indirector-puppet-extensions-points-3/)
for understanding what the indirector is all about) sets the environment [if none is returned from the
data source](https://github.com/puppetlabs/puppet/blob/53d391b37f0a5f8f5937fcd7584fb6aae6db424b/lib/puppet/indirector/node/exec.rb#L24). Something about this call was wrong, apparently. I still had no idea why the override in
`Application::Apply` had any relation to this call, but it looked like a good clue.

I compared the `exec` terminus to the way that other termini handle this, and immediately found my answer
in the [LDAP](https://github.com/puppetlabs/puppet/blob/53d391b37f0a5f8f5937fcd7584fb6aae6db424b/lib/puppet/indirector/node/ldap.rb#L38) terminus.
Digging through its [history](https://github.com/puppetlabs/puppet/commits/53d391b37f0a5f8f5937fcd7584fb6aae6db424b/lib/puppet/indirector/node/ldap.rb), it became obvious that his was also [quite recent](https://github.com/puppetlabs/puppet/commits/53d391b37f0a5f8f5937fcd7584fb6aae6db424b/lib/puppet/indirector/node/ldap.rb).
My hunch that sending the environments name (or string representation) to the node was wrong seemed increasingly likely.
[The fix](https://github.com/ffrank/puppet/commit/3451ecdcaff31e786193021cd3f66e77a314edbb#diff-587caec60973192c53e61501ce698080L24) was trivial of course.

So what had been happening? The `exec` terminus initialized the `node` object with the environment `"production"`, the name
of the current environment (which was actually the specially prepared `apply_environment`). This caused
a [lookup](https://github.com/puppetlabs/puppet/blob/53d391b37f0a5f8f5937fcd7584fb6aae6db424b/lib/puppet/node.rb#L74)
among the available environments. This returned the unaltered `production` environment, so that the override
of the `manifest` setting for `puppet apply` was short-circuited. Passing the actual `environment` object would retain
the override. The issue had cropped up because `Environments::Static` had previously hidden all environments away,
so that the lookup of `"production"`, or whatever environment, still did the right thing.

When I realized this, I grew suspicious that we still might have a problem if this default case was not hit. In other
words, I worried that an ENC script that did actually return an environment name would still trigger the issue.
This could be confirmed quickly, but would have to wait for yet another ticket.

### Creating a test

As so often, the greater challenge was to devise a test that would keep this regression from cropping up again
by indicating it through a failure. I looked at the [unit tests](https://github.com/puppetlabs/puppet/blob/53d391b37f0a5f8f5937fcd7584fb6aae6db424b/spec/unit/indirector/node/exec_spec.rb)
for the `exec` node terminus, which does already cover some scenarios concerning environments.
But I wasn't really sure what I would want to test here. The problem was that a symbol or string was being sent
to `Node#environment=` but I couldn't immediately find an expectation that was broken by the bug.
I also felt that this check wouldn't adequately represent the problem we had faced here.

So I took a look at the integration tests for `puppet apply`, and found a test that was
[pretty close](https://github.com/puppetlabs/puppet/blob/53d391b37f0a5f8f5937fcd7584fb6aae6db424b/spec/integration/application/apply_spec.rb#L33-50) to what I had in mind.
I cloned it and extended it to include two manifests and an ENC script, just like my manual test.
I got my extended test to work quickly enough, but then spent quite a while trying
to make it actually use my prepared site manifest while the bug was still present. The key was
to [manipulate the settings for the test](https://github.com/ffrank/puppet/blob/3451ecdcaff31e786193021cd3f66e77a314edbb/spec/integration/application/apply_spec.rb#L81) to include the `manifest` setting.

### Final thoughts

I never really saw a use for ENCs (and honestly, still don't :-) and hence hadn't ever bothered looking at the code.
I would have never grabbed this bug if I hadn't gotten involved in that mailing list thread. It's always a good idea
to stay in touch with our peers in the user base. It's not the first time that I learned something new this way.
It was especially refreshing to finally poke around in environments and the work that has gone into the new (and
awesome) directory environments.
