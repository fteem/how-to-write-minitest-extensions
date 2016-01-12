# Easy to Share Failures

## The idea

In this chapter, I will show you how to write a more complicated Minitest
extension. The idea behind the extension is that usually, when we have some tests
that we have trouble fixing, we need to consult with a colleague. Well, there's
no better way of sharing some code (or, in this case, a stacktrace) with a
colleague then Pastebin, or a similar service.

For this chapter, we will use [DPaste](dpaste.com), because it has a free and
open API, which requires no API keys and/or configuration.

The plugin will enable the `--paste` switch, which will collect all of the
failing tests stacktraces and upload them to a new DPaste entry. In addition,
it will add the DPaste link to you clipboard, so you can just paste the link
into your colleague's chat. Nifty, eh?

For this plugin, we will use the [Clipboard](https://github.com/janlelis/clipboard)
gem which will help us with adding the DPaste link to the clipboard. To
communicate, with the DPaste API, we will write a dumb Ruby wrapper.

## The API wrapper

The DPaste API wrapper will be quite contrived. We are only interested in saving
the content of the failures and uploading them to the site. In return, we expect
to get a link, that we can save to the clipboard.

{ pagebreak }

First, let's write the API wrapper:

```ruby
# lib/dpaste/api.rb

require 'net/http'

module DPaste
  class API
    API_URL = "http://dpaste.com/api/v2/"

    def self.save(string)
      uri = URI(API_URL)
      response = Net::HTTP.post_form(uri, expiry_days: "10", syntax: "rb", content: string)
      response.body.strip
    end
  end
end
```

As you can see, the example is quite simple. The `DPaste::API.save` method takes
a string, builds the POST form and sends it to the `API_URL`. In return, it gets
the response body, which is the dpaste link where our string got saved, and
strips it of any special characters.

Nothing fancy, just a plain Ruby class with magic from `net/http`. That's all.

## Adding the plugin file

Because we now know why and how we can work with plugins in development, this
time we will add our code directly to a gem. Let's generate one:

```bash
bundle gem minitest-paste --test=minitest
```

The `--test=minitest` flag will setup Minitest as the testing framework for the
gem, intead of the default (RSpec). When we run the command, we can see the
output:

```bash
Creating gem 'minitest-paste'...
      create  minitest-paste/Gemfile
      create  minitest-paste/.gitignore
      create  minitest-paste/lib/minitest/paste.rb
      create  minitest-paste/lib/minitest/paste/version.rb
      create  minitest-paste/minitest-paste.gemspec
      create  minitest-paste/Rakefile
      create  minitest-paste/README.md
      create  minitest-paste/bin/console
      create  minitest-paste/bin/setup
      create  minitest-paste/.travis.yml
      create  minitest-paste/test/test_helper.rb
      create  minitest-paste/test/minitest/paste_test.rb
Initializing git repo in /Users/ie/dev/minitest-paste
```

In our skeleton, we will add the `lib/minitest/paste_plugin.rb` file, which is
where our plugin configuration and initialization will be done.

### Plugin configuration

At the beginning of this chapter, we agreed that the command line switch which
will enable this plugin will be `--paste`. Let us make our plugin react to this
switch:

```ruby
# lib/minitest/paste_plugin.rb

module Minitest
  def self.plugin_paste_options(opts, options)
    opts.on "--paste", "Upload failures to (a service like) Pastebin.com" do
      options[:paste] = true
    end
  end

  def self.plugin_paste_init(options)
    self.reporter << PasteReporter.new(options) if options[:paste]
  end
end
```

The `Minitest.plugin_paste_options` does not do much. It just sets the `paste`
flag to `true` in the `options` hash. The `Minitest.plugin_paste_init` does a bit
more - it adds a new reporter to Minitest, called `PasteReporter`. This is the
class that will capture the output of the failed tests, compile them into one
file (or string) and upload it to DPaste.

## Writing the reporter

Reporters in Minitest are quite simple to write, because of good documentation
and good decisions made by the author. Minitest reporters use the
[composite pattern](https://sourcemaking.com/design_patterns/composite), which
enables plugging in additional reporters to Minitest, as long as they follow
a certain class design.

Minitest, has an abstract class called `AbstractReporter` which defines the
API for the reporters. This means that, it's a good practice, to subclass the
`AbstractReporter` when building custom reporters while overriding any of the
methods whose behaviour we want to customize. The `AbstractReporter` class can
be seen in the
[Minitest source code](https://github.com/seattlerb/minitest/blob/master/lib/minitest.rb#L406).

There are other reporter composite classes that we can work with, namely the
[Statistics Reporter](https://github.com/seattlerb/minitest/blob/master/lib/minitest.rb#L479)
and the
[Summary Reporter](https://github.com/seattlerb/minitest/blob/master/lib/minitest.rb#L539).
These two reporters are added onto the composite reporter list by default.

The first one, `StatisticsReporter`, is a reporter that gathers statistics about
a test run. I/O is not it's purpose, it only gathers statistics. The idea behind
it is to represent a parent class, so other reporters can use it and the data it
gathers.

The second one, `SummaryReporter`, is a reporter that prints the header,
summary, and failure details at the end of the run. If we want to change the
output of this reporter, we need to pull it out of the composite reporter and
add a plugin of our own (which we did in Chapter 3).

Our reporter, called `PasteReporter`, does not needs to wrap the I/O stream,
because we are interested only in collecting the test errors and sending them to
dpaste. If our reporter's intention was to manipulate the output, in that case
we would have needed to work with the I/O stream, which usually is `STDOUT`.

Because we do not want to complicate things by subclassing the `AbstractReporter`,
for our `PasteReporter` we will subclass the `StatisticsReporter`. The only
methods whose behaviour we will need to modify are the `initializer`, the `record`
method and the `report` method.

```ruby
module Minitest
  class PasteReporter < StatisticsReporter

    def initialize opts
      super
      @failures = []
    end

    def record result
      super

      if result.failures.size > 0
        result.failures.each {|f| @failures << f.message }
      end
    end

    def report
      super

      # Send @failures to dpaste.net...

    end
  end
end
```

In the `initialize` method we call `super` and we create a new array called
`@failures`. In this array we will store all of the failed test runs which, at the
end, will be sent to dpaste.

The `record` method is run after every test run. It will check if the result of
the test run has any failures. If it does, it will add all of the failure
messages to the `@failures` array.

Now, when it comes to the `report` method, it will take a bit of work.
