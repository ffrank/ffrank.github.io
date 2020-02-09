---
layout: post
category: features
tags: [ puppet, module, resource, shell ]
summary: Introducing a new mode of running Puppet and syncing resources
---

## Abstract

In the course of improving the Puppet support in mgmt, I created yet another
functionality for the `puppet yamlresource` module. It allows writing
arbitrary resource descriptions to Puppet's standard input in a YAML format.
Puppet synchronizes each received resource in turn, printing a line of
diagnostics in JSON format in response. This allows more flexible scenarios
than running the compiler for each resource, or even just a simple process
like `puppet resource`, which still implies heavy overhead.

## A faster `puppet resource` command

Through my work on the Puppet integration in mgmt, I had need to [improve
on the concept]({% post_url 2016-08-19-translating-all-the-things %}) of
the `puppet resource` command before. It lacks support for all but the
most plain and simple resources. To make it more universal, I supplemented it
with the `puppet yamlresource` command from the `ffrank-yamlresource` Puppet
module. This new command works just like `puppet resource`, but accepts
parameter input in YAML format. This way, it can receive arbitrary parameter
values, such as arrays.

```
puppet apply -e "cron { 'twice hourly': minute => [ 10, 40 ] }"
# same without compiling Puppet code:
puppet yamlresource cron "twice hourly" '{ minute: [ 10, 40 ] }'
```

This enabled us to use Puppet flexibly from mgmt, sending any untranslated
resources back to Puppet for synchronization. The performance suffered
extremely in this scenario, because each single resource event required its own
`puppet yamlresource` invocation, with several seconds of overhead (on most
systems) for spinning up the Ruby interpreter and loading Puppet.

My idea for overcoming this performance issue was derived from this excerpt
of the man page for rrdtool:

> REMOTE CONTROL
>
> When you start RRDtool with the command line option '-' it waits for input via standard input ( STDIN ). With this feature you can improve performance by attaching RRDtool to another process ( MRTG is one example) through a set of pipes.

Piping instructions to a persistent process is an elegant way to avoid
launch and shutdown overhead (or so I have come to believe). So I set out
to teach the same to Puppet. The earlier `puppet yamlresource` experience
had shown that it's important to accept structured data for the resource
descriptions. In fact, the YAML approach was working pretty well already.
So I concluded that this format would do well enough in a pipe as well.
As such, the `puppet yamlresource` face seemed to be as good a place as any
to add a pipe receiving implementation.

I followed through, so this is what you can do as of right now:

 1. Get the `ffrank-yamlresource` module from the Puppet Forge (make sure
    you're at least at version 0.2.0)

        puppet module install ffrank-yamlresource

 2. Run the command 
 
        puppet yamlresource receive

 3. Enter resources in the following format

    ```
    file /path/to/my/file { ensure: file, mode: "0644" }
    service cron { ensure: running, enable: true }
    ```

Some remarks on the input format:

 * The YAML object is always a hash or dictionary.
 * Neither resource type nor title nor YAML string can be quoted.
 * Quotes do work inside the YAML document.
 * Since quotes are not supported, resource titles cannot contain spaces.
 * It is therefor also possible (but not covered in detail here) to pass
   the entire resource as a JSON representation.

Each line of input will yield a JSON object in return, e.g.

```
{"resource":"File[/tmp/a-whole-new-file]","failed":false,"changed":true,"noop":false,"error":false,"exception":null}
```

It indicates the following

 * the descriptor of the resource (`resource`)
 * whether there was a synchronization issue (`failed`)
 * whether Puppet applied a change (`changed`)
 * whether Puppet was in no-op mode (`noop`)
 * whether there was a problem with the input (`error`)
 * the error message, if any (`exception`)

This is how an error is reported:

```
file haha { ensure: file }
Error: (Applying File[haha]) Parameter path failed on File[haha]: File paths must be fully qualified, not 'haha'
{"failed":true,"changed":false,"error":true,"exception":"Parameter path failed on File[haha]: File paths must be fully qualified, not 'haha'"}
```

The `Error:` line is written by Puppet's resource engine to the standard error
stream (stderr). The JSON object indicates the same message in its content.
This is helpful when connecting Puppet with other software. The client needs
not try and decipher messages that may or may not appear on stderr. All
required information can be found in the JSON response.

As a very simple usage example, you can create a shell script that writes
suitable resource descriptions, and pipe its output to Puppet like this:

```
./generate-resources | puppet yamlresource receive
```

When run this way, Puppet's output (JSON objects) will just be written to your
terminal. In order to consume them, the output needs to be redirected.

```
./generate-resources \
	| puppet yamlresource receive 2>/dev/null \
	| ./check-puppet-output
```

In this example, `check-puppet-output` might be a script that uses `jq` to
inspect the objects sent by Puppet.

We have not yet explored the range of possible uses of Puppet that this will
open up, apart from a closer integration in mgmt. (There will be a dedicated
post on this topic soon.) Credit for the title "Puppet Scripting Host" go to
[Ben Ford](https://twitter.com/binford2k). I had not thought of the tool this
way, but it can certainly fill such a role.

# Summary

The new `puppet yamlresource receive` command is publicly available from the
Forge module `ffrank-yamlresource`. It enhances Puppet's mgmt integration
and allows the user to flexibly synchronize resources ad hoc, saving much
overhead compared to the `puppet resource` tool.
