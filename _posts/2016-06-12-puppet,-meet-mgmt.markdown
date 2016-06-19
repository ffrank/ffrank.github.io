---
layout: post
category: features
tags: [ puppet, mgmt, dsl, ruby, go ]
summary: An update on the bridge between Puppet and mgmt.
---

### Welcome back

I just realized that it has been almost 4 months since my [last post]({% post_url 2016-02-18-from-catalog-to-mgmt %}).
If you missed it, you should probably read it before this one.
I kicked off quite a bit of work then and much has come to fruition in the meantime.
This is my report.

**Update**: This post is quite extensive and covers the guts of the implementation.
For a succinct overview of the new functionality, see the [follow-up]({% post_url 2016-06-19-puppet-powered-mgmt %}).

### Clearing the TODO list

With this post, I finally and proudly present the `puppet mgmtgraph` subcommand, available
through the `ffrank-mgmtgraph` module. Back in February, I had outlined three immediate goals:

 1. Re-implement the catalog translator as a Puppet face. I managed to do that by the end of February (shout-out to 
 Erik Dalén, I took lots of cues from his [puppetls](https://github.com/dalen/puppetls) face).
 2. Bundle that up in a module. This was straight-forward enough. Notably, the only code that needed to remain
 in the `Puppet::` Ruby namespace is the actual face API. The bulk of the implementation was segregated into
 a `PuppetX::` module.
 3. Create a DSL for describing translation rules. This was ready by mid March. I'm fairly proud of it and
 it's past time to present it here. Do read on.

In addition to that, I got together with [James](https://ttboj.wordpress.com/) and we figured out how to allow
`mgmt` to connect to the translator directly. It was a great time, and just recently I managed
to cobble up a patch and got my first ever `go` contribution into an open source project!

Let's take a closer look at all that has happened.

### At face value

So Puppet's `faces` API is pretty neat. Researching its basics is inadvertently funny: you will most likely
end up at the [introductory blog post](https://puppet.com/blog/puppet-faces-what-heck-are-faces) by Daniel
Pittman that may leave you wondering, and inspired some mocking comments - it teaches all about semver and
little about what Puppet faces are good for.

I ended up picking up the basics from a cool example - as mentioned above, a quick Google search will lead you to
an [open source module](https://github.com/dalen/puppetls) by esteemed friend and community member Erik.
A great piece of code to read, it taught me a number of useful things.

 1. The `puppet ls` command is just a module install away and it's really useful when handling lots of
 systems with heterogenous Puppet code, especially if you have not written it yourself.
 2. A face primarily exposes a Ruby API, but Puppet makes it easy to create a subcommand
 and allow the user to use the face right from the command line.
 3. It's quite simple to access other faces from within a face and
 4. Puppet comes with some handy faces already, such as the `catalog` face.

So as long as you don't need any specifics beyond invoking the face methods, the code for the
subcommand is [really simple](https://github.com/ffrank/puppet-mgmtgraph/blob/e01ca96fa24271a0e92257172bfbfdd2e4d9a66f/lib/puppet/application/mgmtgraph.rb).
In essence, you just need to inherit the `Application::FaceBase` class in a new class with
an appropriate name. In my case: `Puppet::Application::Mgmtgraph`.

The [face code](https://github.com/ffrank/puppet-mgmtgraph/blob/e01ca96fa24271a0e92257172bfbfdd2e4d9a66f/lib/puppet/face/mgmtgraph.rb)
is not complex either. In the case of `mgmtgraph`, there is more boilerplate than anything else,
because the legwork happens in the `PuppetX::CatalogTranslation` module.
The face offers two methods: `print` is intended for use from the command line and dumps YAML data
to the console. Internally, it relies on the `find` method, which generates the raw, unserialized
graph data.

Currently, I don't think that more methods will be needed, but depending on further development in
both Puppet, `mgmt`, and even the translator itself, something may come up eventually.

### Drink for Ruby DSLs

At PuppetConf 2015, I proposed a game: examine Ruby code bases, and drink whenever you encounter a Domain
Specific Language.
As [Adrien](http://somethingsinistral.net/) once said (I think it was him), whereas Perl solves all problems
using hashes, Ruby does it using DSLs. Puppet's codebase itself implements at least three such languages
I could name, and it uses some existing ones, such as `rspec`'s and `mocha`'s.

I hadn't shown you my original proof-of-concept code that did my first timid translations. It was basically
the declaration of hashes, based on the information in the resource hashes from the Puppet compiler. The code
would just impose the new structure, and change keys and values as necessary. For example:

```puppet
file { '/tmp/my_flag': ensure => present }
```

This resource needs some transformation for `mgmt`, where it will look kind of like this:

```yaml
file:
  - name: /tmp/my_flag
    path: /tmp/my_flag
    state: exists
```

Instead of `ensure`, it needs the keyword `state`, and either `present` or `file` must be rendered into `exists`.
The translator DSL should make it possible to express this just like that: Rename the `ensure` attribute and transform
its value as follows. Specifically:

```ruby
rename :ensure, :state
```

The value transformation can be done in the code block:

```ruby
rename :ensure, :state do |value|
  case value
  when :present, :file
    :exists
  else
    raise "cannot translate file/ensure value '#{value}'"
  end
end
```

First point of order was for me to learn how to create Ruby DSLs at all.
Since the translator lives in (or near) Puppet's codebase, I was inclined to leverage the helper code from there.
For example, Puppet's own types are defined through yet another DSL. It's a fine example for such constructs,
but on the other hand, [as I have explained before](http://ffrank.github.io/presentations/2015-10-puppetconf-hacking-types-and-providers/hacking-types-providers.html#/6/2),
the whole type system uses some arcane Ruby constructs. To make it work, the DSL code needs to generate
classes, layering meta-programming quite a lot.

To make a long story short, this particular example is way too complex for the needs of the `mgmt` translator.
I conducted a quick research for basic instructions on the topic.
It yielded a very helpful article written by [Leigh Halliday](https://www.leighhalliday.com/creating-ruby-dsl).
It inspired me to take a closer look at Ruby's `instance_eval` method, which lives at the heart
of his implementation.

This reminds me of an odd effect: when I got curious about `instance_eval`, I looked for information
on how it compares to `class_eval`, which I had seen in some code example before. There was just one
small problem: the majority of articles I found
(like [this one](https://www.jimmycuadra.com/posts/metaprogramming-ruby-class-eval-and-instance-eval/))
limit themselves to pointing out how *weird* it is that methods defined in an `instance_eval` block
end up as class methods, whereas those from within `class_eval` blocks become instance methods.
Luckily, there is [this explanation](http://web.stanford.edu/~ouster/cgi-bin/cs142-winter15/classEval.php)
from a class at Stanford. I derived this mnemonic:

> Both `class_eval` and `instance_eval` can be used for more than just defining methods. Come on!

Alright, rant finished. I feel better now. How are you?

Armed with these lessons, creating the DSL became almost simple. The next section will outline
the code. As a final remark, an autoloading facility became desirable at some point.
I wanted each DSL stanza to inhabit its own respective source code file (as seen in
[the git repo](https://github.com/ffrank/puppet-mgmtgraph/tree/master/lib/puppetx/catalog_translation/type)),
so Ruby needs to be told to load them.
Initially, the code just got explicit `require` calls for each type that had been added so far.
In the long run, this should not be necessary, though.

Once more, I turned to Puppet's code. After all, the automatic loading of types and providers
from any installed Puppet modules is pretty much what I wanted for my translator units.
Some poking reveals the [Puppet::Util::Autoload](https://github.com/puppetlabs/puppet/blob/master/lib/puppet/util/autoload.rb)
class. It's a private API, so this is me potentially limiting compatibility with future versions.
On the other hand, do you figure that Puppet will change the autoloading API anytime soon?
Before they migrate away from Ruby anyway? We'll see.

To be honest, I did not even make these considerations at the time of writing this code.
Nor at any other time before writing this article, so...yeah. Here we are. This is what happens
when life (or, the impressive number of 19 code contributors) gives you pretty rubies.
You just go ahead and create a simple and powerful loading method:

```ruby
class PuppetX::CatalogTranslation::Type
  def self.loader
    @loader ||= Puppet::Util::Autoload.new(self, "puppetx/catalog_translation/type")
  end

  def self.load_translator(type)
    loader.load(type)
  end
end
```

### Actual engineering?

This last bit was actually one of the implementation details of the `Type` class, the
engine for the tranlation DSL. Each instance of this class represents a translator object
for a specific Puppet resource type. Any translator should be loaded at most once, so
the type class keeps a cache of all instances:

```ruby
class PuppetX::CatalogTranslation::Type
  @instances = {}

  def self.translation_for(type)
    unless @instances.has_key? type
      load_translator(type)
    end
    @instances[type] || @instances[:default]
  end
end
```

The cache isn't populated by the loading methods. Instead, whenever an instance is
created by whatever means, it *registers* itself with the class using this trivial method:

```ruby
class PuppetX::CatalogTranslation::Type
  def self.register(instance)
    @instances[instance.name] = instance
  end
end
```

I wanted instance declarations to take the following form, similar to Puppet resource type
definitions:

```ruby
PuppetX::CatalogTranslation::Type.new :package do
  emit :pkg

  rename :ensure, :state do |value|
    case value
    when :installed, :present
      :installed
    # ...
  end
end
```

This means that the constructor for the `Type` class needs to accept one argument (the
symbolic type name) and a block. The statements in the block, such as `emit` and `rename`
must have some meaning to the instance being constructed. Therefor, those have to
be implemented as *instance methods* of the `Type` class.

```ruby
class PuppetX::CatalogTranslation::Type
  def initialize(name,&block)
    @name = name
    @translations = {}
    instance_eval(&block)
    self.class.register self
  end
end
```

The `@translations` hash is an instance attribute that caches all translation rules
in a comprehensive format. Before we look at its structure, let's discuss the different
elements that currently describe the translation.

1. Direct correspondence: Some resource attributes (such as `file/path`) exist in both
`puppet` and `mgmt`. The parameter value might need changing, but otherwise everything
is just kept.
2. Similar parameters with different names: A typical example is Puppet's `ensure`
property. The concept is there in `mgmt`, but the resource attribute is typically called
`state`. The values can need transformation as well.
3. Addition: Some attributes in `mgmt` simply have no equivalent in Puppet's model,
such as `exec/watchcmd`. Since Puppet does not process events outside of its agent
transaction, it has no need for lingering processes that will trigger `exec` resources again.
4. Outside of the resource attributes and values, there is the resource descriptor
itself. Some resource types are just named differently in `mgmt`, such as `pkg`
as opposed to `package`.

I dubbed the latter element a translator instance's `output`, and it defaults to the
name of the translator. That is to say, the resource names are kept, such as `file`
and `service`. From the DSL, it is set using the `emit` method.

```ruby
PuppetX::CatalogTranslation::Type.new :package do
  emit :pkg
end
```

When this block is run through `instance_eval`, it just sets an instance member variable.

```ruby
class PuppetX::CatalogTranslation::Type
  def emit(output)
    raise "emit has been called twice for #{name}" if @output
    @output = output
  end
end
```

As for the other translation elements, these add hashes to the `@translations`
member variable. The most simple first case, "direct correspondence",
implemented by the `carry` DSL method, adds the following data.

    { :title => <arg1>, :block => <block> }

The block is the one that is given to the `carry` invocation, if there is one.
Let's look at the following examples from the `exec` type translator:

```ruby
PuppetX::CatalogTranslation::Type.new :exec do
  carry :timeout
end
```

It has no block, so the translation hash will be simple. Here's the code
that fills it.

```ruby
class PuppetX::CatalogTranslation::Type
  def carry(*attributes,&block)
    attributes.each do |attribute|
      @translations[attribute] = { :title => attribute, }
      if block_given?
	@translations[attribute][:block] = block
      end
    end
  end
end
```

It takes an arbitrary number of attribute names, and does the same for
each: create a hash that just contains the `:title => <attribute>` pair,
and add the block if it exists (with the `:block` hash key).

It turned out that for the current set of supported `mgmt` resources,
there was no opportunity to pass a block to `carry` (or, for that matter,
directly carry any other attribute). So let's just assume for a minute
that `mgmt` required a different format for the timeout and adds a unit suffix:

```ruby
PuppetX::CatalogTranslation::Type.new :exec do
  # NOT ACTUAL TRANSLATOR CODE!
  carry :timeout do |value|
    "#{value}s"
  end
end
```

This block is just stored in the `@translations` hash for the time being.
It won't be evaluated until the translator is put to work through the
`translate!` method. Let's build it piece by piece:

```ruby
class PuppetX::CatalogTranslation::Type
  def translate!(resource)
    result = {}
    @translations.each do |attr,translation|
      next if !resource.parameters[attr]
      result[attr] = if translation.has_key? :block
        translation[:block].call(resource[attr])
      else
        resource[attr]
      end
    end
    result
  end
end
```

So the translation works by iterating through the elements that have been
defined by the DSL code. In the `:timeout` example, an `exec` will keep
the value as seen in the Puppet resource. If the block is given (as in the
example above), this block will now be invoked, and given the value from
the resource as the parameter. A timeout of 300 will translate to "300s".

> Attentive readers might be wondering why the code uses both `resource[attr]`
and `resource.parameters[attr]`. The distinction is important. Usually,
you want to access parameter values through `resource[<attribute name>]`.
However, this is not a hash lookup. The `[]` method is implemented by
`Puppet::Type` and has a distinct semantics. Specifically, it won't just
return `nil` if the requested attribute is not declared in the resource. That's why
the check is done on the actual hash `resource.parameters`.

Attributes that are not carried directly but need renaming end up
with an additional key/value pair in their respective translation hashes:

```ruby
class PuppetX::CatalogTranslation::Type
  def rename(attribute,newname,&block)
    if block_given?
      carry(attribute) { |x| block.call(x) }
    else
      carry attribute
    end
    @translations[attribute][:alias] = newname
  end
end
```

Everything is just passed to `carry`, and the `:alias` key is added.
The `translate!` loop looks it up:

```ruby
@translations.each do |attr,translation|
  next if !resource.parameters[attr]
  title = translation[:alias] || attr
  result[title] = if translation.has_key? :block
    translation[:block].call(resource[attr])
  else
    resource[attr]
  end
end
```

Perhaps the most interesting part are `mgmt` attributes that are not
in Puppet, but need to be defined in translated resource hashes.
For example, the `exec` type has a `pollint` parameter in `mgmt`.
Puppet has no use for intervals whatsoever, so it's safe to assume
a default of 0:

```ruby
PuppetX::CatalogTranslation::Type.new :exec do
  spawn :pollint { 0 }
end
```

The block given to the `spawn` method takes no argument.

```ruby
class PuppetX::CatalogTranslation::Type
  def spawn(*attributes,&block)
    attributes.each do |attribute|
      carry(attribute) { yield }
      @translations[attribute][:spawned] = true
    end
  end
end
```

Again, it uses `carry` internally. Since the block takes no parameters,
it can be passed on using the `yield` keyword.
In addition, the marker key `:spawned` is added to the translation hash.
Here's the final touch to the `translate!` loop:

```ruby
@translations.each do |attr,translation|
  title = translation[:alias] || attr

  if translation[:spawned]
    result[title] = translation[:block].call
    next
  end

  next if !resource.parameters[attr]

  result[title] = if translation.has_key? :block
    translation[:block].call(resource[attr])
  else
    resource[attr]
  end
end
```

> Anecdote: The logic in this loop had a subtle bug until I sat down
and wrote this article. Explaining things to others is a useful exercise.

You might be wondering at this point: Why even bother calling the
translation block for spawned parameters in each single resource
that gets handed to `translate!`? After all, it takes no parameter,
so can we not compute the result once and cache it for use in
every resource?

Well, there is one more commodity in the `translate!` method. Before
it starts iterating the translation elements, it sets a member variable
`@resource` to the very `resource` parameter value. This makes it
possible for DSL blocks to examine various properties of the input
resource to make decisions about the result.

For example, this is used to compute the `path` value for `file`
resources. In `mgmt`, managing directories works by declaring files
with a trailing slash on the path, such as `/tmp/` or `/var/lib/`
as opposed to `/etc/passwd`. In `puppet`, this distinction is made
through the `ensure` property (`ensure => file` vs. `directory`).
This is the translation rule:

```ruby
PuppetX::CatalogTranslation::Type.new :file do
  spawn :path do
    if @resource[:ensure] == :directory
      @resource[:name] + "/"
    else
      @resource[:name]
    end
  end
end
```

Note that this is a `spawn` rule rather than `carry`. This is a
consistent pattern, found in all the type translator rule sets.
Even though `puppet` supports a `file/path`
parameter as well, we cannot safely use a `carry` rule. That's
because in `puppet`, the parameter is typically not needed or used.
That's because of Puppet's elegant concept of the "namevar".

A Puppet resource type can declare one parameter to be its "namevar",
which implies that the value of the parameter becomes an alias
name of the resource. Also, if the parameter is not specified,
it implicitly uses the resource title as its value. Therefor,
the following Puppet resources actually try to manage the same file:

```puppet
file { '/etc/passwd': owner => root }
file { 'passwd': group => root, path => '/etc/passwd' }
```

This is not valid, by the way, because Puppet does not support
duplicated resources.

To close, there are two design patterns that I have consistently
copy-pasted to all current translations. One is to spawn the
defining parameter from the namevar, as shown above by example
of `file/path`. The other is to consume the resource title:

```ruby
spawn :name do
  @resource.title
end
```

I guess this one will get hardcoded eventually. Do note that it
is not a `rename` of `title` to `name`, because the resource
title must be obtained through the `title` method as opposed
to `@resource[:title]` or something similar, which won't work.

### Lost in translation

One thing that is not yet consistent across the DSL codebase is
error handling. Early on, I'd add some `raise` calls to the
translator blocks:

```ruby
rename :ensure, :state do |value|
  case value
  # ...
  else
    raise "cannot translate file ensure:#{value}"
  end
end
```

Without exception handling on the engine or face level, this will
lead to failure to generate `mgmt` YAML from certain catalogs at all.
This is definitely not wanted.

To salvage this situation, a proper collection of exception types would
be required, with appropriate handlers and reactions on the respective
levels of code nesting.

In more recent code, I went for a compromise: Errors are logged using
`Puppet.err`. Per default, Puppet will print this in a bright red.
This works nicely for the time being. Once people start using this
for actual Puppet code, we'll see if this is sufficient in all situations.

----

After all this code, let's shift down a little and relax with a
quick look at the test suite.

### The fun part: Writing tests

For the face backend and the DSL engine, I tried to build a sensible set
of unit tests. Despite my firm believe that Test Driven Development is
great, I have yet to actually use it to build a new project. When coding
from scratch, I just need to tinker a while before I can settle into
a more structured workflow. By now, finally, all changes are backed (or even
preceded) by proper tests.

The unit tests are straight forward enough, and it all has been inspired
by the tests in core Puppet a lot. Especially the integration tests take
some direct cues. Puppet's tests use quite a number of actual manifests
to test not only parser functionality, but also the `crontab` provider.
This mode of testing is very compelling to me, because it covers such
a broad range of subsystems.

Especially where translation DSL code is concerned, unit testing is a poor fit.
Instead, I leaned completely on integration
tests and the actual translation of mock manifests to make sure that
the translator does what is expected. This is how that looks:

```ruby
it "only keeps edges between supported resources" do
  catalog = resource_catalog("file { '/tmp/foo': } -> file { '/tmp/bar': } -> resources { 'cron': }")
  graph = PuppetX::CatalogTranslation.to_mgmt(catalog)
  expect(graph['edges']).to_not be_empty
  expect(graph['edges'][0]['from']).to include('name' => '/tmp/foo')
  expect(graph['edges'][0]['to']).to   include('name' => '/tmp/bar')
  expect(graph['edges'].length).to be == 1
end
```

This test makes sure that the relationship edge between the two `file` resources
from the example manifest is translated. It also makes sure that there are
no other edges in the output. The second edge must be dropped, because
the `resources` type does not translate to `mgmt`.

Getting this to work was a bit of an adventure, actually. The compiling code
lives in my spec helper and was copied almost verbatim from Puppet's own
spec library (where it is part of the `PuppetSpec::Compiler` module).

```ruby
def resource_catalog(manifest)
  Puppet[:code] = manifest
  node = Puppet::Node.new('spec.example.net')
  Puppet::Parser::Compiler.compile(node).to_ral
end
```

As an aside, analyzing this test code was what made me aware of Puppet's `--code`
option (accessible through `Puppet[:code]`), which is really helpful for
debugging things, or to run subcommands in a fashion akin to `puppet apply -e`:

    puppet master --compile --code 'notify { "what will happen?": }'
    puppet mgmtgraph print --code 'exec { "/usr/games/cowsay puppet+mgmt ftw": }'

So that was a neat discovery. Still, the test code did not quite work and
threw errors like the following:

```
  12) PuppetX::CatalogTranslation::Type::Package prints an error message about unsupported ensure values
      Failure/Error: Puppet::Parser::Compiler.compile(node).to_ral

      Mocha::ExpectationError:
        unexpected invocation: Puppet.err('no 'environments' in {:current_environment=><Puppet::Node::Environment:24443980 @name=\'*root*\' @manifest=\'no_manifest\' @modulepath=\'\' >, :root_environment=><Puppet::Node::Environment:24443980 @name=\'*root*\' @manifest=\'no_manifest\' @modulepath=\'\' >} at top of [[0, nil, nil]] on node spec.example.net')
```

So the compiler is not sufficiently initialized to render the manifest into
a catalog (the first step before the translation to an `mgmt` graph is possible).
I'm not going to lie: the prospect of properly debugging this failure was
quite daunting. That's why, instead, I opted to do more digging through Puppet
core's test code to find the missing ingredient.

Digging through the code did not take too long, actually. There's quite a lot
of `rspec` initialization in Puppet's own `spec_helper`, but most of it is
quite specific. Pinpointing the likely code stanzas was not too hard, so
I added the following to the `puppet-mgmtgraph spec_helper`:

```ruby
Puppet::Test::TestHelper.initialize

config.before :all do
  Puppet::Test::TestHelper.before_all_tests()
end

config.after :all do
  Puppet::Test::TestHelper.after_all_tests()
end

config.before :each do
  Puppet::Test::TestHelper.before_each_test()
end

config.after :each do
  Puppet::Test::TestHelper.after_each_test()
end
```

It was much harder (I felt) to try and find out what I was actually doing.
I perused the documentation of the `puppetlabs-spec_helper` module to find
out about the actual best practices. But, alas, I did not manage to find
anything resembling this code. Still, these invocations evidently allow
the compiler specs to run, so...er...all is well that ends well?

Not much else to say about the tests, I guess, except that yes, I'm not
adding features or patching code without adding new tests. But no, you can
contribute code without bothering with the whole `rspec` dance. I can
do that for you in post. After all - writing tests is the good part ;-)

### Designing the mgmt interface

I met with James during his recent visit in Berlin, where he presented `mgmt` to
a stunned crowd at CoreOS Fest (shout out to CoreOS, the conference was great!)
Afterwards, we got together for some hacking, looked at the translator code
and did some intensive testing.

James was able to immediately point out some translation details that I needed to
change in my DSL code, and he explained some advanced `mgmt` concepts to me.
It was his idea to make the translator add a `watchcmd` to `exec` resources
from Puppet:

```ruby
"while : ; do echo \"puppet run interval passed\" ; /bin/sleep #{Puppet[:runinterval]} ; done"
```

This scriptlet runs ad infinitum and writes lines to its `stdout` in regular intervals.
The interval is whatever Puppet would use for `puppet agent`. This way, `mgmt` will mimic
the behavior of the Puppet agent: Every half hour, an event is generated for each `exec`
resource to trigger a new check and (if necessary) a run of the `exec` command.

We also extended the `file` translation to support directories, even though directory support
in `mgmt`'s `file` resource is not yet implemented in its `master` branch (but that's
a topic for a future post).

Finally, we tackled an issue that had been in the back of our heads ever since the `puppet-mgmtgraph`
module has been originally conceived. A thing that I had wanted to do from the start,
in the interest of simple testing, was a one-line invocation of `mgmt` with a manifest
for translation. This is what I tried:

    mgmt run --file <(puppet mgmtgraph print ...)

I hoped that `bash`'s process substitution would allow me to seamlessly feed generated YAML
to `mgmt`. No one will be the wiser, right? Well, `mgmt` is nobody's fool apparently. Or, not mine.
Whatever. I did learn that you cannot just hand a FIFO pipe to the tool and claim it was a file.
That's because `mgmt` will perform buffered reads from the file very quickly after startup,
and not bother blocking until the FIFO writer terminates.

And granted, support for this is probably not what anybody really wants or needs right now.
Instead, James' immediate idea was to make Puppet a first class citizen: add a parameter
other than `--file` that makes `mgmt` invoke the `puppet mgmtgraph` tool directly, and read
the graph data from its output. Now, during our mini hackathon, we sat down to specify
how that should actually work.

The `mgmtgraph` module will work in one of three ways:

1. With a manifest right from the command line, with the `--code` parameter.
2. With a manifest in a local file, with the `--manifest` parameter.
3. With a catalog from the Puppet master, when invoked without parameters.

James proposed to use similar semantics for the new `--puppet` switch in `mgmt`:

1. Call `puppet mgmtgraph --manifest` when the parameter value matches `/\.pp$/`
2. Call `puppet mgmtgraph --code` when the parameter value does not match rule 1.
3. Call `puppet mgmtgraph` without a switch when the `--puppet` option gets no value at all.

> Spoiler: There were technical issues with implementing rule 3, so we settled
for an alternative, see below.

At this point, we had already determined that Puppet's `runinterval` setting
is of interest to `mgmt`, because the translated `exec` resources should use
it to receive periodic events. But I quickly realized that it would be valuable
to expose other parameters as well.

Users will want to use alternate certificates
so that `mgmt` will request distinct catalogs (where `puppet agent` runs alongside
`mgmt`). For that matter, there could well be dedicated Puppet masters for the
sole purpose of compiling catalogs for `mgmt`. The list goes on. In essence,
it is important to have control over the `puppet` process inside `mgmt`.

I found it easiest and most intuitive to add one more option to `mgmt`:
with `--puppet-conf`, it should be possible to specify an alternate `puppet.conf` file
that overrides whatever settings the user needs. This is how that could look
like:

```
# /etc/mgmt/puppet.conf
[main]
server=mgmt-master.example.net
certname=mgmt.this-agent.example.net
runinterval=1720
vardir=/var/lib/puppet-mgmt
```

And here's how to use it:

    mgmt run --puppet --puppet-conf /etc/mgmt/puppet.conf

### Go code

The honor of implementing this specification was reserved for your's truly (courtesy
of James). I had dabbled in another patch before (which is still WiP at the time of writing
this), but other than that, this was my first dance with Go. What fun. It makes me quite
nostalgic for my C days.

I will try and give yet another rough description of some code I wrote, but please do
note that `mgmt` is still very much in flux and any or all code excerpts may become
very non sequitur to the future codebase.

First off, the whole command line parsing logic is delegated to an external
[go package](https://github.com/urfave/cli). What I realized later is that the
parameter tokenization is actually part of Go's `stdlib`. Anyway, registering
the new CLI switches is straight forward (in `main.go`):

```go
app.Commands = []cli.Command{
	// ...
	Flags: []cli.Flag{
		// ...
		cli.StringFlag{
			Name:  "puppet, p",
			Value: "",
			Usage: "load graph from puppet, optionally takes a manifest or path to manifest file",
		},
		cli.StringFlag{
			Name:  "puppet-conf",
			Value: "",
			Usage: "supply the path to an alternate puppet.conf file to use",
		},
	},
}
```

However, I hinted earlier that the "do X if that switch gets no parameter" did not work out.
That's because you cannot use a `string` type parameter like a `boolean` flag. Instead, the
user would be required to actually pass `'--puppet ""'` in order to allow `mgmt` to parse that
as an empty parameter. I deemed this too cumbersome, and pretty bad UX.

So we settled for a magic value instead: `mgmt run --puppet agent`. Technically speaking, this
is ambiguous, because "agent" is syntactically a valid Puppet manifest. Granted, it could only
ever work if the user happened to have a Puppet module that implements a function named `agent`
that requires no parameters. It appears improbable that this manifest will be very popular.
(Even if it would, implementing a workaround for this clash would be simple.)

Next was the program logic that is responsible for loading the resource graph. The `--file`
mode is implemented right in the
[main routine](https://github.com/purpleidea/mgmt/blob/9f56e4a582473840418794c2f916f4e22cbf4213/main.go#L109)
of `mgmt`. 

```go
if c.IsSet("file") {
	config = ParseConfigFromFile(file)
} else if c.IsSet("puppet") {
	config = ParseConfigFromPuppet(c.String("puppet"), c.String("puppet-conf"))
}
```

The `ParseConfigFromPuppet` function is part of a small new code module in `mgmt` in the `puppet.go`
file. Its purpose is the construction and running of appropriate `puppet` commands, and also
feeding translated YAML data to the existing graph loading function. The interface specification
translates to code quite literally:

```go
var cmd *exec.Cmd
if puppetParam == "agent" {
	cmd = exec.Command("puppet", "mgmtgraph", "print", puppetConfArg)
} else if strings.HasSuffix(puppetParam, ".pp") {
	cmd = exec.Command("puppet", "mgmtgraph", "print", puppetConfArg, "--manifest", puppetParam)
} else {
	cmd = exec.Command("puppet", "mgmtgraph", "print", puppetConfArg, "--code", puppetParam)
}
```

The `puppetConfArg` variable is empty per default, but if `--puppet-conf` is passed to `mgmt`, it
will have a value such as `--config /path/to/puppet.conf` for Puppet.
The output of the selected `puppet mgmtgraph` command is stored in an array slice of type `[]byte`,
and can be passed right to the YAML deserializer function `Parse` from the `config` module.

The other significant functionality in the `puppet.go` module is the retrieval of the `runinterval`
configuration option of the Puppet agent. We use this value in `mgmt` to determine how often the
catalog should be retranslated (and received from the master if so configured). It works with a
similar approach:

```go
if puppetConf != "" {
	cmd = exec.Command("puppet", "config", "print", "runinterval", "--config", puppetConf)
} else {
	cmd = exec.Command("puppet", "config", "print", "runinterval")
}
```

And that's the gist of it. If you're interested in the minutiae, head right over to
[GitHub](https://github.com/purpleidea/mgmt/blob/8f83ecee65e070da53fc884e5a7ddbf93b7af1f6/puppet.go)
and let me know what I should have done better :-) Do cut me some slack though,
this is the first Go code I've contributed anywhere.

Writing Go is remarkably enjoyable to me. It feels like revisiting all the good parts
about C, with many of the rough edges filed off. And while I enjoyed Perl and still have
good times with Ruby, it's *so good* to take a break from duck-typing in a truly modern
language.

Here's wishing I could also claim that running `mgmt` is more pleasant than operating
Ruby and its gem jungle, but alas, there were a couple of hickups right from the start.
One issue that cost me several days is what appears to be a weird race that makes some
go routines get stuck in `select` calls. This would happen quite consistently in a
simple acceptance test I had whipped up. Being unable to rely on tests makes for a poor
coding experience.

In another instance, my programs would freeze when working with `inotify` watches,
adding and removing them from files. When hunting odd behavior in an application like
`mgmt`, that relies heavily on such watches, this kind of issue is quite problematic.
All the more frustrating when your fellow hackers cannot reproduce the errors.

But then, the last part makes it safe to assume that some circumstance is to blame.
My workstation runs Debian stable, which is usually a good choice. However, the
included `golang` packages are in version `1.5.1`, which is a little dated. At the
time of writing this, the `1.6` line is quite mature and `1.7` is on the horizon.

Topping my to-do list is to go ahead and try version `1.5.4` of golang, to see if that
resolves the issues I'm seeing. It would be very surprising if it's actually the kernel
or OS that leads to those malfunctions. Props to [Brice](http://www.masterzen.fr/)
for pointing me towards Travis's [Gimme](https://github.com/travis-ci/gimme) script.
Stay tuned for news about how all that turned out.

### What's next now

It's hard to overstate how much fun it is to write Go code, and I find myself
increasingly fascinated by `mgmt`. Still working on that patch for managing directory
trees. The Puppet translator is far from finished as well. There are three
specific areas that need attention:

1. Classes and defined types in and of themselves need no special handling, because
the resources declared within are part of the catalog, so the translator will find them.
However, the *relationships* between such resource containers are currently not
recognized. A relationship such as `require => Class[...]` will have no effect
in a translated catalog.
2. Exported resources will probably be difficult. I do believe that the export is
represented in the catalog, so it can also be translated. The import is another
story. Supposedly, the import will not show up explicitly, though. This one will
take some more research.
3. Unsupported resources should run through `puppet resource`, as described in
the previous post.

Lots of work on these projects, and I'd also like to take some time to finally help
out over at [Vox Pupuli](https://voxpupuli.org/) again. No idea how to fit all that into
one spare time calendar, but on the bright side, at least I can scratch this gigantic
blog post off my list. Thanks for reading!
