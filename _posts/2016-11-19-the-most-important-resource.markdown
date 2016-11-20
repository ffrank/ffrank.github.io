---
layout: post
category: features
tags: [ puppet, mgmt, types, notify, msg, systemd, syslog ]
summary: A short recount of writing a new resource for mgmt.
---

**Note**: This article was written in early October 2016, then
sat in my local git clone for almost two months, waiting for final
review. This happened because of an intense conference season and
some other distracting circumstance. So, it is with great relief
and satisfaction that I can finally throw this at you now :-)
Enjoy, and thanks for reading!

----

Current status: Itching to write some Go. Now what? Adding
[tests](https://github.com/purpleidea/mgmt/commit/f5dd90a8dd71b5f7883642eb8051a0a702353048)
to [mgmt](https://ttboj.wordpress.com/tag/mgmtconfig/)
is a good exercise, and I will be doing a lot more of it
in the months to come. However, [James](https://ttboj.wordpress.com/)
had suggested I write a new resource, so I sat down and thought about
what would make a useful resource type.

Another piece of technology that recently inspired me is `systemd`.
(PSA: If you feel the urge to tweet an opinion about that to me, you
probably shouldn't.) So I went spelunking through `systemd`'s facilities
to see if there was something that would be nice to manage.
`systemd` exposes a number of APIs, and because it's a central hub
for many essential system functions, it's potentially a good source
for `mgmt`'s event driven management approach.

The [journal API](https://github.com/coreos/go-systemd#journal)
caught my eye. It's a powerful feature of the software, and as a source of
events, the journal has huge potential. Alas, reading and writing the journal
via Go APIs are quite different beasts. For writing, there is a pure-Go package
`journal`. Reading, on the other hand, relies on `journald`'s C APIs. Building a
client requires [cgo](https://golang.org/cmd/cgo/). Now I wasn't keen on adding
complexity to the `mgmt` build chain just yet, so I settled for a resource that writes
messages to the journal.

### Getting started

As so many features in all(?) open source projects, this one started by
mimicking a similar piece of existing code. On
[IRC](http://irc.netsplit.de/channels/details.php?room=%23mgmtconfig&net=freenode),
there has been talk of the `timer` resource a good while ago, so
[timer.go](https://github.com/purpleidea/mgmt/blob/master/resources/timer.go)
became my cue sheet for `journal.go`. In a recent hackathon (when we got the
resource ready to merge), James clarified that
[noop.go](https://github.com/purpleidea/mgmt/blob/master/resources/noop.go) is
a better choice for this purpose. It demonstrates all the important patterns.

Let's look at the implementation for the new resource.  The heart of a `mgmt`
resource is its `CheckApply` method. It makes the call whether the resource
needs applying (the "check"), and if necessary, does the work. James has made
some [nice slides](https://dl.dropboxusercontent.com/u/48553683/slides/mgmt-workshop-systemdconf-2016.pdf)
that summarizes the logic flow (see page 8).

The `apply` action for the new resource initially was a plain
[journald API call](https://godoc.org/github.com/coreos/go-systemd/journal#Send). Its
parameter values must be filled from resource parameters. This made it quite
clear what the set of parameters should look like:

```go
type MsgRes struct {
        BaseRes        `yaml:",inline"`
        Body           string            `yaml:"body"`
        Priority       string            `yaml:"priority"`
```

Any resource type embeds `BaseRes`. The `Body` parameter maps to the `message` parameter for
`journal.Send`. For good measure, I not only
included the `Priority` parameter, but also added `Fields` in order to make
journald's `vars` parameter available:

```go
        Fields         map[string]string `yaml:"fields"`
}
```

This means that such a resource can be serialized to YAML as follows:

```yaml
msg:
- name: a-journal-resource
  message: This entry was made by mgmt
  priority: Notice
```

When James sat down with me to iron out the wrinkles in the code, we came up with some
additions. Writing to the journal is now optional, and syslog output will be
added as an additional option. The regular `mgmt` log is always written.

### Lessons learned

Some useful notes from our hacking session for future reference:

 * The following piece of boilerplate code is important for resource type code.
   So is importing the `encoding/gob` package. Both are needed for loading the
   type from YAML.

```go
func init() {
        gob.Register(&MsgRes{})
}
```

 * Speaking of YAML, it also requires that all resources are part of the
   `GraphConfig struct` in `yamlgraph/gconfig.go`.

 * The `Watch` function contains quite a bit of boilerplate code itself. It's
   easy to miss important parts of it, and if you do, the resource will likely
   not work quite right. To be safe, start with the implementation from the
   `noop` resource (after all, copy/paste is the most common form of code
   reuse). Specific implementation will usually be added in the `event :=
   <-obj.Events()` case.

 * Be sure that the new resource implements the `CheckApply` method with the
   correct semantics. It's not quite obvious how to arrive at the correct `bool`
   return value from a given situation. Reading existing examples is of limited
   use. It's best to follow the guide from
   [James's slides](https://dl.dropboxusercontent.com/u/48553683/slides/mgmt-workshop-systemdconf-2016.pdf).

 * Most resources will want to cache the sync status. The `CheckApply` method
   can rely on the cache, and the `Watch` loop handles cache invalidation.

 * Running `go lint` is very helpful in order to spot missing documentation.
   This is important because generated documentation is now available at
   [GoDoc](https://godoc.org/github.com/purpleidea/mgmt).

### Exposing msg to Puppet code

The new `msg` resource maps quite well to Puppet's `notify`. These were the
steps to arrive at the translator code using the [translator
DSL](/features/2016/06/12/puppet,-meet-mgmt/):

 1. The translated type is `notify` and it results in a `msg`.

    ```ruby
    PuppetX::CatalogTranslation::Type.new :notify do
      emit :msg
    ```

 2. As per usual, the `mgmt` resource `name` is derived from the `title` in
    Puppet. Note that this is a `spawn` action rather than a `rename`, because
    the `title` is a special method, not a field in `resource.parameters`.

    ```ruby
    spawn :name do
      resource.title
    end
    ```

 3. The `msg.body` is derived from the `message` parameter in Puppet. This one
    is not a `rename` either, because `message` is the NameVar, so it must be read
    as follows.

    ```ruby
    spawn :body do
      resource[:name]
    end
    ```

 4. The `loglevel` metaparameter in Puppet translates quite directly to
    `msg.priority`. Now, finally, this can be implemented using `rename`. One
    case that needs special handling is Puppet's `verbose` value, which is just
    an alias for `notice`.

    ```ruby
    rename(:loglevel, :priority) do |value|
      if value == 'verbose'
        'Notice'
      else
        value.capitalize
      end
    end
    ```

 5. That's it. All `msg` parameters that can be populated from a `notify`
    resource are being handled. The last step involves looking at other
    parameters that `notify` accepts. Puppet offers a `withpath` parameter. This
    is currently meaningless to `mgmt`, because there is no language that could
    provide any context to a `msg` resource. 

    ```ruby
    # mgmt (currently) has no notion of scope
    ignore :withpath
    ```

The original translator code for Puppet's `notify` does not include anything
beyond the above snippets. It has hit the `master` branch of the
`ffrank-mgmtgraph` module and is currently pending release.

### New ideas

Now that the *tremendously* important `msg` resource is available, I'd like to see
it fully usable from Puppet code. As with most Puppet code translator modules, there
are some limitations. In the case of `notify`/`msg`, it's simply the fact that the `journal` and
`syslog` parameters are not available to a Puppet manifest.

That's because
`notify` has no parameter that can be used to carry this information. The
translator cannot just make up new parameters either, because the manifest code
has to pass Puppet's own internal validator first, before a catalog is sent to
the translator.

So now I'm looking for ways to expose such parameters to Puppet. Presumably,
this will work by either adding one or more additional metaparameters, or using
an existing one to carry encoded key/value pairs meant for the translator.
