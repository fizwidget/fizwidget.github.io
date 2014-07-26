---
layout: post
title: "Function Pipelines"
date: 2014-07-26 12:45:10 +0930
comments: true
categories: [functional programming, pipelines]
---

The concept of pipelining is pretty simple - components are connected in series such that the output from one flows into the input of the next. Piping data between programs in Unix shell scripts is an example that everyone's familiar with:

<!-- more -->

``` bash
cat employees | grep "Akane" | uniq | sort
```

This style of programming doesn't have to be relegated to shell scripts though. It's a very powerful technique that can help us write better code in many different languages.

Instead of piping data from one program to another, we'll compose functions (or equivalently, chain method calls) to transform data within a program.

An example
----------

Say we've got a list of student objects, and we want to retrieve the email addresses of the ten youngest first-year CS students (perhaps to send them a survey or something). Using the traditional imperative approach, we might write something along these lines:

``` ruby
# Select all the first-year CS students.
level_1_cs_students = []
for student in students
  if student.level == 1 && student.degree == "CS"
    level_1_cs_students.push(student)
  end
end

# Take the 10 youngest of these students.
level_1_cs_students.sort_by!(&:age)
youngest_level_1_cs_students = level_1_cs_students[0, 10]

# Collect their email addresses.
emails = []
for student in youngest_level_1_cs_students
  emails.push(student.email)
end

return emails
```

This works, but the code is quite verbose, and it's not particularly easy to understand or modify (meaning it's also more error-prone).

Let's instead try structuring our solution as a pipeline. We can translate the problem description almost directly into code by chaining together methods from Ruby's [`Enumerable`](http://www.ruby-doc.org/core-2.1.1/Enumerable.html) module:

``` ruby
students.select { |s| s.degree == "CS" }
        .select { |s| s.level == 1 }
        .sort_by(&:age)
        .take(10)
        .collect(&:email)
```

This is clearly a big improvement over the previous attempt. It's concise, easy to understand, and easy to modify. It also has the potential to run more efficiently, as the `select` and `collect` operations could theoretically be distributed over multiple cores.

The code above is written in Ruby, but the concepts are universal. In Python, list comprehensions, generator expressions, and the [`itertools`](http://jmduke.com/posts/a-gentle-introduction-to-itertools/) module come in handy. In Java 8, the Streams API provides the equivalent toolset (and it supports parallel execution!). In C#, LINQ is what you're after.

This style of coding has its roots in functional programming, and it's the standard way of doing things in languages like Haskell and Scheme. There are also obvious similarities to SQL. This is no coincidence, as functional languages and SQL are both [declarative](http://en.wikipedia.org/wiki/Declarative_programming) rather than [imperative](http://en.wikipedia.org/wiki/Imperative_programming) (meaning we describe *what* we want rather than spell out exactly *how* to achieve it).

Conclusion
----------

The next time you find yourself writing loops and if-statements, consider whether you'd be better served by a more functional approach. Once you become familiar with this style of programming, you'll realise that opportunities to use it are all over the place!
