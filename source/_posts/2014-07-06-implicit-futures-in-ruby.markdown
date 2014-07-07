---
layout: post
title: "Implicit Futures in Ruby"
date: 2014-07-06 02:20:05 +0930
comments: true
categories: [concurrency, ruby, futures]
---

Say we're writing a program that performs several time-consuming operations, like network requests, disk accesses, or complex calculations. We want them to execute concurrently, but we also want our code to remain simple and easy to understand. There are many ways of approaching this problem, but in this post I'll focus on *implicit futures* and how they can be implemented in Ruby.

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

Neat! How on Earth do we implement this though...? At first glance, it might seem like we'd need to build it into the language itself. As we'll see though, it's actually quite easy to implement in a dynamic language like Ruby.

Ruby's thread class
-------------------

It turns out that Ruby's `Thread` class can do a lot of the work for us. Here's a quick primer on it:

``` ruby
t = Thread.new { 2 + 2 }
t.value # => 4
```

`Thread.new` spawns a thread to execute the given code block, and `t.value` returns the thread's result (waiting for the it to finish running if necessary).

Let's try rewriting our original code using `Thread`:

``` ruby
a = Thread.new { foo.expensive_call_1 }
b = Thread.new { bar.expensive_call_2 }
c = Thread.new { baz.expensive_call_3 }

other_stuff

puts a.value + (b.value * c.value)
```

This *almost* gets us where we want to be. 

Explicit vs implicit
--------------------

In the code above, we have to explicitly retrieve the results by calling `value` on `a`, `b`, and `c`. This isn't quite what we're after - these are *explicit* futures rather than *implicit* ones.

This mightn't seem like much of an issue, but consider what would happen if we wanted to pass the results along to other pieces of code. We could either:

2. Call `value` and pass the final result along.
1. Pass along the future object, and let the other piece of code call `value` when it needs the final result.

Neither of these options are very good. Option 1 can result in suboptimal performance, because we're calling `value` before we really need to (remember that `value` might block execution if the result isn't ready yet).

Option 2 is also bad, because it limits the reusability of the other piece of code (which would now only be able to work with future objects).

Implicit futures don't have either of those issues. The code that uses them can remain blissfully ignorant of what they are, and no blocking can occur until a method is actually called on the result object.

Method missing
--------------

To summarise what we're trying to achieve, we want the future object to appear as though it were the final result. When we call a method on it, it should delegate the call to the result object (blocking if necessary until the result becomes available).

It's not immediately obvious how we can delegate methods like this. The future should be able to work with all types of result objects, so we don't know in advance which methods need forwarding. Luckily, Ruby lets us intercept calls to undefined methods. If we define a method called `method_missing`, Ruby will call it if the object can't otherwise respond to the method.

Here's a quick demo:

``` ruby
class Foo
  def method_missing(name, *args, &block)
    puts "Someone called the '#{name}' method on me!"
    puts "I was given arguments: #{args}"
    puts "Now let's call the block I was given..."
    block.call
  end
end

well = Foo.new
well.this_is_weird!("yup") { puts "I don't even..." }
```

If we run this, we get the following output:

``` plain
Someone called the 'well_this_is_weird!' method on me!
I was given arguments: ["yup"]
Now let's call the block I was given...
I don't even...
```

Implementing implicit futures
-----------------------------

We've now got everything we need to implement a `Future` class that:

1. Uses a `Thread` to asynchronously execute the computation.
2. Uses `method_missing` to transparently forward all methods calls to the thread's result.

The code actually ends up being very simple:

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

To make it slightly easier to use, we'll define a standalone `future` function that creates and returns a `Future` object:

``` ruby
def future(&block)
  Future.new(&block)
end
```

We can now create future objects as follows:

``` ruby
f = future { expensive_call }
```

This looks exactly like what we set out to achieve. Brilliant! Might be a good idea to check if it works though...

Testing time!
-------------

Let's start with a sanity-check:

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

Now let's try a real-world example. We'll download a bunch of webpages and record how long it took, with and without futures:

``` ruby
require 'benchmark'
require 'open-uri'

urls = [
  "https://www.google.com", "https://www.microsoft.com", "http://www.abc.net.au",
  "http://www.reddit.com", "https://www.atlassian.com", "https://duckduckgo.com",
  "https://www.facebook.com", "https://www.ruby-lang.org", "http://whirlpool.net.au",
  "https://bitbucket.org", "https://github.com", "https://news.ycombinator.com"
]

def with_futures(urls)
  pages = urls.map do |url|
    future { open(url) { |f| f.read } }
  end
  pages.map { |page| page.length }
end

def without_futures(urls)
  pages = urls.map do |url|
    open(url) { |f| f.read }
  end
  pages.map { |page| page.length }
end

Benchmark.bm do |b|
  b.report { with_futures(urls) }
  b.report { without_futures(urls) }
end
```

The results:

``` plain
       user     system      total        real
   0.160000   0.040000   0.200000 (  2.561927)
   0.110000   0.030000   0.140000 ( 13.774315)
```

13.8 seconds down to 2.6 seconds. Nice!

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
