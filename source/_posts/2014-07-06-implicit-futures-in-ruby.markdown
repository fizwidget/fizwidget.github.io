---
layout: post
title: "Implicit Futures in Ruby"
date: 2014-07-06 02:20:05 +0930
comments: true
categories: [concurrency, ruby, futures]
---

Say we're writing a program that performs several time-consuming operations, like network requests, disk accesses, or complex calculations. We want them to execute concurrently, but we also want our code to remain simple and easy to understand. There are many ways of approaching this problem, but in this post I'll focus on *implicit futures* and they can be implemented in Ruby.

<!-- more -->

This snippet of code demonstrates the scenario we're dealing with:

``` ruby
a = foo.expensive_call_1
b = bar.expensive_call_2
c = baz.expensive_call_3

other_stuff

result = a + (b * c)
```

We perform one or more time-consuming operations, potentially do some other things, then use the results of the operations.

In the code above, the expensive calls are executed sequentially. This can lead to *very* poor performance - for example, the CPU might be forced to sit completely idle while a network request completes. Additionally, the call to `other_stuff` won't begin running until all three of the previous calls have finished, even though it doesn't depend on their results.

This is the solution we're working towards:

``` ruby
a = future { foo.expensive_call_1 }
b = future { bar.expensive_call_2 }
c = future { baz.expensive_call_3 }

other_stuff

result = a + (b * c)
```

