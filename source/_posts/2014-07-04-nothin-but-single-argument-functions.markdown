---
layout: post
title: "Nothin' but Single-Argument Functions"
date: 2014-07-04 09:17:13 +0930
comments: true
categories: [functional programming, partial application, turing complete]
---

In a [previous post](/blog/2014/07/03/nothin-but-functions/), I talked about the fact that functions are the only thing a language needs to be Turing-complete. In this post I'll take it a little further: *single-argument functions* are all a language needs to be Turing-complete.

At first glance this seems counter-intuitive. Surely there are a *lot* of things we wouldn't be able to do...adding two numbers, comparing two strings, searching for a number in a list, etc. These functions would all require more than one argument.

<!-- more -->

If you read my previous post on functions, you might be thinking we could pass around linked lists to simulate multiple arguments. There's one problem with that though...the `make_list` function accepts two arguments. We can't use a function that takes multiple arguments to implement multi-argument functions. That'd be assuming what we want to prove.

Closures to the rescue
----------------------

Remember closures? In addition to their arguments, they can access variables from their enclosing scope. Turns out we can use this ability to simulate multi-argument functions. Let's try writing a function to concatenate two strings:

``` python
def concatenate(a):
  def concatenate_aux(b):
    return a + b
  return concatenate_aux

concatenate("Hello")("World") # => "HelloWorld"
```

*(Ignore the fact that I've implemented concatenation with `+`, which is essentially a multi-argument function. That's just for brevity.)*

Tada! The top-level `concatenate` function returns another function, which returns the final result when called. The `concatenate_aux` function is a closure, meaning it still has access to `a` when we call it later on.

So, we can decompose an n-argument function into a series of n nested single-argument functions (this is known as [currying](http://en.wikipedia.org/wiki/Currying)).

This opens up another interesting possibility...

Partial function application
----------------------------

If a function takes n arguments, we usually have to call it with n arguments - anything less results in an error. If we define functions like we did above though, we can *partially apply* them:

```python
greet = concatenate("Hello, ")
```

We've now got a new function (an instance of `concatenate_aux`) that will prepend `"Hello, "` to whatever we give it:

``` python
greet("Fred") # => "Hello, Fred"
greet("Bob")  # => "Hello, Bob"
```

This can actually be a useful technique. The way we did it above is kinda messy, but Python has a [library function](https://docs.python.org/3/library/functools.html#functools.partial) to simplify the process. In some languages (e.g. Haskell) any function can be partially applied - no extra work needed.

Conclusion
----------

-   Single-argument functions are all a language needs to be Turing-complete.
-   Partial function application lets us build specialised functions out of more general ones.
