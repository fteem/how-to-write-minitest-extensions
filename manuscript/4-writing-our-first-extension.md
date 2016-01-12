# Our first Minitest Extension

In this chapter, I will take you through the journey of building a really tiny
Minitest extension. For the purpose of this chapter, we will build a gem that
applies colors to the final test results of a Minitest suite.

This is what the our end result should look like:

![The extension in action](images/clr-end-result.png)

At the end of this section, you will know how to build a small Minitest
extension, packed in a gem. We will also learn about writing and loading plugins
into the Minitest test run.

## Setting up the plugin skeleton

We will call this plugin `minitest-clr`, an acronym of *Minitest Color Results*.
First, open up your terminal and type:

```bash
bundle gem minitest-clr
```

This command will generate a skeleton for a gem, which will allow us to easily
work with our extension and upload it later on Rubygems.org. The command's output
will look something like:

```bash
➜  bundle gem minitest-clr
Creating gem 'minitest-clr'...
      create  minitest-clr/Gemfile
      create  minitest-clr/.gitignore
      create  minitest-clr/lib/minitest/clr.rb
      create  minitest-clr/lib/minitest/clr/version.rb
      create  minitest-clr/minitest-clr.gemspec
      create  minitest-clr/Rakefile
      create  minitest-clr/README.md
      create  minitest-clr/bin/console
      create  minitest-clr/bin/setup
      create  minitest-clr/.travis.yml
      create  minitest-clr/.rspec
      create  minitest-clr/spec/spec_helper.rb
      create  minitest-clr/spec/minitest/clr_spec.rb
Initializing git repo in /some/path/here/minitest-clr
```

Having our gem skeleton in place, let's take a step back and cover some important
rules about writing Minitest plugins (or, extensions). Minitest, on start-up,
looks for files in Ruby's `LOAD_PATH` whose names match `minitest/*_plugin.rb`.
So, by naming our gem `minitest-clr`, the actual name of the library will become
`minitest/clr`. On the code level, it means that the `Clr` class will live under
the `Minitest` namespace.

As you can notice in the skeleton output above, as expected, bundler has no idea
that we want to build a Minitest extension, therefore it's not creating a
`minitest/clr_plugin.rb` file.

## Development flow

Although having the plugin skeleton is nice, for the development flow I would
recommend using a project (preferably a smaller one) that uses Minitest as the
testing framework.