(*If you're unfamiliar with Ruby, `foo { bar }` is equivalent to `foo(lambda: bar())` in Python*)

The expensive calls are now executed asynchronously, and `other_stuff` can begin running immediately. If line 7 is reached before all three results have been calculated, the main thread will automatically block and wait for them to finish.

Neat! How on Earth do we implement this though...? At first glance, it might seem like we'd need to built it into the language itself. As we'll see though, it's actually quite easy to implement in dynamic languages like Ruby and Python.

Ruby's thread class
-------------------

It turns out that Ruby's `Thread` class does a lot of the work for us. Here's a quick primer on it:

``` ruby
t = Thread.new { 2 + 2 }
t.value # => 4
```

`Thread.new` spawns a thread to execute the code block, and `t.value` returns the thread's result (waiting for the thread to finish running if necessary).

This *almost* gets us where we want to be. Let's rewrite our original code using the `Thread` class:

``` ruby
a = Thread.new { foo.expensive_call_1 }
b = Thread.new { bar.expensive_call_2 }
c = Thread.new { baz.expensive_call_3 }

other_stuff

puts a.value + (b.value * c.value)
```

Notice that we have to explicitly retrieve the results by calling the `value` method on `a`, `b`, and `c`. This isn't quite what we're after - they're *explicit* futures rather than *implicit* ones.

This mightn't seem like much of an issue...calling `value` isn't exactly difficult. Consider this though: what if we wanted to pass `a`, `b`, or `c` into other functions or methods, or return them as results? Things would start getting messy, because every piece of code that touches them would need to know that they're explicit futures.

This is why implicit futures are so nice - the code that uses them can remain blissfully ignorant of the fact that they're futures. If `a` were an implicit future representing a numeric result, we could freely pass it around to any piece of code that works with numbers.

Transparent proxy objects
-------------------------

To summarise what we're trying to achieve, we want to be able to treat `a` as though it were the result of the computation:

``` ruby
a = future { some_computation }
```

We don't want to have to explicitly retrieve the result by calling `value`.

To achieve this, we want `future` to return a *transparent proxy object*. A transparent proxy is a wrapper around an object that forwards all method calls to the object it's wrapping. Looking at it from the outside, it appears identical to the wrapped object.

It's not immediately obvious how we can achieve this, but it actually ends up being very easy in Ruby. Ruby allows us to define a special method called `method_missing`, which gets called automatically if an object can't otherwise respond to a method.

Here's a quick demo:

``` ruby
class Useless
  def method_missing(name, *args, &block)
    puts "Someone called the '#{name}' method on me!"
    puts "I was given arguments: #{args}"
    puts "Now let's call the block I was given..."
    block.call
  end
end

well = Useless.new
well.this_is_weird!("yup") { puts "I don't even" }
```

Running this code produces the following output:

``` plain
Someone called the 'well_this_is_weird!' method on me!
I was given arguments: ["yup"]
Now let's call the block I was given...
I don't even
```

We can use `method_missing` to make a transparent proxy:

``` ruby
class TransparentProxy
  def initialize(target)
    @target = target
  end

  def method_missing(name, *args, &block)
    @target.send(name, *args, &block)
  end
end

proxy = TransparentProxy.new("I'm being wrapped by the proxy!")
proxy.reverse # => "!yxorp eht yb depparw gnieb m'I"
```

We're able to treat `proxy` as though it were a string. The call to `reverse` was intercepted by `method_missing`, then forwarded to the wrapped object.

Implementing implicit futures
-----------------------------

Using the building blocks described above, we can write a `Future` class that:

1. Uses a `Thread` to:
    - Asynchronously execute a block of code.
    - Block execution if we attempt to use the result before it's available.
2. Acts as a transparent proxy, forwarding all methods calls to the thread's result.

The code for this ends up being surprisingly simple:

``` ruby
class Future
  def initialize(&block)
    @thread = Thread.new { block.call }
  end
  
  def method_missing(name, *args, &block)
    @thread.value.send(name, *args, &block)
  end
end
```

To make it slightly easier to use, we'll define standalone `future` function that creates and returns a `Future` object:

``` ruby
def future(&block)
  Future.new(&block)
end
```

We can then use it as follows:

``` ruby
f = future { expensive_call }
```

This looks exactly like what we set out to achieve - yay! Might be a good idea to check if it works though...

Testing time!
-------------

Let's start with a sanity-check...

``` ruby
f = future do
  puts "Future: Started."
  sleep(4)
  puts "Future: Finished."
  "yeeeeaaaaaaaahh"
end

sleep(2)

puts "Attempting to use the result..."
puts f.upcase
```

The future should begin executing immediately, and the main thread should be forced to wait couple of seconds before it can access the result. Here's the output we get:

``` plain
Future: Started.
Attempting to use the result...
Future: Finished.
YEEEEAAAAAAAAHH
```

It worked!

Now let's try a real-world example. We'll download a bunch of webpages and report how long it took, with and without futures:

``` ruby
require 'benchmark'
require 'open-uri'

urls = [
  "https://www.google.com", "https://www.microsoft.com", "http://www.abc.net.au",
  "http://www.reddit.com", "https://www.atlassian.com", "https://www.duckduckgo.com",
  "https://www.facebook.com", "https://www.ruby-lang.org", "http://whirlpool.net.au"
]

Benchmark.bm do |b|
  # Without futures.
  b.report do
    pages = urls.map do |url|
      open(url) { |f| f.read }
    end
    pages.map { |page| page.length }
  end

  # With futures.
  b.report do
    pages = urls.map do |url|
      future { open(url) { |f| f.read } }
    end
    pages.map { |page| page.length }
  end
end

```

The results:

``` plain
       user     system      total        real
   0.120000   0.040000   0.160000 ( 11.494227)
   0.070000   0.030000   0.100000 (  2.793592)
```

Using futures resulted in a big speed-up (11.5 seconds down to 2.8 seconds). Great!

Caveats
-------

The above code isn't exactly production-ready. If you were using this in a real project, you'd want the `Future` class to:

- Capture exceptions that occur during the computation and re-throw them when the result is accessed.
- Inherit from `BasicObject` instead of `Object` (so methods defined on `Object` are forwarded to the result).
- Use a thread-pool to prevent the system from being overloaded by a large number of threads.
- Probably a bunch of other things I haven't thought of.

Conclusion
----------

Futures are awesome. Ruby is awesome. That is all.
