---
layout: post
title: "Pipelines!"
date: 2014-07-26 12:45:10 +0930
comments: true
categories: [functional programming, pipelines]
---

The concept of pipelining is pretty simple - a series of components are chained together such that the output from one flows into the input of the next. In the world of software, Unix shell scripting is perhaps the canonical example:

<!-- more -->

``` bash
cat names | grep "fred" | uniq | sort
```

Pipelining is also commonly used as a high-level system architecture. Consider a program that captures images from a camera, tags them with metadata, compresses them, then writes them to disk. It's not hard to see how this program could be structured as a series of four modules connected together in a pipeline.

So, pipelining's a neat idea, and it's used quite commonly. Unfortunately though, there's one area where it isn't used anywhere near as much as it should be: the lower-level implementation details of programs.

An example
----------

Say we've got a list of student objects, and we want to retrieve the email addresses of the ten youngest first-year CS students. How do we go about doing this? Most programmers would probably write something along these lines:

``` ruby
emails = []
sorted_students = students.sort_by(&:age)

for student in sorted_students
  if student.level == 1 && student.degree == "CS"
    email = student.email
    emails.push(email)
  end
  if emails.length == 10
    break
  end
end

return emails
```

This code works, but it's difficult to read, error-prone, and relatively verbose. If we instead try to structure our solution as a pipeline, we'll see that what we really want to do is:

1. Start with the list of all students.
2. Select the first-year students.
3. Select the students enrolled in CS degrees.
4. Take the 10 youngest students.
5. Retrieve their email addresses.

Translating this into Ruby, we get:

``` ruby
students.select { |s| s.level == 1 }
        .select { |s| s.degree == "CS" }
        .sort_by(&:age)
        .take(10)
        .collect(&:email)
```

Ruby is particularly well suited to this style of coding, but it's also practical in many other languages. In Python for example, list comprehensions and the `itertools` module will come in handy.

It shouldn't be hard to see that the second code snippet is a big improvement over the first. It's easier to understand, easier to modify, and is therefore less error-prone. It also has the *potential* to run more efficiently (the runtime system could theoretically distribute the `select` and `collect` operations over multiple cores).

The difference here is really procedural vs functional, or more broadly, imperative vs declarative. The functional pipeline approach is declarative because we're describing *what* we want, rather than spelling out the details of *how* to achieve it.

Conclusion
----------

The next time you think about writing a for-loop or an if-statement, consider whether you'd be better served by a `collect` or a `select` ;)
