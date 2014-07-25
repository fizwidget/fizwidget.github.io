---
layout: post
title: "Python or Ruby?"
date: 2014-07-25 22:10:23 +0930
comments: true
categories: [ruby, python]
published: true
---

Python and Ruby are the two heavyweights in the general-purpose, high-level, dynamic language category. They're both awesome, but given the choice, I'll generally go with Ruby. What follows is my attempt at explaining why...

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

In Python though, a mixture of functions and methods is the norm:

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
"world hello".split.reverse.join(' ') # => "hello world"
```

The equivalent Python code is much harder to make sense of. We have to read some parts left-to-right and other parts inside-out. Yuk!

``` python
' '.join(reversed("world hello".split())) # => "hello world"
```

Ruby's blocks are awesome
-------------------------

One of Ruby's best features is the elegant syntax it has for passing anonymous functions ("blocks") as arguments to methods. Blocks are like Python's lambda expressions, only more powerful, more widely used, and visually cleaner.

Blocks allow us to perform many different tasks using a simple, uniform syntax. For example:

``` ruby
[1, 2, 3].each   { |x| puts x }         # Iteration using blocks.
File.open('a')   { |f| puts f.read }    # Automatic resource management using blocks.
[1, 2, 3].select { |x| x > 2 }          # List processing using blocks.
data.sort_by     { |x| x[1] }           # Sorting using blocks.
files.lazy.map   { |f| Image.new(f) }   # Lazy evaluation using blocks.
```

In Python, we'd typically use no fewer than *five* different language constructs to perform the same tasks:

``` python
for x in [1, 2, 3]: print(x)            # Iteration using 'for'.
with f as open('a'): print(f.read())    # Automatic resource management using 'with'.
[x for x in [1, 2, 3] if x > 2]         # List processing using list comprehensions.
sorted(data, key=lambda x: x[1])        # Sorting using 'lambda'.
(Image(f) for f in files)               # Lazy evaluation using generator expressions.
```

One flexible construct is preferable to multiple rigid ones in my book.

Clear conditionals
------------------

Pretty much everything with a length/size/magnitude of zero evaluates to 'false' in Python. This isn't just a quirk of the language, it's promoted in [PEP 8](http://legacy.python.org/dev/peps/pep-0008/#programming-recommendations) as the Pythonic way of checking whether a sequence is empty:

``` python
if not some_sequence:
    # Whatever happened to "explicit is better than implicit"??
```

In Ruby, `false` and `nil` are the only values that evaluate to 'false'. This means we must explicitly ask objects questions:

``` ruby
if some_sequence.empty? 
  # Obvious meaning is obvious.
end
```

Ruby's approach seems both more readable and less error-prone.

Private parts
-------------

Instance variables are private by default in Ruby. This encourages proper encapsulation of implementation details.

``` ruby
class Foo
  def initialize
    @secret = 42
  end
end

Foo.new.secret # => Error!
```

In Python though, everything defaults to public. This means we're more likely to end up with nasty dependencies on implementation details.

``` python
class Foo:
    def __init__(self):
        self.not_so_secret = 42

Foo().not_so_secret # => 42
```

Naming and documentation
------------------------

The names used in Python's standard library are a bit of a mess (even [PEP 8](http://legacy.python.org/dev/peps/pep-0008/#naming-conventions) admits as much). Cryptic abbreviations and inconsistent naming conventions are not an uncommon sight. Ruby's standard libraries are much nicer to work with in comparison.

In terms of documentation quality, Ruby also has the edge. Descriptions are clearer and more detailed, and usage examples are almost always given.

Consider the documentation for `str.capitalize` in Python:

``` plain
str.capitalize = capitalize(...)
    S.capitalize() -> string
    
    Return a copy of the string S with only its first character
    capitalized.
```

Then compare it with Ruby's `String#capitalize` documentation:

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

Unless you were paying close attention, you might have missed the fact that `str.capitalize` converts the remainder of the string to lowercase (e.g. "ABC" goes to "Abc"). The Ruby documentation makes this behaviour much clearer, and clarifying examples are also given.

This is just one method of course, but these kinds of differences are not atypical in my experience.

Ruby's pretty!
--------------

OK, I'll admit this is ever so slightly subjective. Having said that, compare this:

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

Conclusion
----------

So those are a few of the reasons why I like Ruby. Admittedly this was a rather one sided comparison - Ruby has plenty of downsides and Python plenty of benefits that I neglected to mention! Perhaps those will be the topic of a future post...
