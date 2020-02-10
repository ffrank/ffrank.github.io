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

[basic puppet resources are in mgmt]
[much puppet code can just be run]
[some arcane resource types only in Puppet]
[some features of mirror types only in Puppet]

[nodes get substituted]
[Puppet does the sync work]
[original substitute was Puppet exec]
[yamlresource receive offers better performance]
[requires ability in mgmt to launch and track receiver]
[graph resources must target the receiver]
[translator must emit appropriate nodes]

## Running a persistent Puppet

[yamlresource receive needs to run in parallel]
[diagram from talk]
[not really a common pattern]
[not tricky to build: fire and forget, but keep pipes]

[created using Singleton pattern]
[each resource accesses global struct]
[passes interface to a function that handles sync]

[aside: was originally method of the runner struct]
[makes testing hard]
[see below about testing]

## Graph nodes for Puppet

[sending strings to puppet cannot go into an exec]
[also not desirable]
[dedicated resource instead]

[structure is simple, meant for easy marshalling]
[actually cleaner than exec workaround]
[parameters are not structured, but held as YAML instead]
[note: passed to Puppet as YAML wrapped in JSON]
[JSON is there to make sure titles can be arbitrary]

## New graphs from Puppet

[replace exec workaround]
[actually added new default, old one still available]
[further savings: skip noop query, should not be required]

## A word about tests

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
