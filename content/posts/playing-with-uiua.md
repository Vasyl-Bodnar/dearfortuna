+++
author = "V"
title = "Playing with Uiua for Optimal Mindustry Mining"
date = "2026-07-08T18:00:00-04:00" 
description = "Using Uiua for some minor problems I have in Mindustry."
tags = ["Uiua", "Dynamic Programming", "Mindustry"]
style = "custom.css"
draft = true
+++

## Intro
I happened to be playing [Mindustry](https://mindustrygame.github.io) one evening.
But I kept getting annoyed by a minor issue, my drills and mining are probably suboptimal.

![Picture of resources in Mindustry with a small drill over a few tiles of them](/images/mindustry-drills.jpg)

If you can see the picture and can understand it, the drills cover 2x2 tiles, 
whereas the resources are a large uneven cluster.
It is unfortunately an innate and natural desire to make those drills cover all possible resources you can get.
Thankfully, I have an outlet for such unrefined desires in my life.
I could just make an optimal algorithm for it. And there was a language I was trying to learn for a while too...

## The Question to Ask
First we need to state the problem properly so that the algorithm can be well-defined.
This is generally an easy problem to state, especially if we ignore some extra details 
(cluster overlap, the big problem). 

We have a matrix of ones and zeros as an input, e.g. A is
```
0 1 1 0 
1 1 1 1 
0 1 1 0
```
Then, we must return the optimal packing (i.e. maximum ones covered for our purposes) of non-overlaping 2x2 squares, result would be e.g.
```c
a a b b
a a b b
- - - -
```
Where a is one square, b is another square, and dashes are a waste and my tears.

## Easy Enough
At first I thought about solving this problem in Scheme, 
but given that it would involve matrices, 
I felt that it is time to learn Uiua. 
By the way, while I do use Uiua386 font for this page, 
if you forbid page fonts or do not have it for whatever other reasons, 
the glyphs might look off if they exist at all.

This is a walkthrough of how my code works, with some explanations of how Uiua works.
It is no tutorial, unless you like to be kept in the dark on most things and forged through fire.
There is a good tutorial on the official website.

Anyway, I felt that a good start in uiua would be `⧈∘2_2`. 
This produces 2x2 windows over the matrix (it is `stencil identity [2 2]` if you want names and a more common array notation).
Exactly what we need.
Using the example matrix we would get:
```
0 1  1 1  1 0
1 1  1 1  1 1
               
1 1  1 1  1 1
0 1  1 1  1 0
```
Unfortunately, we hit an ouch here. 
This seems to be Maximum Weight Independent Set, an NP-Hard problem.
Thankfully this gives me an opportunity to test Uiua for something more interesting than just nice array operations.
It does mean that my solution is probably not as idiomatic or optimized as it could be by a Uiua expert.
At least it is safe from LLMs, those things seem to be useless when working with Uiua.

We can start with preparing our data a little further, and introduce the language a little more extremely:
```uiua
Prep ← ◴⊸⊚ ≡₂(/+/+) ⧈∘2_2
```
Our windows part is to the right. `Prep ←` is just binding to a variable.
Uiua is a stack language like Forth so right-to-left is a more proper reading order.
Next we run a reduce with plus `/+` over second rank (using `≡` with `₂` for second rank), that is the actual matrix. 
Because we use 2x2 matrices, we need to reduce twice to first combine rows and then the columns.
Note that `≡` is necessary because many Uiua functions by default apply to every element/row, which is not what we want here.
Finally we run the last part where we get indices to non-zero boxes (coordinates to be more precise), and deduplicate them. 

One of the more interesting parts is the `⊸`, which preserves the argument to `⊚` and puts it after the output.
Thus from our example we get:
```uiua
╭─
╷ 0 0
  0 1    ╭─
  0 2    ╷ 3 4 3
  1 0      3 4 3
  1 1            ╯
  1 2
      ╯
```
Two matrices on the stack with fancy formatting. Again it is a stack language.

## Recursion is Ill-Advised
I thought of a top-down dynamic programming solution, not the fastest but it is optimal.
Yet, Uiua docs seem to say Uiua is not good with recursion.
Conflictingly, Uiua also has `memo`, which memoizes the arguments to the function argument.
We shall proceed.

We will start with our base case complete and begin our recursive case:
```uiua
Dyn ← |2 memo⍣(
  ⊢
| 0)
```
`|2` is just signature for how many arguments we take, `memo` is memoize, and so we are left with `⍣`.
This is `try`, and it is pattern matching by failing. 
There are ways to avoid it and prettier tricks, but to pattern match we genuinely throw an error. 
In this case `⊢` is `first`, that is, it takes the first element of the array. 
This function can fail on the empty list, hence our pattern matching is possible.

Next, we extend our recursive case
```uiua
⌵ ˜◡ ≡₁- ⊸⊢
```
For the first `⊸⊢`, as before, 
that `⊸` preserves the argument on the right, 
so we can do `⊢` while still having our coordinates after.
Next, subtract on each row, the `₁` tells us it is for a vector, i.e. 1-D.
If you ask "subtract what?", you have not been listening.
We already have our first coordinate and we have a list of coordinates, thus we subtract one from every other.
We reach `˜◡`, which preserves all arguments to the previous function. 
We do so `backward`s as you can see from `˜`, which is necessary to keep our values in correct order.
Finally we do `⌵`, which is just absolute value in case our subtraction left negatives.

Back to the stack after this operation:
```uiua
╭─       ╭─
╷ 0 0    ╷ 0 0
  0 1      0 1           ╭─
  0 2      0 2           ╷ 3 4 3
  1 0      1 0    [0 0]    3 4 3
  1 1      1 1                   ╯
  1 2      1 2
      ╯        ╯
```
In this case, since we subtracted [0 0], most of those did not matter.

Next step is more fun:
```uiua
▽≠2≡/++=1⤙=0
```
`=0` runs on all coordinates, and `⤙` saves the coordinates for `=1` as well, this time by putting it ahead instead of behind.
`+` runs between the results giving us a mask of all coordinates equal to 0 or 1.
We then do another row reduce `≡/+`, this time without specifying a rank. 
This summed the x and y coordinate masks together into one coordinate mask.
We can then filter `▽≠2` which takes a mask (the coordinate mask we had and `≠2` to remove those that have 1 or 0 in both x and y) 
as well as the original pre-subtraction coordinates which we still have on the stack.

Let us check up on the stack after all of that:
```uiua
╭─
╷ 3 4 3
  3 4 3
        ╯
[0 0]
╭─
╷ 0 2
  1 2
      ╯
```
Don't get distracted by it being top-down, as a stack language, it is both right-to-left and top-to-bottom.
It does look quite small now that we filtered so many neighbors away. 
Oh and most of this work so far was just filtering neighbors of the coordinate. 
If it seems like a lot, remember that it would likely be a lot of cases/long if statements in most other languages.

__

## The First Hurdle Passed
Barely any words (in the actual code) and we get a beautiful terse Uiua result.
```uiua
Prep ← ◴⊸⊚≡₂(/+/+)⧈∘2_2
Dyn ← |2 memo⍣(
  ↥⊃(
    Dyn↘1
  | ⌵˜◡≡₁-⊸⊢
    ▽≠2≡/++=1⤙=0
    +⊃(⊡⋅⊙∘|Dyn⊙⋅∘))
| 0)
F ← Dyn Prep

```
You can see, run, and play with the code [here in the Uiua online interpreter]().

Now you might realize that I originally stated:
> we must return the optimal packing of non-overlaping 2x2 squares

While this code would only return the optimal value of that packing.
However, since we already track coordinates, it is a trivial change to fix it.
I shall leave it as an exercise for the reader (after some Uiua tutorial though).

One bigger problem though, Mindustry has belts.

## The Actual Problem

__
