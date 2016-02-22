---
layout: post
category: features
tags: [ puppet, catalog, compiler, mgmt, graph ]
summary: An account of the way towards the Puppet "transpiler" for mgmt.
---

Have you heard of `mgmt` yet? It's (currently) a prototype config management
engine written by [James](https://ttboj.wordpress.com/) and brings some exciting
new ideas to the table, building on the proven concepts of Puppet. You should probably
[read up on it](https://ttboj.wordpress.com/2016/01/18/next-generation-configuration-mgmt/)
right now.

James gave the inaugural demo at ConfigMgmtCamp 2016 in Gent, Belgium, and mentioned
that he can picture a sort of "transpiler" that will create resource graphs from
Puppet manifest code. That strung a true cord with me and I couldn't really rest
until I had that working. Alas, the endeavor was not as simple as I had anticipated at first.

### Compile and rearrange

At first I thought this would be easy. The very first thing I tried was `puppet master --compile`
to see if the plain catalog is usable. It looks promising enough:

```puppet
# /tmp/demo.pp
file { 'demo-file':
  path => '/tmp//foo',
  ensure => 'file',
  content => "Testing graph compilation\n",
}
->
exec { 'demo-process':
  command => '/bin/true',
  path => '/bin:/usr/bin',
}
```

It gets compiled into the following catalog:

```
$ puppet master --compile demo.example.net --manifest /tmp/demo.pp
Notice: Compiled catalog for demo.example.net in environment production in 0.21 seconds
{
  "tags": ["settings"],
  "name": "demo.example.net",
  "version": 1455828757,
  "code_id": null,
  "catalog_uuid": "6167049c-f1ea-4dd8-a460-e32a312f7318",
  "environment": "production",
  "resources": [
    {
      "type": "Stage",
      "title": "main",
      "tags": ["stage"],
      "exported": false,
      "parameters": {
        "name": "main"
      }
    },
    {
      "type": "Class",
      "title": "Settings",
      "tags": ["class","settings"],
    },
    {
      "type": "Class",
      "title": "main",
      "tags": ["class"],
      "exported": false,
      "parameters": {
        "name": "main"
      }
    },
    {
      "type": "File",
      "title": "demo-file",
      "tags": ["file","demo-file","class"],
      "file": "/tmp/demo.pp",
      "line": 2,
      "exported": false,
      "parameters": {
        "path": "/tmp//foo",
        "ensure": "file",
        "content": "Testing graph compilation\n",
        "before": [
          "Exec[demo-process]"
        ]
      }
    },
    {
      "type": "Exec",
      "title": "demo-process",
      "tags": ["exec","demo-process","class"],
      "file": "/tmp/demo.pp",
      "line": 8,
      "exported": false,
      "parameters": {
        "command": "/bin/true",
        "path": "/bin:/usr/bin"
      }
    }
  ],
  "edges": [
    {
      "source": "Stage[main]",
      "target": "Class[Settings]"
    },
    {
      "source": "Stage[main]",
      "target": "Class[main]"
    },
    {
      "source": "Class[main]",
      "target": "File[demo-file]"
    },
    {
      "source": "Class[main]",
      "target": "Exec[demo-process]"
    }
  ],
  "classes": [
    "settings"
  ]
}
```

Write this in YAML, and it almost looks like a graph fit for use with `mgmt`.
Here's an example of those:

```YAML
---
graph: mygraph
types:
  exec:
  - name: exec1
    cmd: sleep 10s
    shell: ''
    timeout: 0
    watchcmd: ''
    watchshell: ''
    ifcmd: ''
    ifshell: ''
    pollint: 0
    state: present
  - name: exec2
    cmd: sleep 10s
    shell: ''
    timeout: 0
    watchcmd: ''
    watchshell: ''
    ifcmd: ''
    ifshell: ''
    pollint: 0
    state: present
  - name: exec3
    cmd: sleep 10s
    shell: ''
    timeout: 0
    watchcmd: ''
    watchshell: ''
    ifcmd: ''
    ifshell: ''
    pollint: 0
    state: present
  - name: exec4
    cmd: sleep 10s
    shell: ''
    timeout: 0
    watchcmd: ''
    watchshell: ''
    ifcmd: ''
    ifshell: ''
    pollint: 0
    state: present
edges:
- name: e1
  from:
    type: exec
    name: exec1
  to:
    type: exec
    name: exec2
- name: e2
  from:
    type: exec
    name: exec2
  to:
    type: exec
    name: exec3
```

The best part is that transforming the former into the latter is just a data transformation.
Both are just hashes, their values comprising structured data (more hashes, arrays and strings).
The structures are different. Some rehashing is required. Most importantly, the data is not 100%
compatible and needs translating.

Certain keys are specific to either tool. For example, Puppet has a `path` parameter for
the `exec` type, and `mgmt` supports keys such as `watchcmd` or `ifshell`. Sometimes keys
are named differently, such as `ifcmd` instead of `onlyif`. Sometimes `mgmt` uses different
symbolic values, such as `exists` for `file` resources, instead of `present` or `file`.

All of these transformations can be performed in a straight-forward procedural fashion.
My operations DNA kicked in and I sat down to build a script. When interacting with YAML,
Ruby is my current go-to language. Perl is viable at this level of complexity, but its
capacity to work with YAML is limited (from my experience).

The original script worked pretty well. It flattened the resource representations and
sorted them away into one array per resource type. The parameter values and names were
translated as needed. The graph edges were filtered, with  only connections between
actual resources being kept. Fine and good. There are a few issues, though.

### Getting a better catalog

If you have followed very carefully, you might have spotted something missing from
the catalog above. If you did, you have a better eye for detail than me. This one had me
puzzled for a minute.

Take another look at the edges in the output of `puppet master --compile`:

```YAML
  "edges": [
    {
      "source": "Stage[main]",
      "target": "Class[Settings]"
    },
    {
      "source": "Stage[main]",
      "target": "Class[main]"
    },
    {
      "source": "Class[main]",
      "target": "File[demo-file]"
    },
    {
      "source": "Class[main]",
      "target": "Exec[demo-process]"
    }
  ],
```

It's an array, nice and easy. It does reference both resources from the manifest.
However, the one edge that is of interest to me here, the one *between* the `file`
and the `exec`, is missing! These edges only represent containment (with `Class[main]`
being a part of `Stage[main]` and both resources inside `Class[main]`), but no
dependencies.

This was a show-stopper for this approach, but there's even more. There is something
to remember about the Puppet master. It's kind of dumb. Or rather, it is quite pragmatic
about its output. It only does the most basic code validation, and if a manifest
looks at least vaguely valid, it will send a catalog down the pipes to the agent.
All the magic and powerful safeties are implemented agent side. (This is why
sometimes you get to wait the full compile time, only to have the agent inform you
that some detail is off.)

Specifically, the agent will perform *munging*, the transformation of input values
to their canonical representation. For example, my test manifest intentionally
contains a slight error.

```puppet
file { 'demo-file':
  path => '/tmp//foo',
  ensure => 'file',
  content => "Testing graph compilation\n",
}
```

The double slash in the `path` attribute is not an issue on any supported platform,
but it is still important that Puppet munges the path into the canonical form.
Otherwise, the following manifest would produce a valid catalog:

```puppet
file { '/etc/passwd': ensure => present }
file { '/etc//passwd': ensure => absent }
```

Two resources manage the same entity, and ensure conflicting state. Puppet tries
hard to avoid this type of situation. Munging input values is an important aspect
of this mechanism. By transforming the second path to its canonical representation,
Puppet notices that it is indeed identical to the other one.

In `mgmt`, this situation would be even more dangerous:
Both resources are event based, and will join in a veritable battle to try and
make the filesystem converge.

So what I wanted `mgmt` to consume is the cleaned up catalog that the agent derives
from the more plain data it receives from the master. The agent also creates
an actual dependency graph. I wanted to make use of that as well. Creating edges
on my own based on `before` and `require` values would have felt pretty silly.

Enter another go-to tool of mine: Puppet's own `apply` subcommand. Not only will it
launch an "instant" agent transaction from a local manifest, it even allows for
one-liners right from the shell through its `-e` parameter. What's not to like?

### Prying into the workings

The new plan was to base the translation tool on `puppet apply`. It should perform
all the preparatory steps, but instead of kicking off an agent transaction, it must
print the resource graph in a YAML format suitable for `mgmt`.

First, I was curious about just where in the source of `puppet apply` these preparations
are initiated. For this research, I used a shortcut. I created a temporary `munge` hook
for the `message` parameter of the `notify` type.

```ruby
munge do |value|
  puts caller * "\n\t"
end
```

It does not perform any munging at all. Instead, as a side effect if you will, it dumps
its own stack trace on the console. The following invocation causes this:

```
$ bundle exec puppet apply -e 'notify { "test": }'
Notice: Compiled catalog for fflaptop.local in environment production in 0.09 seconds
/home/ffrank/git/puppet/lib/puppet/parameter.rb:419:in `munge'
	/home/ffrank/git/puppet/lib/puppet/property.rb:484:in `block in should='
...
	/home/ffrank/git/puppet/lib/puppet/resource/catalog.rb:552:in `to_catalog'
	/home/ffrank/git/puppet/lib/puppet/resource/catalog.rb:442:in `to_ral'
	/home/ffrank/git/puppet/lib/puppet/application/apply.rb:263:in `block in main'
...
```

I shortened the output to the pertinent parts. Through the course of its `main` method,
the `apply` command calls a catalog method called `to_ral`. It makes sense that this is
the missing piece. The compiler sends a "symbolic" catalog if you will, and the agent
has to raise it into its Resource Abstraction Layer. I actually [described this in a
presentation](http://ffrank.github.io/presentations/2015-10-puppetconf-hacking-types-and-providers/hacking-types-providers.html#/8/1) at PuppetConf.

As an aside, while examining [the responsible piece of code](https://github.com/puppetlabs/puppet/blob/e945d75e6843de39304804bc57fe089c34e3af12/lib/puppet/application/apply.rb#L262-L272),
I noticed that there is the option to invoke another catalog method called
`write_resource_file`. I examined this resource file, but it is a mere list
and not helpful in the endeavour of creating a graph for `mgmt`. In addition
to the resources file, Puppet will create a `graphs` directory in its state
cache (see the `graphdir` config setting), but this is for the GraphViz .dot
files only. This is not very helpful either.

The RAL catalog, on the other hand, is just what I was looking for. It contains
representations of all resources that have been blessed by Puppet's powerful
abstraction code. I did spend quite some time with [Pry](http://pryrepl.org/)
to find the relationship edges in there, until I realized (by trudging through
the [transaction code](https://github.com/puppetlabs/puppet/blob/master/lib/puppet/transaction.rb))
that the graph is never written to an attribute. It is always generated on-the-fly
by the `relationship_graph` catalog method.

Another aside: The relationship graph emerges from two inputs. There is the
set of resources from the catalog, of course. Each can declare one or more
relationships. This happens explicitly through resource parameters like
`before` and `require`, or automatically through `autorequire` hooks.

But the resulting graph can put different weights on its edges through
*priorities*. Each resource gets a numeric priority during graph generation.
This is how manifest ordering works, for example. All it does is to select
a certain *prioritizer* that is used when choosing resource priorities.
The prioritizer is chosen by the user through the `ordering` configuration value.

### Coaxing out the mgmt graph

After gathering all the technical background information, it was now time
to build a working translator. I will admit, at this point I was growing a little
impatient. I had spent several evenings just doing research,
when I had originally estimated to build a script in just a few hours' time.

So by now, I was out for the path of least resistance. My original idea had been
to finally give the `faces` API a spin and build the translator as a Puppet face.
But at this point, I settled for the most simple approach: Copy the `puppet apply`
code to a new subcommand (working title, `puppet mgmtgraph`), remove the transaction
code and insert conversion and YAML output instead.

The great part is that, in theory, I would inherit `puppet apply`'s powerful
goodies such as its `-e` switch for manifest one-liners. How cool would it be
to just crank out graphs with a quick call to `puppet mgmtgraph -e 'file { ... }'`
and so forth? Alas, it turns out that the code that makes the "portable" compiler
in `puppet apply` work is quite a handful indeed.

The `apply` subcommand has to keep quite a couple of balls in the air, and I did
not feel like doing this just yet in order to get a graph out of very simple
manifests. For example, `apply` has to secure an applicable source for fact values.
It will create a throw-away Puppet *environment* if needed. The code was all there
in my copy, but I still wanted to keep the proof of concept as DRY as possible,
relying only on the basic structure of the subcommand code.

Yet in order to allow `puppet mgmtgraph` to consume an arbitrary manifest
(including modules from the loading path), I would have needed to retain 90%
of the `apply` code. Not an appealing prospect, seeing how I want the new
code to stand on its own feet eventually.

When looking for a more simple code path in the `apply` subcommand, I made
another neat discovery. You can use `puppet apply --catalog <json-file>`
to run a transaction with a catalog that has been prepared using
`puppet master --compile <node>`. I was delighted. After all, I still
had that JSON from earlier in my `/tmp` directory. So, off I went to make
this mode the only one for `puppet mgmtgraph`, for the time at least.

It then dawned on me that it should be simple to compose the code for
`puppet master --compile` and `puppet apply --catalog`. This would allow
me to save the intermediate step of serializing to JSON. And sure enough,
the heart of the [puppet master](https://github.com/puppetlabs/puppet/blob/e945d75e6843de39304804bc57fe089c34e3af12/lib/puppet/application/master.rb#L171-L175)
code basically consists of just one call to the indirector:

```Ruby
catalog = Puppet::Resource::Catalog.indirection.find(options[:node])
```

I took this snippet and, rather unceremoniously, transferred it to my
`mgmtgraph` subcommand. And lo and behold: It works! The code was finally
capable of retrieving a RAL catalog from a manifest, without much
boilerplate code at all.

Back on track towards the quickest possible implementation, I found myself
faced with the challenge to recreate the translation logic I had already
built in the original Ruby script. Again, ops person DNA dictated that I
handle this situation with as much laziness as possible in order to get
the most reliable results. (Look, we didn't attain the current levels of
automation because we like programming languages so much.)

My next step was to apply lots of Pry again, and scour the catalog and
its relationship graph for representations that were similar to those
in the resource catalog I had translated already. Looking at a RAL catalog
is fun, because the resources within are so self-referential. Because
the resources have many relationships among each other, you will find
one and the same resource multiple times in one catalog data structure.
To make things even more confusing, some resources will appear in different
aliased forms. Remember how I intentionally misspelled the path to the
`file` resource in my demo manifest? Puppet munges the path into a
suitable form, and creates a properly named resource as an alias. Of course,
I would need to filter those away before translation.

Other things that need filtering are pseudo-resources for classes, stages,
and possibly other constructs that were not covered in my tests so far
(I'm looking at you, *defined types*). The good news is that the `resources`
method of a given catalog produces a simple list that is limited to canonical resources,
with no aliases, cross-references or other shenanigans. Skipping instances of the
`Puppet::Type::Component` and `Puppet::Type::Stage` classes is just a formality
then.

Reverting these resources back to a data form requires two steps. First,
get a catalog resource representation for each RAL resource (method
`to_resource`), then transform that into a flat hash (method `to_data_hash`).
This hash is almost identical to the one found in the JSON catalog.

Similarly, the graph object returned by the `relationship_graph` catalog
method produces an array of `edges` through the method of that name.
Each of these edge methods has a `to_data_hash` method of its own. Now
the edges are in a familiar format as well.

### Putting it all together

The code that does it all turned out quite ugly. In fact, I won't even
link to it here. More on that choice later. Let's just dissect it right now.
The heart of it is the following method:

```Ruby
def translate_catalog(catalog)
  result = {
    :graph => catalog.name,
    :comment => "generated from puppet catalog for #{catalog.name}",
  }
  result[:types] = {}
  edge_counter = 1

  catalog.resources.select { |res|
    # <snip>
  }

  catalog.relationship_graph.edges.map(&:to_data_hash).each do |edge|
    # <snip>
  end

  puts YAML.dump desymbolize(result)
end
```

The resulting graph must be a hash. It is initialized with two boilerplate
keys. There is also a counter for the edges, since I'm following James' example
of just numbering the edges. Finally, everything is converted to YAML and
written to the console. I added a `desymbolize` method that will recursively
convert all Ruby symbols into string values, since the former are more convenient
to work with.

The real work happens in the two truncated loops, one for the resources and
one for the edges. Here is the first one:

```Ruby
catalog.resources.select { |res|
  case res
  when Puppet::Type::Component
    false
  when Puppet::Type::Stage
    false
  when Puppet::Type
    true
  else
    false
  end
}.map(&:to_resource).map(&:to_data_hash).each do |resource_hash|
  next unless node = mgmt_type(resource_hash)
  result[:types][node[:type]] ||= []
  result[:types][node[:type]] << node[:content]
end
```

As described above, the code first filters for actual RAL resources
that are neither `component` (like a class) nor `stage`. It applies
the appropriate conversion methods `to_resource` and `to_data_hash`,
and finally invokes a custom method `mgmt_type`.

```Ruby
def mgmt_type(resource)
  result = {}
  resource["parameters"] ||= {} # resource w/o parameters
  case resource["type"]
  when 'File'
    # snip
  when 'Exec'
    result[:type] = :exec
    result[:content] = {
      :name => resource["title"],
      :cmd  => resource["parameters"][:command] || resource["title"],
      :shell => resource["parameters"][:shell] || "",
      :timeout => resource["parameters"][:timeout] || 0,
      :watchcmd => "",
      :watchshell => "",
      :ifcmd => resource["parameters"][:onlyif] || "",
      :ifshell => "",
      :pollint => 0,
      :state => :present
    }
    result
  end
end
```

The method is about as straight-forward as it gets. Depending on the
specifics of the resource in question, it sets some values in the
hash it ultimately returns. It currently supports only the `file`
and `exec` types. Some values for `mgmt` just remain blank or
at their default values, if Puppet has no equivalent.

As for the edges, their handling is just a bit more involved.
The only edges that are supposed to make it into the `mgmt`
graph are those that connect supported resources. Here's how
this is currently implemented:

```Ruby
catalog.relationship_graph.edges.map(&:to_data_hash).each do |edge|
  from = parse_ref(edge["source"])
  to = parse_ref(edge["target"])

  next unless from and to
  next_edge = "e#{edge_counter += 1}"

  result[:edges] ||= []
  result[:edges] << { :name => next_edge, :from => from, :to => to }
end
```

Again, the actual work is performed by a custom method.

```Ruby
def parse_ref(ref)
  if ! ref.match /^(.*)\[(.*)\]$/
    raise "unexpected reference format '#{ref}'"
  end
  type = $1.downcase
  title = $2
  return nil unless [ 'file', 'exec', 'service' ].include? type
  return { :type => type, :name => title }
end
```

Behold the regex. (As if I need to prove that I have roots in Perl scripting.)
More interesting, the parser just returns `nil` if the reference doesn't
target a resource of type `file`, `exec` or `service` (even though the
latter is not yet supported for translation). This selection criteria
makes sure to avoid all containment edges and other overhead.

Without further ado, this is the `mgmt` compatible graph produced by the
proof-of-concept code:

```YAML
$ bundle exec puppet mgmtgraph --manifest /tmp/demo.pp 
---
graph: fflaptop.local
comment: generated from puppet catalog for fflaptop.local
types:
  file:
  - name: demo-file
    path: "/tmp/foo"
    state: absent
  exec:
  - name: demo-process
    cmd: "/bin/true"
    shell: ''
    timeout: 300.0
    watchcmd: ''
    watchshell: ''
    ifcmd: ''
    ifshell: ''
    pollint: 0
    state: present
edges:
- name: e2
  from:
    type: file
    name: demo-file
  to:
    type: exec
    name: demo-process
```

And yes, it works with `mgmt`.

```
$ ./mgmt run --file /tmp/demo.yaml 
00:07:57 main.go:65: This is: mgmt, version: 0.0.1-78-gc47418b
00:07:57 main.go:66: Main: Start: 1456096077128477809
00:07:57 main.go:196: Main: Running...
00:07:57 main.go:106: Etcd: Starting...
00:07:57 etcd.go:132: Etcd: Watching...
00:07:57 configwatch.go:54: Watching: /tmp/demo.yaml
00:07:57 main.go:149: Graph: Vertices(2), Edges(1)
00:07:57 main.go:152: Graphviz: No filename given!
00:07:57 main.go:163: State: graphNil -> graphStarting
00:07:57 main.go:165: State: graphStarting -> graphStarted
00:07:57 exec.go:245: Exec[demo-process]: Apply
00:07:57 exec.go:295: Exec[demo-process]: Command output is empty!
```

### Next steps

So if you really want to run this right now, send me a tweet. But honestly,
it's not really worth the effort yet. The implementation has numerous
weaknesses and is generally ugly and unmaintainable.

Here's where I want to go from here:

* Rebuild this into a face after all. I'm especially keen on finding out whether
the existing `catalog` face can be exploited for the manifest compilation.
Otherwise, the face will hopefully be able to reuse the approach of the current
PoC code.
* Move it to a Puppet Forge module. We will eventually need that. If `mgmt`
takes off and we want to allow a broader user base to feed manifest code to it,
they should just be able to grab the module and use the face from the shell.
Keeping the code in a git branch of Puppet core would be very impractical.
* Create a DSL for translation rules. This is a big one. The PoC was written
in a quick'n'dirty fashion, implementing its logic in a procedural fashion.
The more features get added to `mgmt`, the more complex the translation ruleset
will have to become. It's hardly feasible to keep maintaining this in its
current form. (Please note that I cut the more messy parts from the
`mgmt_type` method above. The `exec` type is the easier one to translate.)
Instead, there should be a simple DSL similar to the one that
Puppet uses internally to describe its own resource types.

I will try and revisit all these aspects in follow-up posts once I get around
to actually implement them. Y'all can expect me to be busy with this stuff
for quite a while.
