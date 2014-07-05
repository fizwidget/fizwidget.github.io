---
layout: post
title: "Functional Pipelines"
date: 2014-07-05 12:45:10 +0930
comments: true
categories: [functional programming]
---

Let's say we've got a list of student objects, and we want to get the ID numbers of the 10 youngest first-year students. Perhaps to send them a survey or something. I dunno.

The point is, how do we do it? If you'd asked me a few years ago, I probably would've written something horrible like this:

<!-- more -->

``` ruby
youngest_ids = []

youngest_to_oldest = students.sort_by { |s| s.age }
i = 0

for student in youngest_to_oldest
  if student.level == 1
    youngest_ids.push(student.id_number)
    i += 1
  end
  if i == 10
    break
  end
end

return youngest_ids
```

If you asked me today, I'd write something more like this:

``` ruby
students.select  { |s| s.level == 1 }
        .sort_by { |s| s.age }
        .take(10)
        .collect { |s| s.id_number }
```

This is a much, much better approach. The main difference is that it's declarative rather than imperative. We're describing what we want, rather than spelling out exactly how to achieve it. This approach is easier to understand, easier to modify, less error-prone, more concise, and has the *potential* to perform better (the system could, for example, distribute the `select` operation over multiple cores).

I've used Ruby in the above example, but you can write this kind of code in many different languages (Python, C#, and Java 8 to name a few of the more popular ones). You might also notice similarities to SQL - this is no coincidence, as SQL is also a declarative language.

So go forth and be functional!
