---
layout: post
title: "Python or Ruby?"
date: 2014-07-25 22:10:23 +0930
comments: true
categories: [ruby, python]
---

Python and Ruby are the two heavyweights in the general-purpose, high-level, dynamic language category. They're both awesome, but given the choice, I'll generally go with Ruby. This is my attempt at explaining why...

<!-- more -->

Ruby is consistently object-oriented
------------------------------------

Pretty much everything we do in Ruby involves calling methods on objects:

``` ruby
-42.abs               # => 42
[1, 2, 3].length      # => 3
[1, 2, 3].reverse     # => [3, 2, 1]
[1, 2, 3].reverse!    # Reverses list in-place.
[1, 2, 3].min         # => 1
"foo".capitalize      # => "Foo"
File.new("bar")       # => File object
file.close            # Closes file object.
```

In Python though, a mixture of functions and methods are typically used:

``` python
abs(-42)              # => 42
len([1, 2, 3])        # => 3
reversed([1, 2, 3])   # => [3, 2, 1]
[1, 2, 3].reverse()   # Reverses list in-place.
min([1, 2, 3])        # => 1
"foo".capitalize()    # => "Foo"
open("bar")           # => File object
file.close()          # Closes file object.
```

One of the benefits of Ruby's consistency is that we can always read code left-to-right. This makes functional-style pipelines much easier to read and write:

``` ruby
"world hello".split.reverse.join(" ") # => "hello world"
```

The equivalent Python code is much harder to make sense of. We have to read some parts left-to-right and other parts inside-out. Yuk!

``` python
" ".join(reversed("world hello".split())) # => "hello world"
```

Blocks are awesome
------------------

One of Ruby's best features is the elegant syntax it has for passing anonymous functions ("blocks") as arguments to methods. Blocks are like Python's lambda expressions, only more powerful, more widely used, and visually cleaner.

Blocks allow us to perform many different tasks using a single uniform syntax. For example:

``` ruby
[1, 2, 3].each   { |x| puts x }         # Iteration using blocks.
File.open('a')   { |f| puts f.read }    # Automatic resource management using blocks.
[1, 2, 3].select { |x| x > 2 }          # List processing using blocks.
data.sort_by     { |x| foo.bar(x) }     # Sorting using blocks.
urls.lazy.map    { |u| download(u) }    # Lazy evaluation using blocks.
```

In Python, we'd typically use no fewer than *five* language constructs to perform the same tasks:

``` python
for x in [1, 2, 3]: print(x)            # Iteration using 'for'.
with f as open('a'): print(f.read())    # Automatic resource management using 'with'.
[x for x in [1, 2, 3] if x > 2]         # List processing using list comprehensions.
sorted(data, key=lambda x: foo.bar(x))  # Sorting using 'lambda'.
(download(u) for u in urls)             # Lazy evaluation using generator expressions.
```

One flexible construct > multiple rigid ones in my book.

Clear conditionals
------------------

In Python, empty collections, empty strings, and certain other objects are logically false. This isn't just a quirk of the language, it's promoted in [PEP 8](http://legacy.python.org/dev/peps/pep-0008/#programming-recommendations) as the Pythonic way of checking for empty sequences. In addition to this, the number zero is logically false, and non-zero numbers are logically true.

``` python
if not some_sequence:
    # Whatever happened to "explicit is better than implicit"??

if some_number:
    # Admittedly this one isn't very Pythonic, but it does work.
```

Ruby doesn't allow these shenanigans, so the equivalent code is much more explicit:

``` ruby
if some_sequence.empty?
  # Obvious meaning is obvious.
end

unless some_number.zero?
  # Could use 'if some_number != 0', but this is nicer ;)
end
```

It's also worth noting that we can use `unless` and `until` in Ruby instead of `if not` and `while not` in Python. Method names can also include the '?' character, further improving readability. Compare this Python snippet:

``` python
while not buffer.is_full():
    # Add data to buffer.
```

With this Ruby one:

``` ruby
until buffer.full?
  # Add data to buffer.
end
```

Private parts
-------------

Instance variables are private by default in Ruby, which encourages proper encapsulation of implementation details.

``` ruby
class Foo
  def initialize
    @secret = 42
  end
end

Foo.new.secret # => Error!
```

Everything defaults to public in Python, and privacy can only be suggested a naming convention. This means we're more likely to end up with inappropriate dependencies on implementation details.

``` python
class Foo:
    def __init__(self):
        self.not_so_secret = 42

Foo().not_so_secret # => 42
```

Ruby's approach seems the wiser one to me. Encapsulation is a Good Idea™, so why shouldn't the language support and encourage it?

*(By the way, Ruby's approach doesn't result in C++/Java-style boilerplate. We can generate default getters/setters by placing `attr_accessor :property_name` in the class body, and override them later if need be.)*

Ruby's pretty!
--------------

OK, I'll admit this one is ever so slightly subjective. Having said that, compare this:

``` ruby
class Person
  def initialize(age)
    @age = age
  end

  def to_s
    "Person is #{@age} years old."
  end
end

person = Person.new(34)
puts person
```

To this:

``` python
class Person(object):
    def __init__(self, age):
        self.age = age

    def __str__(self):
        return "Person is {} years old.".format(self.age)

person = Person(34)
print(person)
```

The Ruby code has fewer parentheses, colons, and underscores, and doesn't need to specify `object`, `self`, or `return`. Python's whitespace sensitivity does allow it to omit `end`, but this doesn't make up for all the other clutter in my opinion.

Naming and documentation
------------------------

The naming conventions used in Python's standard library are a bit of a mess (even [PEP 8](http://legacy.python.org/dev/peps/pep-0008/#naming-conventions) admits as much). Ruby's standard library is much more consistent in comparison.

In terms of documentation quality, Ruby also has the edge. Descriptions are clearer and more detailed, and usage examples are almost always given. Consider the documentation for `str.capitalize` in Python:

``` plain
str.capitalize = capitalize(...)
    S.capitalize() -> string
    
    Return a copy of the string S with only its first character
    capitalized.
```

Then compare it with Ruby's `String#capitalize`:

``` plain
= String#capitalize

(from ruby core)
------------------------------------------------------------------------------
  str.capitalize   -> new_str

------------------------------------------------------------------------------

Returns a copy of str with the first character converted to uppercase
and the remainder to lowercase. Note: case conversion is effective only in
ASCII region.

  "hello".capitalize    #=> "Hello"
  "HELLO".capitalize    #=> "Hello"
  "123ABC".capitalize   #=> "123abc"
```

Unless you were paying close attention, you might have missed the fact that `str.capitalize` converts the remainder of the string to lowercase (e.g. "ABC" goes to "Abc"). The Ruby documentation makes this behaviour much more obvious, and gives clarifying examples.

This is just one example of course, but these kinds of differences are not atypical in my experience.

Conclusion
----------

So those are a few of the reasons why I prefer Ruby to Python. Admittedly this was a rather one sided comparison - Ruby has plenty of downsides and Python plenty of upsides that I neglected to mention. Perhaps those will be the topic of a future post!
