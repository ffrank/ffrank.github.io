---
layout: post
category: misc
tags: [ mgmt, etcd, code ]
summary: How will an mgmt cluster be used in real life? Let's find out.
---

Continuing my blog series on figuring out the basics of mgmt networking and
etcd, it's time to tackle one of the most important aspects. At least that's
how I feel about it. After all, it will be nice to show flashy inotify tricks
to your team, but eventually you will want the tool to do some actual work.

But how? We won't be launching mgmt manually on each managed node to read the
respective code file to run. (I mean, you can do that if that's your jam, but
it will not be the canonical use case.)

I have read of "deploys" in the code, so let's go find out how these work.

## Not all help is created equal

First stop: The output of `mgmt help`:

```
COMMANDS:
   run, r     run
   deploy, d  deploy
   get, g     get
   help, h    Shows a list of commands or help for one command
```

Oh neat - not only is there an actual `deploy` subcommand, there is also a
`help` command, so we can just get `mgmt help deploy`. Right? Right.

```
# mgmt help deploy
NAME:
   mgmt deploy - deploy

USAGE:
   mgmt deploy [command options] [arguments...]

OPTIONS:
...
```

I puzzled over this quite a bit. Sure, I can run `mgmt deploy <whatever>`.
But what should the arguments be, specifically?

The way to get the answer to this question is as simple as it is perplexing:
In order to get all information,
you need to call `mgmt deploy --help` rather than `mgmt help deploy`. It shows
a more comprehensive description, including the next layer of subcommands.

```
# mgmt deploy --help
NAME:
   mgmt deploy - deploy

USAGE:
   mgmt deploy command [command options] [arguments...]

COMMANDS:
   empty       deploy using the `empty` frontend
   lang        deploy using the `lang` frontend
   langpuppet  deploy using the `langpuppet` frontend
   puppet      deploy using the `puppet` frontend
   yaml        deploy using the `yaml` frontend
   help, h     Shows a list of commands or help for one command

OPTIONS:
   --seeds value  default etc client endpoint [$MGMT_SEEDS]
...
```

Funnily, you can use `help` on this subcommand level as well, so `mgmt deploy
help` will give you the same information. You can even get `mgmt deploy help
lang`, for example. In all, the interface for `mgmt deploy` is very similar to
that of `mgmt run`, so it seems intuitively clear how it's supposed to work.

## First deployment

To get started, I bring up my cluster as described 
[before]({% post_url 2020-04-02-an-actual-cluster %}), but do not give an
mcl or YAML based input to any of my members. They just come up with the
special `empty` GAPI.

```
# mgmt run --hostname h1 --ideal-cluster-size 3 \
    --client-urls http://138.68.104.187:2379 \
    --server-urls http://138.68.104.187:2380 empty
# mgmt run --hostname h2 \
    --seeds http://seed.playground.net:2380 \
    --client-urls http://134.122.78.105:2379 \
    --server-urls http://134.122.78.105:2380 empty
# mgmt run --hostname h3 \
    --seeds http://138.68.104.187:2379 \
    --client-urls http://134.122.90.164:2379 \
    --server-urls http://134.122.90.164:2380 empty
```

Then, from any machine that can reach these, I deploy a graph from some mcl
code:

```
mgmt deploy --seeds http://138.68.104.187:2379 lang examples/lang/env1.mcl
```

The `env1` example from the mgmt source prints the value of the GOPATH
environment variable. Lo and behold, I get this expected output on each of my
mgmt instances.

```
18:59:08 engine: print[print0]: resource: Msg: GOPATH is: /root/gopath
```

```
18:59:08 engine: print[print0]: resource: Msg: GOPATH is missing!
```

```
18:59:08 engine: print[print0]: resource: Msg: GOPATH is missing!
```

## But what about the second deployment

Satisfied with the immediate success on the first try, I want to make a small
adjustment. In order to let my cluster do something useful, I want my code
to consider the HOSTNAME variable rather than the GOPATH. I make a copy of
env1.mcl in /tmp/hostname.mcl and make a slight change:

```
import "fmt"
import "sys"

$env = sys.env()
$m = maplookup($env, "HOSTNAME", "")

print "print0" {
        msg => if sys.hasenv("HOSTNAME") {
		fmt.printf("HOSTNAME is: %s", $m)
	} else {
		"HOSTNAME is missing!"
	},
}
```

It should be possible to deploy this to my running cluster the same way as the
example code was. But alas:

