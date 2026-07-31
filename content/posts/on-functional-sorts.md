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
This is adapted from a simple solution I found online, the key parts is, of course, this being a merge-sort,
and that it uses `splitAt`, another not standard function. This one is even easier to implement though.
Most importantly, it sorts, and it sorts fine enough. 
My current use case is a couple of tiny lists that I sort once.
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
- Insertion sort, for being great
- Selection sort, for being `O(n^2)`
- Bubble sort, for being known
- Radix sort, for lack of comparisons
- Bogosort, to have a terrific baseline

I will exclude e.g. Bucket sort and others that add a lot of constraints. 
The only exception to that would be Radix sort (the cooler Bucket), just so it is not all comparison-based sorts.
I will exclude other algorithms like heapsort that depend heavily on the array and would do extremely poorly without it.
I will also try out different variations for some of these. 
Naturally, as much as I can, but mostly for common options.
For the bogosort and few others, timeout shall exist for sanity-related reasons.

Note that I will use lists for most algorithms, for pure input and output.
SML does have immutable and mutable arrays and does allow direct mutation unlike e.g. Haskell,
but it would be no fun to compare a mix of in-place array algorithms vs linked list algorithms.
However, I will promise quicksort with both list and array solutions. 
This will still allow us to compare how well arrays would do.

I will be benchmarking them in SML, though there are caveats.
Firstly, since I will be mostly doing these by hand, and I am not an SML expert,
there could be defects in implementations.
These defects are especially likely where I will be adapting array algorithms to lists.
Naturally, the implementations might not be the best, especially in relation to functional languages.
Still, worst, average, and best-cases are likely to dominate over my implementation specifics.
I will try to do best reasonable effort and keep algorithms relatively simple.

Additionally, note that I will be using Poly/ML implementation of SML.
Given whatever optimizations and quirks Poly/ML does and does not, 
these results may not apply directly to other implementation of SML, 
much less other functional languages. 
Lastly, benchmarking is easy to mess up in general, keep that in mind.
Primary goal is fun and humane accuracy.

I will cover most algorithms here, though by no means equally. 
I will also include some benchmarks from the REPL testing, though they will only be on large either random or sorted input.
You can see the full implementations at the [repo](TBD) for this project and in the benchmark results.

## Onto the Algorithms
### Wise man has nothing to merge
We have already covered the basic mergesort from above. 
It is by no means complex, especially in the functional style.
Generally most functional languages seem to use the mergesort as the preferred sorting algorithm.
It is supposedly very fitting for lists, and it is stable too.
Note that I said "basic" however.

```sml
fun sort _ [] = []
  | sort _ [x] = [x]
  | sort ge xs = merge ge ((fn (xs, ys) => (sort ge xs, sort ge ys))
                               (splitAt xs ((List.length xs) div 2)))
```

Looking at just the sort part, indeed, this mergesort does `splitAt`, an expensive operation. 
By itself it is `O(n/2)`, but we continue to split the list in half on each recursive call. 
Thus, we get `O(nlogn)` as we keep halving `n` `logn` times.
We have yet to merge and we already have to do `O(nlogn)` work, not ideal.

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
Now this is a more proper mergesort. Still `O(nlogn)`, but now we don't have to do as much work.
We can do better with a "natural" optimization though:
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
Other parts remain unchanged. Our mergesort is a little longer now, it is probably worse on pure random inputs too. 
However, the more sorted (including reverse-sorted) chunks there are, the more this approaches `O(n)` as we simply do less of the actual mergesort.
On the fully sorted sequence, we are done before we split or merge a single time. 
There are still a few other potential though minor improvements we can try, but this is plenty good.

### Quicksort ain't quick
What better place to start with than the legendary Haskell quicksort solution:
```haskell
qsort []     = []
qsort (p:xs) = qsort lesser ++ [p] ++ qsort greater
    where
        lesser  = filter (< p) xs
        greater = filter (>= p) xs
```
Tiny, simple, functional, elegant, degenerate, and more so infamous than legendary. 
I am obliged to point out that there are many places to tell you how bad it is.
The double filter instead of a single partition, the expensive appends, the always left pivot. 
It is not at all in-place of course, so some don't even consider it a real quicksort.

