---
layout: post
title: "Haskell Awesomeness"
date: 2014-08-12 07:54:57 +0930
comments: true
categories: [functional programming, haskell]
published: false
---

``` haskell
main = interact (unlines . map reverse . lines)
```

This snippet of code reads lines from standard input, reverses them, and prints them to standard output.

If I were to run the program in my terminal, I might get this output:

``` plain
> Hey there!
!ereht yeH
> Well, it seems to be working.
.gnikrow eb ot smees ti ,lleW
```

Let's break it down and examine how it works...

`main` is what you'd expect - it's the function that runs when the program is launched. `interact` is a function defined in Haskell's standard `Prelude` library. It's a higher-order function because it accepts a function as its input.

The above snippet demonstrates a few cool features of Haskell:

1. Higher-order functions.
2. Partial function application.
3. Function composition notation.
4. Lazy evaluation.
5. Type interence.
6. Recursion instead of loops.
7. No variables, only constants.
8. Pure functions.


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