```
# mgmt deploy  --seeds http://138.68.104.187:2379 lang /tmp/hostname.mcl
This is: mgmt, version: 0.0.21-73-gd0f971f
Copyright (C) 2013-2020+ James Shubin and the project contributors
Written by James Shubin <james@shubin.ca> and the project contributors
19:01:32 main: start: 1588964492173753354
19:01:32 deploy: hash: d0f971f69dff0c187ee6e9e910eb50e55fb8ac29
19:01:32 deploy: previous deploy hash: 10aa80e8f57f4b37c9204fe104e2e8c11e5bf861
19:01:32 deploy: previous max deploy id: 1
19:01:32 cli: lang: lexing/parsing...
19:01:33 cli: lang: init...
19:01:33 cli: lang: interpolating...
19:01:33 cli: lang: building scope...
19:01:33 cli: lang: running type unification...
19:01:34 cli: lang: input: /tmp/hostname.mcl
19:01:34 cli: lang: tree:
.
------ metadata.yaml
------ hostname.mcl
19:01:34 deploy: goodbye!
19:01:34 deploy: error: could not create deploy id `2`: could not create deploy id 2
```

It didn't work. Let's find out why.

### The mandatory code trawl

The error message is found in two places.

```
t# git grep "could not create deploy"
etcd/deployer/deployer.go:              return fmt.Errorf("could not create deploy id %d", id)
lib/deploy.go:          return errwrap.Wrapf(err, "could not create deploy id `%d`", id)
```

This is why the message is repeated in the output: The second match (the one
with the backticks around the id number) wraps the first one. That is to say,
the error is being generated in `etcd/deployer/deployer.go`. This is the code
piece, right at the end of the `AddDeploy` function:

```
result, err := obj.Client.Txn(ctx, ifs, ops, nil)
if err != nil {
        return errwrap.Wrapf(err, "error creating deploy id %d", id)
}
if !result.Succeeded {
        return fmt.Errorf("could not create deploy id %d", id)
}
```

It looks like the "deployer" is attempting to make a client transaction towards
the etcd cluster. The `Client.Txn` call does not raise an error, but the
transaction is not successful either. It is not immediately clear what the
possible cause for this is. I suspect I will have to raise the etcd log level
once more.

## Lab reproduction

The environment in which I have first observed this behavior is somewhat
unwieldy. It consists of three running mgmt instances on separate virtual
machines, creating an ad hoc etcd cluster. I have not devised a convenient way
to restart this environment from scratch. (To make it worse, it does not seem
to cleanly restart without discarding some persistent data either, which will
warrant yet another article soon.)

As such, my next step is to try and reproduce in a more simple context.
Instead of configuring my seed server for network interaction, I will run mgmt
plainly.

```
mgmt run empty
```

An idle mgmt process starts up, waiting for connections on 127.0.0.1.

```
# mgmt deploy --seeds http://127.0.0.1:2379 lang examples/lang/env1.mcl
```

As expected, this indeed deploys a graph to my standalone mgmt instance. Better
yet, a second deployment yields the exact same error observed in the cluster
environment described above. I'm a lot more comfortable debugging this.

## Some etcd inspection

First dumb check: What happens when I try and deploy the exact same thing twice
in a row?