Now, do note that it does work, it will sort all of your lists. 
By the time it is the problem you should probably be using arrays anyway.
Still, it is greatly inefficient. Some of the deficiencies are super easy to fix too.
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
We can move on to the improvements:
```sml
fun sort _ [] = []
  | sort _ [x] = [x]
  | sort ge (p::xs) =
    let val (lesser, greater) = List.partition (fn x => ge (p, x)) xs
    in (sort ge lesser) @ [p] @ (sort ge greater)
    end
```
There is already a partition function that gathers trues in one list and falsies in the other.
It is perfectly suited for this case while being a single optimized call.

Now there are other minor fixes that we can do, but I wanted something more. 
Thus, I was able to find a different version of quicksort on [Literate Programming wiki](https://www.literateprograms.org/quicksort__haskell_.html),
where the implementation uses accumulators to improve upon even mergesort from GHC (the main Haskell implementation):
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
It is relatively small and nice for what is supposedly better than GHC's mergesort.
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

Then, I shall introduce a snippet of the benchmark for just these functions with random int list of n=10000:
| Algorithm                   | Mean    | StdDev  | Err     |
|-----------------------------|---------|---------|---------|
| Basic mergesort             | 2.43 ms | 0.75 ms | 0.33 ms |
| Bottom-up mergesort         | 2.18 ms | 1.03 ms | 0.46 ms |
| Natural bottom-up mergesort | 2.60 ms | 0.38 ms | 0.17 ms |
| Bad quicksort               | 3.25 ms | 0.16 ms | 0.07 ms |
| Partition quicksort         | 2.70 ms | 0.38 ms | 0.17 ms |
| Accumulator quicksort       | 1.64 ms | 0.30 ms | 0.13 ms |

This is time, so lower is better. 
As you can see, on fully random input and on a somewhat large input size,
I was able to get a quicksort to outperform mergesort just like in the article!
Even the partition version is comparably fast.
However, this is the best-case scenario for quicksort, because if input is sorted:
| Algorithm                   | Mean      | StdDev    | Err       |
|-----------------------------|-----------|-----------|-----------|
| Basic mergesort             | 1.96 ms   | 0.56 ms   | 0.25 ms   |
| Bottom-up mergesort         | 1.71 ms   | 0.40 ms   | 0.18 ms   |
| Natural bottom-up mergesort | 0.18 ms   | 0.04 ms   | 0.02 ms   |
| Bad quicksort               | 997.12 ms | 244.28 ms | 109.24 ms |
| Partition quicksort         | 775.36 ms | 57.45 ms  | 25.71 ms  |
| Accumulator quicksort       | 345.93 ms | 25.49 ms  | 11.40 ms  |

The bad version is taking nearly a second on the input of mere 10000 elements.
Even the accumulator version, which easily beats mergesort on random input, is two orders worse now.
You might be able to see the issue with "functional" quicksort.
You can also see the benefit of the natural version. 
Performance is up by an order for the sorted input, 
whereas the cost on the random input is not as high.

### Arrays?
What if we instead try the quicksort array solution:
```sml
fun part ge arr lo hi =
    let val p = Array.sub (arr, (hi + lo) div 2)
        val lo = ref (lo - 1)
        val hi = ref (hi + 1)
        val ret = ref true
    in while !ret do (
            lo := !lo + 1;
            while not (ge (Array.sub (arr, !lo), p)) do
                  lo := !lo + 1;
            hi := !hi - 1;
            while not (ge (p, Array.sub (arr, !hi))) do
                  hi := !hi - 1;
            if !lo >= !hi then
                ret := false
            else
                let val l = Array.sub (arr, !lo)
                    val h = Array.sub (arr, !hi)
                in
                    Array.update (arr, !lo, h);
                    Array.update (arr, !hi, l)
                end
        );
       !hi
    end

fun qsort ge arr lo hi =
    if lo >= hi orelse lo < 0 orelse hi < 0 then ()
    else
        let val p = part ge arr lo hi
        in
            qsort ge arr lo p;
            qsort ge arr (p + 1) hi
        end

fun sort ge arr = qsort ge arr 0 (Array.length arr - 1)
```
This is the Hoare's solution, though note that the pivot is the middle element.
While it would be more fair to keep pivot to the first element like in the functional quicksorts,
the ability to pick the middle element is exactly the big difference between them.
Middle element with linked lists is O(n) and with arrays O(1).
Although, you might be surprised with random input (still n=10000):
| Algorithm                   | Mean    | StdDev  | Err     |
|-----------------------------|---------|---------|---------|
| Natural bottom-up mergesort | 2.71 ms | 0.24 ms | 0.11 ms |
| Accumulator quicksort       | 2.06 ms | 0.45 ms | 0.20 ms |
| List array quicksort        | 2.19 ms | 0.28 ms | 0.12 ms |
| Array quicksort             | 1.08 ms | 0.17 ms | 0.07 ms |

I also included the solution where a list is converted into an array, 
sorted using the array quicksort, 
and then converted back into a list.

Naturally, the array quicksort is nearly three times faster than the accumulator version. 
However, I expected a much larger difference.
These are arrays vs linked lists need I remind you.
Not sure what Poly/ML does in the background, possibly related to these being polymorphic arrays. 
Though, it is enough of a difference where converting to and from array is not a bad solution.
Regardless, the key difference can be seen in the sorted input:
| Algorithm                   | Mean      | StdDev    | Err      |
|-----------------------------|-----------|-----------|----------|
| Natural bottom-up mergesort | 0.13 ms   | 0.00 ms   | 0.00 ms  |
| Accumulator quicksort       | 449.95 ms | 151.97 ms | 67.96 ms |
| List array quicksort        | 0.94 ms   | 0.01 ms   | 0.00 ms  |
| Array quicksort             | 0.94 ms   | 0.03 ms   | 0.01 ms  |

With the middle pivot choice, array quicksort on sorted input is even slightly better than on random.
Accumulator quicksort cannot begin to compare. 
You can also see the benefit of the natural mergesort again, four times faster than the array version.
Doing barely any work on lists is better than doing lots on arrays after all.
Note that the list array quicksort is slightly slower, but even without rounding (nearest number for the curious) it is quite close

### Building the Forest
Another interesting algorithm is treesort. 
In some ways it is quite elegant:
```sml
functor TreeSort(Tree : TREE) :> LIST_SORT = struct
fun sort ge xs = Tree.inorder (Tree.produce ge xs)
end
```
This example includes a functor that takes in some tree structure to produce a new module. 
Very handy for cases like these. 
Naturally, we do need to define `produce` and `inorder` in some tree structure to pass in.
These are simple functions at their core. 
`produce` takes a list and returns a tree,
`inorder` takes a tree and returns a list.

A good start would certainly be the humble binary tree:
```sml
fun insert _ x Nil = Node (Nil, x, Nil)
  | insert ge x (Node (l, y, r)) =
    if ge (x, y) then
        Node (l, y, insert ge x r)
    else
        Node (insert ge x l, y, r)

fun produce ge xs = List.foldl (fn (x, t) => insert ge x t) Nil xs

fun inorder Nil = []
  | inorder (Node (l, v, r)) = (inorder l) @ (v::(inorder r))
```
Indeed, very simple. 
The main function we care about is `insert` in this case, since `produce` and `inorder` are trivial.
While this tree is capable, one big issue is that it is not *self-balancing*.
A sorted input will turn it into a linked list which loses all the benefits of the binary tree.
For this reason, I shall also include a Splay tree in bottom-up fashion:
```sml
fun rotLeft Nil = Nil
  | rotLeft (p as Node (l, x, Nil)) = p
  | rotLeft (Node (l, x, Node (rl, rx, rr))) = (Node (Node (l, x, rl), rx, rr))

fun rotRight Nil = Nil
  | rotRight (p as Node (Nil, x, r)) = p
  | rotRight (Node (Node (ll, lx, lr), x, r)) = (Node (ll, lx, Node (lr, x, r)))

fun splay ge [] = Nil
  | splay ge [n] = n
  | splay ge [s as Node (l, x, r), Node (pl, px, pr)] =
    if ge (x, px) then
        rotLeft (Node (pl, px, s))
    else
        rotRight (Node (s, px, pr))
  | splay ge ((s as Node (l, x, r))::Node (pl, px, pr)::Node (gl, gx, gr)::ts) =
    (case (ge (x, px), ge (px, gx)) of
         (true, true) => 
         splay ge (rotLeft (Node (gl, gx, rotLeft (Node (pl, px, s))))::ts)
       | (true, false) => 
         splay ge (rotRight (Node (rotLeft (Node (pl, px, s)), gx, gr))::ts)
       | (false, true) => 
         splay ge (rotLeft (Node (gl, gx, rotRight (Node (s, px, pr))))::ts)
       | (false, false) => 
         splay ge (rotRight (Node (rotRight (Node (s, px, pr)), gx, gr))::ts))
  | splay ge _ = raise Impossible


fun insert' _ x Nil acc = (Node (Nil, x, Nil))::acc
  | insert' ge x (Node (l, y, r)) acc =
    if ge (x, y) then
        insert' ge x r ((Node (l, y, r))::acc)
    else
        insert' ge x l ((Node (l, y, r))::acc)

fun insert ge x t = splay ge (insert' ge x t [])
```
The code is fundamentally simple, though there is a number of cases and auxiliaries.
In some ways it improves upon the humble non-balancing tree, in others, it degrades. 
Firstly, Splay trees can still turn into mostly linked lists if balancing is unlucky.
Additionally, the rotations are a lot of extra work over the plain tree.
This becomes a significant trade off.

However, we can reduce that work by making a top-down algorithm. 
Top-down we would only need to go once through the tree instead of twice like in the bottom-up.
My solution is a little messy but it is based on a standard triple trees method:
```sml
fun insertLeftmost n Nil = n
  | insertLeftmost n (Node (l, x, r)) = (Node (insertLeftmost n l, x, r))

fun insertRightmost n Nil = n
  | insertRightmost n (Node (l, x, r)) = (Node (l, x, insertRightmost n r))

fun rotLeft Nil left right = (Nil, left, right)
  | rotLeft (Node (l, x, r)) left right = (r, insertRightmost (Node (l, x, Nil)) left, right)

fun rotRight Nil left right = (Nil, left, right)
  | rotRight (Node (l, x, r)) left right = (l, left, insertLeftmost (Node (Nil, x, r)) right)

fun insert' _ x Nil left right = Node (left, x, right)
  | insert' ge x (Node (l, y, r)) left right =
    if ge (x, y) then
        let val (mid, left, right) = rotLeft (Node (l, y, r)) left right
            val (r, left, right) = case r of
                        Nil => (Nil, left, right)
                      | Node (rl, ry, rr) =>
                        if ge (x, ry) then
                            rotLeft (Node (rl, ry, rr)) left right
                        else
                            rotRight (Node (rl, ry, rr)) left right
        in
            insert' ge x r left right
        end
    else
        let val (mid, left, right) = rotRight (Node (l, y, r)) left right
            val (l, left, right) = case l of
                        Nil => (Nil, left, right)
                      | Node (ll, ly, lr) =>
                        if ge (x, ly) then
                            rotLeft (Node (ll, ly, lr)) left right
                        else
                            rotRight (Node (ll, ly, lr)) left right
        in
            insert' ge x l left right
        end

fun insert ge x t = insert' ge x t Nil Nil
```
Overall, this should improve the results, though unlikely to compete with the good sorts.
There could be some other trees to try. I even considered B+, but the ideal solution will definitely betray the functional style.
Still, we need to test how binary and Splay would be doing, starting with random values:
| Algorithm                   | Mean    | StdDev  | Err     |
|-----------------------------|---------|---------|---------|
| Natural bottom-up mergesort | 3.42 ms | 0.84 ms | 0.38 ms |
| Simple treesort             | 2.98 ms | 0.29 ms | 0.13 ms |
| Splay treesort              | 8.24 ms | 0.83 ms | 0.37 ms |
| Top-down Splay treesort     | 8.31 ms | 1.30 ms | 0.58 ms |
| Array quicksort             | 1.11 ms | 0.09 ms | 0.04 ms |

Simple binary tree does quite well, even faster than our mergesort. 
Splay trees do a lot of extra work, which is not necessary with random data.
Top-down came out slower here, so I have to start blaming my implementation.
Array quicksort is as fast as always.

Now, the more tricky question is sorted input:
| Algorithm                   | Mean      | StdDev   | Err      |
|-----------------------------|-----------|----------|----------|
| Natural bottom-up mergesort | 0.21 ms   | 0.03 ms  | 0.01 ms  |
| Simple treesort             | 591.08 ms | 18.85 ms | 8.43 ms  |
| Splay treesort              | 377.89 ms | 10.28 ms | 4.60 ms  |
| Top-down Splay treesort     | 397.07 ms | 23.74 ms | 10.62 ms |
| Array quicksort             | 1.17 ms   | 0.24 ms  | 0.11 ms  |

Yeah, not great. 
Splay trees are nearly twice as fast as the binary tree at least. 
My top-down is slower again. 
Other solutions are as fast as you expect.

Fun thing about the reverse sorted input (i.e. descending):

> Process poly killed

Oops, when testing on my REPL, it did not even finish.
The issue happened in the Splay tree seemingly going on infinitely. 
This is either my error or a limit to the bottom-up for PolyML here, as it does relatively well on smaller inputs.
Anyway, let us just exclude it for now, we will consider it later anyway:
| Algorithm                   | Mean      | StdDev   | Err      |
|-----------------------------|-----------|----------|----------|
| Natural bottom-up mergesort | 0.83 ms   | 0.08 ms  | 0.04 ms  |
| Simple treesort             | 958.55 ms | 33.19 ms | 14.84 ms |
| Top-down Splay treesort     | 0.37 ms   | 0.04 ms  | 0.02 ms  |
| Array quicksort             | 1.41 ms   | 0.03 ms  | 0.01 ms  |

What I wanted to display is how fast the top-down splay treesort is for this specific use case. 
Likely a quirk of my implementation and the way recursion occurs in some places. 
It is completely different from input sorted ascending to begin with.
An interesting anomaly nonetheless. 
However, these treesorts have proven themselves quite incompetent for a general case.
Thankfully, we still have more algorithms to test.

### Square into `nlogn` hole
We shall take a look at insertion sort, selection sort, and bubble sort. 
These are relatively simple algorithms, which is why they are quire popular.
Starting with insertion sort:
```sml
fun insertion ge [] ys = ys
  | insertion ge (x::xs) ys =
    let val (ys, b) = List.foldr (fn (y, (acc, b)) =>
                                     if b andalso ge (x, y)
                                     then (y::x::acc, false)
                                     else (y::acc, b)) ([], true) ys
    in if b
       then insertion ge xs (x::ys)
       else insertion ge xs ys
    end

fun sort ge [] = []
  | sort ge [x] = [x]
  | sort ge xs = insertion ge xs []
```
The implementation is indeed simple (can also be done with partition if you want compactness at a cost of some speed).
However, performance leaves much to be desired even in the quick checks.
How about selection sort then:
```sml
fun selection ge [] ys = ys
  | selection ge (x::xs) ys =
    let val (min, rest) = List.foldl (fn (x, (y, acc)) =>
                                         if ge (x, y) then
                                             (x, y::acc)
                                         else
                                             (y, x::acc))
                                     (x, []) xs
    in selection ge rest (min::ys)
    end

fun sort ge [] = []
  | sort ge [x] = [x]
  | sort ge xs = selection ge xs []
```
Similarly complicated, and should in general be comparable.
Performance is not great.
Maybe the bubble sort is the solution:
```sml
fun bubble ge [] = ([], false)
  | bubble ge [x] = ([x], false)
  | bubble ge (x::y::xs) =
    if ge (x, y) then
        let val (l, _) = bubble ge (x::xs)
        in (y::l, true)
        end
    else
        let val (l, b) = bubble ge (y::xs)
        in (x::l, b)
        end

fun sort ge [] = []
  | sort ge [x] = [x]
  | sort ge xs =
    let val (xs, b) = bubble ge xs
    in if b then
           sort ge xs
       else
           xs
    end
```
A little more complicated in some ways. Generally comparable still.
Performance is, uhh, how about the benchmarks:
| Algorithm                   | Mean      | StdDev    | Err      |
|-----------------------------|-----------|-----------|----------|
| Natural bottom-up mergesort | 3.52 ms   | 0.55 ms   | 0.24 ms  |
| Insertion sort              | 750.23 ms | 76.92 ms  | 34.40 ms |
| Selection sort              | 703.18 ms | 121.30 ms | 54.25 ms |
| Array quicksort             | 1.47 ms   | 0.33 ms   | 0.15 ms  |

Bubble sort is not included. It is so slow that I can't run it on input of a 1000 numbers, 
and these are n=10000 need I remind you.
Thus, similarly to bottom-up splay tree, either there is a mistake in my port to functional style or it genuinely is that bad.

Regardless, insertion and selection are quite disappointing, predictably so however, given the algorithms.
Insertion is a little better on sorted input (500 ms from a quick test), 
but there isn't even a point in showing a table for that alone.
Overall, I am heavily disappointed with the `O(n^2)` algorithms for the test inputs.
Although, it is hard to be surprised when I am adapting what would otherwise be at least in-place algorithms on arrays.

### Shall not compare
Thankfully, we have what are considered by many the superiour sorting algorithm, 
as long as your values are easy to bucket, the Radix sort.
Now, buckets generally favor arrays heavily, since O(1) is unbeatable there.
For the sake of fairness I included two solutions, both are LSB for some number of bits.
One will be in pure lists, the other will have an array for buckets, but nothing else.
Onto the list one:
```sml
functor RadixListSort(val bits : Word.word) :> INT_LIST_SORT = struct
fun radix xs i =
    if i = (0w64 div bits) then xs else
    let val shift = Word.*(i, bits)
        val mask = Word64.<<(Word64.-(Word64.<<(0w1, bits), 0w1), shift)
        val xs = List.map (fn x => (x, Word64.toInt (Word64.>>(
                                        Word64.andb(x, mask),
                                        shift)))) xs
        val xs = List.foldr (fn ((x, d), acc) =>
                                let val befor = List.take (acc, d)
                                    val after = List.drop (acc, d)
                                in case after of
                                       [] => befor @ [[x]]
                                     | l::ls => befor @ ((x::l)::ls)
                                end)
                            (List.tabulate(Word64.toInt (Word64.<<(0w1, bits)),
                                           fn _ => [])) xs
        val xs = List.concat xs
    in
        radix xs (Word.+(i, 0w1))
    end

fun sort _ [] = []
  | sort _ [x] = [x]
  | sort _ xs = List.map Word64.toInt
                         (radix (List.map Word64.fromInt xs) 0w0)
end
```
Another functor! When you see three in a casual use like this, you must consider it a good day.
This solution is general to the number of bits that are provided at construction. 
You can also see the big negative of lists showing up here, 
as I am forced to traverse many buckets until I can put the element in, rather than a simple `O(1)` of an array.
A `fold` can be used instead of `take` and `drop` for extra performance, but it is already complicated enough.

Anyway, the bit option opens a question of how many bits should we use for best performance.
Bits are typically a tradeoff of iterations vs space, in arrays at least. 
This is of course true for lists as well, but the balance will be different.
Thus, let us compare different bit sizes in a benchmark:
| Algorithm                   | Mean     | StdDev  | Err     |
|-----------------------------|----------|---------|---------|
| Natural bottom-up mergesort | 4.15 ms  | 0.50 ms | 0.22 ms |
| 2-bit List Radix sort       | 19.37 ms | 3.33 ms | 1.49 ms |
| 3-bit List Radix sort       | 14.67 ms | 3.82 ms | 1.71 ms |
| 4-bit List Radix sort       | 9.42 ms  | 0.72 ms | 0.32 ms |
| 5-bit List Radix sort       | 9.86 ms  | 0.33 ms | 0.15 ms |
| 6-bit List Radix sort       | 13.97 ms | 0.61 ms | 0.27 ms |
| 8-bit List Radix sort       | 41.12 ms | 2.07 ms | 0.92 ms |
| Array quicksort             | 1.05 ms  | 0.02 ms | 0.01 ms |

As one would expect for lists, having more iterations is not as bad as having more items to traverse.
4-bit version is 4 times faster than the 8-bit version. 
3 and 2-bit are, just like 5 and 6-bit, slower, at least a little bit. From this, 4-bit seems the ideal bit size for this version.
I did think about 16-bit once, but that one takes forever.

All of these versions are slower than mergesort and especially array quicksort.
However, unlike quadratics, at least these are in the same league.
Now, what about using the more proper style, with array buckets:
```sml
functor RadixArraySort(val bits : Word.word) :> INT_LIST_SORT = struct
local val buckets = Array.tabulate(Word64.toInt
                                       (Word64.<<(0w1, bits)),
                                   fn _ => [])
in
fun radix xs i =
    if i = (0w64 div bits) then xs else
    let val _ = Array.modify (fn _ => []) buckets
        val shift = Word.*(i, bits)
        val mask = Word64.<<(Word64.-(Word64.<<(0w1, bits), 0w1), shift)
        val _ = List.app (fn x =>
                             let val idx =
                                     Word64.toInt
                                         (Word64.>>(
                                               Word64.andb(x, mask),
                                               shift))
                             in Array.update (buckets, idx,
                                              x::(Array.sub (buckets, idx)))
                             end) xs
        val xs = Array.foldr (op List.revAppend) [] buckets
    in
        radix xs (Word.+(i, 0w1))
    end
end

fun sort _ [] = []
  | sort _ [x] = [x]
  | sort _ xs = List.map Word64.toInt
                         (radix (List.map Word64.fromInt xs) 0w0)
end
```
There are a couple of differences, but honestly not that many. 
The core of the algorithm is much the same. 
Big difference is buckets are an array instead of list.
Still, we need to know the best number of bits,
and there is a benchmark I can sell you:
| Algorithm                   | Mean     | StdDev  | Err     |
|-----------------------------|----------|---------|---------|
| Natural bottom-up mergesort | 3.74 ms  | 1.69 ms | 0.75 ms |
| 4-bit List Radix sort       | 12.70 ms | 2.08 ms | 0.93 ms |
| 2-bit Array Radix sort      | 6.23 ms  | 0.90 ms | 0.40 ms |
| 4-bit Array Radix sort      | 2.89 ms  | 0.40 ms | 0.18 ms |
| 6-bit Array Radix sort      | 1.62 ms  | 0.10 ms | 0.04 ms |
| 8-bit Array Radix sort      | 1.36 ms  | 0.04 ms | 0.02 ms |
| 10-bit Array Radix sort     | 0.98 ms  | 0.34 ms | 0.15 ms |
| 12-bit Array Radix sort     | 1.16 ms  | 0.13 ms | 0.06 ms |
| 14-bit Array Radix sort     | 1.13 ms  | 0.12 ms | 0.05 ms |
| 16-bit Array Radix sort     | 1.41 ms  | 0.12 ms | 0.06 ms |
| List array quicksort        | 2.18 ms  | 0.23 ms | 0.10 ms |
| Array quicksort             | 1.10 ms  | 0.11 ms | 0.05 ms |

As expected, it is a significant improvement over the list one. 
Indeed some of these even matched or exceeded the array quicksort 
while still processing a list at their core.
I added a list array quicksort for comparison, since it is a little closer in spirit, 
yet it is twice as slow. Radix really is great when it is able to win with a handicap. 
Shame it is not as general and limited to list a input here.

Note that while in this case 10-bit version is faster, 
I noticed that the general region of 10-14 seemed to be good, 
with no obvious winner at repeating my sample sizes. 
More testing required.

### Random
Bogosort is a joke entry, but using random shuffles is still a legal move.
Note that bogosort typically depends on arrays, using lists would make it abysmal.
As such, I will still use arrays for shuffling, in the style of the quicksort list-to-array-to-list solution.
The result is such:
```sml
fun check _ [] b = b
  | check _ [x] b = b
  | check ge (x::y::xs) b =
    if not b then b
    else check ge (y::xs) (ge (y, x))

local val grng = ref (Random.init 23)
in
fun shuffle xs =
    let val xs = Array.fromList xs
        val len = Array.length xs
    in
        Array.modify (fn x =>
                         let val ((y, j), rng) =
                                 Random.map (fn r =>
                                                let val r = r mod len
                                                in (Array.sub (xs, r), r)
                                                end) (!grng)
                         in
                             grng := rng;
                             Array.update (xs, j, x);
                             y
                         end) xs;
        List.tabulate (len, fn i => Array.sub (xs, i))
    end
end

fun sort _ [] = []
  | sort _ [x] = [x]
  | sort ge xs = if check ge xs true
                 then xs
                 else sort ge (shuffle xs)
```
A little messy, but the core is simple enough. 
Check if it is sorted, otherwise shuffle and try again.
Note that I kept arrays to `shuffle` only, since list shuffle is not great, but checking is fine. 
This does imply a big performance degradation in the constant conversion from list to array and back.
This is bogosort however, so making it run slower would be in the spirit.
Here is an example on **n=5**, random input, note the time being microseconds as opposed to milliseconds:
| Algorithm                   | Mean     | StdDev   | Err     |
|-----------------------------|----------|----------|---------|
| Natural bottom-up mergesort | 0.27 us  | 0.14 us  | 0.01 us |
| Bogosort                    | 67.89 us | 20.53 us | 1.45 us |
| Array quicksort             | 0.20 us  | 0.16 us  | 0.01 us |

At n=6, it already goes up to 500 ns, so going further is not a bright idea.
Still, n=5 is not too bad in comparison to nothing.
Nothing surprising however.

It does pretty well on sorted, n=10000 though:
| Algorithm                   | Mean    | StdDev  | Err     |
|-----------------------------|---------|---------|---------|
| Natural bottom-up mergesort | 0.22 ms | 0.02 ms | 0.01 ms |
| Bogosort                    | 0.07 ms | 0.00 ms | 0.00 ms |
| Array quicksort             | 1.09 ms | 0.10 ms | 0.04 ms |

This insane speedup is thanks to `check` being before `shuffle`.
If you only get sorted input, bogosort is the winner. 
It at least has that over the quadratic algorithms.

[^1]: Ignoring the Powerbook since all kinds of devices are used in RAM shortages, the GHC version was 6.4.1, released September 19 2005.
I did check what kind of mergesort GHC had in that version, and it was a simple bottom up solution without natural runs.
Subsequent addition was natural runs, seemingly few major versions later.
These days, GHC has a beefy 4-way bottom-up mergesort with natural runs, not to mention many optimizations in the compiler itself since then.
