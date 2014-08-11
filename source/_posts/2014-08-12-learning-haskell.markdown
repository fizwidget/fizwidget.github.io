---
layout: post
title: "Learning Haskell"
date: 2014-08-12 07:54:57 +0930
comments: true
categories: [functional programming, haskell]
published: false
---

``` haskell
fibs = 0 : 1 : zipWith (+) fibs (tail fibs)
```

``` haskell
primes = sieve [2..]
  where sieve (p:xs) = 
    p : sieve [x | x <- xs, x `mod` p /= 0]
```

``` haskell
data Color = Red | Green | Blue
data Person = Person String String Integer
data Tree a = Leaf | Node (Tree a) a (Tree a)
```
