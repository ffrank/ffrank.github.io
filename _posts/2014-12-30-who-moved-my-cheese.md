---
layout: post
category: bugs
tags: [puppet, bugs, git, rebase, bisect, testing]
summary: Another tale of history bisection in the search for the root of today's problems.
---

Finding out why Travis fails your latest PR - *2 minutes*. Reproducing locally
and figuring out how to fix the issue - *3 hours*. Squashing and rebasing to
master, then finding out that the issue reappeared due to the rebase - *priceless*.

### What is it now?

The work on [PUP-3341](https://tickets.puppetlabs.com/browse/PUP-3341)
had basically been finished weeks earlier. The mailing list had
[consented](https://groups.google.com/d/msgid/puppet-dev/5467E757.2050904%40Alumni.TU-Berlin.de)
(or something like that) and implementation had gone rather swimmingly.
As always, the tests had been the most challenging aspect. Imagine my
chagrin when Travis failed on my branch.

I quickly discerned where I had gone wrong and touched the code up. A local
spec run was succesful. Yay! Do the dance:

{% highlight bash %}
git commit -am fixup
git rebase -i origin/master
{% endhighlight %}

Squashed the fix into the appropriate commit in my branch and pushed to
GitHub. This should have been it.

Upon revisiting the [list of my PRs](https://github.com/puppetlabs/puppet/pulls/ffrank)
I was disappointed. Had it not worked? Were my changes not pushed? Nope, the checksums
matched my local repository. Had I made a mistake when verifying the fix locally?
Engage time machine:

{% highlight bash %}
git reflog
{% endhighlight %}

Checking out the state before that rebase cleared things up: Yes, my fix to the spec
had worked. No, it would not work after the rebase. Some upstream change re-broke
my test. What fun.

### Running bisection

I had presented a bisection in an [earlier post]({% post_url 2014-09-26-applying-external-nodes %})
but let's do this again, because

* it bears repeating
* a local branch breaking courtesy of upstream is especially nasty

The idea for bisecting this kind of issue is based around this workflow:

1. Check out the respective commit to test.
2. Apply the branch.
3. Run whatever test and save its result.
4. Clean the WORKDIR so that the next iteration can commence.
5. Return the saved result.

An easy way to apply changes from a branch to the current HEAD,
with the option to trivially undo that, is `git-cherry-pick` in no-commit mode:

{% highlight bash %}
#!/bin/bash

git cherry-pick -n 3c8fb2417 .. 253f52296
bundle exec rspec spec/integration/application/apply_spec.rb:44
RC=$?
git checkout -f
exit $RC
{% endhighlight %}

The actual test is running `rspec` through Bundler, of course. The hashes
belong to the head of my branch when it still worked, and its branch point
at the time.

### Resolution

For the curious, I once again ended up with a change of [Josh's](https://github.com/jpartlow),
which is not surprising, seeing as he has done lots of work on the environment
code throughout the whole switch to directory environments. Full disclosure,
I have no clear idea how [968695](https://github.com/puppetlabs/puppet/commit/96869598009e1e39b122e65284006a428f91d97c)
broke [this test](persona://github.com/ffrank/puppet/blob/6c3ab52c20f2777d2ee6138e4c786405c5560f9c/spec/integration/application/apply_spec.rb#L44-70),
but suspecting that the changes in `test_helper.rb` might be involved, I tried
not fiddling with the `Puppet[:environmentpath]` setting in favor of working
with what `rspec` gave me. [This](https://github.com/ffrank/puppet/blob/b161fb21731701e5485dbbbd4f7678ecb7481009/spec/integration/application/apply_spec.rb#L44-67) is what I ended up with.

A bit of an anticlimax, I know. In other cases, this mode of bisection yields
more helpful results. I probably won't keep you posted though - repitition
is good, but let's not get carried away. How about "more on this story as
something outrageously amazing happens some time in the future, which I cannot
*not* post about".
