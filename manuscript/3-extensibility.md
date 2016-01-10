# Extensibility

Minitest by design accepts plugins - classes that follow a certain shape and
rules. Basically, Ryan Davis added an expected interface in the Minitest
sourcecode that every extension implementer must follow.

Defining a plugin in Minitest is super simple - just create a file named
`minitest/name_of_plugin.rb` to the project/gem that you are working on. One
thing to keep in mind is that the file must be discoverable via Ruby's
`LOAD_PATH`.

In the first paragraph, we said that each plugin is a class that follows a
certain interface. This is due to Minitest's behaviour. At runtime, it expects
all plugins to have methods named in a convention. When Minitest is starting up,
it will try to call the `plugin_<name-of-plugin-here>_init` method. The option
processor, which handles the command line options, will also try to call the
```plugin_<name-of-plugin-here>_options``` method passing the `OptionParser`
instance and the current options hash.

A very tiny example of a Minitest plugin:

```ruby
# minitest/bogus_plugin.rb:

module Minitest
  def self.plugin_bogus_options(opts, options)
    opts.on "--myci", "Report results to my CI" do
      options[:myci] = true
      options[:myci_addr] = get_myci_addr
      options[:myci_port] = get_myci_port
    end
  end

  def self.plugin_bogus_init(options)
    self.reporter << MyCI.new(options) if options[:myci]
  end
end
```

(Note: Example blatantly taken from Minitest's
[README](https://github.com/seattlerb/minitest#writing-extensions))

Having said that, let's write our first, very simple, Minitest Extension.
