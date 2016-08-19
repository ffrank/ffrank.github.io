---
layout: post
category: features
tags: [ puppet, mgmt, resource, type, types, notify, cron, default ]
summary: A misnomed post about running arbitrary Puppet code through mgmt.
---

Through the last months, I've written a lot about my work on the translator
module for Puppet, which allows us to control [mgmt](https://github.com/purpleidea/mgmt/)
with manifest code.

One gimmick we had imagined early was the ability to support arbitrary
manifests in `mgmt`, by invoking `puppet resource` for vertices that `mgmt`
itself cannot handle. This works now, in principle, and it nicely demonstrates
the amount of work that still needs doing.

### Shaving the yak

Adding the generic pseudo-translator code to the module was relatively
straight-forward. You can still see it on [GitHub](https://github.com/ffrank/puppet-mgmtgraph/blob/fcb0b82a7e609283cb1b49fb7838ba22228ad49c/lib/puppetx/catalog_translation/type/default_translation.rb).
It's basically a copy of the `exec` translator, but the `cmd`
value is constructed from the original resource's type, title, and all
attributes.

The first thing I tried was a very simple `cron` resource.

    puppet mgmtgraph print --code \
      'cron { "/bin/true": ensure => present, hour => 0, minute => 12 }'

The intention was to generate the following command within an `exec` vertex
for `mgmt`:

    puppet resource cron /bin/true ensure=present hour=0 minute=12

However, this was the actual result:

    puppet resource cron '/bin/true' \
      provider='crontab' ensure='present' \
      minute='["12"]' hour='["0"]' \
      target='ffrank' loglevel='notice'

Now, the additional parameters `provider`, `target` and `loglevel` make sense and are not
harmful. The values for `hour` and `minute`, on the other hand, look sensible from a
Ruby point of view, but are not suitable for the `puppet resource` command at all.

Should the generator code perform more processing to get rid of the array braces?
No, because there is a very good reason for them to appear here. So instead of
looking for a workaround for this specific example, I got thinking about how the
following resource should be translated:

```puppet
cron {
  "/bin/true":
    ensure => present,
    hour => [ 0, 12 ], # <- array!
    minute => 12,
}
```

Quite a few of Puppet's core resource types accept array values for some properties,
but `puppet resource` will not actually have any of that. After all, it wasn't built
for the kind of heavy lifting that we have in mind here.
This essentially means that we cannot use `puppet resource` in this fashion after all.
An alternative is needed.

It would be possible to fall back to `puppet apply` and create one-shot manifests
for each resource, instead of the `puppet resource` invocation. However, I feel that
this would be even more painful than the original plan. Not only would Puppet need
to launch for every resource, it would also need to parse a manifest, build a catalog
and create a complete transaction context for every single vertex. So what's the
middle ground here?

I ended up creating [yet another Puppet module](https://github.com/ffrank/puppet-yamlresource)
that adds the `puppet yamlresource` subcommand. It's essentially a clone of
`puppet resource`, but as the name implies, it accepts YAML input:

```yaml
puppet mgmtgraph print --code \
  'cron { "/bin/true": ensure => present, hour => 0, minute => 12 }'
# ...
resources:
  exec:
  - name: Cron:/bin/true
    cmd: |-
      puppet yamlresource cron '/bin/true' '{"provider": "crontab",
        "ensure": "present", "minute": ["12"], "hour": ["0"],
	"target": "ffrank", "loglevel": "notice"}'
# ...
```

The translator actually feeds it with JSON, but that is still valid YAML (as
[Adrien](https://twitter.com/nullfinch) once pointed out to me), and pleasantly
compact on the command line.

### Now what

All right, we now have rules to accept not only [all class and resource
relationships]({% post_url 2016-07-12-edging-it-all-in %}), there is even a way
to usher all the resources over to `mgmt`, if only to have them handed right
back to Puppet in the end. Does that mean that we can go ahead and throw
any manifest at the translator and let `mgmt` run along? Not quite.

After all, there are still important details that get lost in translation
(bonus question: pun or fair use). Simple example, `mgmt` does not yet
manage file permissions. Consider a `file` resource in Puppet that
specifies a value for the `mode` property:

```yaml
puppet mgmtgraph print --code 'file { "/etc/passwd": mode => "644" }'
# ...
  file:
  - name: "/etc/passwd"
    path: "/etc/passwd"
    content: 
```

The result is a plain resource that does not manage anything at all.
There are about 25 other parameters that can be passed to `file` resources
in Puppet that lack a counterpart in `mgmt`. The other supported resource
types have similar issues, if not quite so many different parameters.

This is a major problem, because a large manifest can have an almost perfect
translation, but a single property that gets ignored from an early resource
can throw all dependent resources off. That's why it's important that this
effect is at least very obvious to the user. To that end, the translator
module now emits warnings about any parameter it gets from Puppet that
has no translator handling.

The first tests of this feature went quite a bit noisier than expected.
All resources would lead to output such as this:

    Warning: cannot translate: Package[cowsay] { loglevel => :notice } (attribute is ignored)

A simple `file` resource would produce 18 such warnings! These were spurious
for the most part. It is not really important what log level the user chose
for a given resource. The catalog will still work as intended.

False positives like these are a pet peeve of mine. Even in moderate numbers, they
can render a monitoring or logging system practically unusable. Avoid if
at all possible. In the case of resource translators, this called for a means
to declare a white-list of attributes that can just pass without comment.
It takes the following form in [translator DSL](/features/2016/06/12/puppet,-meet-mgmt/):

```ruby
ignore :validate_replacement, :provider, :sourceselect
```

The above example is from the `file` translator. The ignored parameters are
of no interest because

1. The `validate_replacement` parameter only has effect in tandem with
`validate_cmd` (which *will* cause a warning).
2. The `file` type has only one provider (plus one for Windows).
3. The `sourceselect` parameter is only effective on files that specify
more than one `source`, which is not translatable in the first place.

Other parameters require a little more sophisticated handling. Consider
the `purge` parameter of the file resource. It makes Puppet remove
unmanaged files from directory trees that are under management.
When left at its default of `false`, it has no effect whatsoever.
That's why it doesn't make sense to emit a warning in this case.
However, if a `file` uses a `purge => true` parameter, the warning is
very important, because `mgmt` cannot replicate Puppet's behavior
appropriately. We can express this in translator DSL as follows:

```
ignore :purge do |value|
  if value
    Puppet.warning "#{@resource.ref} uses the purge attribute, which cannot be translated. Unmanaged content will be ignored."
  end
end
```

All currently available translator instances have received such handlers.
Puppet's parameters tend to have default values, so without the `ignore`
statements, translating the resources would become quite noisy.

### Putting it to the test

To see how well the whole translation engine worked by now, I let it
run against one of my favourite manifests. It goes like this:

    puppet apply -e 'include puppetdb'

Doing this on a fresh system takes a while, but you end up with a
functional PuppetDB stack, so that's pretty neat.

I did not actually intend to run it (not really feeling the need to
get PuppetDB onto my hacking laptop right now), but it's a cool
showcase for the translator. Sure enough, the `mgmt` graph came out
just fine (I think). It's comprised of 223 vertices and 465 edges
between them. 130 of the vertices are of type `noop`; in other words,
we are looking at 65 classes and defines. The 93 bona fide resources
are mostly `exec` (for unsupported resource types), but also around
40 translatables like `file` and `service`.

The number of translator warnings was quite alarming at first, more
than 570. However, upon closer examination, it became clear that
the newly introduced default translator did in fact complain about almost each and
every attribute it encountered, despite the fact that it actually
accounted for all of them. By fixing this, I more than halved the
number of warnings to about 200.

These warnings from a real-live catalog are really valuable, because
they allow us to estimate a few things:

* How many false positives are still raised to the user?
* What attributes are frequently reported and should be added to the translator?
* Which ones are not (yet) supported by `mgmt`?

All quite useful to guide future development.

### What's still missing

The translator itself is in a nice shape. In fact, I feel that this is a time
at which it makes sense for advanced Puppet users to go ahead and start playing
with the translator, `mgmt` itself, and the glue code. Feedback and patches
most welcome. (The [Forge](https://forge.puppet.com/ffrank/mgmtgraph) is a
good place to get started with the translator.)

There is still the question about exported resources that I'd like to tackle
eventually (but here, also, patches are welcome if you feel so inclined).

A big item is performance. The default translation needs to invoke Puppet
at least once for each vertex that is visited. Launching the Ruby runtime
and loading the necessary Puppet code is very expensive. I guess this will
be something I will address before getting up to other work again.

Thanks for reading, and stay tuned for more updates.
