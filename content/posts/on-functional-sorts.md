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
the SML standard's standard library if you will. Yet, there was no mention of `sort` or anything similar. 
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
This is adapted from a simple solution I found online, the key parts is, of course, this being a mergesort,
and that it uses `splitAt`, another not standard function. This one is even easier to implement though.
Most importantly, it sorts, and it sorts fine enough. 
My current usecase is a couple of tiny lists that I sort once.
Really, even a best-case `O(n^3)` cubic sorting algorithm would work great. 
Hell, even the shuffle and pray of bogosort would work fine. 

This excuse for a post did make me think, however. 
What would be the best sorting algorithm for typical functional lists, 
i.e. immutable singly linked lists? And why not just benchmark them all in SML for fun.

## The Choices laid out
Well there is a long list of `sort`s to consider, but I will keep to a couple of known ones:
- Mergesort, supposedly better for functional languages
- Quicksort, with its many variations
- Treesort, since trees are nice
- Heapsort, for seeing how it would turn out
- Insertion sort, for being great
- Selection sort, for being `O(n^2)`
- Bubble sort, for being known
- Radix sort, for lack of comparisons
- Slowsort and bogosort, to have a terrific baseline

I will exclude e.g. Bucket sort and others that add a lot of constraints. 
The only exception to that would be Radix sort (the cooler Bucket), just so it is not all comparison-based sorts.
I will also try out different variations for some of these. 
As much as I can, but mostly for common options.
For the slowsort and bogosort, timeout shall exist for sanity-related reasons.

Note that I will use lists for all algorithms, for pure input and output.
SML does have immutable and mutable arrays and does allow direct mutation unlike e.g. Haskell,
but it would be no fun to compare a mix of in-place array algorithms vs linked list algorithms.
However, I will have quicksort with both list and array solutions. 
This will still allow us to compare how well arrays would do.

I will be benchmarking them in SML, though there are caveats.
Firstly, since I will be mostly doing these by hand, and I am not an SML expert,
there could be defects in implementations.
These defects are especially likely where I will be adapting array algorithms to lists.
Naturally, the implementations might not be the best, especially in relation to functional languages.
I will try to do best reasonable effort and keep algorithms relatively simple.

Additionally, note that I will be using Poly/ML implementation of SML.
Given whatever optimizations and quirks Poly/ML does and does not, 
these results may not apply directly to other implementation of SML, 
much less other functional languages. 
Still, worst, average, and best-cases are likely to dominate over my implementation specifics.
Lastly, benchmarking is easy to mess up in general, keep that in mind.
Primary goal is fun and humane accuracy.

I will cover most algorithms here, though by no means equally. 
You can see the full implementations at the [repo](TBD) for this project and in the benchmark results.

## Onto the Algorithms
### Split and Merge
We have already covered the basic merge sort from above. 
It is by no means complex, especially in the functional style.
Generally most functional languages seem to use the merge sort as the prefered sorting algorithm.
It is supposedly very fitting for lists, and it is stable too.
Note that I said "basic" however.

```sml
fun sort _ [] = []
  | sort _ [x] = [x]
  | sort ge xs = merge ge ((fn (xs, ys) => (sort ge xs, sort ge ys))
                               (splitAt xs ((List.length xs) div 2)))
```

Looking at just the sort part, indeed, this merge sort does `splitAt`, an expensive operation. 
By itself it is `O(n/2)`, but we continue to split the list in half on each recursive call. 
Thus, we get `O(nlogn)` as we keep halving `n` `logn` times.
We have yet to merge and we already have to do `O(nlogn)` work, unideal.

