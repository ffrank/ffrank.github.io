---
layout: post
category: features
tags: mgmt puppet graph language compiler
---

This blog is not dead. It hasn't seen any new content in a long while,
which is a shame, but I did use the hiatus to recharge some creative
energy. Here's hoping progress will be more steady next year.

**Narrator voice** It wasn't more steady. In fact, this article
was kept around as an unfinished draft throughout 2019, not to be
published before Config Management Camp 2020. It still lacks
diagrams that make the graph code understandable. These may or may
not be added at a future date.
I am publishing this now because the narrative around GAPI
development maybe helpful for future contributors (or myself).
I hope it still makes for a somewhat enjoyable read.
(**End edit 2020-02-06**.)

Speaking of 2019, another installment of
[Config Management Camp](https://cfgmgmtcamp.eu/) is coming up,
and I felt compelled to get in position to present a cool new
thing. Call it Config Management Camp Driven Development, if you will.
My motivation nonwithstanding, I built a new feature for
[mgmt](https://github.com/purpleidea/mgmt/).

Specifically, I meant to implement an
[idea](/hacks/2018/02/13/thinking-about-migration-from-puppet-to-mgmt/)
that was hatched during Config Management Camp 2018.
It's about allowing mgmt to run code that is mixed from Puppet's DSL
and mgmt's own language. This article is mainly about code,
but let it be mentioned that the code now works, is merged
into mgmt's master branch, and has passed its most basic smoke test.
The amount of testing was way lower than I would prefer, but there is
only so many hours in the week.

This article really only deals with source code internals
of mgmt (as seen in December 2018). It may some day serve as an
extended guide to how this code works, but might also help some
aspiring mgmt contributors and give them some starting pointers.
The code has since mutated and will likely do it again, but this
post will stay frozen in the code's original look and feel.

## Creating the outline

Before I start bragging about the amazing hacker feats that lead to
this particular patch, let me put emphasis on the very beginning
of the process. I started by chatting up [James](https://purpleidea.com/about/)
on IRC (`#mgmtconfig` on Freenode) and asking for help. If you want
to get into hacking on the project, I suggest you do the same. We hopped
into a video call basically instantly, and he walked me through
the code I needed to base my work on.

Specifically, we talked about GAPIs. The GAPI subsystem is the part
of mgmt that provides all
the alternative ways of supplying configuration instructions
in the form of resource graphs.
Right now, the following GAPIs exist:

* `lang` interprets mgmt's own DSL
* `puppet` receives graphs generated from Puppet manifests
* `yamlgraph` allows constructing raw graphs in YAML
* `empty` is a pseudo-interface that falls back to running with an empty graph

In the course of this article I will describe how we added
the newest GAPI, `langpuppet`. Yes, naming it was Very Hard.

In general, the GAPI is a Go interface defined in `gapi/gapi.go` by the
full name of `gapi.GAPI`. This is important, because the current
convention sees all its implementations using the same base name
of `GAPI` as well, but in respective packages, such as `lang.GAPI`
or `puppet.GAPI`. The rest of the article discusses all aspects
of the GAPI interface.

The new `langpuppet.GAPI` wraps the existing `lang` and `puppet`
GAPIs. All methods basically multiplex the functionality to the
"child" GAPIs. This was daunting without a complete architectural
overview of the code base, but here is where James' help was most
critical.

Essentially, we started just as if we were building
any new GAPI. That is, we copied an existing implementation and
threw out its specifics. I picked the Puppet GAPI, because that's
what I'm most familiar with.

So, step one was to copy `puppet/gapi.go` to `langpuppet/gapi.go`,
remove anything that alluded to Puppet, and adjust the package name.

## Designing the CLI

The first interaction of the `mgmt` core with any GAPI is its
`CliFlags` method. mgmt uses the
[urfave/cli](https://godoc.org/github.com/urfave/cli)
library for its CLI needs. The `CliFlags` method returns an
array of `cli.Flag` structures that describe CLI flags that are
specific to the respective GAPI. Here's how this looked for Puppet
at the time:

```go
func (obj *GAPI) CliFlags() []cli.Flag {
        return []cli.Flag{
                cli.StringFlag{
                        Name:  "puppet, p",
                        Value: "",
                        Usage: "load graph from puppet, optionally takes a manifest or path to manifest file",
                },
                cli.StringFlag{
                        Name:  "puppet-conf",
                        Value: "",
                        Usage: "the path to an alternate puppet.conf file",
                },
        }
}
```

This is sufficient in order to display the `--puppet` and `--puppet-conf`
flags for this GAPI
in the output of `mgmt help run`. The parameters will also be accepted
from the `mgmt run` command line.

GAPI-specific parameters are quite important, because when invoked,
`mgmt run` will pick the GAPI it should use, by inspecting the parameters.
The `lang` GAPI will be used when `--lang` is present, and `puppet` if there
is a `--puppet`. This is why we decided to not allow a call such as the following:

```
mgmt run --lang /path/to/code.mcl --puppet /other/path/to/manifest.pp
```

If the new `langpuppet` GAPI was to expose these same parameters, the engine
would pick either one of the `lang`, `puppet`, or `langpuppet` GAPIs in a
semi-deterministic fashion. In other words, building this interface would
have required changes in the way that GAPIs are selected. We didn't want this
just now.

What we decided to make a "mirror CLI" that has each flag of the `lang`
and `puppet` GAPIs, but prefixed with `lp-` for `langpuppet`. I was
afraid this would make a bad user experience, but found that typing the
prefixed flags comes rather naturally after all.

This is the code in the `langpuppet` GAPI to make it happen:

```go
func (obj *GAPI) CliFlags() []cli.Flag {
        langFlags := (&lang.GAPI{}).CliFlags()
        puppetFlags := (&puppet.GAPI{}).CliFlags()

        var childFlags []cli.Flag
        for _, flag := range append(langFlags, puppetFlags...) {
                childFlags = append(childFlags, &cli.StringFlag{
                        Name:  FlagPrefix + strings.Split(flag.GetName(), ",")[0],
                        Value: "",
                        Usage: fmt.Sprintf("equivalent for '%s' when using the lang/puppet entrypoint", flag.GetName()),
                })
        }

        return childFlags
}
```

With this code, the new GAPI announces itself to the CLI subsystem and
makes its flags known. To be more precise, this piece of code does the
customization for this. There are two pieces of boilerplate code that
are necessary for the actual registration. One is this function, that
appears verbatim in each GAPI implementation:

```go
func init() {
        gapi.Register(Name, func() gapi.GAPI { return &GAPI{} }) // register
}
```

It returns a pointer to a generic GAPI interface, but `&GAPI{}` always refers
to the local GAPI structure, such as `langpuppet.GAPI`.

The other required step is the import of the package in `lib/deploy.go`:

```go
import (
	...
        _ "github.com/purpleidea/mgmt/lang"
        _ "github.com/purpleidea/mgmt/langpuppet"
        _ "github.com/purpleidea/mgmt/puppet"
        _ "github.com/purpleidea/mgmt/yamlgraph"
	...
```

So now that the CLI basics are here, let's make sure that mgmt will initialize
the GAPI object properly.

## Getting up and running

My focus was on the `Init` method first, but much later I learned
that the `Cli` method is actually more interesting and crucial, and
that it runs earlier. The last bit should not have been surprising,
because this method is the GAPI's way to determine whether it should
even run, given a specific `mgmt` CLI invocation. The method makes this
decision right at its start:

```go
func (obj *GAPI) Cli(c *cli.Context, fs engine.Fs) (*gapi.Deploy, error) {
        if !c.IsSet(FlagPrefix+lang.Name) && !c.IsSet(FlagPrefix+puppet.Name) {
                return nil, nil
        }
```

Returning `nil,nil` means that the method determined that its GAPI should
not activate. It activates when either `--lp-lang` or `--lp-puppet` is
specified. `FlagPrefix` is a constant that is specific to the `langpuppet`
package and is set to the string `lp-`. The suffixes `lang` and `puppet`
are not used here directly, either. Instead, the code imports the `Name`
constants from the GAPI packages `lang` and `puppet` respectively.

Next, the method does some consistency checking and aborts if only one of the
required flags is passed:

```go
if !c.IsSet(FlagPrefix+lang.Name) || c.String(FlagPrefix+lang.Name) == "" {
        return nil, fmt.Errorf("%s input is empty", FlagPrefix+lang.Name)
}
if !c.IsSet(FlagPrefix+puppet.Name) || c.String(FlagPrefix+puppet.Name) == "" {
        return nil, fmt.Errorf("%s input is empty", FlagPrefix+puppet.Name)
}
```

In order to do the actual work of passing the flag values to the `lang` and `puppet`
child GAPIs, the method then needs to remove the `lp-` prefixes from them. This
is more involved than I had originally anticipated. That's because the `Cli` method
does not accept some simple array of flag values. It needs a structured value of the
type `cli.Context` (from `urfave/cli`).

I opted to create a new `Context` object using `cli.NewContext`, as one does. Doing
so first requires a `flag.FlagSet`. The `flag` package is part of Go's standard library.
Here's how that looks in code:

```go
flagSet := flag.NewFlagSet(Name, flag.ContinueOnError)

for _, flag := range c.FlagNames() {
	if !c.IsSet(flag) {
		continue
	}
	childFlagName := strings.TrimPrefix(flag, FlagPrefix)
	flagSet.String(childFlagName, "", "no usage string needed here")
	flagSet.Set(childFlagName, c.String(flag))
}
```

I took a shortcut and just assumed that each flag that is of interest here will be
of type string. Hence the flags are registered using `flagSet.String()`. Each flag is
explicitly given its value using `flagSet.Set`. You can also specify a default
value when registering the flag (with the `flagSet.String` call). But the
`IsSet` check only passes when the value is specified explicitly. The child GAPI methods
rely on this check, so the `Set` call is required.

Finally, the child GAPI methods can be called, and then something important happens.
The `Cli` methods return `gapi.Deploy` objects, which in turn point to respective
`GAPI` objects. These are the workers that we actually hold on to, because they
will do the work of generating graphs from mgmt and Puppet code respectively.

```go
if langDeploy, err = (&lang.GAPI{}).Cli(cli.NewContext(c.App, flagSet, nil), fs); err != nil {
	return nil, err
}
if puppetDeploy, err = (&puppet.GAPI{}).Cli(cli.NewContext(c.App, flagSet, nil), fs); err != nil {
	return nil, err
}

return &gapi.Deploy{
	Name: Name,
	Noop: c.GlobalBool("noop"),
	Sema: c.GlobalInt("sema"),
	GAPI: &GAPI{
		langGAPI:   langDeploy.GAPI,
		puppetGAPI: puppetDeploy.GAPI,
	},
}, nil
```

The important thing to note here is that the `Cli` method is invoked on a given
`langpuppet.GAPI` object, but it returns a brand new object, wrapped in a
`gapi.Deploy` object. The engine will continue making use of that wrapped object.
Doing any initialization on the receiver `obj` in either `Cli` or `CliFlags`
will not work, because those receivers are ephemeral. This is also the reason
that the child GAPI methods are invoked on ephemeral receivers that are summarily discarded.

The `Init` method has fairly little work left to do now. Its basis is some
boilerplate code that you will find in any GAPI implementation.

```go
func (obj *GAPI) Init(data gapi.Data) error {
	if obj.initialized {
		return fmt.Errorf("already initialized")
	}
	obj.data = data // store for later

	obj.closeChan = make(chan struct{})
	obj.initialized = true
	return nil
}
```

In the middle part, I just added some code that takes care of passing on
the initialization `data` to the child GAPIs.

```go
dataLang := gapi.Data{
	Program:       obj.data.Program,
	Hostname:      obj.data.Hostname,
	World:	       obj.data.World,
	Noop:	       obj.data.Noop,
	NoConfigWatch: obj.data.NoConfigWatch,
	NoStreamWatch: obj.data.NoStreamWatch,
	Debug:	       obj.data.Debug,
	Logf: func(format string, v ...interface{}) {
		obj.data.Logf(lang.Name+": "+format, v...)
	},
}
dataPuppet := gapi.Data{
	Program:       obj.data.Program,
	Hostname:      obj.data.Hostname,
	World:	       obj.data.World,
	Noop:	       obj.data.Noop,
	NoConfigWatch: obj.data.NoConfigWatch,
	NoStreamWatch: obj.data.NoStreamWatch,
	Debug:	       obj.data.Debug,
	Logf: func(format string, v ...interface{}) {
		obj.data.Logf(puppet.Name+": "+format, v...)
	},
}

if err := obj.langGAPI.Init(dataLang); err != nil {
	return err
}
if err := obj.puppetGAPI.Init(dataPuppet); err != nil {
	return err
}
```

James helped me with the `Logf` functions. They prepend a `lang:` or `puppet:`
marker to each log message generated within a child GAPI.

This concludes the initialization methods. Once these worked, the engine was
able to get this new GAPI in position to actually feed graphs to it. The mechanism
for this is perhaps the most complicated part.

## Collecting graphs

The graph generating code in mgmt is quite asynchronous in nature. That's because
the tool is designed to receive new data from the user or other systems at
any time, and incorporate them as timely as possible. For example, in its default
configuration, the `mgmt` process will watch your mcl automation code, and when you
make changes (e.g., push to your git repository), it compiles the new
code into a graph and patches the current graph in memory with the differences.

The GAPI has three parts to enable this mechanism. The first part is the `Next`
method that each GAPI needs to implement. The engine calls it to express the
desire to receive graphs from the GAPI. The seconds part is a goroutine that
is launched by the `Next` method. It is responsible for the actual signaling
whenever a new graph can be produced by the GAPI. It writes structured values
into a channel that is returned to the engine by the `Next` method itself.

The third part is the `Graph` GAPI method. On first glance it appears to be
the most important part, but do note that implementing it will often be quite straight
forward, and it cannot properly work without a robust `Next` loop.
The whole construct assumes that the engine plays nice and obeys the (G)API
rules:

* Always call `Next` first
* Wait for a `Next` event before calling `Graph`
* Call `Graph` at most once per event

The `langpuppet` GAPI has to obey the same rules, because
it lodges itself in between the engine and its child GAPIs. Let's look at
the `Graph` method first, because it is structurally much simpler than `Next`.

```go
func (obj *GAPI) Graph() (*pgraph.Graph, error) {
        if !obj.initialized {
        	return nil, fmt.Errorf("%s: GAPI is not initialized", Name)
        }
        
        var err error
        if obj.langGraphReady {
        	obj.langGraphReady = false
        	obj.currentLangGraph, err = obj.langGAPI.Graph()
        }
        if err != nil {
        	return nil, err
        }
        
        if obj.puppetGraphReady {
        	obj.puppetGraphReady = false
        	obj.currentPuppetGraph, err = obj.puppetGAPI.Graph()
        }
        if err != nil {
        	return nil, err
        }
        
        g, err := mergeGraphs(obj.currentLangGraph, obj.currentPuppetGraph)
        return g, err
}
```

It only calls the respective child `Graph` method if that GAPI has indicated
that a graph is ready. Finally, it merges the graphs and returns the result.
The flags `langGraphReady` and `puppetGraphReady` are set in the `Next` loop,
so let's look at that now.

```go
// Next returns nil errors every time there could be a new graph.
func (obj *GAPI) Next() chan gapi.Next {
        ch := make(chan gapi.Next)
        obj.wg.Add(1)
        go func() {
                defer obj.wg.Done()
                defer close(ch) // this will run before the obj.wg.Done()
                if !obj.initialized {
                        next := gapi.Next{
                                Err:  fmt.Errorf("%s: GAPI is not initialized", Name),
                                Exit: true, // exit, b/c programming error?
                        }
                        ch <- next
                        return
                }
		
		//
		// ACTUAL EVENT LOOP HERE
		//
	}()
	return ch
}
```

`obj.wg` is a `sync.WaitGroup` and should be used in all GAPI implementations.
It allows the GAPI to cleanly shut down, including the goroutine launched here.
It's notable that the goroutine does not return errors, but feeds them into
the event channel instead.

The meat of this goroutine is the event loop. Let's dissect it a little:

```go
nextLang := obj.langGAPI.Next()
nextPuppet := obj.puppetGAPI.Next()

firstLang := false
firstPuppet := false

for {
        var err error
        exit := false
        select {
		...
```

The loop initialization is two-fold: The child GAPIs's `Next` methods
must be called so that their own event loops start running. The respective
channels are held in local variables `nextLang` and `nextPuppet`. The significance
of the boolean flags `firstLang` and `firstPuppet` will be described
a little farther below. First, let's look at the first `select` block.

```go
select {
case nextChild := <-nextLang:
	if nextChild.Err != nil {
		err = nextChild.Err
		exit = nextChild.Exit
	} else {
		obj.langGraphReady = true
		firstLang = true
	}
case nextChild := <-nextPuppet:
	if nextChild.Err != nil {
		err = nextChild.Err
		exit = nextChild.Exit
	} else {
		obj.puppetGraphReady = true
		firstPuppet = true
	}
case <-obj.closeChan:
	return
}
```

Last things first: When the engine calls the `Close` method of a GAPI
(which was not really mentioned here yet), the `closeChan` gets closed,
and the event loop can terminate immediately.

Otherwise, child GAPI events will flip both respective flags to `true`,
or in case of errors, remember the details in local variables, so that
a wrapped error can be handed out. Let's talk about the dual boolean flags
first. The `firstLang` and `firstPuppet` flags are checked first,
right after the `select` block.

```go
if (!firstLang || !firstPuppet) && err == nil {
	continue
}
```

This is a weird statement, so let's unpack it by examining the possible
scenarios. Normally, the `select` block above will finish after receiving
an event from one of the child GAPIs, say `lang`. The `firstLang` flag is
`true` now, while `firstPuppet` is still false, and `err` is `nil`. The
`continue` is hit and the loop starts over. It is supposed to get an event
from the Puppet GAPI then, and now `firstPuppet` is `true` as well.
Now the "or" expression in parenthesis evaluates to `false`, and it will
never flip back. The `firstLang` and `firstPuppet` flags never get reset.

Long story short, this little block makes sure that both child GAPIs have
sent signals before any processing is done on them. However, if one of them
produces an error, that *will* be processed immediately, and the `continue`
is skipped even though the "first" flags are not yet set.

This is what the processing looks like:

```go
if err == nil {
	obj.data.Logf("generating new composite graph...")
}
next := gapi.Next{
	Exit: exit,
	Err:  err,
}

select {
case ch <- next: // trigger a run (send a msg)
// unblock if we exit while waiting to send!
case <-obj.closeChan:
	return
}
```

Usually, `exit` and `err` will have their zero values of `false` and `nil`
respectively. The event does not carry additional information.

Handing out the event to the engine happens in yet another `select` block,
so that `closeChan` events can be processed even when nobody is listening
on our event channel `ch` anymore.

This is all. The event loop is complete, and will send appropriate events
to the engine, which will then invoke our `Graph` method, which was
described earlier. It's almost time to look at the `mergeGraphs` function,
but let's revisit some sections of `Next` and `Graph` first. I have been
lying to you a little, so that I can emphasize this now.

### Let's talk about race conditions

The first iteration of the `Next` loop didn't use boolean flags to communicate
with the `Graph` method. It just set `obj.currentLangGraph` and `obj.currentPuppetGraph`
to `nil`, and there were no booleans at all. This is what `Graph` looked like:

```go
if obj.currentLangGraph == nil {
	obj.currentLangGraph, err = obj.langGAPI.Graph()
	if err != nil {
		return nil, err
	}
}

if obj.currentPuppetGraph == nil {
	obj.currentPuppetGraph, err = obj.puppetGAPI.Graph()
	if err != nil {
		return nil, err
	}
}
```

This worked, but had a big problem: The event loop in `Next` will set
the child graph pointers to `nil` at arbitrary times, effectively
discarding the graphs. For example,
this could happen right after the `Graph` method checked and found
that they are in fact not `nil`.

This is a pretty terrible race. Adding the boolean flags made this
business a lot safer already. Yet still, the flag itself is
a value that is written by two independent threads. Bear in mind
that the goroutine launched in `Next` runs concurrently with
everything else. That's why there is still a race, in the following
scenario.

 1. The `Next` loop receives an event. It sets the boolean flag
    and passes on the event to the engine.
 2. The engine invokes `Graph`, which looks at the boolean flag,
    finds that it has been set.
 3. At this very moment, the `Next` loop receives another event
    and sets the flag to `true` again.
 4. Only now does the `Graph` method get around to resetting the
    flag to `false`.

The most recent event is lost, and the child GAPI is missing a call
to `Graph`. The most recent input update is ignored, and mgmt will be
none the wiser until randomly another event occurs. Latent subtle
failures like this are frustrating for users, and exhausting for
those who would try to debug them.

(FIXME: Is this even correct? Will the `Graph` method, after losing
the race, request the most recent graph from the child GAPI and all
is well anyway? Could we get away without locking after all?)

To be safe against this, the operation of testing and resetting
the flags must be made atomic. Go has no atomic test-and-set function,
so this calls for a classic mutex. This is how the code actually looks:

```go
var err error
obj.graphFlagMutex.Lock()
if obj.langGraphReady {
	obj.langGraphReady = false
	obj.graphFlagMutex.Unlock()
	obj.currentLangGraph, err = obj.langGAPI.Graph()
	if err != nil {
		return nil, err
	}
} else {
	obj.graphFlagMutex.Unlock()
}
```

The `graphFlagMutex` is shared by both the `langGraphReady` and the
`puppetGraphReady` flags. This isn't helpful, but it makes the variable
name a little easier to read.

```go
obj.graphFlagMutex.Lock()
if obj.puppetGraphReady {
	obj.puppetGraphReady = false
	obj.graphFlagMutex.Unlock()
	obj.currentPuppetGraph, err = obj.puppetGAPI.Graph()
	if err != nil {
		return nil, err
	}
} else {
	obj.graphFlagMutex.Unlock()
}
```

What's important here is that any reset
of a flag is in the same critical section as the test of its current value.

In the `Next` method, the lock is respected with the following modification:

```go
select {
case nextChild := <-nextLang:
	if nextChild.Err != nil {
		err = nextChild.Err
		exit = nextChild.Exit
	} else {
		obj.graphFlagMutex.Lock()
		obj.langGraphReady = true
		obj.graphFlagMutex.Unlock()
		firstLang = true
	}
```

Setting the flag is guarded by the same lock. An equivalent change was applied
to the handling of `obj.puppetGraphReady`.

```go
obj.graphFlagMutex.Lock()
obj.puppetGraphReady = true
obj.graphFlagMutex.Unlock()
firstPuppet = true
```

Since these boolean flags are the only piece of information that is written
from independent threads, this is all the synchronization that is required here.

## Merging graphs

Up to this point, the code we have examined was arguably not very creative.
(The reality is that actually writing and debugging it with little knowledge
of the general architecture of the project was quite daunting.)
It was all mechanics in order to properly integrate the lang and Puppet GAPIs
with our new wrapper. However, we skipped a crucial part in the whole process
of generating a graph out of two existing ones, implemented in the
`mergeGraphs` function (see the last step of the `Graph` method above).

The algorithm for allowing useful graph merging was described
[in an earlier post](/hacks/2018/02/13/thinking-about-migration-from-puppet-to-mgmt/).
If you intend to follow along with the logic here, I suggest you read
that article first (the last part, that is).
The general approach is to build a new graph by
including all vertices from both input graphs.

Some vertices are merged between the two input graphs.
For example, the resource `noop[puppet_handover]` from mcl code will be merged
with the empty Puppet class `mgmt_handover` (resulting in a new
`noop[handover]`).
Consistency checks should generate errors when either input includes
at least one "mergeable" vertex that has no counterpart in the other input.

The complete implementation lives in `langpuppet/merge.go`. Let's start
with the boilerplate consistency checks.

```go
func mergeGraphs(graphFromLang, graphFromPuppet *pgraph.Graph) (*pgraph.Graph, error) {
	if graphFromLang == nil || graphFromPuppet == nil {
		return nil, fmt.Errorf("cannot merge graphs until both child graphs are loaded")
	}

	result, err := pgraph.NewGraph(graphFromLang.Name + "+" + graphFromPuppet.Name)
	if err != nil {
		return nil, err
	}
```

Once these passed,
the first step for the actual algorithm is to copy the graph produced
by the `lang` GAPI verbatim. All vertices and edges are incorporated
in the `result` graph. Along the way, the program takes note of vertices
that represent `noop` resources with a name starting in `puppet_`. These
are candidates for merging, which will be kept in a `map`.

```go
mergeTargets := make(map[string]pgraph.Vertex)

// first add all vertices from the lang graph
for _, vertex := range graphFromLang.Vertices() {
	if strings.Index(vertex.String(), "noop["+MergePrefixLang) == 0 {
		resource, ok := vertex.(engine.Res)
		if !ok {
			return nil, fmt.Errorf("vertex %s is not a named resource", vertex.String())
		}
		basename := strings.TrimPrefix(resource.Name(), MergePrefixLang)
		resource.SetName(basename)
		mergeTargets[basename] = vertex
	}
	result.AddVertex(vertex)
	for _, neighbor := range graphFromLang.OutgoingGraphVertices(vertex) {
		result.AddVertex(neighbor)
		result.AddEdge(vertex, neighbor, graphFromLang.FindEdge(vertex, neighbor))
	}
}
```

Do note the typecast of `vertex` to an `engine.Res` pointer. The `pgraph.Pgraph`
is very generic. Any stringable object can be a vertex. (Edges are just as indistinct.)
So, in order to use any information from a vertex, its actual content type must
be inferred and extracted with a cast like above.

In order to make merging easier down the line, the "base names" of the pertinent resources
are extracted here. For example, the resource `noop[puppet_post_deployment]` gets mapped
to the string `post_deployment`. In the `result` graph, this string also becomes the
name for this resource, e.g. `noop[post_deployment]`.

The `MergePrefixLang` constant in the code above is
defined as follows, along with its counterpart:

```go
const (
        // MergePrefixLang is how a mergeable vertex name starts in mcl code.
        MergePrefixLang = "puppet_"
        // MergePrefixPuppet is how a mergeable Puppet class name starts.
        MergePrefixPuppet = "mgmt_"
)
```

Finally, the respective vertex is added to the `result` graph, along with all its
outgoing edges. The `pgraph` API requires that the vertices on both ends of the edge
are in the graph before an edge is added. That's why each "neighbor" vertex is
also added during this loop iteration.

This implies that most vertices are added at least twice, one time during their
respective loop iteration, but also from whatever other vertices
have outgoing edges to them. This is fine. The `pgraph` API implements `AddVertex`
as an idempotent action that can be repeated at will.

After the `lang` graph is processed entirely, the `puppet` graph gets
incorporated. The algorithm does two passes on this graph. First, the
algorithm builds an internal map of vertices that will be merged.
During this pass, the program also identifies the "first" vertex of Puppet's
graph (the start marker of Puppet's "main stage") and saves a pointer to it.
This is used to make the second pass more orderly.

```go
var anchor pgraph.Vertex
mergePairs := make(map[pgraph.Vertex]pgraph.Vertex)

// do a scan through the Puppet graph, and mark all vertices that will be
// subject to a merge, so it will be easier do generate the new edges
// in the final pass
for _, vertex := range graphFromPuppet.Vertices() {
	if vertex.String() == "noop[admissible_Stage[main]]" {
		// we can start a depth first search here
		anchor = vertex
		continue
	}
```

The top of the loop concludes the handling for the "anchor" vertex.
The rest of the loop body deals with merging preparation.

```go
// at this stage we don't distinguis between class start and end
if strings.Index(vertex.String(), "noop[admissible_Class["+strings.Title(MergePrefixPuppet)) != 0 &&
	strings.Index(vertex.String(), "noop[completed_Class["+strings.Title(MergePrefixPuppet)) != 0 {
	continue
}

resource, ok := vertex.(engine.Res)
if !ok {
	return nil, fmt.Errorf("vertex %s is not a named resource", vertex.String())
}
```

The consistency check here should never fail, because Puppet's graphs have no vertices
that aren't resources (after translation to mgmt, that is). Note the use of `strings.Title`, which
capitalizes start of the word in the `MergePrefixPuppet` constant. This is required
because of Puppet's convention to capitalize references in this fashion.
The start and end markers of `class mgmt_post_deployment` in Puppet are named
`admissible_Class[Mgmt_post_deployment]` and `completed_Class[Mgmt_post_deployment]`
respectively. That's why the program scans for `Mgmt_` rather than `mgmt_`.

```go
// strip either prefix (plus the closing bracket)
basename := strings.TrimSuffix(
        strings.TrimPrefix(
                strings.TrimPrefix(resource.Name(),
                        "admissible_Class["+strings.Title(MergePrefixPuppet)),
                "completed_Class["+strings.Title(MergePrefixPuppet)),
        "]")
```

Wow, who the hell wrote this? The idea is, we can just strip *both* possible
prefixes from whatever vertex we're looking at, because one of these operations
will be a no-op. Also, since I'm lazy, the closing bracket gets stripped off
in the same statement. A much better way to write this is:

```go
basename := strings.TrimSuffix(resource.Name(), "]")
basename = strings.TrimPrefix(basename,"admissible_Class["+strings.Title(MergePrefixPuppet))
basename = strings.TrimPrefix(basename,"completed_Class["+strings.Title(MergePrefixPuppet))
```

This refactoring is probably already in mgmt's `master` by the time you read this.

By this point, the algorithm has identified a vertex in the graph from the Puppet
translator that can be merged. It tries and saves a mapping from this vertex
to its counterpart in the `result` graph.

```go
if _, found := mergeTargets[basename]; !found {
	// FIXME: should be a warning not an error?
	return nil, fmt.Errorf("puppet graph has unmatched class %s%s", MergePrefixPuppet, basename)
}

mergePairs[vertex] = mergeTargets[basename]
```

Finally, perform another consistency check and make sure that merge classes
are really empty.

```go
if strings.Index(resource.Name(), "admissible_Class["+strings.Title(MergePrefixPuppet)) != 0 {
	continue
}

// is there more than one edge outgoing from the class start?
if graphFromPuppet.OutDegree()[vertex] > 1 {
	return nil, fmt.Errorf("class %s is not empty", basename)
}

// does this edge not lead to the class end?
next := graphFromPuppet.OutgoingGraphVertices(vertex)[0]
if next.String() != "noop[completed_Class["+strings.Title(MergePrefixPuppet)+basename+"]]" {
	return nil, fmt.Errorf("class %s%s is not empty, start is followed by %s", MergePrefixPuppet, basename, next.String())
}
```

If such a class was non-empty, the algorithm would go ahead and merge vertices
in a way that would lead to circles in the resulting graph. (TODO: diagram!)

Now for the final pass of the input graph from Puppet. This one is a little
peculiar. In order to not repeat the merging logic too often in code,
vertices are only added indirectly. The loop visits each vertex,
but only adds its "neighbors" (vertices at the end of outgoing edges) and
not the current vertex itself. This will still happen multiple times
to vertices with more than one incoming edge, but the code is a little
more compact this way, and thus easier to reason about.

```go
merged := make(map[pgraph.Vertex]bool)
result.AddVertex(anchor)
// traverse the puppet graph, add all vertices and perform merges
// using DFS so we can be sure the "admissible" is visited before the "completed" vertex
for _, vertex := range graphFromPuppet.DFS(anchor) {
	source := vertex

	// when adding edges, the source might be a different vertex
	// than the current one, if this is a merged vertex
	if _, found := mergePairs[vertex]; found {
		source = mergePairs[vertex]
	}
```

The first vertex must be added outside the loop, because it has no incoming edges.
As mentioned in the code comments, a depth first search is performed so that
I can make assumptions about the order in which some vertices are seen.
The code above also takes care of half the merging logic. It ensures that
edges that are outgoing from merging vertices will actually go out of the
merged vertex.

TODO: diagram

Next is the actual work of adding all neighbors, i.e. vertices with edges
that are incoming from the vertex currently being iterated.

```go
// the current vertex has been added by previous iterations,
// we only add neighbors here
for _, neighbor := range graphFromPuppet.OutgoingGraphVertices(vertex) {
	if strings.Index(neighbor.String(), "noop[admissible_Class["+strings.Title(MergePrefixPuppet)) == 0 {
		result.AddEdge(source, mergePairs[neighbor], graphFromPuppet.FindEdge(vertex, neighbor))
		continue
	}
	if strings.Index(neighbor.String(), "noop[completed_Class["+strings.Title(MergePrefixPuppet)) == 0 {
		// mark target vertex as merged
		merged[mergePairs[neighbor]] = true
		continue
	}
	// if we reach here, this neighbor is a regular vertex
	result.AddVertex(neighbor)
	result.AddEdge(source, neighbor, graphFromPuppet.FindEdge(vertex, neighbor))
}
```

Note the assumption in the comment on top. It is safe because this is a
depth first search through the graph received from Puppet.

The two `if` blocks
in the loop body implement the second half of the vertex merge. Both cases
skip the `AddVertex` call, because the resulting graph will contain the
merged vertex instead of these class containment nodes. This merge vertex is
already in the result graph. It was added in the earlier import of the
graph that was compiled from mcl code.

The first `if` block handles the vertex that marks the start of the
respective class. Since the merged vertex (`mergePairs[neighbor]`) is
already in the result graph, only the edge to that vertex needs adding.
It uses the local `source` vertex from the previous code snippet.

The other block is entered when the neighbor in question is the end node
of a merging class. This vertex can only have one incoming edge, because
it is mandated that the merge classes are empty. The only edge is from
the `admissible` vertex to the `completed` (or if your will, start to end).
This edge is dropped completely, because both vertexes are merged into one.
The only thing that happens here is marking the "target" vertex in the
result graph as completely merged, for later consistency checking.

Note that whenever the code adds an edge to the result graph,
it looks up the very edge object in the source graph.
It does not even really care what kind of object serves as an edge.
Remember that any stringable structure can be an edge (or
a vertex, for that matter).

The aforementioned consistency check is the last bit of code
in the `mergeGraph` function.

```go
for _, vertex := range mergeTargets {
	if !merged[vertex] {
		// FIXME: should be a warning not an error?
		return nil, fmt.Errorf("lang graph has unmatched %s", vertex.String())
	}
}
```

I've left some `FIXME` notes like the one above, because I'm currently
not sure what will make for a better UX: Fail a graph that cannot be
cleanly merged, or simply log a warning about the fact. At the moment
I lean towards failing outright, so that's what the code still does.

## Summary

![Diagram of the GAPI lifecycle in mgmt](https://user-images.githubusercontent.com/436765/51065512-7b40c280-1605-11e9-976a-64b93a976571.png)

The GAPI system in mgmt's code base is not terribly complex, but it's
hard to infer its design from just reading code. Do ask James for help
with new features. Once you grok the basics, making a new GAPI is not
so hard.

The `langpuppet` GAPI almost completely relies on existing code from
the `lang` and `puppet` GAPIs. It carefully implements the `Next`/`Graph`
loop that is responsible for the GAPI's functionality along most of its
life cycle.

Although there is not much testing data yet, the support for mixed
mcl/Puppet code is essentially functional now.
