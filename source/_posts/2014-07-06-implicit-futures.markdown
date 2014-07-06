---
layout: post
title: "Implicit Futures"
date: 2014-07-06 02:20:05 +0930
comments: true
categories: [concurrency, ruby, futures]
---

Implicit futures...sounds like an album title or something. In the world of CS though, it's a concurrency construct. Let's say we need to perform some time-consuming tasks (downloading webpages, performing expensive computations, loading data from disk, ...), and we want to execute them as efficiently as possible. This means we want to run them in parallel, and we don't want to block the rest of the program while they're running.

This is our starting point (the language is Ruby):

``` ruby
# Calculate results.
a = expensive_call_1
b = expensive_call_2
c = expensive_call_3

# Do other stuff.
stuff

# Use results.
puts a + (b * c)
```

...this will perfom horribly. The expensive calls will execute sequentually, meaning the CPU might be forced to sit twiddling its thumbs while a network request finishes. Additionally, `stuff` won't get to start running until after all three calls have finished, even though it doesn't depend on any of their results.

This is what we're working towards:

``` ruby
# Calculate results.
a = future { expensive_call_1 }
b = future { expensive_call_2 }
c = future { expensive_call_3 }

# Do other stuff.
stuff

# Use results.
puts a + (b * c)
```

If you're unfamiliar with Ruby's block syntax, `foo { puts "Hi!" }` passes an anonymous function that prints "Hi!" into the method `foo` (equivalent to `foo(lambda: print("Hi!))` in Python). We need to pass in our computations like this because we don't want them to execute immediately - we want `future` to execute them in the background.

This allows the three calls to execute in parallel, and it frees up the system to perform other tasks in the meantime. If the results have all been calculated by the time we reach the final statement, the final result will immediately be calculated and printed. If any of them haven't yet finished though, the code will block until they do.

Perfect! How on Earth can we implement this though...?

Implementation attempt #1
-------------------------

It turns out Ruby's `Thread` class can do most of the work for us. A quick primer...

``` ruby
t = Thread.new { 2 + (10 * 4) }
puts t.value
```

Calling `Thread.new { some_code }` spawns a thread that executes `some_code`. Calling `t.value` will block until the thread finishes and return the thread's result. The thread's result in this case is 2 + (10 * 4) = 42. So, the above code executes `2 + (10 * 4)` on a background thread, then prints `42` after the thread has finished running.

This *almost* gets us where we want to be. Let's try rewriting our original code using the `Thread` class:

``` ruby
# Calculate results.
a = Thread.new { expensive_call_1 }
b = Thread.new { expensive_call_2 }
c = Thread.new { expensive_call_3 }

# Do other stuff.
stuff

# Use results.
puts a.value + (b.value * c.value)
```

This is actually an *explicit* future. It's explicit because `a`, `b`, and `c` aren't the results of our computations - they're `Thread` objects, and we need to call `value` on them to get the results. Ideally we want `a`, `b`, and `c` to *actually be* the results (or at least appear as though they are).

This might seem like a minor distinction, and in the example I've given, it really doesn't make much difference. In a more realistic example though, we might want to pass `a` into a function, or return it as a result. Things now start getting messy - every piece of code that touches `a` needs to be aware of the fact that it's not *actually* the result, it's an object that we call `value` on to ask for the result. This is why *implicit* futures are so nice - the code that uses them can be blissfully ignorant of the fact that they're executing asynchronously. When code tries to use an implicit future, it will behave as though it were the result (blocking if necessary until the result becomes available).

So, back to the drawing board.

Implementation attempt #2
-------------------------

To summarise the problem we're dealing with, we want `a` in the following code to behave as though it were the result of the computation:

``` ruby
a = future { some_computation }
```

We could make the call to `future` block until the result is available, but that'd defeat the whole point of the exercise - we're trying to *prevent* unnecessary blocking.

We can't magically replace `a` with the result when it becomes available. As far as I'm aware, you simply can't do that in any mainstream programming language. What we *can* do is make `a` a transparent proxy object - that it, make it forward all method calls onto the result (blocking if necessary until the result becomes available). At first glance this might also seem impossible, but in dynamic languages like Ruby or Python, it's actually pretty easy.

In Ruby, you can define a special method called `method_missing`. As the name suggests, this method is automatically called when an object can't respond to a method. Normally this would cause a runtime error, but if `method_missing` is defined, it'll be called instead. Let's check it out:

``` ruby
class Useless
  def method_missing(name, *args, &block)
    puts "Yay, someone called '#{name}' on me!"
    puts "I was given these arguments: #{args}"
    puts "Now let's run the block I was given..."
    block.call
  end
end

well = Useless.new
well.this_is_weird(1, 2, 3) { puts "wat" }
```

Running this produces the following output:

``` plain
Yay, someone called 'this_is_weird' on me!
I was given these arguments: [1, 2, 3]
Now let's run the block I was given...
wat
```

Now let's try something slightly less useless:

``` ruby
class TransparentProxy
  def initialize(target)
    @target = target
  end

  def method_missing(name, *args, &block)
    @target.send(name, *args, &block)
  end
end

proxy = TransparentProxy.new(20)
puts 2 + (proxy * 2)
```

The result: `42`. The proxy forwarded the call to `*` to the integer it was wrapping (...numbers are objects in Ruby by the way).

We've now got everything we need to implement implicit futures. The final code is ridiculously simple:

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
puts f + 2
```

To make it slightly more usable, we'll define standalone function called `future`:

``` ruby
def future(&block)
  Future.new(&block)
end

f = future { 20 * 2 }
puts f + 2
```

Sanity check
------------

Having done all that work, let's make sure the damn thing actually works how we expect it to:

``` ruby
puts "Let's create a future..."

f = future do
  puts "Future: Just started running."
  sleep(2)
  puts "Future: Done!"
  20
end

sleep(1)

puts "Shortly after the future was created."

sleep(2)

puts "Future should have finished by now. Let's use the result:"
puts 2 + (f * 2)
```

This gives us the output:

``` plain
Let's create a future...
Future: Just started running.
Shortly after the future was created.
Future: Done!
Future should have finished by now. Let's use the result:
42
```

Perfect! Now let's try a scenario where we attempt to use the future before the result has been computed...

``` ruby
f = future do
  puts "Future: Just started running."
  sleep(2)
  puts "Future: Done!"
  20
end

sleep(1)

puts "Let's try and use the future before it's ready..."
puts 2 + (f * 2)
```

This gives us:

``` plain
Future: Just started running.
Let's try and use the future before it's ready...
Future: Done!
42
```

This is exactly what we'd expect to happen: the final result couldn't be calculated until the future finished running. Neat!

Conclusion
----------

I should probably mention that the above code isn't exactly production-ready. If you were going to use this in a real project, you'd want to properly capture and re-throw exceptions, and you'd also want to extend from `BasicObject` rather than `Object` (to ensure methods defined on `Object` are properly forwarded to the result). This would've made the code harder to understand though, so I stuck with the most basic implementation above.

tl;dr Futures are awesome and Ruby is awesome.
