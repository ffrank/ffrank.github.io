---
layout: post
category: features
tags: puppet file http indirector
---

Many users (myself included) would like to specify `source => http://...`
for `file` resources in Puppet manifests. It turns out that the existing
infrastructure in the Puppet core makes this quite easy to implement.

### File serving in Puppet

Among the most basic functions of any configuration management system is the
central maintenance and programmatic distribution of various configuration files.
Data files such as binary applications or tarballs are frequently managed as well.
Puppet is no exception to this rule.

The backbone of Puppet's management paradigm are discrete resources with
current and desired states. A convenient way to describe a file's state
is to provide the Puppet master with an actual file that the agent should mirror exactly
to the system it manages.  In this mode of file management, the user will supply a URI to point the agent
to the prepared file. The `puppet://` URI scheme is prevalent. Its semantics
depends on the mode of Puppet operation.

 * URLs that **include a host name**, such as `puppet://server/path/to/file`,
   will always make Puppet look up the target state on the named Puppet master,
   through Puppet's fileserver API
 * URLs **without a host name** encountered by the **Puppet agent** locate
   the source file on the master that generated the current catalog
 * URLs **without a host name** that are passed to **puppet apply** lead to
   source files being retrieved from the local configuration repository

So far, the only alternative supported URI scheme has been `file://`.
A resource that uses such a file source makes Puppet sync the managed
file with another file from the local filesystem tree. Such URIs can
be specified as a file path.  In other words, the following are
synonymous.

    file { '/etc/motd':
        source => "file:///var/shared/motds/$fqdn"
    }

    file { '/etc/motd':
        source => "/var/shared/motds/$fqdn"
    }

...with the latter being more common.

As an aside, it would appear that historically, the only available form of
management was the `content`
property, and that the `source` parameter was apparently added as an afterthought.
This is what [Charlie Sharpsteen](http://www.sharpsteen.net) concluded upon
researching [a bug](https://tickets.puppetlabs.com/browse/PUP-1208) that is
actually closely related to the feature I'm going to discuss in this article.

More on this story as it develops. Now back to the topic at hand.

### Metadata

For efficiency, the process of fileserving is split into two discreet phases.
For each file that is synchronized from the server, Puppet will first
[retrieve its metadata](https://github.com/ffrank/puppet/blob/6b96e48ea3fb060ee3d6ac6fcc75f9b2b34bcb76/lib/puppet/type/file/source.rb#L178). This puts much less strain on the network than the actual
[download of the file](https://github.com/ffrank/puppet/blob/6b96e48ea3fb060ee3d6ac6fcc75f9b2b34bcb76/lib/puppet/type/file/source.rb#L103).

Metadata describes each file in terms of

* owner
* group
* mode (permissions)
* content checksum
* type

The checksum is usually created using the MD5 algorithm. At the time of
writing, the aforementioned [bug](https://tickets.puppetlabs.com/browse/PUP-1208)
still causes all other available algorithms to not actually work.

The path to the managed file on the agent system is also part of the metadata as
used by the agent.
However, it is not received from the fileserving component of the Puppet master.
This value is defined in the manifest and included in the catalog by the compiler.

### Local and remote sources

The agent always uses Puppet's indirector framework to receive metadata
for `file` resources that make use of the `source` attribute. If you would like
to learn more about the indirector, I recommend [Masterzen's
excellent post](http://www.masterzen.fr/2011/12/11/the-indirector-puppet-extensions-points-3/)
on the topic.  In short, the
indirector models arbitrary data paths for all parts of Puppet to get information
from various sources.

Once the metadata has been received and compared to the filesystem, Puppet might
decide that a sync operation is necessary. In this case, it will also request
the actual file content. This is done via indirector as well.

If the `source` is specified as a URL with the `puppet://` scheme, both indirector calls
map quite directly to the REST API. The desired data is handed out by the master,
or whatever alternate fileserver was specified, in the server response.
For `sources` that use URLs with the `file://` scheme or just a filesystem path,
the indirector terminus will not engage in any network communication at all.
It can gather the required metadata information through the `stat` system call and the
respective checksumming function.
If a sync is necessary, file contents are copied locally.

The indirector [picks](https://github.com/ffrank/puppet/blob/6b96e48ea3fb060ee3d6ac6fcc75f9b2b34bcb76/lib/puppet/file_serving/terminus_selector.rb#L15-26)
the appropriate terminus according to the URI scheme.

### HTTP servers as remote sources

In the introduction, I claimed that it was pretty simple to add HTTP support
to the file `source` attribute. The reason for that is the indirector structure
that I just described. Basically, all it takes is adding a couple of termini, right? Right.

Where to start? First things first: Puppet needs file metadata to determine
whether the local file needs to be synchronized to the server's data. Common
HTTP servers such as Nginx or Apache will not offer the kind of information
that the Puppet agent usually manages. File ownership in terms of users and groups
are of no concern to most HTTP clients, nor are permissions or the
content checksum.

HTTP resources do carry some meta-information in their headers, such as the
[last time of modification](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.29)
or the [file size](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.13).
The headers can be retrieved using the HTTP `HEAD` request. So that's what
the indirector can do to receive a limited set of metadata. Unsupported fields
will have to remain unmanaged by the `source` parameter - if the manifest
does not specify an `owner` or `mode`, then the agent will not care about those
attributes of the local file. Defaults will be used for newly created files.

File content is available as part of the response's body, requested through
the HTTP `GET` method. This is all that the indirector really needs to do
in order to fetch data for the agent to synchronize.

### Limitations of the protocol

The previous section already described how most of the relevant file attributes
are not available via HTTP. There are some other implicit limitations that arise
from the protocol.

HTTP resources can only ever be mapped to plain files. Symbolic links cannot
be sensibly served. The closest equivalent in the protocol is the redirection.
But it is not suitable for representing an actual link, because the user would
expect Puppet to follow all HTTP redirects. After all, a server side reorganization
can always make it sensible to add new redirections, and clients should just cope
with that.

Directories can be served in the form of the respective index. It is even feasible
to implement a spider algorithm in Puppet to allow the recursive synchronization
of a whole directory tree to the agent system. But I decided to put this beyond
the scope of the initial implementation. It can be subject to another feature
request down the line. As it stands, Puppet will consider a directory index as
text file content, which is perfectly acceptable for the time being, I feel.

Also on the bright side, the protocol does specify a
[checksum header](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.15)
that can be used to allow for actual synchronization of content based on
this hash value. Thanks to [Ken Barber](https://uk.linkedin.com/in/kennethbarber)
for pointing this out to me. It is not mandatory for servers to include this header,
and most will not bother. But users who set up servers for the express purpose of
serving content to Puppet agents will want to enable it in order to enrich
the available file metadata.

### Conclusion

Puppet's own facilities are quite fit for the addition of HTTP as an available protocol
for file synchronization. The nature of the protocol imposes some limits on the
number of availabe file attributes. Implementing a simplified
algorithm based on the `mtime` should be straight forward.

Seeing as the theoretical background ended up being quite a mouth full,
I'm postponing the overview of the implementation details to a later post. See you there!
