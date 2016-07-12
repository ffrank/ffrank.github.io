---
layout: post
category: features
tags: [ puppet, mgmt, class, define, stage, dependency, edge, graph, catalog ]
summary: Of relationships between Puppet's classes and defines, and their translation to mgmt.
---

The `mgmt` translator for `puppet` catalogs was truly created from the bottom
up. We started with a few resource types, and the relationships between
the translated resources. This falls short for many catalogs, of course,
because dependencies must often put whole classes in order, or do the same
for instances of defined types.

Allowing the [translator module]({% post_url 2016-06-19-puppet-powered-mgmt %})
to accept such macro-dependencies was not much work, but it did require some
intense digging. Before I describe this process, let's take another look at
what happened so far.

### Review: Arriving at the original implementation

When I first wrote about [mgmt integration with Puppet]({% post_url 2016-02-18-from-catalog-to-mgmt %}),
I already showed my approach to analyzing the catalog.
It's really what anybody should do when looking at the details of a Ruby
code base: Fire up `pry` and step right *into* the objects in question.
Doing this allows you to take a close look at (in this case) the catalog data, but it
cannot always answer all questions.

The first surprise was the absence of relationship edges in the graph
data structure. These edges emerge from the `prioritizer`, upon invoking
the catalog's `relationship_graph` method.

To look at the kind of catalog object we need, add a call to `pry` in the
`PuppetX::CatalogTranslation::to_mgmt` method:

```
@@ -6,6 +6,7 @@ module PuppetX; end
 
 module PuppetX::CatalogTranslation
   def self.to_mgmt(catalog)
+    require 'pry' ; binding.pry
     result = {
       :graph => catalog.name,
       :comment => "generated from puppet catalog for #{catalog.name}",

```

Jump in there with a nice and simple catalog:

    $ bundle exec puppet mgmtgraph print --code 'notify { "This is the only resource": }'
    ...
    [1] pry(PuppetX::CatalogTranslation)>

You will immediately want to `cd` into the `catalog` object:

```
[1] pry(PuppetX::CatalogTranslation)> cd catalog
```

From here you can look around using `ls`, or immediately call the
`relationship_graph` method:

```
[2] pry(#<Puppet::Resource::Catalog>):1> relationship_graph
```

This sends you into a viewer with a large wall of text, giving you
a full description of the `relationship_graph` object. This is a lot of data
because it includes the `notify` resource object, which in turn holds
a reference to the whole catalog. All is displayed.

Again, it's more useful to `cd` into that to get a better feel for
the object.

```
[3] pry(#<Puppet::Resource::Catalog>):1> g = relationship_graph
[4] pry(#<Puppet::Resource::Catalog>):1> cd g
[5] pry(#<Puppet::Graph::RelationshipGraph>):2> 
```

Here, an `ls` will reveal that there are helpful methods such as `edges` and
`vertices`:

```
[6] pry(#<Puppet::Graph::RelationshipGraph>):2> edges
=> [{ Class[Main] => Notify[This is the only resource] },
    { Class[Settings] => Stage[main] },
    { Class[Main] => Stage[main] },
    { Stage[main] => Class[Settings] },
    { Class[Settings] => Class[Settings] },
    { Stage[main] => Class[Main] },
    { Notify[This is the only resource] => Class[Main] }]
```

Quite a lot of edges. The original translator implementation ignored all class
dependencies, so all of the above were considered to be overhead.
The `notify` resource has no explicit relationships, and these implicit ones
(with the containing class) were ignored as well.

