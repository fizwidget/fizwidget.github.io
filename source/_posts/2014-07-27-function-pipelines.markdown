---
layout: post
title: "Function Pipelines"
date: 2014-07-27 12:45:10 +0930
comments: true
categories: [functional programming, pipelines]
---

The concept of pipelining is pretty simple - components are connected in series such that the output from one flows into the input of the next. Piping data in shell scripts is an example most programmers are familiar with:

``` bash
cat logs | grep 'ERROR' | sed 's/shit/darn/g' | sort | uniq > bad_things
```

This style of programming doesn't have to be relegated to shell scripts though - it's a universal concept that can help us write better code in many different languages. Instead of piping data from one program to another, we'll compose functions (or equivalently, chain methods).

<!-- more -->

Let's jump straight into some Ruby examples, then I'll discuss how these ideas apply to other languages.

Example #1
----------

Say we've got a list of student objects, and we want to get the email addresses of the ten youngest third-year compsci students (perhaps to send them a survey or something). Using a traditional imperative approach, we might write something along these lines:

``` ruby
# Select all the third-year CS students.
third_year_cs_students = []
for student in students
  if student.level == 3 && student.degree == "CS"
    third_year_cs_students.push(student)
  end
end

# Take the 10 youngest of these students.
third_year_cs_students.sort_by!(&:age)
youngest_third_year_cs_students = third_year_cs_students.slice(0, 10)

# Collect their email addresses.
emails = []
for student in youngest_third_year_cs_students
  email = student.email_address
  emails.push(email)
end

return emails
```

This works, but the code is quite verbose, and it's not particularly easy to understand or modify.

Let's instead try structuring our solution as a pipeline. We can translate the problem description almost directly into code by chaining together methods from Ruby's [`Enumerable`](http://www.ruby-doc.org/core-2.1.1/Enumerable.html) module:

``` ruby
students.select { |s| s.level == 3 }
        .select { |s| s.degree == "CS" }
        .sort_by(&:age)
        .take(10)
        .collect(&:email_address)
```

This is clearly a big improvement over the previous attempt. It's concise, easy to understand, and easy to modify. It also has the potential to run more efficiently, as the `select` and `collect` operations could potentially be distributed over multiple cores.

A key thing to note is that none of these methods are modifying the objects they're called on. They're instead applying some kind of filter or transformation, then outputting a new sequence to the next stage of the pipeline.

It's also worth pointing out that we can use intermediate variables and/or helper methods if we wish - the code doesn't *have* to be an unbroken chain of method calls. So long as we cleanly separate the stages and avoid mutating state (reassigning variables, appending to lists, etc), we're still using a pipelined approach.

Example #2
----------

For this example, we want to get the extensions of the smallest file and the largest file in a directory. If our directory contained a small text file and a large video file, example output might be `[".txt", ".mkv"]`. Taking an imperative approach, we might write something like this:

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

Conceptually, there are four distinct stages to the computation: list all directory entries, select the entries that correspond to files, select the smallest and largest of these files, and lastly, get their extensions. In the code above, these stages are all intermingled. This makes it harder to understand, and limits the reusability of the logic.

Taking a pipelined approach with `Enumerable` methods, we can cleanly separate the stages and reuse existing logic:

``` ruby
Dir.entries(dir_path)
   .select    { |path| File.file?(path) }
   .minmax_by { |path| File.size(path) }
   .collect   { |path| File.extname(path) }
```

Much nicer!

In other languages
------------------

The examples above are written in Ruby, but the concepts are universal. In Python, list comprehensions, generator expressions, and the `itertools` module all come in handy (plus builtins like `map` and `filter`). In Java 8, the Streams API provides a similar toolbox. In C#, check out `Enumerable` and LINQ. Your language of choice probably supports these concepts in one form or another.

This style of coding has its roots in functional programming, and it's the standard way of doing things in languages like Haskell and Scheme. There are also obvious similarities to SQL - this is no coincidence, as functional languages and SQL are both [declarative](http://en.wikipedia.org/wiki/Declarative_programming) rather than [imperative](http://en.wikipedia.org/wiki/Imperative_programming) in nature (meaning we describe *what* we want rather than spell out exactly *how* to achieve it).

Conclusion
----------

The next time you find yourself writing loops and if-statements, consider whether a more functional approach would be more appropriate. Once you're familiar with this style of programming, you'll notice lots of new opportunities to simplify and streamline your code.
