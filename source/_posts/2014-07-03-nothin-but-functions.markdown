---
layout: post
title: "Nothin' but Functions"
date: 2014-07-03 16:59:34 +0930
comments: true
categories: [functional programming, lambda calculus]
---

Imagine we're using a programming language that doesn't give us any way of defining data structures. No arrays, no tuples, no structs, no classes. Nothing. All we can do is define and call functions.

Now imagine we need to work with a collection of items. Are we screwed?

<!-- more -->

Building data structures from functions
---------------------------------------

Turns out the answer is no. We can create data structures, quite literally, out of functions. Here's a linked list:

``` python
def make_list(head, tail=None):
  def node(operation):
    if operation == "head":
      return head
    elif operation == "tail":
      return tail
  return node
```

The `make_list` function accepts a `head` (the data item we want to store), and a `tail` (the next node in the list). It returns an instance of the `node` function, which in turn accepts a single argument and returns the head or tail of the list.

We can use `make_list` to construct the list `[1, 2, 3]` as follows:

``` python
numbers = make_list(1, make_list(2, make_list(3)))
```

Let's try extracting the values from it:

``` python
numbers("head")                 # => 1
numbers("tail")("head")         # => 2
numbers("tail")("tail")("head") # => 3
```

Now let's take a look at the nodes themselves:

``` python
numbers                         # => <function make_list.<locals>.node at 0x10341d620>
numbers("tail")                 # => <function make_list.<locals>.node at 0x10341d510>
numbers("tail")("tail")         # => <function make_list.<locals>.node at 0x10341d488>
numbers("tail")("tail")("tail") # => None
```

We do indeed have a list made of functions (an immutable, singly-linked list to be precise). It's a bit tedious to work with at the moment, but there's nothing stopping us from defining utility functions to operate on it. For example, here's a simple function that'd print the list:

``` python
def print_list(lst):
  if lst:
    print(lst("head"))
    print_list(lst("tail"))
```

Closures
--------

What we've done above is only possible because Python supports *closures*. A closure is a function that has access to the variables from the scope it was defined in. The `node` function is a good example of this: `head` and `tail` aren't explicitly passed in as arguments, but it can access them all the same. Each time we call `make_list`, a new instance of `node` is returned that references the new versions of `head` and `tail`.

Our list is nothing more than a chain of nested closures. When we call the outermost closure and pass it `"tail"` as an argument, it returns the next closure in the chain (i.e. the next node in the list).

Conclusion
----------

We've really just scratched the surface here - it's not hard to see that more complex data structures could be defined using similar techniques. We could even use functions to define *numbers* by representing the *n*th integer as a series of *n* nested functions.

This is all hinting at a deeper fact: any language that allows us to define and apply functions is Turing-complete (meaning we can use it to compute anything that can be computed). The [lambda calculus](http://palmstroem.blogspot.com.au/2012/05/lambda-calculus-for-absolute-dummies.html) is the ultimate example of this.

Neat, huh?
