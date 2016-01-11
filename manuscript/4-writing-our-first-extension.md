# Our first Minitest Extension

In this chapter, I will take you through the journey of building a really tiny
Minitest extension. For the purpose of this chapter, we will build a gem that
applies colors to the final test results of a Minitest suite.

This is what the our end result should look like:

![alt text](images/clr-end-result.png)

At the end of this section, you will know how to build a small Minitest
extension, packed in a gem. We will also learn about writing and loading plugins
into the Minitest test run.

## Setting up the plugin

We will call this plugin `minitest-clr`, an acronym of *Minitest Color Results*.
First, open up your terminal and type:

```bash
bundle gem minitest-clr
```

This command will generate a skeleton for a gem, which will allow us to easily
work with our extension and distribute it.

