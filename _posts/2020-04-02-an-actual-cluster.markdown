---
layout: post
category: misc
tags: [ mgmt, etcd, network ]
summary: Extending the automatic clustering example to make it work in an actual network
---

In this little blog series, I've been wrestling with mgmt's embedded etcd
server, trying to get it to form a functioning cluster with mgmt 0.0.21. This
finally worked on the local machine, so let's take the next step and take it to
a little network.

## You never listen

Everything starts with the seed server. For mgmt 0.0.21, I'm launching it like
this:

```
# mgmt run --tmp-prefix --hostname h1 --ideal-cluster-size 3 yaml examples/yaml/etcd1a.yaml
```

I'm old, so this is how I look at ports a process is listening on:

```
# netstat -tlnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      14884/systemd-resol
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      808/sshd
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      7507/mgmt
tcp        0      0 127.0.0.1:2380          0.0.0.0:*               LISTEN      7507/mgmt
tcp6       0      0 :::22                   :::*                    LISTEN      808/sshd
```

Seriously though, `netstat` has so much nicer output than `ss` it's not even
funny. But I digress.

Listening on the loopback network device will be no good if we want remote
processes to connect. We will have to tell mgmt to listen on external
network devices. Now to figure out which port is supposed to do what.
Fortunately, I'm still running a patched build of mgmt from an earlier
adventure. It logs etcd messages at the INFO level, so I'm able to spot the
following helpful messages in the output of mgmt:

```
{"level":"info","ts":"2020-03-17T22:35:35.686Z","caller":"embed/etcd.go:117","msg":"configuring peer listeners","listen-peer-urls":["http://localhost:2380"]}
{"level":"info","ts":"2020-03-17T22:35:35.687Z","caller":"embed/etcd.go:127","msg":"configuring client listeners","listen-client-urls":["http://localhost:2379"]}
```

In short, `http://localhost:2380` is the "peer URL", while port 2379 belongs to
the "client URL". Here are relevant output lines from `mgmt help run`:

```
--client-urls value \
	list of URLs to listen on for client traffic [$MGMT_CLIENT_URLS]
--server-urls value \
	list of URLs to listen on for server (peer) traffic [$MGMT_SERVER_URLS]
```

It's nice and all that we can use environment variables instead of switches,
but let's keep everything in one place for now.

```
# mgmt run --tmp-prefix --hostname h1 --ideal-cluster-size 3 \
	--client-urls http://138.68.104.187:2379 \
	--server-urls http://138.68.104.187:2380 \
	yaml examples/yaml/etcd1a.yaml
```

Another look at `netstat` (and in the etcd output) reveals that it listens to
`127.0.0.1:2379` in addition to the supplied client URL, but the external ports
do seem to be available now.

## Cross the streams

On to the next one.  Jumping to a peer server,
let's first check that the server URL of the seed is reachable:

```
# curl http://138.68.104.187:2380
404 page not found
```

Perfect. Both Droplets are part of the same DigitalOcean project, and can
freely exchange packets on those high ports. (Less thrilled to realize that
my home client can reach this port as well, so I guess I will not leave this
cluster on over night...also should probably figure out firewalling or
whatever. No, yes, I *am* an ops professional, obviously. Thank you for your
concern.)

So on the local host, the first peer was supposed to use ports 2381 and 2382
respectively. In a networked setup, there hardly is a point, so this is my
proposal:

```
# mgmt run --tmp-prefix --hostname h2 --seeds http://138.68.104.187:2379 \
	--client-urls http://134.122.78.105:2379 \
	--server-urls http://134.122.78.105:2380 \
	yaml etcd1b.yaml
```

Works like a charm. Server comes up and creates these files:

```
# grep . /tmp/mgmtB/*
/tmp/mgmtB/fA:i am fA, exported from host A
/tmp/mgmtB/fB:i am fB, exported from host B
```

Final one:

```
# mgmt run --tmp-prefix --hostname h3 --seeds http://138.68.104.187:2379 \
	--client-urls http://134.122.90.164:2379 \
	--server-urls http://134.122.90.164:2380 \
	yaml etcd1c.yaml
```

The cluster is up, and this is what it looks like from the original seed:

```
# grep . /tmp/mgmtA/f?
/tmp/mgmtA/fA:i am fA, exported from host A
/tmp/mgmtA/fB:i am fB, exported from host B
/tmp/mgmtA/fC:i am fC, exported from host C

# ssh playground02 'grep . /tmp/mgmt[ABC]/f?'
/tmp/mgmtB/fA:i am fA, exported from host A
/tmp/mgmtB/fB:i am fB, exported from host B
/tmp/mgmtB/fC:i am fC, exported from host C

# ssh playground03 'grep . /tmp/mgmt[ABC]/f?'
/tmp/mgmtC/fA:i am fA, exported from host A
/tmp/mgmtC/fB:i am fB, exported from host B
/tmp/mgmtC/fC:i am fC, exported from host C
```

Very nice, just as designed: Each host collects the exported files, rewrites
the path to /tmp/mgmtA, mgmtB, or mgmtC respectively, and manages the content
as defined by the exporting server.

## Rounding things off

This concludes my mini-series on getting etcd back up and clustered with mgmt
0.0.21. I hope you enjoyed following along with my screw-ups and found the
resulting information useful. After conferring with James, I should add that,
yes, the exporting and collecting of resources was only built as a
proof-of-concept feature. It is not available from the mgmt language as of
version 0.0.21, and it's not high on our list of priorities. Nevertheless, it
was good fun to use it to test and show off the etcd functionality here.

Here is the complete set of commands required to build a three node auto
cluster with mgmt 0.0.21, assuming that you have control of a domain
`playground.net` (or a nice set of /etc/hosts files).

```
# on seed.playground.net:
mgmt run --tmp-prefix --hostname h1 --ideal-cluster-size 3 \
  --client-urls http://seed.playground.net:2379 \
  --server-urls http://seed.playground.net:2380 \
  yaml examples/yaml/etcd1a.yaml

# on peer1.playground.net:
mgmt run --tmp-prefix --hostname h2 --seeds http://seed.playground.net:2379 \
  --client-urls http://peer1.playground.net:2379 \
  --server-urls http://peer1.playground.net:2380 \
  yaml examples/yaml/etcd1b.yaml

# on peer2.playground.net:
mgmt run --tmp-prefix --hostname h3 --seeds http://seed.playground.net:2379 \
  --client-urls http://peer2.playground.net:2379 \
  --server-urls http://peer2.playground.net:2380 \
  yaml examples/yaml/etcd1c.yaml
```

Enjoy your ad hoc clusters, and let me know via Twitter if you've managed to
build anything cool with it.
