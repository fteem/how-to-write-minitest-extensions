# Easy to Share Failures

## The idea

In this chapter, I will show you how to write a more complicated Minitest
extension. The idea behind the extension is that usually, when we have some tests
that we have trouble fixing, we need to consult with a colleague. Well, there's
no better way of sharing some code (or, in this case, a stacktrace) with a
colleague then dpaste.com.

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

