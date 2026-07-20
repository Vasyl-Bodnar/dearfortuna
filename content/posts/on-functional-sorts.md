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
- Heapsort, for seeing how it would turn out
- Insertion sort, for being great
- Selection sort, for being `O(n^2)`
- Bubble sort, for being known
- Radix sort, for lack of comparisons
- Slow sort and bogosort, to have a terrific baseline

I will exclude e.g. Bucket sort and others that add a lot of constraints. 
The only exception to that would be Radix sort (the cooler Bucket), just so it is not all comparison-based sorts.
I will also try out different variations for some of these. 
As much as I can, but mostly for common options.
For the slow sort and bogosort, timeout shall exist for sanity-related reasons.

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
Now this is a more proper mergesort. Still `O(nlogn)`, but now we don't have to do as much work. [](TODO:.RIGHT.VS.LEFT.FOLDS)
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
[](TODO:.WHAT.ABOUT.MAPPING-AS-WE-GO)

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

[](TODO:TEMP.NUMBERS)
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
Not sure what Poly/ML does in the background. 
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
This becomes a significant tradeoff.

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

These trees are all good of course, but there is one tree that I always wanted to implement, a B+ tree.
This would be a great time to see how it would fair as a base for a treesort too:

[^1]: Ignoring the Powerbook since all kinds of devices are used in RAM shortages, the GHC version was 6.4.1, released September 19 2005.
I did check what kind of mergesort GHC had in that version, and it was a simple bottom up solution without natural runs.
Subsequent addition was natural runs, seemingly few major versions later.
These days, GHC has a beefy 4-way bottom-up mergesort with natural runs, not to mention many optimizations in the compiler itself since then.
