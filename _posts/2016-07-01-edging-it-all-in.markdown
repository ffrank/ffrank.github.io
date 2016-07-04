---
layout: post
category: features
tags: [ puppet, mgmt, class, define, stage, dependency, edge, graph, catalog ]
summary: Of relationships between Puppet's classes and defines, and their translation to mgmt.
---

The `mgmt` translator for `puppet` catalogs was truly created from the bottom
up. We started with some resource types, and picking up the relationship between
any resources that got translated. This falls short for many catalogs, of course,
because dependencies must often put whole classes in order.

Allowing the [translator module]({% post_url 2016-06-19-puppet-powered-mgmt %})
to accept such macro-dependencies was not much work, but it did require some
intense digging. Before I describe this process, let's take another look at
what happened so far.

### Review: Arriving at the original implementation

When I first wrote about [mgmt integration with Puppet]({% post_url 2016-02-18-from-catalog-to-mgmt %}),
I already showed my approach to analyzing the catalog.
It's really what anybody should do when looking at the details of a Ruby
code base: Fire up `pry` and walk right into the objects in question.
The first surprise was the absence of relationship edges in the graph
data structure. These edges emerge from the `prioritizer`, upon invoking
the `relationship_graph` method.

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
(with the containing class) were to be ignored as well.

Now that class dependencies were actually wanted, it was necessary to
arrive at a better understanding of this structure. At first glance,
it's quite confusing. Some edges appear redundant, and there are
circles, even tight one such as this:

```
{ Class[Main] => Stage[main] },
{ Stage[main] => Class[Main] },
```

Fortunately, `pry` made it easy to pierce this particular mystery as well.

### Looking closer

Originally, upon discovering these apparent contradictions, I assumed that
this "compiled" graph was actually allowed to contain cycles. The idea was
that cycles were only forbidden for the explicit edges from `before` and
`require` parameters, `autorequire` rules and so forth.
However, this thesis had little explanatory power. How are these cyclic edges
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
Let's `cd` right in there as well:

```
[7] pry(#<Puppet::Relationship>):3> cd source
[8] pry(#<Puppet::Type::Whit>):4>
```

Note the `pry` prompt. You just changed into the scope of a `Puppet::Type::Whit` object.
In other words, the source of this edge (presented as `Class[Settings]`) is really
a resource of Puppet's `whit` type. Now if you've never heard of this type, don't worry:
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

[some final analysis]

### Holding on to the edge

[idea to rely on mgmt's noop resource had been there from the start]

[tricky part was supposedly to generate appropriate place-holders]

[the whits make it simple though: just translate them to noop]

[whits don't appear in catalog.resources -> use relationship graph vertices instead]

[these changes suffice to cover all needed relationships]

[observe the greater graph complexity]

[not a problem at all - graphs are not supposed to be edited]

### Almost there

[amazing: can now write complex manifests with modules etc.]

[unsupported resources still get dropped, along with transitive relationships (example!)]

[as soon as these are incorporated, arbitrary manifests should translate completely]

[exports would be nice, but not needed]

## Limiting the scope

[after all, puppet language support is only a stop-gap]

[there is no feature parity, and it's probably not worth teaching missing parameters to puppet]

[more importantly, puppet's language was designed specifically for puppet]

[mgmt probably needs a language that captures its reactive nature]

[still: hope is that the puppet support will enable many people to start testing]
