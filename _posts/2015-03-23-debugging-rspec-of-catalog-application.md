---
layout: post
category: tips
tags: puppet rspec catalog logs debug
---

Running catalogs from RSpec tests is tricky. There are some examples
in the existing test base that help doing it. One basic pattern goes like this:

{% highlight ruby %}
describe SomeThing
  let(:catalog) { Puppet::Resource::Catalog.new }
  context "when doing something else" do
    let(:resource) do
      Puppet::Type::MyType.new(
        :ensure => :present,
	:param  => 'value',
      )
    end

    it "behaves a certain way" do
      catalog.add_resource(resource)
      catalog.apply
      expect( check_result() ).to be_truthy
    end
  end
end
{% endhighlight %}

Of course, such tests can fail for various reasons. The test environment
can be incredibly fragile at times. But since all the interesting things
happen in the `catalog.apply` method, it's difficult to probe into it
without resorting to [pry](https://github.com/nixme/pry-debugger) and friends.

Fortunately, Puppet's RSpec suite is [set up](https://github.com/puppetlabs/puppet/blob/76afe2e8c7d7942c805e99f83f115a27d78684a8/spec/spec_helper.rb#L132)
to collect all log output in the special member variable `@logs`.
The easiest way (that I know of) to examine this data in a failing test
is temporarily raising it as an exception.

{% highlight ruby %}
it "behaves a certain way" do
  catalog.add_resource(resource)
  catalog.apply
  raise "#{logs * "\n"}"
  expect( check_result() ).to be_truthy
end
{% endhighlight %}

When this test is run, it will fail due to the unexpected exception,
dumping the exception details on the terminal. These details consist
of the Puppet output as produced when applying the catalog. It closely
resembles the output of `puppet agent` or `puppet apply`. A very good
first step for debugging this kind of test.
