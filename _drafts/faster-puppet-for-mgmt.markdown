---
layout: post
category: features
tags: [ mgmt, puppet, resource, pippet ]
summary: Explaining the improved puppet resource workaround in mgmt
---

## Abstract

When running a graph that was created from a Puppet manifest, mgmt has so far
used a workaround for each resource that does not translate. This workaround
involved an `exec` type resource, shelling out to a variant of `puppet
resource`. Using the new `puppet yamlresource receive` subcommand, mgmt can now
send resources to a persistent Puppet process instead. The native support for
this is implemented through the new pseudo-resource `pippet` in mgmt.
It's not meant to be used in mgmt code, but the Puppet code translator will
rely on it when required.

## Ensuring semantics

I have blogged quite extensively about the [Puppet compatibility
layers]({% post_url 2018-02-13-thinking-about-migration-from-puppet-to-mgmt %})
in the past. They work because the most important core resource types from
Puppet (such as `file` and `package`) can be found in mgmt as well. We expect
that much Puppet code that is out there could be run through native mgmt
resources.

The Puppet ecosystem of custom resource types in module is quite large,
however. Even the full set of "core adjacent" types is nothing to scoff at.
Any resources that mgmt does not support need to get synchronized in
another way.

```
microwave_oven { "CN-4PK":
    connected => true,
    ...
}
```

Additionally, any actual instances even of mgmt supported types
cannot be modeled correctly by mgmt if they use properties that are
exclusive to Puppet. For example, mgmt does not support downloading files from
a Puppet server.

```
file { "/etc/motd":
  source => "puppet:///modules/motd/default",
}
```

To provide for such resources, we implemented [a
workaround]({% post_url 2016-08-19-translating-all-the-things %}) that would
substitute such unsupported resources by a stand-in. The idea was to let
Puppet do the work of synchronizing these resources. The basic approach was
to generate `exec` resources that roughly do the following:

```
exec "substitute Cron[refresh-my-stuff]" {
    cmd => "puppet resource cron refresh-my-stuff command=/path/to/script minute=...
```

The actual implementation is a little more involved (see article linked above),
and had me introduce the `puppet yamlresource` face. We have recently boosted
its performance using [a new
subcommand]({% post_url 2020-02-09-puppet-scripting-host %}) `puppet
yamlresource receive`. It allows users or programs to send resource
descriptions directly to Puppet's stdin stream. Puppet will sync these
resources directly and immediately, one by one.

In order to allow mgmt to take advantage of this latest addition, we needed
to add three properties to mgmt with respect to its Puppet integration:

 * mgmt needs to launch and track the Puppet receiver process
 * individual resources in mgmt's graph must be able to use this receiver
 * the Puppet manifest translator must emit such resources for mgmt

Let's look into each of these steps in turn.

## Running a persistent Puppet

When using `puppet yamlresource receive` in order to connect Puppet to
another piece of software (in this case, mgmt), the processes communicate
via I/O pipes (see following diagram).

