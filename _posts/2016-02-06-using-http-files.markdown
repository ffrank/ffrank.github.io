---
layout: post
category: features
tags: [ puppet, http, proxy, ssl, security, file, metadata, md5, checksum, mtime ]
---

A quick guide to make the most of HTTP file sources in Puppet.

### Overview

In the [introductory post](/articles/2015/12/21/building-http-support/) about the new
support for `source => "http://..."` parameters in Puppet's `file` resource type,
I promised a follow-up that would explain the technical details of the implementation.
This is not that post.

Instead, seeing as Puppet `4.4` will hopefully be released
soon and make this feature available to you, I will explain the caveats in greater
detail first and add some perspective on how to best circumvent them.

### Apparent incoherence

So you switched your first files to common HTTP sources.
You will likely notice that Puppet keeps presenting a message like the following to you:

    ffrank@fflaptop:~/git/puppet$ bundle exec puppet apply -e'
    file { "/tmp/http-test":
      source => "http://ffrank.github.io/articles/2015/12/21/building-http-support/"
    }'
    Notice: Compiled catalog [...]
    Notice: /Stage[main]/Main/File[/tmp/http-test]/content: \
      content changed '{md5}0e5a75aecc19936a3a80c9e05040269e' \
      to '{mtime}2015-12-27 23:51:55 +0100'
    Notice: Applied catalog in 0.97 seconds

This does not appear to be sensible and might actually look like a bug. Why is an MD5 sum compared
to a time stamp? Well, it's not a bug - it is indeed a feature.

The fundamental problem (see the [previous post](/articles/2015/12/21/building-http-support/))
is that the HTTP server will not supply 
a full set of metadata to the agent. On the contrary - most web servers will not even
include the MD5 sum of the server-side file in the response headers. Fortunately, the
`Last-Modified` header is quite popular, so Puppet can fall back to that.

Still, without explicit instructions, Puppet will use `md5` for checksumming the local copy regardless.
Going from there, it cannot help but compare these incompatible sums and fail.
The undesirable result is that Puppet will download such a file **during each transaction**.

### The workaround

For a clean run with the same resource, add the `checksum => "mtime"` parameter to select
this "checksumming" strategy for the file in question.

    ffrank@fflaptop:~/git/puppet$ bundle exec puppet apply -e '
    file { "/tmp/http-test":
      source => "http://ffrank.github.io/articles/2015/12/21/building-http-support/",
      checksum => "mtime",
    }'
    Notice: Compiled catalog for [...]
    Notice: Applied catalog in 0.76 seconds

This is not even too bad. Comparing time stamps is a quick way to detect upstream changes.
It assumes that clocks are sufficiently synchronous and that there is
no tampering with file contents on the agent side. If either assumption is broken, Puppet will not
be able to recognize diverged state.

This is why you probably don't want to synchronize your files from random remote HTTP servers.

### The Right Way™

Getting Puppet to synchronize files with plain old HTTP servers with the usual convenience
is possible, once we bring MD5 checksums back into the game. Of the more popular HTTP servers,
Apache is the only one that will readily provide the `Content-MD5` header. Puppet will consume
this header to verify local file content.

Still, even with Apache's popularity, most if not all public sites don't choose to enable
this header. How is this helpful then? Well, the best way to take advantage of the HTTP support
feature is to run some local Apache servers to distribute those large tarballs and binary blobs
to your agents.

Setting up an appropriate server is easy. Here is a minimalistic manifest that will do the
footwork:

    include apache
    apache::vhost {
      'puppet':
        docroot => '/var/lib/puppet-files',
        custom_fragment => "ContentDigest on\n",
    }

Apache will now serve files from `/var/lib/puppet-files` and include their MD5 sums in the
`Content-MD5` header. Using appropriate URLs as Puppet file sources will Just Work.

### But why?

You gained the ability to serve files to your Puppet agents without using masters as file servers.
But now you need to provision Apache servers. How is this beneficial?

In most setups, this adds a convenient place to store those pesky large archives on which
some manifests rely for bootstrapping your software. Doing that outside your masters
has at least two benefits.

 1. Most users have all local Puppet modules in version control. This is not where you want
    large tarballs.
 2. Serving large files to agents binds valuable resources that your masters cannot use
    for meaningful work until transfers finish.

This small list is not exhaustive. There are a number of further operational advantages.

### Summary

Use `http://` URLs for Puppet files responsibly. Only point them at upstream servers if the
URL is unambiguous (e.g. pointing at a versioned file. Avoid `puppet-latest.tar.gz` in favor
of `puppet-4.4.0.tar.gz`). If you do this, make sure to specify
`checksum => 'mtime'`.

In most cases, you will want to use this support to download large files from your own
local Apache servers. The vhosts that are responsible for this task will likely be dedicated to it.
Make sure to set the `ContentDigest` setting for the appropriate vhost or directory.
Do *not* set Puppet to use `mtime` in this scenario.
