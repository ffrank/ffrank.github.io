---
layout: post
category: features
tags: [ puppet, http, proxy, ssl, security ]
summary: A non-exhaustive summary and outlook on Puppet's HTTP client capabilities.
---

### Making HTTP requests from within Puppet

Puppet without HTTP support is unthinkable. Most if not all communications
between the different parts of a Puppet setup use REST APIs. Puppet modules
from the Forge are retrieved through HTTPS exclusively.

There are convenient methods available in the existing code base, which
can be used by [new features](https://tickets.puppetlabs.com/browse/PUP-1072)
that need to perform HTTP requests. There are some details and limitations
to keep in mind, though.

### The Connection Pool

The preferred implementation around
[net/http](http://ruby-doc.org/stdlib-2.0/libdoc/net/http/rdoc/Net/HTTP.html)
is the [HTTP connection pool](https://github.com/puppetlabs/puppet/blob/master/lib/puppet/network/http_pool.rb).
It allows you to conveniently fetch a new or existing
[HTTP object](https://github.com/puppetlabs/puppet/blob/9ca4d221864beba82272fe4146255741f99906b1/lib/puppet/network/http_pool.rb#L33)
and make all manners of [requests](https://github.com/puppetlabs/puppet/blob/9ca4d221864beba82272fe4146255741f99906b1/lib/puppet/network/http/connection.rb#L73).

You can just create such objects for accessing HTTPS servers (the `use_ssl`
parameter even defaults to `true` to that end) as well as using unencrypted
HTTP. However, it is important to keep in mind that Puppet will only trust
servers whose certificates are signed by Puppet's own CA. In particular, this
means that such a connection cannot be established to your webserver that uses
a valid certificate that you purchased from a common vendor, and which is
trusted by most clients such as browsers or mobile apps.

This is somewhat limiting, but serves as a nice safety mechanism. Developers
don't need to keep track of just which certificates are trusted for each single
pooled connection that their code uses. *All* connections are limited to the same
single CA.

### The proxy class

It is due to this limitation that the Connection Pool is not suitable for making
the `puppet module` face work. The primary purpose of this subcommand is downloading modules from the
Forge servers, which obviously have to use a common certificate that cannot be
verified using the CA from any Puppet setup out there.

Enter the [HTTP proxy](https://github.com/puppetlabs/puppet/blob/master/lib/puppet/util/http_proxy.rb)
class. It is part of `Puppet::Util` because it does not implement any network
communication itself. It's but a helper to
[properly configure](https://github.com/puppetlabs/puppet/blob/9ca4d221864beba82272fe4146255741f99906b1/lib/puppet/forge/repository.rb#L133)
a vanilla [Net::HTTP::Proxy](http://ruby-doc.org/stdlib-2.0/libdoc/net/http/rdoc/Net/HTTP.html#method-c-Proxy)
object. Doing this has two advantages:

 1. Puppet respects the system's proxy settings when accessing the forge.
 2. It is quite safe to trust [common certificates](https://github.com/puppetlabs/puppet/blob/9ca4d221864beba82272fe4146255741f99906b1/lib/puppet/forge/repository.rb#L137-142) for a single connection, since there is no real risk of accidental reuse

### Adding HTTP support to the file type

As nice as it would be to tap the flexibility and power of the Connection Pool
for file resources that Puppet is supposed to retrieve once PUP-1072 is completely
implemented, you definitely want the semantics of the proxy approach.
However, there are more advantages to using the HTTP pool than just shorter
method parameter lists.

One particular property that is irrelevant to the Forge client but crucial to
a generic HTTP downloader is the ability to seamlessly
[follow redirections](https://github.com/puppetlabs/puppet/blob/9ca4d221864beba82272fe4146255741f99906b1/lib/puppet/network/http/connection.rb#L163).
Without it, the user would be required to use canonical URLs for all resources.
With some services, this is not even possible, because physical locations of
data are volatile and redirections are managed dynamically.

I brainstormed the issue with [Kylo](https://twitter.com/kylog) and we championed
two possible approachs:

 1. Cautiously add the ability to make some of the pooled HTTP connections unsafe
    wrt. the verification of server certificates. Note that "unsafe" is a very relative
    term in this context, because SSL certificates are still validated in the same
    way that each (sane) HTTPS client will. But many requests that and agent sends
    are directed at a master or similar component. It is paramount that a master
    cannot be impersonated using a cheap vendor certificate.
 2. Move the redirection logic to a general helper class, for use from both the
    HTTP pool code, as well as new users such as the file type and its properties.

The second variant turned out to be non-trivial because the current method is
[rather](https://github.com/puppetlabs/puppet/blob/9ca4d221864beba82272fe4146255741f99906b1/lib/puppet/network/http/connection.rb#L171)
[specific](https://github.com/puppetlabs/puppet/blob/9ca4d221864beba82272fe4146255741f99906b1/lib/puppet/network/http/connection.rb#L209-231)
to the `Puppet::Network::HTTP::Connection` class. Not only is the first approach
quite difficult due to the same structures. It is also tricky to get right, because
"unsafe" server connections must never bleed into unintended code paths. Such an
issue could easily cause all kinds of security problems. Call me chicken, but
I wouldn't want to be responsible for introducing this kind of hole in the code.

### Conclusion

For the time being, I settled for the approach that we had initially rejected out
of principle: duplicating the existing redirection code for use with standalone
`HTTP::Proxy` objects. For the price of a few lines of redundant code, we avoid
a couple of potential pitfalls and also get
better adherence to [KISS](http://en.wikipedia.org/wiki/KISS_principle). This
may seem petty and lazy, but it's a hard truth that

 * we have more than our share of complexity in the Puppet code base
 * I can be lazy at times

Look forward to a simple and robust implementation of HTTP file sources in core
Puppet. We're almost there.
