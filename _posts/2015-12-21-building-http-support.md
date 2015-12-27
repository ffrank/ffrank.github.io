---
layout: post
category: articles
tags: puppet bugs features code
---

An account of the creation of yet another long awaited feature.
First part of two.

### Overview

One of the most requested capabilities for Puppet has been to support code
such as the following:

    file {
      '/opt/jdk.tar.gz':
        source => 'http://filerepo.local/java/jdk.tar.gz',
    }

Retrieving files (especially larger archives) via HTTP is a very convenient
practice. People have been doing it through custom `exec` resources and
defined types all over the place, for years. There are
[several](https://forge.puppetlabs.com/camptocamp/archive) sophisticated
[modules](https://forge.puppetlabs.com/nanliu/staging) that bring this
capability along.

Still, all these approaches lack some of the convenience that Puppet's `file`
type brings to the table.

 * The `file` type will use checksums or time stamps to decide whether
 a file needs downloading again.
 * With `file` resources, Puppet can back files up locally or to a master
 node prior to overwriting.

There may be more benefits of which I am not even aware.

Implementing this capability appears to be straight forward enough:

 1. Allow HTTP URLs for the `source` property
 2. Extract metadata from the HTTP headers
 3. Connect to a webserver in lieu of the Puppet fileserver for retrieval

I believe that it was primarily the second step that kept the Puppet Labs
engineers from just forking off some time at some point. HTTP headers don't
carry information that is quite as rich as Puppet's fileserver metadata.
Fortunately, this gap can be filled with a few compromises.

### Designing the semantics

The agent synchronizes local files with content from the master in two steps.
First, an API call is made to retrieve the master's file metadata. This data
is sufficient for the agent to decide whether it also needs to retrieve the
contents. If so, it makes a separate call to the master.

Allowing Puppet to download content through HTTP and HTTPS is simple to the
point of almost being trivial. Subsystems such as the Puppet Module Tool
(i.e., `puppet module install` and friends) already implement a client.
The file metadata portion is all the more challenging. The Puppet fileserver
presents the following aspects to the agent:

 * `path`
 * `owner`
 * `group`
 * `mode`
 * `file type`
 * `destination` (applicable to symbolic links)
 * `checksum` (along with the `checksum_type`)

The `path` is actually passed by the agent. It holds no information about the
server side file. `owner`, `group` and `mode` are typical file system attributes.
As for the `type` of files that Puppet will manage, there are but three
supported alternatives:

 * plain file
 * directory
 * symbolic link

Finally, there are a number of different `checksum` algorithms that Puppet supports:

 * `md5` (and `md5lite`)
 * `sha1` (and `sha1lite`)
 * `sha256` (and `sha256lite`)
 * `mtime` and `ctime`
 * `none`

The `md5`, `sha1` and `sha256` types are actual hash sums that can safely (and
in the case of `sha256`, securely) represent a whole file's content succinctly
and in an easily comparable fashion. The respective `lite` variant will only consider the first
half kilobyte (or rather, kibibyte) of data. This will safe some computing power
(lots of it in case of large files), but it leaves your agents prone to miss some
upstream changes such as appending new data to an old file.

Selecting `none` is usually not sensible for files that are getting synchronized
from any kind of fileserver, because each agent run will cause another download
of the whole file. `mtime` and `ctime` are more sensible: You get a light-weighed means
of estimating whether a file was changed upstream after downloading it. (General
piece of advice: You almost never want `ctime` for most practical queries. Just
stick to `mtime`.)

### Enter HTTP

So Puppet supports a rather rich set of file metadata. This is only natural,
given that file management is probably one of the most frequent activities
of the Puppet agent. Most of these properties are not represented in the
HTTP standard at all. Web servers can choose to include information beyond
the specification in custom headers. However, the currently predominant web servers
don't, for the most part.

Specifically, `owner`, `group` and `mode` are not *meant* to be communicated
to browsers and other HTTP clients. As for `file type`, its different valences
can, in theory, be mapped to different HTTP responses:

 * plain files, represented by regular old responses with code 200 and a body,
 which comprises the file content
 * directories, transferred in the form of directory indexes. Technically,
 these are plain resources that contain links to other resources (directory
 entries) in their body. 
 * symbolic links, along with the `destination` attribute. Their HTTP equivalent
 is the 30X redirect. It carries a destination in the `Location` header.

However, actually mapping redirects to symlinks would be very awkward. First off,
the redirect location will point to a path that is only meaningful in the context of
the HTTP server it references. It cannot be safely assumed that the *path* portion
of the target URL can be used as a symlink target on the agent system.
What's more, the user will often expect Puppet to actually follow redirects,
so that "friendly" URLs, shortened URLs, and so forth, just work.

As for directory indexes, adding special handling for those would make Puppet
an actual web spider. I can see use cases for that, but then synchronizing
whole directory trees is not a very common nor commendable practice. For these
reasons, I settled for no special handling of neither directory indexes nor
redirects. All HTTP content is implicitly considered to represent plain files.

### Checksums from headers

We have discussed the mapping of almost all metadata fields to HTTP headers,
except the checksum. The good news is that `md5` sums at least
used to be supported in the form of the `Content-MD5` header. It was
[removed](http://tools.ietf.org/html/rfc7231#appendix-B) from the standard
because it was "inconsistently implemented with respect to partial responses",
but Apache will generate it for static resources when the `ContentDigest`
option is set. For generated content and alternate server software,
we're still stuck with relying on an alternative.

The `sha` variants are not provided by any HTTP standard. When push comes
to shove, metadata can always be construed as having a `none` type checksum
(i.e., there is no checksum whatsoever). However, most webservers include
`Last-Modified` headers in their responses if possible. As I mentioned earlier,
`ctime` is rarely appropriate. It is also a bad fit for the HTTP header, so
I opted to map it to the `mtime` checksum type. It is the most common type
of "checksum" that is extractable from HTTP headers.

Unfortunately, the `file` type is hardcoded to try and use an `md5` checksum
by default. It will not automatically fall back to a different type, even when
the server won't present a suitable checksum. In practice that means that
the following resource leads to a result that is less than desirable:

    file {
      '/opt/jdk.tar.gz':
        source => 'http://filerepo.local/java/jdk.tar.gz',
    }

Even when the `filerepo.local` server does not supply a `Content-MD5` header with
the requested resource, the Puppet agent will compute the `md5` sum of the local
file's content. It cannot possibly match the `mtime` type checksum that is
derived from the HTTP headers. Therefor, Puppet will insist in continuously
synchronizing the file with the upstream server, until the checksum type is
changed explicitly:

    file {
      '/opt/jdk.tar.gz':
        source   => 'http://filerepo.local/java/jdk.tar.gz',
        checksum => 'mtime',
    }

This effect is a little hard to explain, but will become more obvious during the
next part. This post is part one of two. The second installment is to appear
soon, and will deal with the technical details of the feature implementation.
