---
layout: post
category: misc
tags: [ mgmt, etcd, local state, history ]
summary: Running tests with mgmt's embedded etcd server, learning about its local state directory
---

In the [previous post]({% post_url 2020-03-09-getting-etcd-back %}), you got to
watch me try and get to the bottom of a suspected bug in mgmt or its embedded
etcd server. What I found was an FAQ entry to match my issue, and some limited
knowledge about etcd's go packages for embedding servers in other software.

The FAQ information gave me an idea about what I was seeing. In my first attempt
of building a cluster, I had quite faithfully reproduced the commands from [the
original
post](https://purpleidea.com/blog/2016/06/20/automatic-clustering-in-mgmt/) that
introduced the clustering feature in mgmt. Notably, none of the mgmt invocations
use the `--tmp-prefix` or other related switches.

The mentioned FAQ entry suggests that the local state directory of etcd got
corrupted. What or who would corrupt it? Starting mgmt had never caused issues
until I tried the clustering. Was it this attempt? Had it caused lasting damage
that kept the embedded server from launching at all afterwards?

## Planning

Here's my theory: When running the clustering example, all three mgmt processes
started writing etcd state to `/var/lib/mgmt/`. This caused immediate data
corruption, which made it impossible for the cluster to come up. Furthermore,
mgmt cannot recover from this on its own, so the embedded etcd will not start
anymore. The commands worked in the past because the behavior used to default to
what `--tmp-prefix` does now, or something else that prevented this overlap.

In order to test this theory, I will try the following steps:

1. Run the seed server with `--tmp-prefix`, which should remedy the base issue
of etcd not starting at all.
2. Removing the content of `/var/lib/mgmt/` and verifying that this also allows
a clean mgmt start, even without `--tmp-prefix`.
3. Running the full cluster example, but using `--tmp-prefix` with at least two
of the instances.
4. If everything worked so far, reproducing the original failure mode by running
everything against `/var/lib/mgmt/`.
5. If everything checks out to this point, dig into the git history and try to
find how all this worked in June 2016, when James blogged about etcd clustering
originally.

I'll be amazed if this plan ends up working. The most recent track record of
mgmt experimentation suggests otherwise. Let's try and reverse the trend.

## An all new state

First let's avoid the (corrupted?) local state directory by switching to an all
new ephemeral directory.

```
# mgmt run --tmp-prefix --hostname h1 --ideal-cluster-size 3 yaml examples/yaml/etcd1a.y
aml
This is: mgmt, version: 0.0.21-72-g10aa80e-dirty
...
00:20:03 etcd: server: starting...
<snip: lots of etcd output because I'm still on INFO level...>
00:20:04 etcd: server: ready
00:20:04 etcd: connect...
{"level":"warn","ts":"2020-03-10T00:20:04.902Z","caller":"grpclog/grpclog.go:60","msg":"Adjusting keepalive ping interval to minimum period of 10s"}
00:20:04 etcd: connected!
...
```

mgmt then proceeds to actually sync the resources in the "etcd1" graph. Success!
This concludes step 1 of 5, on to the next one.

## Fighting corruption

Now to see whether mgmt will perform the same clean start without the
`--tmp-prefix` switch, after the (now very likely corrupted) state information
from there.

```
# rm -r /var/lib/mgmt/*
# mgmt run --hostname h1 --ideal-cluster-size 3 yaml examples/yaml/etcd1a.yaml
```

What do you know; mgmt starts up clean, runs the graph, and starts waiting for
events. This test succeeded as well.

## Clustering up - redux

I've been here before. Code all set to go, launching three mgmt processes in
tandem, with appropriate switches to let them form an etcd cluster. Only it
would not work. The second and third process only spouted cryptic etcd related
messages, and did not really start their operation. The original seed seemed
to get unsettled at that point as well.

Let's try to improve. The seed server uses the default state directory in
`/var/lib/mgmt/`. The two others will use a respective temporary prefix.

```
# mgmt run --hostname h1 --ideal-cluster-size 3 yaml examples/yaml/etcd1a.yaml
# mgmt run --tmp-prefix --hostname h2 --seeds http://127.0.0.1:2379 --client-urls http://127.0.0.1:2381 --server-urls http://127.0.0.1:2382 yaml examples/yaml/etcd1b.yaml
# mgmt run --tmp-prefix --hostname h3 --seeds http://127.0.0.1:2379 --client-urls http://127.0.0.1:2383 --server-urls http://127.0.0.1:2384 yaml examples/yaml/etcd1c.yaml
```

As predicted earlier, I'm actually amazed. It works. It actually works. Just
like that, I have a running etcd cluster. The exported files are collected as
designed.

```
# grep . /tmp/mgmt?/f[ABC]
/tmp/mgmtA/fA:i am fA, exported from host A
/tmp/mgmtA/fB:i am fB, exported from host B
/tmp/mgmtA/fC:i am fC, exported from host C
/tmp/mgmtB/fA:i am fA, exported from host A
/tmp/mgmtB/fB:i am fB, exported from host B
/tmp/mgmtB/fC:i am fC, exported from host C
/tmp/mgmtC/fA:i am fA, exported from host A
/tmp/mgmtC/fB:i am fB, exported from host B
/tmp/mgmtC/fC:i am fC, exported from host C
```

It was a bit of a process, but here we are. Back in 2016. Gosh that was a shitty
year. Who knew that evoking something from that time would feel so good.

## Reproducing the failure - for science!

The theory held so far, so I shall go ahead and try to mess everything up again.
I'm stopping the `h2` instance of mgmt and relaunching it without the
`--tmp-prefix` switch. This should hurt.

```
# mgmt run --hostname h2 --seeds http://127.0.0.1:2379 --client-urls http://127.0.0.1:2381 --server-urls http://127.0.0.1:2382 yaml examples/yaml/etcd1b.yaml
```

It seems to start up okay, but after a few seconds, it mentioned this:

```
{"level":"info","ts":"2020-03-10T00:51:55.611Z","caller":"etcdserver/backend.go:85","msg":"db file is flocked by another process, or taking too long","path":"/var/lib/mgmt/etcd/server/member/snap/db","took":"10.000128701s"}
```

But the seed server especially is *not* happy. It gives me quite exactly ten of
these each second:

```
{"level":"warn","ts":"2020-03-10T00:53:02.263Z","caller":"rafthttp/peer.go:267","msg":"dropped internal Raft message since sending buffer is full (overloaded network)","message-type":"MsgHeartbeat","local-member-id":"8e9e05c52164694d","from":"8e9e05c52164694d","remote-peer-id":"467d02f1279bb468","remote-peer-active": false}
{"level":"warn","ts":"2020-03-10T00:53:02.363Z","caller":"rafthttp/peer.go:267","msg":"dropped internal Raft message since sending buffer is full (overloaded network)","message-type":"MsgHeartbeat","local-member-id":"8e9e05c52164694d","from":"8e9e05c52164694d","remote-peer-id":"467d02f1279bb468","remote-peer-active":false}
{"level":"warn","ts":"2020-03-10T00:53:02.463Z","caller":"rafthttp/peer.go:267","msg":"dropped internal Raft message since sending buffer is full (overloaded network)","message-type":"MsgHeartbeat","local-member-id":"8e9e05c52164694d","from":"8e9e05c52164694d","remote-peer-id":"467d02f1279bb468","remote-peer-active": false}
```

Every five seconds or so, it intersperses a

```
{"level":"warn","ts":"2020-03-10T00:52:55.603Z","caller":"rafthttp/probing_status.go:70","msg":"prober detected unhealthy status","round-tripper-name":"ROUND_TRIPPER_RAFT_MESSAGE","remote-peer-id":"467d02f1279bb468","rtt":"0s","error":"read tcp 127.0.0.1:47250->127.0.0.1:2382: i/o timeout"}
{"level":"warn","ts":"2020-03-10T00:52:55.603Z","caller":"rafthttp/probing_status.go:70","msg":"prober detected unhealthy status","round-tripper-name":"ROUND_TRIPPER_SNAPSHOT","remote-peer-id":"467d02f1279bb468","rtt":"0s"}
```

I'm not inspecting the output further to find any other rhythms. This looks
about as bad as I had anticipated. Let the record show that this mgmt process
also will not cleanly terminate from a handful of SIGINTs. The `h2` instance
does stop (I got impatient and sent 3 signals), but has also spouted a number
of additional errors. I'm not going back to compare timings to see if these are
related to the attempted shutdown of the seed.

Lack of details notwithstanding, I declare the penultimate test to be
successful. We let two mgmt instances use the same local state directory, and
etcd immediately ran into an issue and broke everything. (Yes, everything. I
just found the `h3` process in etcd hysteria as well. Sleep well, sweet prince.)

Only one thing left to do, I suppose.

## Archaeology

James's [original
post](https://purpleidea.com/blog/2016/06/20/automatic-clustering-in-mgmt/) on
the topic at hand is dated 20th June 2016. According to
[github](https://github.com/purpleidea/mgmt/releases?after=0.0.12), the 0.0.4
release is from September 2016, which dates the blog post in the 0.0.3 era,
maybe even earlier. Let's start at 0.0.3.

Before that, let's look up the place in the code that currently defines
`/var/lib/messages` as the default location for things like the etcd state.

```
# git grep /var/lib/mgmt
docs/documentation.md:created if it does not exist. This usually defaults to `/var/lib/mgmt/`. This
docs/faq.md:This dir is typically `/var/lib/mgmt/etcd/member/`. If you accidentally use it
docs/puppet-guide.md:vardir=/var/lib/mgmt/puppet
examples/yaml/file2.yaml:    source: "/var/lib/mgmt/files/some_dir/"
lang/lexparse_test.go:          name:  "/var/lib/mgmt",
lang/lexparse_test.go:          //path: "/var/lib/mgmt",
lang/lexparse_test.go:          name:  "/var/lib/mgmt/",
lang/lexparse_test.go:          //path: "/var/lib/mgmt/",
lib/main.go:    // is root, then use /var/lib/mgmt/.
```

No cigar, but close. For now, I will focus on these lines from
[lib/main.go](https://github.com/purpleidea/mgmt/blob/3bce96bbd509ad5ffb35ead52128dec5c1a67abf/lib/main.go#L225):

```
// Use systemd StateDirectory if set. If not, use XDG_CACHE_DIR unless user
// is root, then use /var/lib/mgmt/.
var prefix = fmt.Sprintf("/var/lib/%s/", obj.Program) // default prefix
```

Of course, it's unclear whether this whole code snippet is present in a truly
ancient mgmt version.

```
# git checkout 0.0.3
```

And sure enough, the entire file is not here, and at a glance, the code we're
looking for cannot be found elsewhere either. Time to take a closer look at
how the `prefix` variable is actually used in order to inform the embedded etcd
server of what goes where.
