# Obeying Sandi Metz' four rules

## The Four Rules

Some time ago, the author of [POODR](http://poodr.com), Sandi Metz, introduced
[four rules for developers](https://robots.thoughtbot.com/sandi-metz-rules-for-developers).
The rules stated that:

- Classes can be no longer than one hundred lines of code.
- Methods can be no longer than five lines of code.
- Pass no more than four parameters into a method. Hash options are parameters.
- Controllers can instantiate only one object. Therefore, views can only know
  about one instance variable and views should only send messages to that object.

Although some of the people in the Ruby community found them to be conservative,
the rules got quite popular and people started using them in their day-to-day
work.

In this chapter we will build a Minitest plugin which will check if the tests
that are being run obey Sandi's four rules. We will call it `Minitest::Metz`.

## Static analysis

These type of metric, which are interested in the shape of the code are done
using static analysis. According to [Wikipedia](https://en.wikipedia.org/wiki/Static_program_analysis):

> Static program analysis is the analysis of computer software that is performed
> without actually executing programs (analysis performed on executing programs
> is known as dynamic analysis). In most cases the analysis is performed on
> some version of the source code, and in the other cases, some form of the
> object code.

As you can see, the four rules are not interested in how the program runs, but
in it's structure. That's why, we need a tool that can run this static analysis
on our tests and inform us of the structure of our code.

Fortunately for us, a tool that runs this analysis already exists and it's called
[SandiMeter](https://github.com/makaroni4/sandi_meter). It scans the code and
returns results based on the four rules which we can use on our tests. We will
use this tool in our Minitest plugin, because it provides a nice way of
analyzing our code with descriptive results.

### SandiMeter in action

To get the whole picture of how the SandiMeter works, let's say we have a class
that looks like this:

```ruby
# person.rb
class Person
  def initialize(first_name, last_name, age, sex, weight, height)
    @first_name = first_name
    @last_name = last_name
    @age = age
    @sex = sex
    @weight = weight
    @height = height
  end
end

Person.new("Ilija", "Eftimov", 25, :male, 90, 190)
```
Now, via the command line, we want to check if this class obeys the four rules.
Using the SandiMeter, which is a command line tool, we can run:

```bash
sandi_meter -d person.rb
```

This will produce an output like this:

```bash
1. 100% of classes are under 100 lines.
2. 0% of methods are under 5 lines.
3. No method calls to analyze.
4. No controllers to analyze.

Methods with 5+ lines
  Class name  | Method name  | Size  | Path
  Person      | initialize   | 6     | ./person.rb
```

As you can see, the SandiMeter gem tells us of any rule violations that our
`person.rb` file has, which in this case is the length of the `initialize` method.
Also, the SandiMeter can be used as a library, which is what we are aiming for:

```ruby
require 'sandi_meter/file_scanner'
require 'pp'

scanner = SandiMeter::FileScanner.new
data = scanner.scan(PATH_TO_PROJECT)
require 'sandi_meter/file_scanner'
require 'pp'

scanner = SandiMeter::FileScanner.new
data = scanner.scan(PATH_TO_PROJECT)
pp data
# {:first_rule=>
#   {:small_classes_amount=>916,
#    :total_classes_amount=>937,
#    :misindented_classes_amount=>1},
#  :second_rule=>
#   {:small_methods_amount=>1144,
#    :total_methods_amount=>1833,
#    :misindented_methods_amount=>0},
#  :third_rule=>{:proper_method_calls=>5857, :total_method_calls=>5894},
#  :fourth_rule=>{:proper_controllers_amount=>17, :total_controllers_amount=>94}}
```

Note: Example taken from SandiMeter's
[README](https://github.com/makaroni4/sandi_meter/blob/master/README.md).

Now that we have the SandiMeter usage under our belts, let's move on with
building our Minitest plugin.

{ pagebreak }

## Building the plugin

As usual, we start off by generating a new gem, with the
`bundle gem minitest-metz --test=minitest` command:

```bash
Creating gem 'minitest-metz'...
      create  minitest-metz/Gemfile
      create  minitest-metz/.gitignore
      create  minitest-metz/lib/minitest/metz.rb
      create  minitest-metz/lib/minitest/metz/version.rb
      create  minitest-metz/minitest-metz.gemspec
      create  minitest-metz/Rakefile
      create  minitest-metz/README.md
      create  minitest-metz/bin/console
      create  minitest-metz/bin/setup
      create  minitest-metz/.travis.yml
      create  minitest-metz/test/test_helper.rb
      create  minitest-metz/test/minitest/metz_test.rb
Initializing git repo in /Users/ie/dev/minitest-metz
```

As a reminder, the `--test=minitest` flag changes the default testing framework
of the gem, from RSpec to Minitest.

### Adding dependencies
Let's open the `minitest-metz.gemspec` file and declare the SandiMeter as a
runtime dependency:

{ pagebreak }

```ruby
# coding: utf-8
lib = File.expand_path('../lib', __FILE__)
$LOAD_PATH.unshift(lib) unless $LOAD_PATH.include?(lib)
require 'minitest/metz/version'

Gem::Specification.new do |spec|
  spec.name          = "minitest-metz"
  spec.version       = Minitest::Metz::VERSION
  spec.authors       = ["Ile Eftimov"]
  spec.email         = ["ileeftimov@gmail.com"]

  spec.summary       = %q{Make sure your code (production and tests) obey Sandi
                          Metz' four rules for developers.}
  spec.description   = %q{Minitest::Metz is a Minitest plugin that under it's
                          hood hides the SandiMeter. It allows you to easily
                          apply Sandi Metz's four rules for developers on your
                          tests.}
  spec.homepage      = "https://github.com/fteem/minitest-metz"

  spec.files         = `git ls-files -z`.split("\x0").reject { |f| f.match(%r{^(test|spec|features)/}) }
  spec.bindir        = "exe"
  spec.executables   = spec.files.grep(%r{^exe/}) { |f| File.basename(f) }
  spec.require_paths = ["lib"]

  spec.add_dependency "minitest", "~> 5.8"
  spec.add_dependency "sandi_meter", "~> 1.2"

  spec.add_development_dependency "bundler", "~> 1.10"
  spec.add_development_dependency "rake", "~> 10.0"
end
```

As you can see, because this is a Minitest plugin which uses the SandiMeter gem,
we will have only two runtime dependencies: Minitest and SandiMeter. The rest
of the gemspec file is quite straightforward - a summary and description of the
gem, the github repo and the rest is boilerplate stuff.

{ pagebreak }

### Adding the plugin file

The workings of our plugin will resemble what we have already seen in the earlier
chapters. We will add a command line swtich, `--metz` which can be used to
enable the SandiMeter. Let us add the `lib/minitest/metz_plugin.rb` file:

```ruby
module Minitest
  def self.plugin_metz_options(opts, options)
    opts.on "-z", "--metz", "Check if your code obeys Sandi Metz' four rules for developers" do |z|
      options[:metz] = z
    end
  end

  def self.plugin_metz_init(options)
    if options[:metz]
      self.reporter << Minitest::Metz::StatsReporter.new
    end
  end
end
```

The first method just checks if the `-z` or the `--metz` switch is present, which
updates the `options[:metz]` hash. The second method, `plugin_metz_init`, checks
if the plugin is enabled and appends the `Minitest::Metz::StatsReporter` to
Minitest's reporters array if needed. Pretty straightforward, no magic.

### Scanning

Before building the `Minitest::Metz::StatsReporter` we will take a look at how
we can do the scanning in our library. The first step will be to create a
wrapper class for the SandiMeter's FileScanner class, called
`Minitest::Metz::Scanner`, which should return the results of the scanning.
To make any sense of the results, we will create an additional class, called
`Minitest::Metz::ScanResults`, which will contain wrap the scanning
results with useful methods.

{ pagebreak }

```ruby
require "sandi_meter/file_scanner"

module Minitest
  module Metz
    class Scanner
      def self.scan(file_path)
        new.scan(file_path)
      end

      def initialize
        @scanner = SandiMeter::FileScanner.new
      end

      def scan(file_path)
        r = @scanner.scan(file_path)
        Minitest::Metz::ScanResults.new(r)
      end

    end
  end
end
```

As you can notice, the `Scanner` class is quite simple - it just takes the
file path, scans the document and returns a `Minitest::Metz::ScanResults` object.

{ pagebreak }

```ruby
module Minitest
  module Metz

    class ScanResults
      attr_reader :first_rule_violation, :misindentation_violation,
                  :second_rule_violation, :third_rule_violation
      def initialize(r)
        @first_rule_violation     = r[:first_rule][:total_classes_amount] - r[:first_rule][:small_classes_amount]
        @misindentation_violation = r[:first_rule][:misindented_classes_amount]
        @second_rule_violation    = r[:second_rule][:total_methods_amount] - r[:second_rule][:small_methods_amount]
        @third_rule_violation     = r[:third_rule][:total_method_calls] - r[:third_rule][:proper_method_calls]
      end

      def all_valid?
        first_rule_valid? && misindentation_valid? && second_rule_valid? && third_rule_valid?
      end

      def first_rule_valid?
        first_rule_violation.zero?
      end

      def misindentation_valid?
        misindentation_violation.zero?
      end

      def second_rule_valid?
        second_rule_violation.zero?
      end

      def third_rule_valid?
        third_rule_violation.zero?
      end
    end

  end
end
```

Although the `ScanResults` class is quite longer, it is just as simple as the
`Scanner` class. In the `initialize` method it takes the scanning results hash
and looks for any rules violations in the results. To hide away all of the
complexity of the results hash, we add the `*_valid?` methods, so we can easily
check if there are any rule violations.

### Building the reporter

As we saw, we will need the `StatsReporter`. This class will be used to check
each test file that is being run with the SandiMeter and report if any of the
test classes do not obey to the four rules.

```ruby
# lib/minitest/metz/stats_reporter.rb

class Minitest::Metz::StatsReporter < Minitest::Reporter
  def initialize
    @results = {}
  end

  def record(result)
    file_path, = result.class.instance_method(result.name).source_location
    unless @results[file_path]
      scan_result = Minitest::Metz::Scanner.scan(file_path)
      @results[file_path] = build_results_string(file_path, scan_result)
    end
  end

  def report
    @results.each do |key, output|
      puts output
    end
  end

  private

  def build_results_string(file_path, result)
    str = "\nSandi Meter Rules results:"
    if result.all_valid?
      str << " All valid."
    else
      str << "\n#{file_path}"
      str << "\n  #{result.first_rule_violation} class(es) over 100 lines." unless result.first_rule_valid?
      str << "\n  #{result.misidentation_violation} misindented class(es)." unless result.misidentation_valid?
      str << "\n  #{result.second_rule_violation} method(s) over 5 lines." unless result.second_rule_valid?
      str << "\n  #{result.third_rule_violation} method call(s) accepted are more than 4 parameters." unless result.third_rule_valid?
    end

    str
  end
end
```

Now, there's a bit going on here, so stick with me. In the initializer, we just
create a hash called `@results` which will hold all of our scanning results.
As you can notice, most of the magic happens in the `record` and
`build_results_string` method.

The `record` method is run after each test case is being run. The first line of
the `record` method, looks for the file_path of the test that is being run.
Now, if that path has never been scanned, which means that it is not present in
the `@results` hash, we use the `Minitest::Metz::Scanner` class to scan the file.
The results of the scanning are formatted nicely in the `build_results_string`
method and added to the `@results` hash, with the file path as a key.

An example output of this reporter looks like this, if all of the rules are
obeyed:

```bash
Run options: --metz --seed 31668

# Running:

.

Finished in 0.105939s, 9.4394 runs/s, 9.4394 assertions/s.

1 runs, 1 assertions, 0 failures, 0 errors, 0 skips

Sandi Meter Rules results: All valid.
```

If there's a rule broken, the output can vary:

```bash
Run options: --metz --seed 31668

# Running:

.

Finished in 0.105939s, 9.4394 runs/s, 9.4394 assertions/s.

1 runs, 1 assertions, 0 failures, 0 errors, 0 skips

Sandi Meter Rules results:
test/minitest/some_test.rb
  1 misindented class(es).
  1 class(es) over 100 lines.
```

