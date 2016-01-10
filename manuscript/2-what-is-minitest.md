# What is Minitest?

Minitest is a testing tool written by
[Ryan Davis](http://www.zenspider.com/projects/minitest.html). It's
[first commit](https://github.com/seattlerb/minitest/commit/17bc421f953aac4b8930c6049f6e73ffb87b8b64)
is on October 9th 2008, but it seems like it got renamed from `miniunit` to
`minitest` at the very beginning. Nevertheless, the framework originates since
around 2008.

Minitest, according to it's README, is composed of six parts:

- `minitest/unit` which is the actual unit testing framework, with all of it's
  assertions and magic
- `minitest/spec` which is the spec engine that enables the BDD syntax to work
  on top of `minitest/unit`
- `minitest/mock` which is a mocking/stubbing framework
- `minitest/pride` is a testing reporter which colors the tests output
- `minitest/benchmark` which is a tool that enables asserting on a code's
  performance
- `minitest/autorun` is the actual test runner

What the Minitest maintainers and community are very proud of is Minitest's
simple and explicit approach to testing in general. Unlike RSpec, which has a
lot of custom matchers and expectations, Minitest is super simple. In fact Ryan,
the Minitest author, wanted to make Minitest tests look like plain Ruby, no
magic - no fluff. Just plain old Ruby classes, inheriting from `Minitest::Test`.

Seeing Minitest in action is super simple. Having a class called `Cat`:

```ruby
class Cat
  def sleepy?
    true
  end

  def talk!
    "MEOW!"
  end
end
```

We can test it with the following code:

```ruby
class TestCat < MiniTest::Test
  def setup
    @cat = Cat.new
  end

  def test_that_cat_is_always_sleepy
    assert @cat.sleepy?
  end

  def test_that_it_can_meow
    assert_match /^meow/i, @cat.talk!
  end
end
```

By being very minimal and simple, Minitest is also super fast. And being fast
makes it super extendable, because projects like Minitest, with simplicity in
mind are easy to read, understand and extend.