There is, thankfully, a solution. We can do a bottom-up:
```sml
fun merge _ (xs, []) = xs
  | merge _ ([], xs) = xs
  | merge ge (x::xs, y::ys) = if ge (y, x)
                              then x :: (merge ge (xs, y::ys))
                              else y :: (merge ge (x::xs, ys))

fun combiner [] = []
  | combiner [x] = [(x, [])]
  | combiner (x::y::xs) = (x,y)::(combiner xs)

fun merger _ [] = []
  | merger _ [(x, [])] = x
  | merger ge xs = merger ge (combiner (List.map (merge ge) xs) [])

fun sort _ [] = []
  | sort _ [x] = [x]
  | sort ge xs = merger ge (combiner (List.map (fn x => [x]) xs) [])
```
Merge part can remain, but our splitting step is now much nicer, at the cost of code size.
In some ways this is even more indicative of the "real" functional programming with maps and bunch of functions.
Now this is a more proper merge sort. Still `O(nlogn)`, but now we don't have to do as much work. [](TODO:.RIGHT.VS.LEFT.FOLDS)
Still, we can do better with a "natural" optimization:
```sml
fun extractAsc ge [] = ([], [])
  | extractAsc ge [x] = ([x], [])
  | extractAsc ge (x::y::xs) =
    if ge (y, x) then
        let val (run, rest) = extractAsc ge (y::xs)
        in (x :: run, rest)
        end
    else ([], x::y::xs)

fun extractDes ge [] acc = acc
  | extractDes ge [x] (acc, rest) = (x::acc, rest)
  | extractDes ge (x::y::xs) (acc, rest) =
    if ge (y, x) then (acc, x::y::xs)
    else extractDes ge (y::xs) (x::acc, rest)

fun natural ge [] = []
  | natural ge [x] = [[x]]
  | natural ge (x::y::xs) = if ge (y, x) then
                                let val (run, rest) = extractAsc ge (y::xs)
                                in (x::run)::(natural ge rest)
                                end
                            else
                                let val (run, rest) = extractDes ge (x::y::xs) ([], [])
                                in run::(natural ge rest)
                                end

fun sort _ [] = []
  | sort _ [x] = [x]
  | sort ge xs = merger ge (combiner (natural ge xs))
```
Other parts remain unchanged. Our merge sort is a little longer now, it is probably worse on pure random inputs too. 
However, the more sorted (including reverse-sorted), chunks there are, the more this approaches `O(n)` as we simply do less of the actual merge sort.
On the fully sorted sequence, we are done before we split or merge a single time.
[](TODO:.WHAT.ABOUT.MAPPING-AS-WE-GO)

### Quicksort ain't quick
What better place to start with than the legendary haskell quicksort solution:
```haskell
qsort []     = []
qsort (p:xs) = qsort lesser ++ [p] ++ qsort greater
    where
        lesser  = filter (< p) xs
        greater = filter (>= p) xs
```
Tiny, simple, functional, elegent, degenerate, and more so infamous than legendary. 
I am obliged to point out that there many places to tell you how bad it is.
The double filter instead of a single partition, the expensive appends, the always left pivot. 
It is not at all in-place of course, so some don't even consider it a real quicksort.

Now, do note that it does work, it will sort all of your lists. 
By the time it is the problem you should probably be using arrays anyway.
Still, it is greatly inefficient. Some of the defficiencies are super easy to fix too.
Have to include it nonetheless, so here is the SML version:
```sml
fun sort _ [] = []
  | sort _ [x] = [x]
  | sort ge (p::xs) =
    let val lesser  = List.filter (fn x => not (ge (p, x))) xs
        val greater = List.filter (fn x => ge (p, x)) xs
    in (sort ge lesser) @ [p] @ (sort ge greater)
    end
```
There are slight differences, but nothing serious.
We can move on to the simple improvements:
```sml
fun sort _ [] = []
  | sort _ [x] = [x]
  | sort ge (p::xs) =
    let val (lesser, greater) = List.partition (fn x => ge (p, x)) xs
    in (sort ge lesser) @ [p] @ (sort ge greater)
    end
```
There is already a partition function that gathers trues in one list and falses in the other.
It is perfectly suited for this case while being a single optimized call.

Now there are other minor fixes that we can do, but I wanted something more. 
Thus, I was able to find a different version of quicksort on [LiterateProgramming wiki](https://www.literateprograms.org/quicksort__haskell_.html),
where the implementation uses accumulators to improve upon even merge sort from GHC (the main Haskell implementation):
```haskell
qsort3' [] y     = y
qsort3' [x] y    = x:y
qsort3' (x:xs) y = part xs [] [x] []
    where
        part [] l e g = qsort3' l (e ++ (qsort3' g y))
        part (z:zs) l e g 
            | z > x     = part zs l e (z:g) 
            | z < x     = part zs (z:l) e g 
            | otherwise = part zs l (z:e) g
```
It is relatively small and nice for what is supposedly better than GHC's merge sort.
Note that the wiki only tested large lists of random numbers and that was on an ancient GHC and Apple Powerbook[^1].
Still, let us adapt that to SML:
```sml
fun qsort ge [] acc = acc
  | qsort ge [x] acc = x::acc
  | qsort ge (x::xs) acc = part ge x acc xs [] [x] []

and part ge _ acc [] l e g = qsort ge l (e @ (qsort ge g acc))
  | part ge x acc (y::ys) l e g =
    if ge (y, x)
    then part ge x acc ys l e (y::g)
    else part ge x acc ys (y::l) e g

fun sort ge xs = qsort ge xs []
```
It is not identical, differing in a few ways (e.g. we have `ge` rather than `>` and `<`).
`part` is now a separate but mutually recursive function too.
Nevertheless, the form is much the same.

[^1]: Ignoring the Powerbook since all kinds of devices are used nowadays, the GHC version was 6.4.1, released September 19 2005.
I did check what kind of merge sort GHC had in that version, and it seemed to be a simple bottom up solution without natural runs.
These days, GHC has a beefy 4-way bottom-up merge sort with natural runs, not to mention many optimizations in the compiler itself since then.
