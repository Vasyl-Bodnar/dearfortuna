+++
author = "V"
title = "On Functional Sorts"
date = "2026-07-10T18:00:00-04:00" 
description = "Writing and benchmarking multiple sorting algorithms in SML"
tags = ["Sorting", "SML", "Benchmark"]
draft = true
+++

## Intro
I recently was playing with SML and was in need of a sort function for lists. 
I checked the [List structure](https://smlfamily.github.io/Basis/list.html) in the Basis, 
the SML standard library if you will. Yet, there was no mention of `sort` or anything similar. 
Had to read through each item multiple times, just in case. Still none.

Alright, something was off. A quick search into a [Stack Overflow article](https://stackoverflow.com/questions/14411862/standard-sorting-functions-in-sml).
Indeed, Basis does not have a sort function. A poor decision in my opinion, but it is done. 
Thankfully, many implementations do define a sort function in their standard libraries.
But:
> In Poly/ML, there is no library for sorting, so you have to define your own sorting function.

And I happen to be enjoying and using [Poly/ML](https://polyml.org/), so there goes that.
I even checked the [Poly/ML standard library](https://polyml.org/documentation/Reference/Basis.html) in case they added it recently, 
but it does not seem to be the case.
Oh well, have to just do it myself:
```sml
fun merge _ (xs, []) = xs
  | merge _ ([], xs) = xs
  | merge ge (x::xs, y::ys) = if ge (y, x)
                              then x :: (merge ge (xs, y::ys))
                              else y :: (merge ge (x::xs, ys))

fun sort _ [] = []
  | sort _ [x] = [x]
  | sort ge xs = merge ge ((fn (xs, ys) => (sort ge xs, sort ge ys))
                               (splitAt xs ((List.length xs) div 2)))
```
This is adapted from a solution I found online, the key parts is of course this being a merge sort 
and that it uses `splitAt` which is also not standard. That one is even easier to implement though.
Most importantly, it sorts, and it sorts fine enough. 
I have a couple of tiny lists that I sort once.
For that usecase, even a best-case O(n^3) cubic sorting algorithm would work. 
Hell, even bogosort would work. 

This did make me think, however. What would be the best sorting algorithm for typical functional lists, 
i.e. immutable singly linked lists.
