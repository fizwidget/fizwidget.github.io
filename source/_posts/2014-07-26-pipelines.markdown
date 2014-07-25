---
layout: post
title: "Pipelines!"
date: 2014-07-26 12:45:10 +0930
comments: true
categories: [functional programming, pipelines]
---

The concept of pipelining is pretty simple - a series of components are connected together in series such that the output from one flows into the input of the next. In the world of software, Unix shell scripting is perhaps the canonical example:

<!-- more -->

``` bash
cat employees | grep "akane" | uniq | sort
```

Pipelining is also commonly used as a high-level system architecture. Consider a program that captures images from a camera, tags them with metadata, compresses them, then writes them to disk. It's not hard to see how it could be structured as a series of four modules connected together in a pipeline.

So pipelining is a neat idea, and it's used all over the place. Unfortunately though, there's one area where it isn't used anywhere near as much as it should be: general purpose programming. I'm not talking about high-level architectures here, I'm talking about the nitty-gritty implementation details.

An example
----------

Let's say we're writing a program that manages a list of student objects, and we want to retrieve the email addresses of the ten youngest first-year CS students. Using a traditional imperative coding style, we might write something like this:

``` ruby
level_1_cs_students = []

for student in students
  if student.level == 1 && student.degree == "CS"
    level_1_cs_students.push(student)
  end
end

level_1_cs_students.sort_by!(&:age)
youngest_level_1_cs_students = level_1_cs_students[0, 10]

emails = []

for student in youngest_level_1_cs_students
  emails.push(student.email)
end

return emails
```

This code works, but it's not particularly easy to understand or modify.

Let's instead try structuring our solution as a pipeline. We want to select the first-year CS students, take the 10 youngest ones, and retrieve their email addresses. We can translate this almost directly into code by chaining together methods from Ruby's `Enumerable` module:

``` ruby
students.select { |s| s.degree == "CS" }
        .select { |s| s.level == 1 }
        .sort_by(&:age)
        .take(10)
        .collect(&:email)
```

This is clearly a big improvement over the previous code. It's more concise, easier to understand, easier to modify, and less error-prone. It also has the *potential* to run more efficiently, as the runtime system could theoretically distribute the `select` and `collect` operations over multiple cores.

Ruby is particularly well suited to this style of coding, but it's practical in many other languages as well. In Python for example, list comprehensions and the `itertools` module give us the equivalent tools.

Conclusion
----------

The key difference is that the first attempt was written in an *imperative* style, while the latter was written in a *declarative* style. The functional pipeline approach describes what we want, rather than spelling out the low-level details of how to achieve it.

So the next time you go to write a for-loop or an if-statement, consider whether you'd be better served by a `collect` or a `select`.
