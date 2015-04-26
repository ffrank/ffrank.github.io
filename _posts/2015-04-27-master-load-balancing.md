---
layout: post
category: bugs
tags: puppet master http ssl loadbalancing
---

Brief explanation of load balancing approaches
with a discussion of applicability to the Puppet
master - debunking my own ad hoc ramblings.

### Background

At his amazing presentation of the new features in Puppet 4 at
Puppet Camp Berlin 2015, [Martin](https://twitter.com/tuxmea)
explained how masters can be configured to perform only a subset of
the available master services. This way you can have, say, a single
CA server, some file servers and a swarm of manifest compilers.

Someone in the audience asked how to use this, since typically you
address the master through just the one domain name (e.g. puppet.example.com).
I spoke up to suggest an approach using a layer 7 load balancer
that could be set up to map API endpoints to master servers.

Getting such a cluster to work would be simple enough, for most services,
and you could build a prototype in a manner of minutes (provided
you have some experience with [HAproxy](http://www.haproxy.org/)).
But it will not work with Puppet, as I have come to realize in the
meantime. Let me explain why that is.

### Load balancing in general

The two most ubiquitous approaches to load balancing are doing it
on layer 4 (transport) or layer 7 (application). The former is simple,
fast, requires few resources and is nearly universally applicable.
It lacks the versatility to build more complex setups, however.
That's because the load balancer has no knowledge of the data payload
whatsoever. On layer 7, a load balancer can make decisions based
on protocol information such as HTTP headers or RDP cookies, and can
even alter the packets it forwards.

![Diagram of layer 4 load balancing](https://cloud.githubusercontent.com/assets/436765/7339436/e8b2b086-ec6e-11e4-9719-83cf13e42235.png)

In a layer 4 load balancing setup, all servers typically have the
same role and exhibit the same behavior. The load balancer is
perfectly free to decide where each client connection should be
routed.

Some applications only work correctly when a given client
never switches from one server to the other. This is usually the case
when the server holds some session data locally, with no exchange
of such data among servers in the load balancing group. Load
balancers are frequently required to ensure *session persistence*
to accomodate such applications.

In layer 4 load balancing, this persistence can typically only be achieved
by mapping each client IP address to one of the clustered servers.
This is inflexible, because it cannot take runtime data such as
respective server load into consideration. Assuming random
client addresses, this approach still yields a good balance among
servers, though.

With a layer 7 load balancer, there are more sophisticated methods
of achieving session persistence. For example, an HTTP load balancer
can inspect the request and response headers passed between clients
and servers. It can store a table of session identifiers that it
finds in cookies. It "knows" which session is kept on which server.

![Diagram of layer 7 load balancing](https://cloud.githubusercontent.com/assets/436765/7339438/e8b63544-ec6e-11e4-9a66-b790178ea586.png)

In turn, this allows more liberal balancing schemes. The load balancer
can prefer the least loaded server for new sessions, or apply
arbitrary rules for mapping requests to servers.

Such a load balancer can even serve different applications,
and route requests to the respective server or servers. Examples for
this are

 * several subdomains, e.g. *redmine.example.com*
  * the balancer will route requests according to the `Host` header
 * several services that make up an API
  * the balancer routes requests according to the URL path
 * special appservers are deployed with debug settings
  * load balancers route to these servers only if
    the developer includes a special custom header in the request

![Diagram of load balancing rules](https://cloud.githubusercontent.com/assets/436765/7339426/82de8f50-ec6e-11e4-9d24-09ffe0c205b0.png)

Because appservers will sometimes need to know the IP of the original
client, load balancers will often include it in forwarded requests
as additional information. They will add or modify the `X-Forwarded-For`
or `X-Real-IP` header for this purpose. Web servers like Apache, NGINX
or Tomcat are able to interpret the value from such headers as client
addresses. They need to, because at the network level, their immediate
client is always the load balancer.

(Layer 4 load balancers have means to report the original client IP
as well, but this is not in the focus of this post.)

### SSL offloading

Rules and transformations on layer 7 are only possible if the load
balancer can actually inspect the data payload of the connections.
This places limitations on the end-to-end encryption that can be used.
A common solution is to to implement SSL offloading.

![Diagram of SSL offloading](https://cloud.githubusercontent.com/assets/436765/7339439/e8b763a6-ec6e-11e4-809b-95684e8ecda3.png)

The actual webserver does not use SSL at all. It only sends and receive
plain HTTP. Encryption and decryption are handled by one or more separate
SSL termini. A great example of a pragmatic software solution for this
is [stunnel](https://www.stunnel.org/).

To allow layer 7 load balancing with encryption, the load balancer would
be situated between the servers and the SSL termini. For convenience and
simplicity, some load balancers can act as SSL offloaders themselves.
Both NGINX and HAproxy excel at this.

![Diagram of layer 7 load balancing with SSL](https://cloud.githubusercontent.com/assets/436765/7339437/e8b3afc2-ec6e-11e4-8b33-0a4bf185bc79.png)

### (Not) combining this with Puppet

The above setup could not be applied to Puppet masters, however. The master
**will not** accept unencrypted HTTP connections under any circumstances.
This is not the reason why such a construct cannot work, though. Both NGINX
and HAproxy will happily *re-encrypt* the packages on their way to the master.
The load balancer will effectively act as a Man In The Middle. This is even a viable approach
in some scenarios. It works because (unlike a rogue proxy) the load balancer
has the server certificate's private key at its disposal.

Even given this amount of trust, a load balancer still cannot put itself
between a Puppet agent and master. This is because to Puppet, the **client
certificate** (the one from the agent) is just as important (or more so)
as the server certificate. It is the basis not only of trust, but also
for compiling the appropriate catalogue for each given agent. Even in
file serving, it can be decisive, if your masters use ACLs to control
agent access to some mounts (granted, this is quite an edge case).

When re-encrypting traffic, the load balancer can do so with its own
certificate only. It cannot assume the identity of the original agent.
If such a thing was possible, this would have grave implications for
the security of any Puppet setup. Nor can the master forward the agent's
name as supplemental information (such as a custom HTTP request header),
for the same reason - it would imply that impersonating other agents
was trivially simple.

### Summary

While the implementation of Puppet's networking through a RESTful API
would lend itself rather well to rule based mapping and distribution,
this is not feasible due to the requirements for mutual cryptographic
authentication of both the master and the agent. Distribution as sketched
in the introduction would likely need to be built into Puppet itself.
Given the indirector structure that wraps the API internally, this does
not appear to be readily possible, either.

If you'd like to divide your master services, you will likely need to
address them explicitly.

1. Get the certificate from `puppetca.example.com`

        puppet agent --test --waitforcert 600 --server puppetca.example.com

2. Get the catalog from `puppet.example.com`

        puppet agent --test --server puppet.example.com

3. Get files from `puppetfiles.example.com`

        file { '/etc/ldap-key':
            source => 'puppet://puppetfiles.examples.com/modules/...'
        }

Each of these services could comprise a layer 4 load balancing setup
if desired.
