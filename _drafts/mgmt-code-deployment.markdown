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
with the backticks around the id number) wraps the first one.