```
# mgmt deploy  --seeds http://127.0.0.1:2379 lang e
xamples/lang/env1.mcl
This is: mgmt, version: 0.0.21-73-gd0f971f
Copyright (C) 2013-2020+ James Shubin and the project contributors
Written by James Shubin <james@shubin.ca> and the project contributors
14:55:05 main: start: 1589122505506627793
14:55:05 deploy: hash: d0f971f69dff0c187ee6e9e910eb50e55fb8ac29
14:55:05 deploy: previous deploy hash: 10aa80e8f57f4b37c9204fe104e2e8c11e5bf861
14:55:05 deploy: previous max deploy id: 0
First dumb check: What happens when I try and deploy the exact same thing twice
in a row?

```
# mgmt deploy  --seeds http://127.0.0.1:2379 lang examples/lang/env1.mcl
This is: mgmt, version: 0.0.21-73-gd0f971f
Copyright (C) 2013-2020+ James Shubin and the project contributors
Written by James Shubin <james@shubin.ca> and the project contributors
14:55:05 main: start: 1589122505506627793
14:55:05 deploy: hash: d0f971f69dff0c187ee6e9e910eb50e55fb8ac29
14:55:05 deploy: previous deploy hash: 10aa80e8f57f4b37c9204fe104e2e8c11e5bf861
14:55:05 deploy: previous max deploy id: 0
...
14:55:07 deploy: success, id: 1
14:55:07 deploy: goodbye!
```

I cut away some diagnostic messages from the `lang` GAPI that are not of
interest to me (yet).

Second run:

```
# mgmt deploy  --seeds http://127.0.0.1:2379 lang examples/lang/env1.mcl
...
14:55:23 main: start: 1589122523215602941
14:55:23 deploy: hash: d0f971f69dff0c187ee6e9e910eb50e55fb8ac29
14:55:23 deploy: previous deploy hash: 10aa80e8f57f4b37c9204fe104e2e8c11e5bf861
14:55:23 deploy: previous max deploy id: 1
...
14:55:24 deploy: goodbye!
14:55:24 deploy: error: could not create deploy id `2`: could not create deploy id 2
```

It appears correct that the deployer would attempt to create ID 2, as a deploy
with ID 1 was successfully created in the previous run. It strikes me as odd
that the "previous deploy hash" is apparently *not* updated through the initial
deployment. The second attempt indicates the same value that is seen when
deploying to an empty etcd.

Some printf debugging of the data structures pushed around by the deployer was
proves not so promising, so it appears it's time to read up on how etcd
transactions actually work.

## Adding a deploy

Getting an overview of the `AddDeploy` function, the interface doesn't look too
complicated at all. Broadly speaking, it seems to take three preparatory steps:

 1. It constructs appropriate paths (within etcd's own key-value store, I
    assume).
 2. It builds a list of "ifs", but only if this is not the very first deploy
    (it appears plausible that this code path contains a bug).
 3. It builds a list of "ops", probably operations that should be done
    on the etcd data.

This is fascinating and all, but as said, it's past time to read some reference
material. First stop is the documentation for the [etcd
clientv3](https://godoc.org/go.etcd.io/etcd/clientv3) package that is in broad
use by mgmt. Oddly though, the only `Txn` method described here is for the
`Op` structure, not the `Client` structure as I had expected.

It should be noted that the deployer code in mgmt does not use etcd interfaces
directly, but rather the internal `interfaces.Client` interface from mgmt's own
etcd package. According to its code comments, this interface is implemented by
EmbdEtc, a type from the same package. Looking at its `MakeClient` method,
that just seems to allow deriving a client object from a pre-existing client:

```
func (obj *EmbdEtcd) MakeClient() (interfaces.Client, error) {
        c := client.NewClientFromClient(obj.etcd)
```

Without further digging, I will assume that the deployer will ultimately use
the `SimpleClient` object from this package, but I will try to confirm this
suspicion with a brief dive into the deployer code. This is how in
`lib/deploy.go`, the client object is created:

```
etcdClient := client.NewClientFromSeedsNamespace(
        cliContext.StringSlice("seeds"), // endpoints
        NS,
)
```

NS is a constant from `lib/main.go` with value "/\_mgmt".

The `NewClientFromSeedsNamespace` function will return a `Simple` object which
does wrap an `etcd/clientv3.Client` object. So here we are: The `Txn` method
I have been searching is defined on this `etcd.Simple` type:

```
// Txn runs a transaction.
func (obj *Simple) Txn(ctx context.Context, ifCmps []etcd.Cmp, thenOps, elseOps []etcd.Op) (*etcd.TxnResponse, error) {
        resp, err := obj.kv.Txn(ctx).If(ifCmps...).Then(thenOps...).Else(elseOps...).Commit()
```

It's still more than a little confusing to me. It uses the `Txn` method through
the
KV interface associated to this client object, to create an etcd transaction
(or so I suppose). The semantics are implemented through the `If`, `Then`, and
`Else` method then.

The etcd API documentation is not rich with information about how the response
object returned by the `Commit` call should be interpreted. There is one very
basic example for the `Txn` method in the [description of the KV
interface](https://godoc.org/go.etcd.io/etcd/clientv3#KV), but it does not
go into error handling at all. It feels like this was a wild goose chase after
all.

## Taking ctl

In order to get a better feel for whatever etcd is doing here, I want some more
direct access. From an [earlier
adventure]({% post_url 2020-03-09-getting-etcd-back %}) I still have this
`etcdctl` binary around. It graciously connects to a fresh instance of

```
mgmt run empty
```

Here's what that looks like:

```
# ETCDCTL_API=3 ~/etcdctl --endpoints 127.0.0.1:2380 member list
8e9e05c52164694d, started, ubuntu-s-1vcpu-1gb-fra1-01, http://localhost:2380, http://localhost:2379, false
```

With etcd3, this is how I can inspect all key/value pairs stored:

```
# ~/etcdctl --endpoints 127.0.0.1:2380 get / --prefix
```

In my current state of experimentation, this is very promising, because it
contains pairs like the following:

```
/_mgmt/deploy/1/hash
d0f971f69dff0c187ee6e9e910eb50e55fb8ac29
/_mgmt/deploy/1/payload
P/+JAwEBBkRlcGxveQH/igABBQECSUQBBgABBE5hbWUBDAABBE5vb3ABAgABBFNlbWEBBAABBEdBUEkBEAAAADX/igIEbGFuZwIBAQoqbGFuZy5HQVBJ/4sDAQEER0FQSQH/jAABAQEI
SW5wdXRVUkkBDAAAAEH/jD0BOmV0Y2RmczovLy9mcy9kZXBsb3kvMS02MmI0ODIxZi05MTAyLTQ5M2QtYmI3Yi03YjIxY2QxOTJhZjYAAA==
```

These look like descriptors for the deploy that was actually successful. There
is also this (please note, there is a fair bit of binary data that is not
represented here; this is just what appears in the terminal window):

```
/_mgmt/fs/deploy/1-62b4821f-9102-493d-bb7b-7b21cd192af6
:
superBlock
DataPrefix
          Hash
              TreeHFilePath
                           ModeModTimChildrenHash
                                                 Time
metadata.yaml|@8fe0adbefe197914bd143d2fd3199f8368654405fe135785c81b277b4ac1d33env1.mcl@f54aca743575340a6f10cab5393a7f79df4f34910e0fd05816c7c
49fb172c9ae
```

And also quite interesting:

```
/_mgmt/storage/{sha256}f54aca743575340a6f10cab5393a7f79df4f34910e0fd05816c7c49fb172c9ae
import "fmt"
import "sys"

$env = sys.env()
$m = maplookup($env, "GOPATH", "")

print "print0" {
        msg => if sys.hasenv("GOPATH") {
                fmt.printf("GOPATH is: %s", $m)
        } else {
                "GOPATH is missing!"
        },
}
```

It's also very interesting that the unsuccessful second deploy is represented
as well:

```
/_mgmt/fs/deploy/2-c9807a7e-c1c0-48e2-b3fa-74e458ae0016
:
superBlock
DataPrefix
          Hash
              TreeHFilePath
                           ModeModTimChildrenHash
                                                 Time
metadata.yamlJ@8fe0adbefe197914bd143d2fd3199f8368654405fe135785c81b277b4ac1d33env1.mcl+@f54aca743575340a6f10cab5393a7f79df4f34910e0fd05816c7
c49fb172c9ae
```

However, instead of trying to immediately match that to code excerpts I've seen
in mgmt, I will first take one step back, start over from a clean slate, and
see how the key/value pairs build up through my experiment. It should be easier
to understand the code after this step.

## Starting over

The easiest way to get a pristine etcd key/value store is of course to restart
the mgmt seed as follows:

```
# mgmt run --tmp-prefix empty
```

Let's see what gets initialized directly.

```
# ~/etcdctl --endpoints 127.0.0.1:2380 get / --prefix
/_mgmt/chooser/dynamicsize/idealclustersize
5
/_mgmt/endpoints/ubuntu-s-1vcpu-1gb-fra1-01
http://localhost:2379
/_mgmt/nominated/ubuntu-s-1vcpu-1gb-fra1-01
http://localhost:2380
/_mgmt/volunteer/ubuntu-s-1vcpu-1gb-fra1-01
http://localhost:2380
```

All this data is purely organizational in nature. Good. Next, deploying the
first graph.

```
# ./mgmt deploy --seeds http://127.0.0.1:2379 lang examples/lang/env1.mcl
...
# ~/etcdctl --endpoints 127.0.0.1:2380 get / --prefix --keys-only    /_mgmt/chooser/dynamicsize/idealclustersize
/_mgmt/deploy/1/hash
/_mgmt/deploy/1/payload
/_mgmt/endpoints/ubuntu-s-1vcpu-1gb-fra1-01
/_mgmt/fs/deploy/1-25ab3b42-f8d0-4147-8864-73f01d9963f3
/_mgmt/nominated/ubuntu-s-1vcpu-1gb-fra1-01
/_mgmt/storage/{sha256}8fe0adbefe197914bd143d2fd3199f8368654405fe135785c81b277b4ac1d336
/_mgmt/storage/{sha256}e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
/_mgmt/storage/{sha256}f54aca743575340a6f10cab5393a7f79df4f34910e0fd05816c7c49fb172c9ae
/_mgmt/volunteer/ubuntu-s-1vcpu-1gb-fra1-01
```

So this step has added

 * Hash and Payload for deploy 1
 * an entry in `fs` for deploy 1, keyed with a UUID
 * three storage entries

As for the storage entries, their respective values are

 * nothing at all
 * the mcl code that was deployed
 * another list of key/value pairs:
  * main: env1.mcl
  * path: ""
  * files: ""
  * license: ""
  * parentpathblock: false
  * metadata: null

Now to see exactly what happens with another attempted deployment.

```
# ./mgmt deploy --seeds http://127.0.0.1:2379 lang examples/lang/env1.mcl
...
# ~/etcdctl --endpoints 127.0.0.1:2380 get / --prefix --keys-only
/_mgmt/chooser/dynamicsize/idealclustersize
/_mgmt/deploy/1/hash
/_mgmt/deploy/1/payload
/_mgmt/endpoints/ubuntu-s-1vcpu-1gb-fra1-01
/_mgmt/fs/deploy/1-25ab3b42-f8d0-4147-8864-73f01d9963f3
/_mgmt/fs/deploy/2-5e79a475-b6ad-471c-890a-5e52c1b99baa
/_mgmt/nominated/ubuntu-s-1vcpu-1gb-fra1-01
/_mgmt/storage/{sha256}8fe0adbefe197914bd143d2fd3199f8368654405fe135785c81b277b4ac1d336
/_mgmt/storage/{sha256}e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
/_mgmt/storage/{sha256}f54aca743575340a6f10cab5393a7f79df4f34910e0fd05816c7c49fb172c9ae
/_mgmt/volunteer/ubuntu-s-1vcpu-1gb-fra1-01
```

So the only key that was added is
`/_mgmt/fs/deploy/2-5e79a475-b6ad-471c-890a-5e52c1b99baa`. No values of
existing keys have changed either. With all this insight, it's time to head
back into source code and output, and to try and find out just what keeps going
wrong here.

## Bug hunt

Wandering back into the source code, starting in
[lib/deploy.go](https://github.com/purpleidea/mgmt/blob/e9af8a2595e336542c9dfc656fe808ddc6937a59/lib/deploy.go),
I'm taking another hard look at the `deploy` function, when it strikes me: The
"hash" is retrieved in the following code block:

```
var hash, pHash string
if !cliContext.Bool("no-git") {
        wd, err := os.Getwd()
        if err != nil {
                return errwrap.Wrapf(err, "could not get current working directory")
        }
        repo, err := git.PlainOpen(wd)
        if err != nil {
                return errwrap.Wrapf(err, "could not open git repo")
        }
```

The `deploy` command (silently) opens and inspects the git repository from the
current working directory. I have been trying `mgmt deploy` invocations from
the root of my clone of the mgmt source code this whole time. Was this what
kept throwing the deployment off?

```
# cd
# mgmt deploy  --seeds http://127.0.0.1:2379 \
    lang /path/to/examples/lang/env1.mcl
This is: mgmt, version: 0.0.21-73-gd0f971f-dirty
Copyright (C) 2013-2020+ James Shubin and the project contributors
Written by James Shubin <james@shubin.ca> and the project contributors
10:05:24 main: start: 1590314724368572336
10:05:24 deploy: goodbye!
10:05:24 deploy: error: could not open git repo: repository does not exist
```

Alright then...adding `--no-git`.

```
# mgmt deploy --no-git --seeds http://127.0.0.1:2379 \
    lang /path/to/examples/lang/env1.mcl
...
09:50:54 deploy: success, id: 2
09:50:54 deploy: goodbye!
```

Success. Deploying with the `--no-git` options works without issue.
So it very much seems to be as I suspected all along: This is not a programming
error, but what I consider to be a UX bug. The tool makes an assumption about
what I'm trying to do (deploy mcl code from my git repository), fails to
communicate this properly, and lets me walk into an error because my own
assumptions differ from those of the tool.

In a follow-up post, I will document my reporting of this issue, along with my
little quest for The Path of Least Surprise.

## Summary

Code gets deployed to an mgmt cluster using the `mgmt deploy` subcommand,
which feels much like the `run` subcommand in terms of form and function.
The `deploy` command expects to be invoked from a git clone root directory
(or with other means to locate a git repository) and uses git metadata for
the deployment. This can lead to cryptic errors when trying to deploy code.
The exact circumstances leading to such errors will be detailed in another
blog post.