For example, I am the author of a Ruby gem called
[Forecastr](https://github.com/fteem/forecastr) which uses Minitest. Inside the
`lib` directory of the project, we will add a `minitest` folder. This will be
our development workspace. After we are happy with the extension, we can extract
the logic of the extension to the gem skeleton we created in the last step.

In your own development setup, you can use any Ruby project. Also, the path of
the directory does not have to be `lib`, as long as the path is included in
Ruby's `LOAD_PATH`.

## Adding the plugin

The file will be consisted of two methods, `Minitest.plugin_clr_options` and
`Minitest.plugin_clr_init`. Let's create the file on our own, and add the first
method:

```ruby
# lib/minitest/clr_plugin.rb

module Minitest
  def self.plugin_clr_options(opts, options)
    opts.on "-c", "--clr", "Colorize results" do
      options[:clr] = true
    end
  end

  def self.plugin_clr_init(options)
    # To do later...
  end
end
```

There are couple of important things that you must understand about the naming
and the load order of Minitest extensions:

Note that we are adding the methods to the `Minitest` module. This is due to the
workings of Minitest's internals. By loading the `clr_plugin` on start-up it
will try to invoke the `Minitest.plugin_clr_options` method. If our plugin was
called `lorem_ipsum_plugin`, then the method it would try to invoke on start-up
would be `Minitest.plugin_lorem_ipsum_options`. The same applies to the other
method, `Minitest.plugin_clr_init`, which will cover in a bit.

Now, although Minitest is a testing tool, it is a command line tool as well.
Think about it - it allows us to run our tests from the command line. Having that
behaviour in mind, the role of the method `plugin_clr_options` is to take the
options hash and the opts, which is an object of the `OptionParser` class, and
deal with the various flags/options that this plugin might need.

To show this in action, our method will take the `-c` or `--clr` flags, which
will enable our plugin at runtime. This means that to get colored test results,
the user will have to run his tests with:

```bash
ruby -I lib:test test/a_test.rb --clr
```

Or

```bash
ruby -I lib:test test/a_test.rb -c
```

Please note that, if you are building plugins, try not to override any of the
available flags for Minitest.

{ pagebreak }

## Plugin Initialization

Since our options parsing is in place, we can continue with the other method -
`Minitest.plugin_clr_init`. This is the method where our plugin will be invoked
on runtime.

Although our plugin is small, there are some things that must be understood
before we continue with the plugin.

### Terminal Colors

Without going much in depth on this topic, let's cover the basics. In our test
results, we are interested in using four colors: blue, green, red and yellow.
Each of these colors will represent different information: blue for overall test
runs, green for the number of assertions, red for railures and errors, and yellow
for skipped tests.

Colors in terminals are achieved by using special characters. To see them in
action, try this in your terminal:

```bash
echo "\e[32m green output!"
```

As you can notice, the characters `\e[32m` made the text green. Let's create a
small class, that will apply colors to any strings:

{pagebreak}

```ruby
# lib/minitest/colorize.rb

module Minitest
  class Colorize
    ESC    = "\e[0m"

    GREEN  = "\e[32m"
    BLUE   = "\e[34m"
    RED    = "\e[91m"
    YELLOW = "\e[93m"

    def self.green(string)
      "#{ESC}#{GREEN}#{string}#{ESC}"
    end

    def self.blue(string)
      "#{ESC}#{BLUE}#{string}#{ESC}"
    end

    def self.red(string)
      "#{ESC}#{RED}#{string}#{ESC}"
    end

    def self.yellow(string)
      "#{ESC}#{YELLOW}#{string}#{ESC}"
    end
  end
end
```

The `Colorize` class has five constants, four of which represent a color, and
one that is used to represent color escaping. For example, when we want to make
a string appear in yellow color in the terminal, we can use:

```ruby
Colorize.yellow("Lorem ipsum dolor sit amet")
```

If we try this class in `irb`, we will notice something interesting:

```irb
>> Colorize.yellow("Lorem ipsum dolor sit amet.")
=> "\e[0m\e[93mLorem ipsum dolor sit amet.\e[0m"
```

{ pagebreak }

The `Colorize` class will only append the proper coloring to the text, but the
magic happens when we ouput the text to `STDOUT`, using the well known `puts`
method:

```irb
>> puts Colorize.yellow("Lorem ipsum dolor sit amet.")
```

The result:

![Yellow output in terminal](images/clr-yellow-output.png)

### Manipulating IO

Since we have dealt with the coloring class, let us get back to our plugin and
the `Minitest.plugin_clr_init` method. This method will be invoked on startup,
which means it is the right place to invoke the class that will do the coloring
of the result output.

```ruby
module Minitest
  def self.plugin_clr_options(opts, options)
    opts.on "-c", "--clr", "Colorize results" do
      options[:clr] = true
    end
  end

  def self.plugin_clr_init(options)
    io = Minitest::Clr.new(options[:io])
    self.reporter.reporters.grep(Minitest::Reporter).each do |rep|
      rep.io = io
    end
  end
end
```

Let me take you line by line through this new method. In the first line, we
create a new object of the `Minitest::Clr` class. This class, which we will
cover later, will have the responsibility of coloring the resulting output. Now,
what might not be clear at this point, is that the `Minitest::Clr` class will
basically be a wrapper around an I/O stream, more specifically `STDOUT`. Which
is the reason why it's taking `options[:io]` as a parameter.

Now, I know you might be wondering where `options[:io]` comes from. Let us put
some debugging statements inside the `plugin_clr_init` method and see what the
`options` hash contains:

```ruby
# lib/minitest/clr_plugin.rb

module Minitest
  def self.plugin_clr_options(opts, options)
    opts.on "-c", "--clr", "Colorize results" do
      options[:clr] = true
    end
  end

  def self.plugin_clr_init(options)
    puts options.inspect

    io = Minitest::Clr.new(options[:io])
    self.reporter.reporters.grep(Minitest::Reporter).each do |rep|
      rep.io = io
    end
  end
end
```

When we run the tests in our workspace project, we will see the contents of the
`options` hash:

```ruby
{ :io => #<IO:<STDOUT>>, :seed => 60965, :args => "--seed 60965" }
```

As you can notice, the `:io` key of the `options` hash has `STDOUT` as a value.
`STDOUT` is an I/O stream, which simply put, handles the input and output. What
we want to achieve with the `Minitest::Clr` class is to wrap the I/O stream we
get and make it color the output of the test results.

Now, since `Minitest::Clr` will wrap the I/O stream, in the next section of the
`plugin_clr_init`, we seek all of the reporters that our project has plugged into
Minitest and we change the I/O stream for each of them with the `Minitest::Clr`
object that we created. This ensures us that for any reporter that Minitest uses
in the project, the plugin will work.

{ pagebreak }

## Wrapping the I/O stream

When we got the plugin initialization out of the way, let us see how we can write
our I/O wrapper. First, we will see the code and then I will explain what each
line of does:

```ruby
require "lib/minitest/colorize"

class Minitest::Clr
  attr_reader :io

  def initialize io
    @io = io
  end

  def puts output = nil
    return io.puts if output.nil?

    if final_report?(output)
      io.puts final_report!(output)
    else
      io.puts output
    end
  end

  def method_missing(msg, *args)
    io.send(msg, *args)
  end

  private

  def final_report?(output)
    output =~ /(\d+) runs, (\d+) assertions, (\d+) failures, (\d+) errors, (\d+) skips/
  end

  def final_report!(output)
    [runs, assertions, failures, errors, skips] = output.scan(/\d+/)
    "#{Colorize.blue(runs)} runs, #{Colorize.green(assertions)} assertions, #{Colorize.red(failures)} failures, #{Colorize.red(errors)} errors, #{Colorize.yellow(skips} skips"
  end
end
```

This is the whole magic in the plugin. Just like we mentioned in the last section
of this chapter, the class takes the I/O stream as a parameter in the
initializer. The only method that we want to override in this wrapper is the
`Minitest::Clr#puts` method. It will check if the output which is supposed to be
sent to the I/O stream is the last test report, which is done in the private
method `final_report?`. If it is, it extracts the numbers from the output and
builds the new output, with colors. Then, the output is sent to the I/O stream.

Any other output, that does not match the test results report format, will not
be colored and will be sent just to I/O stream. The last magic happens in
`method_missing` method. Because we are not interested in overriding any other
methods of the I/O stream, we just send any other method that is called on the
`Minitest::Clr` object to the I/O object.

## Putting it in action

Since we have both, the plugin file and the actual I/O wrapper class, playing
together nicely, let's see our plugin in action. In your workspace, run any
Minitest test file that you have, by adding the `--clr` flag at the end:

```bash
ruby -I lib:test test/some_test.rb --clr
```

You should see the last line of the output have colored numbers.

## Final steps and usage

Having our plugin working nicely, we can now move our code to the gem that we
created at the beginning. In the gem, we need to create the `lib/minitest`
directory, and move both, the `clr_plugin.rb` and the `clr.rb` files. After,
building and publishing the gem are quite trivial steps.

After, publishing the gem, we can add it to any Ruby project by including it in
the Gemfile and installing the bundle with `bundle install`. After the install
completes, one other thing is to add it to the `test_helper.rb` file:

```ruby
# test/test_helper.rb
require "minitest/autorun"
require "minitest/unit"

require "minitest/clr"
# Other stuff here..
```

If everything went well, when you run the tests you can see the final test
results in color.

Note: You can see my implementation of `minitest-clr`
[here](https://github.com/fteem/minitest-clr).

