---
layout: post
category: bugs
tags: [ mgmt, etcd, debugging ]
summary: Finding out while the embedded etcd server in mgmt 0.21 will not start, by reading code and documentation
---

In the [previous
post]({% post_url 2020-03-01-attempting-some-resource-collecting %}) I ran into
an issue with the YAML example graphs for the etcd functionality not really
working anymore. Coming up with a solution in these graphs was not hard (and in
fact proved an opportunity to learn a few things along the way). However, I did
not manage to get the cluster up and talking regardless.

I accepted the challenge and sat down to try and find out what was wrong with
etcd.

## A first look at the code

The embedded etcd code lives in the [etcd
subdirectory](https://github.com/purpleidea/mgmt/tree/master/etcd) of the mgmt
source code. In `etcd/etcd.go`, the `EmbdEtcd struct` represents the
"embedded server and client etcd". This file imports the "clientv3" package
from etcd as `etcd`. As the first errors I had encountered seemed to originate
with the client, I looked at this first.

The [online documentation](https://godoc.org/github.com/coreos/etcd/clientv3)
for the clientv3 package is
quite good, complete with a simple code example and everything. Matching this
with the actual implementation in mgmt is rather daunting, however. Still, it
gave me the idea of trying to build a very simple client to mgmt's server, so
that I would have a more limited code base to pry open.

Better yet, I recalled that [the post that introduced the embedded
etcd](https://purpleidea.com/blog/2016/06/20/automatic-clustering-in-mgmt/)
mentions that you can run simple tests using `etcdctl` such as

```
ETCDCTL_API=3 etcdctl --endpoints 127.0.0.1:2379 member list
```

New step 1: See if we can talk to the embedded server this way in the first
place.

## Etsy Dee Cuddle

In order to get a suitably modern `etcdctl`, I followed the instructions in the
article linked just above. In fact, mgmt bundles an appropriate version of etcd
in its vendor directory. I'm not entirely certain that switching into the
`vendor/go.etcd.io/etcd` directory (a git submodule) is an acceptable way to run
the build, but it sure worked.

My approach was to only run the original seed server of mgmt, in order to target
that with a simple query from `etcdctl`, connecting to the API port of the
embedded etcd server.

```
mgmt run --hostname h1 --ideal-cluster-size 3 yaml examples/yaml/etcd1a.yaml
```

I found myself a little surprised that `etcdctl` noped out with a message that
was quite reminiscent of what I had seen mgmt do on repeat while trying to
contact the original cluster member.

```
ETCDCTL_API=3 ./vendor/go.etcd.io/etcd/bin/etcdctl --endpoints 127.0.0.1:2380 member list
{"level":"warn","ts":"2020-03-08T19:25:00.394Z","caller":"clientv3/retry_interceptor.go:61","msg":"retrying of unary invoker failed","target":"endpoint://client-f55a7a6d-3208-45ef-9a7a-f28f55805a05/127.0.0.1:2380","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest connection error: connection error: desc = \"transport: Error while dialing dial tcp 127.0.0.1:2380: connect: connection refused\""}
Error: context deadline exceeded
```

This was promising insofar as it suggested to me that the problem was probably
with the original seed server after all, not with the second mgmt instance that
tries to join the cluster. Sure enough, when examining the output of the seed
process, it turned out that it was failing the embedded server start after the
60 second timeout now.

## Imitating the server

In order to take a better look at what was going on with the seed server, I
searched for one of its log messages, the one appearing before it ran into the
timeout failure.

```
git grep "server: starting"
docs/faq.md:### On startup `mgmt` hangs after: `etcd: server: starting...`.
docs/faq.md:etcd: server: starting...
etcd/server.go: obj.Logf("server: starting...")
```

So this tells us two things. For one, this is the "server" component from
`etcd/server.go`. This is where I started reading code, because I had totally
missed the other point: There is already an FAQ entry for this current problem.
It advises that the local state directory of the embedded etcd server is
probably corrupted. This seems likely enough; after all, my first attempt did
not go quite as disastrous.

This gives me an idea, but allow me to first regale you with the account of my
immediate investigation of what was going on. I first went and studied the
implementation in said `server.go` file. It uses the `embed` type from the
[etcd package](https://godoc.org/github.com/coreos/etcd/embed). The
documentation for this package comes with some sample code to spin up the most
simplistic embedded etcd server. Lo and behold, all the parts of this code
sample can be located in `server.go` [in mgmt](https://github.com/purpleidea/mgmt/blob/3bce96bbd509ad5ffb35ead52128dec5c1a67abf/etcd/server.go#L139`).

So why does it not start? (Mind you, at this point I had not seen the FAQ, and
was not yet on to the local state directory as the likely culprit.) My plan was
simple: Build and run the example code from the etcd documentation, and then
make it resemble mgmt's implementation more and more. See where it breaks.

To get the embedded server to run, I copied the example code to a file
"demo_etcd.go" in the root directory of the mgmt code. I added a `package main`
line at the top and ran it through `go run`.

The example itself worked quite well. The embedded server came up and talked to
`etcdctl` without an issue. I managed to replicate some of mgmt's configuration
settings as well, but not all of them. As I found myself printing the entire
`cfg` structure from both mgmt and my example, and comparing them, one
particular parameter caught my eye. I made an adjustment.

```
-       cfg.LogLevel = "error" // keep things quieter for now
+       cfg.LogLevel = "info"
```

I had marveled at how much less noisy mgmt 0.21 felt compared to the earlier
releases I was used to. I suspect that this setting plays a big role. Changing
this gave me a lot more output to work with, and I finally set back a little
and wrote this blog post.

## Wrapping up

I have not yet had a chance to take a good look at the output generated from
the etcd server. However, while producing this write-up, I had a new thought
that might explain what went wrong here, and why the embedded server is now
permanently broken for me.

This post is already fairly long and pretty rambly. I will stop here and explore
my new theory in yet another post in the near future.