![Visual graph](https://googledrive.com/host/0B15cMf9TSVMzY25Yb01EYmlmak0/class-relationships.svg)

Now that class dependencies are actually wanted, it becomes necessary to
arrive at a better understanding of this structure. At first glance,
it's quite confusing. Some edges appear redundant, and there are
circles, even tight one such as this:

```
{ Class[Main] => Stage[main] },
{ Stage[main] => Class[Main] },
```

or even this:

```
{ Class[Settings] => Class[Settings] },
```

Fortunately, `pry` made it easy to pierce this particular mystery as well.

### Looking closer

Originally, upon discovering these apparent contradictions, I assumed that
this "compiled" graph was actually allowed to contain cycles. The idea was
that cycles were only forbidden wrt. the explicit edges from `before` and
`require` parameters, `autorequire` rules and so forth.
However, this theory had little explanatory power. How are these cyclic edges
used by Puppet's algorithms? What is the nature of these graph vertices?
After all, a Puppet class is not a resource.

Another Ruby mystery, another opportunity to exploit the analytical power of `pry`.
Use the same invocation as earlier, and `cd` down into the `RelationshipGraph` object.
To get more information about a specific edge, you can `cd` right into that now:

```
[4] pry(#<Puppet::Graph::RelationshipGraph>):2> edges[1]
=> { Class[Settings] => Stage[main] }
[5] pry(#<Puppet::Graph::RelationshipGraph>):2> cd edges[1]
[6] pry(#<Puppet::Relationship>):3> ls
Puppet::Network::FormatSupport#methods: mime  render  support_format?  to_json  to_msgpack  to_pson
Puppet::Relationship#methods: 
  callback  callback=  event  event=  inspect  label  match?  ref  source  source=  target  target=  to_data_hash  to_s
...
```

So the edge's `source` and `target` are available through attribute accessors.
You can `cd` right into the source vertex, for example:

```
[7] pry(#<Puppet::Relationship>):3> cd source
[8] pry(#<Puppet::Type::Whit>):4>
```

Note the `pry` prompt. You just changed into the scope of a `Puppet::Type::Whit` object.
In other words, the source of this edge (presented as `Class[Settings]`) is really
a resource of Puppet's `whit` type.

Now, if you've never heard of this type, don't worry:
This is by design. It is only used internally, and not for use in manifest code.
See [its source code](https://github.com/puppetlabs/puppet/blob/8615d23104a34923c05e6ddf777d98498d1fc924/lib/puppet/type/whit.rb)
for more information. The documentation is enlightening and, as it happens, quite funny.

So the `class` at the source of this edge is really a `whit`. Have a closer look:

```
[8] pry(#<Puppet::Type::Whit>):4> self.ref
=> "Whit[Completed_class[Settings]]"
```

The resource reference gives you a distinct description of the resource. In fact,
let's get a proper rendering of the whole graph:

```
[9] pry(#<Puppet::Type::Whit>):4> cd ..
[10] pry(#<Puppet::Relationship>):3> cd ..
[11] pry(#<Puppet::Graph::RelationshipGraph>):2> edges.map { |e| [ e.source.ref, e.target.ref ] }
=> [["Whit[Admissible_class[Main]]", "Notify[This is the only resource]"],
 ["Whit[Completed_class[Settings]]", "Whit[Completed_stage[main]]"],
 ["Whit[Completed_class[Main]]", "Whit[Completed_stage[main]]"],
 ["Whit[Admissible_stage[main]]", "Whit[Admissible_class[Settings]]"],
 ["Whit[Admissible_class[Settings]]", "Whit[Completed_class[Settings]]"],
 ["Whit[Admissible_stage[main]]", "Whit[Admissible_class[Main]]"],
 ["Notify[This is the only resource]", "Whit[Completed_class[Main]]"]]
```

![Visual graph](https://googledrive.com/host/0B15cMf9TSVMzY25Yb01EYmlmak0/whit-relationships.svg)

The naming scheme for the `whit` resources is simple: There is an `admissible`
marker and its counterpart, called `completed`. This ordered pair encloses each
container (stages, classes, and defined types, the latter not being depicted above).
Read on to learn how this insight allows us
to translate the complete set of relationships for `mgmt`.

### Holding on to the edge

To model class containment and dependencies in `mgmt`, we already had an idea
floating around. We actually came up with it
[back in February](https://github.com/purpleidea/mgmt/issues/8#issuecomment-184866939),
right when we started thinking about the translator concept.
The rough plan was to introduce proxy nodes that do nothing (type `noop`) and just help
distribute the relationships.

What I had now learned was that Puppet does just that already, using `whit` pseudo-resources.
Incorporating them in the output graph is simple: Each `whit` can be translated right
into a `noop`. As soon as this happens, relationships between `whit`s and other resources
are kept as well.

Since these `whit` pseudo-resources don't show up in the catalog's resource table, the
resource translation code needed to change. This was the original loop:

```ruby
catalog.resources.each do |res|
  ...
end
```

It now just uses the full graph:

```ruby
catalog.relationship_graph.vertices.each do |res|
  ...
end
```

Accepting the `whit` nodes was implemented in
[translator DSL](https://github.com/ffrank/puppet-mgmtgraph/blob/master/lib/puppetx/catalog_translation/type/whit.rb).

Furthermore, the handling of edges needed to change. It used to rely on the symbolic edge representation,
as generated through the `Edge#to_data_hash` method. This method returns source and target
in the form of resource references such as `Class[Settings]` or `Stage[main]`.

To get at the actual `whit` node edges, the code now foregoes the `to_data_hash` method, and instead
retrieves type and title from the actual respective resource object. Apart from this detail, everything
still works the same as before.

These relatively minor changes were all that was necessary to enable translation of all resource
dependencies that make up the complete Puppet graph. An implication of this change is that the resulting
graph receives a certain amount of boilerplate relationships. Here is a very simple graph from
a short manifest, the way it looked before `whit` support was added:

```yaml
$ bundle exec puppet mgmtgraph print --code 'file { "/etc/nologin": }'
---
graph: fflaptop.local
comment: generated from puppet catalog for fflaptop.local
resources:
  file:
  - name: "/etc/nologin"
    path: "/etc/nologin"
    content: 
edges: []
```

Now that the implicit `Class[main]`, the `main` run stage and other internals are recognized,
the result looks as follows:

```yaml
---
graph: fflaptop.local
comment: generated from puppet catalog for fflaptop.local
resources:
  file:
  - name: "/etc/nologin"
    path: "/etc/nologin"
    content: 
  noop:
  - name: admissible_Stage[main]
  - name: completed_Stage[main]
  - name: admissible_Class[Settings]
  - name: completed_Class[Settings]
  - name: admissible_Class[Main]
  - name: completed_Class[Main]
edges:
- name: Whit[Admissible_class[Main]] -> File[/etc/nologin]
  from:
    kind: noop
    name: admissible_Class[Main]
  to:
    kind: file
    name: "/etc/nologin"
- name: Whit[Completed_class[Settings]] -> Whit[Completed_stage[main]]
  from:
    kind: noop
    name: completed_Class[Settings]
  to:
    kind: noop
    name: completed_Stage[main]
- name: Whit[Completed_class[Main]] -> Whit[Completed_stage[main]]
  from:
    kind: noop
    name: completed_Class[Main]
  to:
    kind: noop
    name: completed_Stage[main]
- name: Whit[Admissible_stage[main]] -> Whit[Admissible_class[Settings]]
  from:
    kind: noop
    name: admissible_Stage[main]
  to:
    kind: noop
    name: admissible_Class[Settings]
- name: Whit[Admissible_class[Settings]] -> Whit[Completed_class[Settings]]
  from:
    kind: noop
    name: admissible_Class[Settings]
  to:
    kind: noop
    name: completed_Class[Settings]
- name: Whit[Admissible_stage[main]] -> Whit[Admissible_class[Main]]
  from:
    kind: noop
    name: admissible_Stage[main]
  to:
    kind: noop
    name: admissible_Class[Main]
- name: File[/etc/nologin] -> Whit[Completed_class[Main]]
  from:
    kind: file
    name: "/etc/nologin"
  to:
    kind: noop
    name: completed_Class[Main]
```

> Note that this example uses a `file` resource rather than a `notify`. The reason is simple:
There is no translation for `notify` yet, so the simple graph would have been fairly empty,
and the new one would be all `noop` vertices.

I actually entertained the thought of filtering some of these default classes and the stage, but this would require
an inacceptable amount of added code complexity. After all, this generated YAML is for
running through `mgmt`, and not for casual editing after the fact. As described
[earlier]({% post_url 2016-06-19-puppet-powered-mgmt %}), the Puppet translator is now
a first class citizen of `mgmt`, so that the YAML does not even need to be saved.

Of course, the translation can never be perfect, so you may have need to alter the
YAML data after all. If this occurs, you should probably write a scriptlet in the language of
your choice (that has YAML support). This way, the confusingly similar edge and node
names of all the `noop` vertices don't get in the way.

### Almost there

This latest bit of progress is actually quite amazing: We are still limited by the
resource types that `mgmt` supports already, but otherwise, you are now free to
write manifests and modules for use with `mgmt`, and they should Just Work!

The fact that unsupported resources (and their edges) still get dropped can be
important. Consider the following contrived manifest:

```puppet
files { "/tmp/a": }
->
notify { "Half time!": }
->
files { "/tmp/b": }
```

The `notify` resources cannot be translated, so it's missing in the output graph.
So are both relationship edges that connect to it. Hence, the translated graph will
regard the `file` resources as unrelated (`mgmt` will actually process them in parallel).
This is very bad, but not an issue that will be addressed directly. Instead, we will
focus on retaining all Puppet resources in one form or another.

As soon as this works, it should be possible to run almost arbitrary manifests through `mgmt`,
along with Forge modules and everything. There will always be limitations, but here's hoping
that this facility will enable a lot of people to start playing with the `mgmt` project.

Another interesting commonality among `puppet` and `mgmt` is the concept of exported resources.
It remains to be seen whether a proper translation of those is possible at all.
It would be nice to have, so that this `mgmt` feature also comes available through `puppet` manifests.
However, it might turn out that this cannot really work. This would be unfortunate, but
no tragedy.

## Limiting the scope

After all, support for `puppet` manifests in `mgmt` is only a stop-gap feature until a
suitable native language can be devised and implemented. I can name a few reasons why
Puppet cannot permanently serve as the code interpreter:

1. There is never going to be full feature parity. Some of Puppet's types are very rich. A `file` resource
can take up to 32 attributes, not counting metaparameters. On the other side of this coin,
`mgmt` introduces some properties that only make sense under its event-based paradigm, such
as the `exec` type's `watchcmd`. This could be worked around by creating artificial Puppet types
that are very similar to the ones in `mgmt`, but that would be missing the point (which is to
take advantage of all the smart engineering that has gone into Puppet's existing type system throughout
the years).
2. The `puppet` compiler can only ever generate a graph for the specific agent that invoked it.
The catalog it emits, and the finished graph that is translated from it, will usually be tailored
to the fact values that this agent supplies. Basing the configuration of a whole cluster on such
a graph does not appear feasible.
3. Puppet's language may just be the most sophisticated one on the configuration management scene.
It was designed with a great deal of deliberation for a sole purpose: To model systems in a suitable
way for management through `puppet agents`. It makes some assumptions that just don't hold true
for `mgmt`, such as the transactional evaluation of the resource graph.
4. On the flip side, `mgmt` has some unique requirements that stem from its event-based nature.
A suitable language should probably be reactive, or have some other properties that help the
user take maximum advantage of the tool.

Still, while there is no DSL for `mgmt` yet, I hope that the `puppet` support will make `mgmt`
more accessible for many potential users, and thereby help its development.

Would you like to be a part of this? Please feel free to get involved to any extent:

* install and try the [puppet-mgmtgraph](https://forge.puppet.com/ffrank/mgmtgraph/readme) module
* read up on `mgmt` at [James' blog](https://ttboj.wordpress.com/tag/mgmt/)
* download and try [mgmt](https://github.com/purpleidea/mgmt/)
* let James know what you think, you can find him on IRC in `#mgmtconfig` on FreeNode
* holler us on Twitter
* send feature request or even patches for either tool, if you're so inclined

Thanks for reading!
