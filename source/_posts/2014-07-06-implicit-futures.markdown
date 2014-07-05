---
layout: post
title: "Implicit Futures"
date: 2014-07-06 02:20:05 +0930
comments: true
categories: [concurrency, ruby, futures]
---

Lorem ipsum.

``` ruby
class Future
  def initialize(&computation)
    @thread = Thread.new { computation.call }
  end
  
  def method_missing(name, *args, &block)
    @thread.value.send(name, *args, &block)
  end
end
```
