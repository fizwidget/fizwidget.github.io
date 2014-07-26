---
layout: post
title: "Function Pipelines"
date: 2014-07-27 12:45:10 +0930
comments: true
categories: [functional programming, pipelines]
---

The concept of pipelining is pretty simple - components are connected in series such that the output from one flows into the input of the next. Piping data between programs in shell scripts is an example everyone's familiar with:

``` bash
cat names | grep "Snow" | sort | uniq
```

This style of programming doesn't have to be relegated to shell scripts though - it's a universal concept that can help us write better code in a wide variety of languages. Instead of piping data from one program to another, we'll compose functions (or equivalently, chain method calls).

<!-- more -->

Example #1
----------

Say we've got a list of student objects, and we want to retrieve the email addresses of the ten youngest first-year compsci students (perhaps to send them a survey or something). Using a traditional imperative approach, we might write something along these lines:

``` ruby
# Select all the first-year CS students.
first_year_cs_students = []
for student in students
  if student.level == 1 && student.degree == "CS"
    first_year_cs_students.push(student)
  end
end

# Take the 10 youngest of these students.
first_year_cs_students.sort_by!(&:age)
youngest_first_year_cs_students = first_year_cs_students[0, 10]

# Collect their email addresses.
emails = []
for student in youngest_first_year_cs_students
  email = student.email_address
  emails.push(email)
end

return emails
```

This works, but the code is quite verbose, and it's not particularly easy to understand or modify.

Let's instead try structuring our solution as a pipeline. We can translate the problem description almost directly into code by chaining together methods from Ruby's [`Enumerable`](http://www.ruby-doc.org/core-2.1.1/Enumerable.html) module:

``` ruby
students.select { |s| s.degree == "CS" }
        .select { |s| s.level == 1 }
        .sort_by(&:age)
        .take(10)
        .collect(&:email_address)
```

This is clearly a big improvement over our previous attempt. It's concise, easy to understand, and easy to modify. It also has the potential to run more efficiently, as the `select` and `collect` operations could potentially be distributed over multiple cores.

Example #2
----------

Let's say we want to get the extensions of the smallest file and the largest file in a particular directory. Taking an imperative approach, we might write something like this:

``` ruby
min_extension = nil
max_extension = nil
min_size = 0
max_size = 0


Dir.foreach(dir_path) do |path|
  if File.file?(path)
    size = File.size(path)
    if size < min_size || min_extension.nil?
      min_extension = File.extname(path)
      min_size = size
    elsif size > max_size || max_extension.nil?
      max_extension = File.extname(path)
      max_size = size
    end
  end
end

return [min_extension, max_extension]
```

Needless to say, this code is a bit nasty. Conceptually, there are four distinct steps we want to perform: list all directory entries, select the entries that correspond to files, select the smallest and largest of these files, then get their extensions. These stages are all intermingled in the code above.

Taking a pipelined approach with `Enumerable` methods, we can cleanly separate the stages:

``` ruby
Dir.entries(dir_path)
   .select    { |path| File.file?(path) }
   .minmax_by { |path| File.size(path) }
   .collect   { |path| File.extname(path) }
```

Such elegance. Much conciseness. Wow.

I should also point out that we're free to introduce intermediate variables and helper methods as needed - we don't need to have a single unbroken chain of method calls. So long as we cleanly separate the stages and avoid mutating state (appending to lists, re-assigning variables, etc), we're still using a pipelined, functional approach.

In other languages
------------------

The examples above are written in Ruby, but the concepts are universal. In Python, list comprehensions, generator expressions, and the [`itertools`](http://jmduke.com/posts/a-gentle-introduction-to-itertools/) module come in handy. In Java 8, the [Streams API](http://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html) is what we're after. In C#, check out [`Enumerable`](http://msdn.microsoft.com/en-us/library/system.linq.enumerable.ASPX) and LINQ. Odds are, your favourite language supports these concepts in one form or another.

This style of coding has its roots in functional programming, and it's the standard way of doing things in languages like Haskell and Scheme. There are also obvious similarities to SQL - this is no coincidence, as functional languages and SQL are both [declarative](http://en.wikipedia.org/wiki/Declarative_programming) rather than [imperative](http://en.wikipedia.org/wiki/Imperative_programming) (meaning we describe *what* we want rather than having spell out exactly *how* to achieve it).

Conclusion
----------

The next time you find yourself writing loops and if-statements, consider whether you'd be better off with a more functional approach. Once you become familiar with this style of programming, you'll realise that opportunities to simplify imperative code are everywhere!
