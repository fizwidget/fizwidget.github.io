---
layout: post
title: Nothing but Functions
date: 2014-07-03 16:59:34 +0930
comments: true
categories: [functional programming, closure, lambda calculus, linked list, turing complete]
---

Imagine we're using a programming language that doesn't give us any way of defining data structures. No arrays, no structs, no objects, no classes. Nothing. All we can do is define and call functions.

We need to work with a collection of items. Are we screwed?

## A list made of functions

Turns out the answer is no. We can create linked lists, quite literally, out of functions. Let's jump straight into some code:

``` python
def make_list(head, tail):
  def node(operation):
    if operation == "head":
      return head
    elif operation == "tail":
      return tail
  return node
```

The `make_list` function accepts a `head` (the data item we want to store), and a `tail` (the rest of the linked list). We can use it to construct the list `[1, 2, 3]` as follows:

``` python
items = make_list(1, make_list(2, make_list(3, None)))
```

When we call `make_list`, it defines and returns a function called `node`. The `node` function accepts a single parameter and returns the head or the tail based on its value.

Let's give it a try:

``` python
items("head")                 # => 1
items("tail")                 # => <function node at 0x107a82b18>
items("tail")("head")         # => 2
items("tail")("tail")("head") # => 3
items("tail")("tail")("tail") # => None
```

Neat! That's a bit tedious though, so let's write a function to print the list:

``` python
def print_list(items):
  if items:
    print(items("head"))
    print_list(items("tail"))

print_list(items) # Output: 1 2 3
```

## Closures

A closure is a function that can access the variables from the scope in which it was created. We can see an example of this in the code above: the `node` function accesses `head` and `tail`, which were passed into `make_list`. Each time we call `make_list`, a new instance of `node` is returned that references the new `head` and `tail` arguments.

Our list is nothing more than a chain of nested closures. When we call the outermost closure and pass it `"tail"` as an argument, it returns the next closure in the chain (i.e. the next node in the list).

## Conclusion

We've really just scratched the surface here - it's not hard to see that similar techniques could be used to define more complex data structures. We could even use functions to define *numbers* by representing the nth integer as a series of n nested functions.

This is all hinting at a deeper fact: any language that allows you to define and apply functions is Turing-complete (meaning it lets you compute anything that can be computed). The [lambda calculus](http://palmstroem.blogspot.com.au/2012/05/lambda-calculus-for-absolute-dummies.html) is the canonical example of such a language.

Neat, huh?
