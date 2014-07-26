---
layout: post
title: "Implicit Futures in Ruby"
date: 2014-07-06 02:20:05 +0930
comments: true
categories: [concurrency, ruby, futures, threads]
---

Say we're writing a program that performs several time-consuming operations, like network requests, disk accesses, or complex calculations. We want them to execute concurrently, but we also want our code to remain simple and easy to understand. There are many different ways of approaching this problem, but in this post I'll focus on *implicit futures* and how they can be implemented in Ruby.

<!-- more -->

This code demonstrates the type of scenario we're dealing with:

``` ruby
a = foo.expensive_call_1
b = bar.expensive_call_2
c = baz.expensive_call_3

other_stuff

result = a.some_method(b + c)
```

We perform one or more time-consuming operations, potentially do something else, then use the results of the operations.

In the code above, the time-consuming operations are executed sequentially. This can result in very poor performance (e.g. CPU might be forced to sit idle while a network request completes). Also, the call to `other_stuff` won't begin running until all three of the previous calls have finished, even though it doesn't depend on their results.

This is the solution we're working towards:

``` ruby
a = future { foo.expensive_call_1 }
b = future { bar.expensive_call_2 }
c = future { baz.expensive_call_3 }

other_stuff

result = a.some_method(b + c)
```

(*If you're unfamiliar with Ruby, `foo { bar }` is equivalent to `foo(lambda: bar())` in Python*)

The expensive calls are now executed asynchronously, and `other_stuff` can begin running immediately. If line 7 is reached before all three results are available, the main thread will automatically block and wait for them to finish.

Nice! How on Earth can this be implemented though...? At first glance, it looks like we'd need to modify the language itself. As we'll see, there's actually a much simpler approach.

Threads in Ruby
---------------

We'll use Ruby's `Thread` class to do most of the heavy-lifting. Here's a quick primer on it:

``` ruby
t = Thread.new { 2 + 2 }
t.value # => 4
```

`Thread.new` spawns a thread to execute the given code block, and `t.value` returns the thread's result (waiting for it to finish if necessary).

Let's rewrite our original code to use `Thread`:

``` ruby
a = Thread.new { foo.expensive_call_1 }
b = Thread.new { bar.expensive_call_2 }
c = Thread.new { baz.expensive_call_3 }

other_stuff

result = a.value.some_method(b.value + c.value)
```

This almost gets us where we want to be, but not quite.

Explicit vs implicit
--------------------

In the code above, we have to explicitly retrieve the results by calling `value` on the futures/threads. This means they're *explicit* futures rather than *implicit* ones.

This might not seem like much of an issue, but what if we wanted to pass one of the results on to another piece of code? We could either:

1. Call `value` on the future, and pass along the result directly.
2. Pass along the future, and let the other piece of code call `value` when it needs the result.

Neither of these options are very good. The first can result in suboptimal performance, because we're calling `value` before we really need to (remember that `value` might block execution if the result isn't ready yet). The second option limits the reusability of the other piece of code, as it will now only be able to work with futures.

Implicit futures don't have either of these issues. The code that uses them can remain blissfully ignorant of the fact that they're futures, and no blocking will occur until the result is actually needed (i.e. a method is called on it).

Delegating method calls
-----------------------

To summarise what we're trying to achieve, we want the future object to behave as though it were the result object. When we call a method on it, it should delegate the call to the result (blocking if necessary until the result becomes available).

It's not immediately obvious how methods can be delegated like this. The future should be able to work with all types of result objects, so we don't know in advance which methods need forwarding.

Ruby has a rather interesting feature called `method_missing` that comes in handy here. Calling a non-existent method would normally result in an error, but if we define a method called `method_missing`, Ruby will call that instead.

Here's a quick demo:

``` ruby
class Useless
  def method_missing(method_name, *args, &block)
    puts "Someone called '#{method_name}' on me."
    puts "I was given arguments: #{args}"
    puts "Now let's call the block I was given:"
    block.call
  end
end

u = Useless.new
u.this_is_weird!("yup") { puts "I don't even..." }
```

