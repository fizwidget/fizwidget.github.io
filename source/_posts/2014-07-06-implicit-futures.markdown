---
layout: post
title: "Implicit Futures"
date: 2014-07-06 02:20:05 +0930
comments: true
categories: [concurrency, ruby, futures]
---

Let's say we're writing a program that needs to perform several time-consuming operations (such as network requests, disk accesses, or complex calculations). How should we structure our code to execute them in an efficient way? There are many possible approaches to this problem, but in this post I'll focus on implicit futures and how we can implement them in Ruby.

<!-- more -->

This snippet of code demonstrates the basic scenario we're dealing with:

``` ruby
a = foo.expensive_call_1
b = bar.expensive_call_2
c = baz.expensive_call_3

other_stuff

result = a + (b * c)
```

We initiate one or more time-consuming operations, do some other stuff, then use the results of the operations. In the code above, the expensive calls are executed sequentially. This will lead to very poor performance - for example, the CPU might be forced to sit idle while a network request is in progress. Additionally, the call to `other_stuff` won't begin running until all three of the previous calls have finished, even though it doesn't depend on their results.

This is the solution we're working towards:

``` ruby
a = future { foo.expensive_call_1 }
b = future { bar.expensive_call_2 }
c = future { baz.expensive_call_3 }

other_stuff

result = a + (b * c)
```

*If you're unfamiliar with Ruby's block syntax, `foo { bar }` is equivalent to `foo(lambda: bar())` in Python.*

In the code above, the expensive calls are executed asynchronously and `other_stuff` can begin running immediately. If line 7 is reached before all three results have been calculated, the main thread will automatically block to wait for them.

Neat! How on Earth can you implement something like this though..?

Explicit Futures
----------------

It turns out that Ruby's `Thread` class does a lot of the work for us. Here's a quick primer on it:

``` ruby
t = Thread.new { 2 + 2 }
t.value # => 4
```

`Thread.new` spawns a thread to execute the block, and `t.value` returns the thread's result (blocking if necessary to wait for it to finish running).

This *almost* gets us where we want to be. Let's rewrite our original code to use the `Thread` class:

``` ruby
a = Thread.new { foo.expensive_call_1 }
b = Thread.new { bar.expensive_call_2 }
c = Thread.new { baz.expensive_call_3 }

other_stuff

puts a.value + (b.value * c.value)
```

Notice that we have to explicitly retrieve the results by calling the `value` method on `a`, `b`, and `c`. This isn't quite what we're after - we want to be able to treat them as though they're actually the results.

This mightn't seem like much of an issue...calling `value` isn't exactly difficult. Consider this though: what would happen if we wanted to pass `a`, `b`, or `c` into other functions or methods, or return them as results? Things would start getting messy, because every piece of code that touches them would need to know that they're not *actually* the results, they're objects we call `value` on to get the results.

This is why *implicit* futures are so nice - the code that uses them remains blissfully ignorant of what they are. If `a` were an implicit future, we could freely pass it around to code that doesn't know anything about futures.

Implementing Implicit Futures
-----------------------------

To summarise the problem, we want to be able to treat `a` as though it were the result of the computation:

``` ruby
a = future { some_computation }
```

We don't want to have to explicitly retrieve the result by calling `value`.

In order to achieve this, we want `future` to return a *transparent proxy object*. A transparent proxy is a wrapper around an object that forwards all method calls to the object it's wrapping. From the outside, it looks like the object it's wrapping. It's not immediately obvious how we can achieve this, but in a dynamic language like Ruby, it actually ends up being quite easy.

In Ruby, we can define a special method called `method_missing`. As the name suggests, this method is called automatically if an object can't otherwise respond to a method. Normally this would result in a runtime error, but if `method_missing` is defined, it'll be called instead.

Here's a quick demo:

``` ruby
class Useless
  def method_missing(name, *args, &block)
    puts "Someone called the '#{name}' method on me!"
    puts "I was given #{args} as arguments."
    puts "Now let's call the block I was given..."
    block.call
  end
end

u = Useless.new
u.well_this_is_weird(1, 2, 3) { puts "wat" }
```

Running this produces the following output:

``` plain
Someone called the 'well_this_is_weird' method on me!
I was given [1, 2, 3] as arguments.
Now let's call the block I was given...
wat
```

If we use `method_missing`, creating a transparent proxy becomes quite easy:

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

The call to `reverse` was intercepted by `method_missing`, then forwarded to the wrapped object (via the `send` method, which allows us to call arbitrary methods on objects at runtime).

### Finally...

Things are starting to come together now. We can write a `Future` class that:

1. Uses a `Thread` to asynchronously execute a block of code.
2. Acts as a transparent proxy, forwarding all methods to the thread's result.

The code for this is actually very simple:

``` ruby
class Future
  def initialize(&block)
    @thread = Thread.new { block.call }
  end
  
  def method_missing(name, *args, &block)
    @thread.value.send(name, *args, &block)
  end
end

f = Future.new { 20 * 2 }
```

To make it slightly easier to use, we'll define standalone `future` function that creates and returns a `Future` object:

``` ruby
def future(&block)
  Future.new(&block)
end

f = future { 20 * 2 }
```

This looks exactly like what we set out to achieve. Yay!

Might be a good idea to check if it works though...

Sanity check
------------

``` ruby
f = future do
  puts "Future: Started."
  sleep(4)
  puts "Future: Finished."
  "yeeeeaaaaaaaahh"
end

sleep(1)

puts "Attempting to use the result..."
puts f.upcase
```

This gives us the output:

``` plain
Future: Started.
Attempting to use the result...
Future: Finished.
YEEEEAAAAAAAAHH
```

Yay. When we attempted to use the result, the main thread blocked until it became available. That's exactly what we wanted to happen.

Caveats
-------

I should probably mention that the above code isn't exactly production-ready. If you were going to use this in a real project, you'd want the `Future` class to:

- Capture exceptions and re-throw them when the future is used.
- Inherit from `BasicObject` instead of `Object` (so `Object`'s methods also get forwarded).
- Use a thread-pool to prevent the system from being overloaded by a large number of threads.
- Probably a bunch of other things I haven't thought of.

Conclusion
----------

Futures are awesome. Ruby is awesome. That is all.
