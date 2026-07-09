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
```c
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

Anyway, I felt that a good start in uiua would be `⧈∘2_2`. 
This produces 2x2 windows over the matrix (it is `stencil identity [2 2]` if you want names and a more common array notation).
Exactly what we need.
Using the example matrix we would get:
```c
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
Uiua is a stack language like Forth so right to left is a more proper reading order.
Next we run a reduce with plus `/+` over second rank (using `≡` with `₂` for second rank), that is the actual matrix. 
Because we use 2x2 matrices, we need to reduce twice to first combine rows and then the columns.
Finally we run the last part where we get indices to non-zero boxes (coordinates to be more precise), and deduplicate them. 

One of the more interesting parts is the `⊸`, which preserves the argument to `⊚` and puts it after the output.
Thus from our example we get:
```
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
I thought of a top-down dynamic programming solution, 
but Uiua docs seem to say Uiua is not good with recursion.
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

## The First Hurdle Passed
Barely any words and we get a beautiful terse uiua result.
```uiua
Prep ← ◴⊸⊚≡₁/+≡₂/+⧈∘2_2
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

One problem though