This gives the following output:

``` plain
Someone called 'this_is_weird!' on me.
I was given arguments: ["yup"]
Now let's call the block I was given:
I don't even...
```

Our future class can use `method_missing` to intercept method calls. However, it still needs to actually forward the intercepted methods to the result. This can be done using `send`, which lets us dynamically call any method on an object:

``` ruby
some_object.send(method_name, args, &block)
```

Putting it all together
-----------------------

We now have everything we need to implement implicit futures. We can define a `Future` class that:

1. Uses `Thread` to:
    - Asynchronously compute the result.
    - Block execution if the result is requested before it's ready.
2. Uses `method_missing` and `send` to delegate all method calls to the result.

The code ends up being very simple:

``` ruby
class Future
  def initialize(&block)
    @thread = Thread.new { block.call }
  end
  
  def method_missing(method_name, *args, &block)
    result = @thread.value
    result.send(method_name, *args, &block)
  end
end
```

When a future is created, a thread is immediately spawned to execute the given code block. When a method is called on the future, it simply forwards it to `@thread.value` (which will block execution if the thread hasn't finished running yet).

To make it slightly easier to use, we'll define a standalone `future` function that creates and returns a `Future` object:

``` ruby
def future(&block)
  Future.new(&block)
end
```

We can now create futures like this:

``` ruby
f = future { expensive_call }
```

This looks exactly like what we set out to achieve! Might be a good idea to check if it works though...

Testing!
--------

Let's start with a quick sanity-check. To mimic an expensive operation, I've placed a call to `sleep` in the code given to the future:

``` ruby
f = future do
  puts "Future: Started."
  sleep(4)
  puts "Future: Finished."
  "I'm the result!"
end

sleep(2)

puts "Attempting to use the result."
puts f.upcase
```

The call to `f.upcase` should cause the main thread to block for several seconds until the result is ready. If we run the code, this is exactly what we see happen. Here's the output:

``` plain
Future: Started.
Attempting to use the result.
Future: Finished.
I'M THE RESULT!
```

It worked!

Now let's try more realistic example. We'll download a bunch of webpages and record how long it takes, with and without futures:

``` ruby
require 'benchmark'
require 'open-uri'

urls = [
  "https://www.google.com", "https://www.microsoft.com", "http://www.abc.net.au",
  "http://www.reddit.com", "https://www.atlassian.com", "https://duckduckgo.com",
  "https://www.facebook.com", "https://www.ruby-lang.org", "http://whirlpool.net.au",
  "https://bitbucket.org", "https://github.com", "https://news.ycombinator.com"
]

def download_with_futures(urls)
  pages = urls.map do |url|
    future { open(url) { |f| f.read } }
  end
  # For the purposes of the benchmark, wait until all downloads have
  # finished by calling a method on each future.
  pages.each { |page| page.length }
end

def download_without_futures(urls)
  urls.map do |url|
    open(url) { |f| f.read }
  end
end

Benchmark.bm do |b|
  b.report { download_with_futures(urls) }
  b.report { download_without_futures(urls) }
end
```

And the results are...

``` plain
       user     system      total        real
   0.160000   0.040000   0.200000 (  2.561927)
   0.110000   0.030000   0.140000 ( 13.774315)
```

13.8 seconds down to 2.6 seconds. Not bad!

Caveats
-------

The `Future` class given above isn't exactly production-ready. If you were using it in a real project, you'd want it to:

- Capture exceptions that occur during the thread's execution and re-throw them when the result is accessed.
- Inherit from `BasicObject` instead of `Object`, so methods defined on `Object` are also delegated to the result.
- Use a thread pool to prevent the system from being overloaded by a large number of threads.
- Probably a bunch of other things I haven't thought of.

A basic implementation in 9 lines of code isn't too shabby though!

Conclusion
----------

Futures are awesome. Ruby is awesome. That is all.
