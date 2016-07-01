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
intense digging.

### Review: Arriving at the original implementation

[see first post, pried at the catalog, invoked relationship-graph method]

[lots of edges even in simple manifests]

[everything not between known resources was dropped]

[especially confusing: not acyclic, classes refer to themselves]

### Looking closer

[thesis: cycles are acceptable in the full rendering, forbidden only in abstract declarations]

[still doubtful, after all, classes are not resources]

[pry cd to the rescue]

[see that the 'class' is actually a whit]

[whit at beginning and end of class/define have distinct names]

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