![mgmt with pipe to Puppet](http://ffrank.github.io/presentations/2020-02-puppet-from-mgmt-on-overdrive/pippet.png)

This design does not really follow a pattern that I have encountered in any
of the projects I have worked on before. Implementing it is not super hard,
fortunately. Essentially the process is just kicked off in the background,
and mgmt just needs to be careful to hold on to its stdin and stdout I/O
channels.

For easy management inside mgmt, I borrowed the Singleton pattern from
object oriented programming. There is a global struct with a synchronized
function that will retrieve it.

```
type pippetReceiver struct {
        stdin         io.WriteCloser
        stdout        io.ReadCloser
        registerMutex sync.Mutex
        applyMutex    sync.Mutex
        registered    int
}

func getPippetReceiverInstance() *pippetReceiver {
        for pippetReceiverInstance == nil {
                pippetReceiverOnce.Do(func() { pippetReceiverInstance = &pippetReceiver{} })
        }
        return pippetReceiverInstance
}
```

`pippetReceiverOnce` is a [sync.Once](https://golang.org/pkg/sync/#Once) object
that makes the method thread safe. ("pippet" is not a typo and will be explained
in the next section.) The `pippetReceiver` is not properly initialized in this
moment. That does not happen before the first consumer _registers_ itself. 

```
func (obj *pippetReceiver) Register() error {
        obj.registerMutex.Lock()
        defer obj.registerMutex.Unlock()
        obj.registered = obj.registered + 1
        if obj.registered > 1 {
                return nil
        }
        // count was increased from 0 to 1, we need to (re-)init
        var err error
        if err = obj.Init(); err != nil {
                obj.registered = 0
        }
        return err
}
```

Users of this data object are typically graph resources that rely on
Puppet. The next section (below) goes into more detail about these resources.
They call yet another function that implements the protocol of
`puppet yamlresource receive`, passing the resource details, and a reference
to the mentioned global struct. The reference uses a minimalistic interface
that this structure implements.

```
type PippetRunner interface {
        LockApply()
        UnlockApply()
        InputStream() io.WriteCloser
        OutputStream() io.ReadCloser
}

// applyPippetRes does the actual work of making Puppet synchronize a resource.
func applyPippetRes(runner PippetRunner, resource *PippetRes) (bool, error) {
        ... #*
```

The `LockApply` and `UnlockApply` method just wrap the `applyMutex` from the
`struct pippetReceiver`. `InputStream` and `OutputStreams` are basically getter
methods for `stdin` and `stdout` in the same struct.

An aside on golang code design: The synchronization function was originally
implemented as a method of `struct pippetReceiver` itself. That's because
I was in the mind trap of thinking in terms of object oriented programming.
I had intended to build tests for all this code by generating a mock of the
(extensive) general interface. (The mind trap leading here was the absurdly
powerful mocking that is available in Ruby.)

What I had to realize is that this made testing very hard. In fact, I am
dedicating a whole section to this topic towards the end of this article.

## Graph nodes for Puppet

Now that mgmt is capable of running Puppet in the background, making it wait
for resources to sync, mgmt will need a way to also supply these resources.
This cannot be done with an `exec` type node, because these write their
output to the system outside the mgmt process. The pipe to Puppet lives
within the mgmt process, though.

Besides, running an `exec` in order to write a resource description into
Puppet's stdin stream would partly defeat the purpose. Overhead is to be
avoided. What is needed here is a dedicated resource type. Because it will
write information into a *pipe* to *Puppet*, it was named "Pippet".

The structure of this type is very simple, designed specifically for easy
marshalling.

```
type PippetRes struct {
        ...
        Type string `yaml:"type" json:"type"`
        Title string `yaml:"title" json:"title"`
        Params string `yaml:"params" json:"params"`

        runner *pippetReceiver
}
```

Notably, the `Params` are not stored as map, but just as a string. They are
expected to be represented by a YAML dictionary. This YAML will be passed
directly to Puppet, wrapped in a JSON object, along with the `Type` and
`Title` of the resource. Puppet does not really require the JSON wrapping,
but it is done in order to avoid any whitespace issues. (For more details,
I would direct you to the [article about `puppet yamlresource
receive`]({% post_url 2020-02-09-puppet-scripting-host %}) again.)
This unusual practice is why the `PippetRes struct` contains JSON field tags.
You will not find these in most mgmt resource structs.

Pippet resources are not really supposed to be used in mgmt code. They are
meant to result from Puppet manifest translations. The next section provides
some details.

## New graphs from Puppet

The `ffrank-mgmtgraph` Puppet module received a new "default translation" rule.
You can see [the definition of this
rule](https://github.com/ffrank/puppet-mgmtgraph/blob/master/lib/puppetx/catalog_translation/type/default_translation_to_pippet.rb)
in the module code, written in the [resource translator
DSL](https://github.com/ffrank/puppet-mgmtgraph/blob/master/DSL.md).
This rule replaces the original `exec` based workaround (which will be kept
for reasons of nostalgia, and possibly compatibilty).

This is really all there is to it. As this changes the default behavior of
the `mgmtgraph` Puppet face, it was part of a "major" release (air quotes
courtesy of the fact that we're still pre-1.0 release). We have dropped
compatibility with mgmt 0.0.20 and older (which do not have the `pippet`
resource yet). We also discovered some code cruft that got cleaned up,
but that is beyond the scope of this article.

## A word about tests

When teaching mgmt to run the Puppet resource receiver process, I first
approached the code with a "Ruby state of mind". A struct with fields for
all the runtime data was created, with methods to perform initialization,
tracking for resources that use Puppet this way, and the actual communication.
My first idea for building tests for this
[implementation started with Ruby state of mind]
[intended to mock pippetrunner, stub sync method]
[found that this makes testing brittle if not impossible]

[looked into dependency injection instead]
[found great blog post]
[incidentally understood Go interface ideas much better]

[sync function explicitly designed for dependency injection]
[mock implementation for pippetrunner interface]

## Summary

[both mgmt and the translator were enhanced for faster puppet integration]
[as a side-effect, Puppet received a scripting host]
[closing in on ability to run arbitrary Puppet code]
