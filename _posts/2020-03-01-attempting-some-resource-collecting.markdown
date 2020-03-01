---
layout: post
category: misc
tags: [ mgmt, yaml, graph, etcd, cluster ]
summary: Attempting the etcd clustering example and learning about mgmt resource exports
---

After some adventures [building a new
feature]({% post_url 2020-02-23-faster-puppet-for-mgmt %}) for mgmt's
Puppet support, and making [a presentation including live
demos](http://ffrank.github.io/presentations/2020-02-puppet-from-mgmt-on-overdrive/puppet-mgmt-overdrive.html)
about it, I find myself quite impressed with how far mgmt's interface and its
configuration language have evolved since last I played with them. One thing
that I found lacking is the documentation. As the tool is now in a nice shape
for people out there to go ahead and give it a spin, I feel that improving the
docs will be especially helpful in order to allow early adopters to get
a smooth start, without the need to read code or asking James and his
contributors for help.

My first area of interest is the automatic clustering feature in mgmt. I want
to set up a minimal cluster, and feel what it's like to run configuration code
on it. I want to find out and describe how configuration code and data will be
deployed to such a cluster. Finally, I want to compile the resulting blog posts
into neat documents for the proper [documentation
section](https://github.com/purpleidea/mgmt/tree/master/docs) of mgmt.

## Clustering it up

The [blog
post](https://purpleidea.com/blog/2016/06/20/automatic-clustering-in-mgmt/)
describing the automatic clustering procedure is somewhat dated, and a few
things have changed in the CLI since. The first step used to be the following
invocation:

```
mgmt run --file examples/etcd1a.yaml --hostname h1 --ideal-cluster-size 3
```

The `--file` switch does not exist anymore, and the YAML frontend has to be
chosen explicitly. The modern invocation should look like this:

```
mgmt run --hostname h1 --ideal-cluster-size 3 yaml examples/yaml/etcd1a.yaml
```

To my disappointment, this does not seem to Just Work anymore. The mgmt process
crashes with a cryptic stack trace.

```
20:39:58 engine: autoedge: adding autoedges...
20:39:58 engine: file[filea]: CheckApply(true)
20:39:58 engine: autogroup: algorithm: wrappedGrouper: NonReachabilityGrouper...
20:39:58 engine: file[filea]: resource: contentCheckApply(true)
20:39:58 engine: file[filea]: CheckApply(true): Return(false, open /tmp/mgmtA/fA: no such file or directory)
20:39:58 main: commit...
20:39:58 engine: graph sync...
20:39:58 main: graph: Vertices(1), Edges(0)
20:39:58 main: waiting...
20:39:58 gapi: yaml: collect: file; pattern: /tmp/mgmtA/
20:39:58 gapi: yaml: collected: file[filea]
20:39:58 engine: autoedge: adding autoedges...
20:39:58 engine: autogroup: algorithm: wrappedGrouper: NonReachabilityGrouper...
20:39:58 main: loop: exited
panic: close of closed channel

goroutine 259 [running]:
github.com/purpleidea/mgmt/engine/graph.(*State).Pause(0xc000462480, 0xc00034f140, 0xc000573cb0)
        /root/gopath/src/github.com/purpleidea/mgmt/engine/graph/state.go:349 +0xea
github.com/purpleidea/mgmt/engine/graph.(*Engine).Pause(0xc000258be0, 0x0, 0x0, 0x0)
        /root/gopath/src/github.com/purpleidea/mgmt/engine/graph/engine.go:392 +0xca
github.com/purpleidea/mgmt/lib.(*Main).Run.func15(0x2005918, 0xc0004ae6b0, 0xc000010be0, 0xc000148330, 0xc000010be8, 0xc00048ab40, 0x7ffd956446de, 0x2, 0xc000$
23e60, 0xc0004a1340, ...)
        /root/gopath/src/github.com/purpleidea/mgmt/lib/main.go:734 +0x15bf
created by github.com/purpleidea/mgmt/lib.(*Main).Run
        /root/gopath/src/github.com/purpleidea/mgmt/lib/main.go:522 +0x1567
```

I could not reliably reproduce this, so it might be related a race condition
that James alluded to in recent conversations. What *was* reproducible at the
time (version `0.0.21-72-g10aa80e`) was a simple graph error:

```
21:18:35 engine: Worker(file[filea]): Exited(open /tmp/mgmtA/fA: no such file or directory
error during Process()
github.com/purpleidea/mgmt/util/errwrap.Wrapf
        /root/gopath/src/github.com/purpleidea/mgmt/util/errwrap/errwrap.go:29
github.com/purpleidea/mgmt/engine/graph.(*Engine).Process
        /root/gopath/src/github.com/purpleidea/mgmt/engine/graph/actions.go:238
github.com/purpleidea/mgmt/engine/graph.(*Engine).Worker
        /root/gopath/src/github.com/purpleidea/mgmt/engine/graph/actions.go:522
github.com/purpleidea/mgmt/engine/graph.(*Engine).Commit.func1.2.1
        /root/gopath/src/github.com/purpleidea/mgmt/engine/graph/engine.go:232
```

I suppose that this has been broken forever, basically. The `etcd1a` example
and its siblings assume that directories like `/tmp/mgmtA` exist already. They
are created by other examples, but the etcd examples cannot run on their own,
apparently. Since this is a clustering demo, I opted for an approach that takes
advantage of the etcd exchange, exporting the required resources from the
initial cluster member. I changed the `etcd1a` example to look as follows:

```
graph: mygraph
resources:
  file:
  - name: "@@filea"
    path: "/tmp/mgmtA/fA"
    content: |
      i am fA, exported from host A
    state: exists
  - name: "@@dir_a"
    path: "/tmp/mgmtA/"
    state: exists
  - name: "@@dir_b"
    path: "/tmp/mgmtB/"
    state: exists
  - name: "@@dir_c"
    path: "/tmp/mgmtC/"
    state: exists
  - name: "@@dir_d"
    path: "/tmp/mgmtD/"
    state: exists
  - name: "@@dir_e"
    path: "/tmp/mgmtE/"
    state: exists
collect:
- kind: file
  pattern: "/tmp/mgmtA/"
edges: []
```

It did not go well. Not only did the mgmt process collect *all* these resources,
it also somehow rendered them into an odd superposition:

```
21:48:02 engine: file[dir_c]: CheckApply(true): Return(false, mkdir /tmp/mgmtA/mgmtC/: no such file or directory)
21:48:02 engine: file[dir_d]: CheckApply(true): Return(false, mkdir /tmp/mgmtA/mgmtD/: no such file or directory)
21:48:02 engine: file[dir_e]: CheckApply(true): Return(false, mkdir /tmp/mgmtA/mgmtE/: no such file or directory)
21:48:02 engine: file[filea]: resource: contentCheckApply(true)
21:48:02 engine: file[filea]: CheckApply(true): Return(false, open /tmp/mgmtA/fA: no such file or directory)
21:48:02 engine: file[dir_a]: CheckApply(true): Return(false, mkdir /tmp/mgmtA/mgmtA/: no such file or directory)
```

More research is clearly required. First of all, a simpler approach to the
problem at hand:

```
---
graph: mygraph
resources:
  file:
  - name: "@@filea"
    path: "/tmp/mgmtA/fA"
    content: |
      i am fA, exported from host A
    state: exists
  - name: "dir_a"
    path: "/tmp/mgmtA/"
    state: exists
collect:
- kind: file
  pattern: "/tmp/mgmtA/"
edges: []
```

This YAML graph actually works and lets mgmt converge the configuration nice
and quickly.

## Why does the export approach not work

This is bugging me though: It was my expectation that I would be able to just
export the base directories for a all cluster members, and that the `collect`
pattern of `/tmp/mgmtA` would ensure that only `dir_a` would get imported to
the local mgmt engine. Furthermore, I expected that it would create `/tmp/mgmtA`
and not `/tmp/mgmtA/mgmtA`, so something is odd on an even more fundamental
level. Let's break this down.

Step one, make the now functional `dir_a` into the exported `@@dir_a` again,
and see what mgmt does with it.

```
---
graph: mygraph
resources:
  file:
  - name: "@@filea"
    path: "/tmp/mgmtA/fA"
    content: |
      i am fA, exported from host A
    state: exists
  - name: "@@dir_a"
    path: "/tmp/mgmtA/"
    state: exists
collect:
- kind: file
  pattern: "/tmp/mgmtA/"
edges: []
```

This graph passed another mgmt run without issue, but it did create
`/tmp/mgmtA/mgmtA` for reasons that eluded me at the time. In a somewhat
desperate attempt, I removed `@@filea` from the graph to see if it had any
strange interactions with `@@dir_a`, but the result was quite the same. Time
for some silly experimentation.

First, I exported the directory `/tmp/x`, and collected using the same pattern
`/tmp/mgmtA/`. This made mgmt create `/tmp/mgmtA/x`, as I had wryly come to
expect at this point. Exporting `/tmp/xy/z` leads to the creation of
`/tmp/mgmtA/z` however, which is a little less obvious. In keeping with this,
however, an exported file `/path/to/a/hidden/dir` had me end up with
`/tmp/mgmtA/dir`.

At this point I can be fairly certain that this is a bug with the export/collect
functionality, so let's dive right in.

## Finding the "bug"

This is a weird one. My very first instinct is to `git grep pattern` to
hopefully find how this field from the YAML graph is used internally.
I did not. I did however find this gem:

```
// CollectPattern applies the pattern for collection resources.
func (obj *FileRes) CollectPattern(pattern string) {
        // XXX: currently the pattern for files can only override the Dirname variable :P
        obj.Dirname = pattern // XXX: simplistic for now
}
```

The wording makes me think that I was quite mistaken from the start. The pattern
is not meant for *filtering* exported resources, and only collecting a subset.
It is *applied* to collected resources, and overrides their properties.
This makes my initial approach of just exporting all the directories from a
single node somewhat problematic. In fact, with this implementation, I cannot
think of a way to export these required directories at all, or create a graph
that collects different resources in different ways for that matter.

Regardless of these findings, I got quite curious how this whole thing was
actually implemented. So I traced the YAML unmarshalling to see where this would
lead me. It was helpful that the GAPI code is one of the parts of mgmt that I am
[already familiar]({% post_url 2020-02-06-building-the-code-mixer %})
with.

In `yamlgraph/gconfig.go`, the function `NewGraphFromConfig` is responsible for
creating an actual graph object from the YAML representation. To my dismay, I
found that this very function also does the whole work of collecting exported
resources:

```
// lookup from backend (usually etcd)
var hostnameFilter []string // empty to get from everyone
kindFilter := []string{}
for _, t := range obj.Collector {
        kind := strings.ToLower(t.Kind)
        kindFilter = append(kindFilter, kind)
}
...
if len(kindFilter) > 0 { // if kindFilter is empty, don't need to do lookups!
        var err error
        resourceList, err = world.ResCollect(context.TODO(), hostnameFilter, kindFilter)
        ...
}
for _, res := range resourceList {
        ...
        for _, t := range obj.Collector {
                kind := strings.ToLower(t.Kind)
                // use t.Kind and optionally t.Pattern to collect from storage
                obj.Logf("collect: %s; pattern: %v", kind, t.Pattern)

                // XXX: expand to more complex pattern matching here...
                if res.Kind() != kind {
                        continue
                }
...
```

In other words, collecting currently *only* works in the context of graphs that
are generated using the YAML input (that is, the very basic crutch we use for
demo purposes), and doing more refined collections than "oh yes, this is the
kind of resource (such as `file`) I'm looking for" is not yet implemented
either.

On a cursory trawl of the list of open issues for mgmt, I did not spot a mention
that matches the missing features I am seeing. My conclusion for now is that
the export/collect features of the YAML GAPI are just that - a demo feature
that can currently show off that yes, etcd *can* be used to exchange
configuration data between nodes in the same cluster. We don't seem to have
concrete plans for a well rounded implementation of this quite yet.

## Moving on

I will focus on getting the etcd example graphs into a runnable state for now,
avoiding any more exporting than we are currently doing. Next post will
hopefully deal with how the clustering actually works.
