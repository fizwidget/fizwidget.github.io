---
layout: post
title: "Nothin' but Single-Argument Functions"
date: 2014-07-04 09:17:13 +0930
comments: true
categories: [functional programming, partial application, turing complete]
---

In a [previous post](/blog/2014/07/03/nothin-but-functions/), I talked about the fact that functions are the only thing a language needs to be Turing-complete. In this post I'm going to take it a little further: *single-argument functions* are all a language needs to be Turing-complete.

At first glance that seems weird. Surely there are a *lot* of things this would prevent us from doing...adding two numbers, comparing two strings, searching for a number in a list, etc. These functions would all require more than one argument.

<!-- more -->

If you read my previous post, you might be thinking of passing a linked list to simulate multiple arguments. There's one problem with that though...the `make_list` function accepts two arguments. We can't use a function that takes multiple arguments to implement multi-argument functions. That'd be assuming what we want to prove :P

Closures come to the rescue
---------------------------

Remember closures? In addition to their arguments, they can access variables from their enclosing scope. We can use this ability to simulate multi-argument functions. Let's try writing a function function to concatenate two strings:

``` python
def concatenate(a):
  def concatenate_aux(b):
    return a + b
  return concatenate_aux

concatenate("Hello")("World") # => "HelloWorld"
```

Tada! The top-level `add` function returns another function, which will in-turn return the final result. The `concatenate_aux` function is a closure, meaning it still has access to `a` when we call it later on.

So, we can decompose an n-argument function into a series of n nested single-argument functions (a process known as [currying](http://en.wikipedia.org/wiki/Currying)).

This opens up an interesting possibility...

Partial application
-------------------

Typically, if a function takes n arguments, you have to call it with n arguments. Anything less results in an error. If we define functions like we did above though, we can *partially apply* them:

```python
greet = concatenate("Hello, ")
```

We've now got a new function (an instance of `concatenate_aux`) that will prepend `"Hello, "` to whatever we give it:

``` python
greet("Fred") # => "Hello, Fred"
greet("Bob")  # => "Hello, Bob"
```

Conclusion
----------

So, single-argument functions are all a language needs to be Turing-complete. Neat, but not very practical.

Partial application on the other hand is actually a useful technique. Not so much in Python (due to the messy way we'd have to define functions) but some languages make it really easy. In Haskell we can partially apply any function, without having to to through the hassle of defining them in a weird way. This lets us build specialised functions from more general ones.
