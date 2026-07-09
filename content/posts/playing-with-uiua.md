+++
author = "V"
title = "Playing with Uiua for Optimal Mindustry Mining Pt. 1"
date = "2026-07-08T18:00:00-04:00" 
description = "Using Uiua for some minor problems I have in Mindustry, the first part with Maximum Weight Independent Set"
tags = ["Uiua", "Dynamic Programming", "Mindustry"]
style = "custom.css"
draft = false
+++

## Intro
I happened to be playing [Mindustry](https://mindustrygame.github.io) one evening.
But I kept getting annoyed by an issue: my mining was suboptimal.

![Picture of resource tiles in Mindustry with a small drill over a 2 by 2 tiles of them](/images/mindustry-drills.jpg)

If you can see the picture, the drills cover 2×2 tiles, 
whereas the resources are a large uneven cluster.
It is, unfortunately, an innate and natural desire to make those drills cover all possible resources you can get.
Thankfully, I have an outlet for such unrefined desires in my life.
I could just write an optimal algorithm for it. And, there was a language I was trying to learn for a while...

## The Question to Ask
First, we need to state the problem properly so that the algorithm can be well-defined.
This is generally an easy problem to state, especially so, as long as we ignore some extra minor details 
(clusters can overlap, the big problem). 

We have a matrix of ones and zeros as an input, A is e.g.
```
0 1 1 0 
1 1 1 1 
0 1 1 0
```
Then, we must return the optimal packing (i.e. maximum number of 1s covered for our purposes) of non-overlapping 2×2 squares, result would be e.g.
```c
a a b b
a a b b
- - - -
```
Where a is one square, b is another square, and dashes are a waste and my tears.

## Easy Enough
At first, I thought about solving this problem in Scheme, 
but given all the matrices, I felt that it was time to learn Uiua. 
By the way, while I do use Uiua386 font for this page, 
if you forbid page fonts or do not have it for whatever other reasons, 
the glyphs might look off if they exist at all. No pretty colors though, see the online interpreter for that.

This is a walkthrough of how my code works, with some explanations of how Uiua works.
It is no tutorial, unless you like to be kept in the dark on most things and forged through fire.
There is a good tutorial on the official website.

Anyway, I felt that a good start in Uiua would be `⧈∘2_2`. 
This produces 2×2 windows/squares over the matrix (it is `stencil identity [2 2]` if you want names and a more common array notation).
Exactly what we need.
Using the example matrix we would get a new matrix:
```
0 1  1 1  1 0
1 1  1 1  1 1
               
1 1  1 1  1 1
0 1  1 1  1 0
```
Unfortunately, we hit an ouch here, NP-Hard.
This has become the Maximum Weight Independent Set problem.
Thankfully, this gives me an opportunity to test Uiua for something more interesting than just nice array operations.
It does mean that my solution is probably not as idiomatic or optimized as it could be by a Uiua expert.
At least it is safe from LLMs, those things seem to be useless when working with Uiua.

We can start with preparing our data a little further, and introduce the language a little faster:
```uiua
Prep ← ◴⊸⊚ ≡₂(/+/+) ⧈∘2_2
```
`Prep ←` is just binding to a variable.
Uiua is a stack language like Forth so right-to-left is a more proper reading order.
Our windows part is to the right. Note that it is a 2×3×2×2 matrix.
As such, we reduce with plus `/+` over second rank (using `≡` with `₂`), i.e. the 2×2 matrices inside the 2×3 matrix. 
If we did not use `≡` (`rows`) it would be 2 rows of squares, the default rank for reduce and most Uiua functions.
Because we use 2×2 matrices, we need to reduce twice to first combine rows and then the columns.
Finally, we run the last part where we get coordinates to non-zero squares, and deduplicate them. 

One of the more interesting parts is the `⊸`, which preserves the argument and puts it to the right of the output. 
In this case, with `⊚`, it preserves our sums on the right side.
Thus, from our example we get:
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
Yet, the Uiua docs seem to say Uiua is not good with recursion.
Conflictingly, Uiua also has `memo`, which memoizes the calls to a function (or combinations thereof).
We shall proceed.

The core idea would be to use the coordinates as a visit list.
On each step, by either visiting the coordinate or skipping it, 
and then keeping the max value. 
When visiting we naturally delete neighbors that would overlap otherwise and recurse further while adding the score at the coordinate.
Remember that our coordinates point to 2×2 square sums behind them on the stack, so we can pick the score we need anytime.

We will start with our base case complete and begin our recursive case:
```uiua
Dyn ← |2 memo⍣(
  ⊢
| 0)
```
`|2` is just a signature for how many arguments we take, `memo` is memoize, and so we are left with `⍣`.
This is `try`, pattern matching by failing. 
There are ways to avoid it and prettier tricks, but to pattern match we genuinely throw an error. 
In this case, `⊢` is `first`, that is, it takes the first element of the array. 
This function can fail on the empty list, hence our pattern matching is possible.

Next, we extend our recursive case
```uiua
⌵ ˜◡ ≡₁- ⊸⊢
```
For the first `⊸⊢`, as before, 
that `⊸` preserves the argument on the right, 
so we can do `⊢` while still having our coordinates after.
Next, subtract on each row, the `₁` tells us it is for a vector, we skip a dimension.
It is not easy to follow the stack if you never used Forth, but think about what is on the stack before subtract.
We already have our first coordinate and we have a list of coordinates, thus we map subtract the chosen coordinate from every other.
We reach `˜◡`, which preserves all arguments to the previous function. 
We do so `backward` as you can see from `˜`, which is necessary to keep our values in correct order.
Finally, we do `⌵`, which is just absolute value in case our subtraction left negatives.

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
In this case, since we subtracted [0 0], most of those operations did not matter, they would in deeper cases though.

Next step is more fun:
```uiua
▽≠2≡/++=1⤙=0
```
`=0` runs on all coordinates, and `⤙` saves the coordinates for `=1` as well, this time by putting it ahead instead of behind like `⊸` does.
`+` runs between the results giving us a mask of all coordinates equal to 0 or 1.
We then do another row reduce `≡/+`, this time without specifying a rank. 
This sums the x and y coordinate masks together into one coordinate mask.
We can then filter `▽≠2` which takes a mask (the coordinate mask we had and `≠2` to remove coordinates where x and y are either 1 or 0) 
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
Remember, as a stack language, it is both right-to-left and top-to-bottom.
It does look quite small now that we filtered so many neighbors away. 
Oh and most of this work so far was just filtering neighbors of the coordinate. 
If it seems like a lot, remember that it would likely be a lot of cases/long if statements in most other languages.

Now we enter an even more fun part, forking!
```uiua
+⊃(⊡⋅⊙∘|Dyn⊙⋅∘)
```
As you can see by `Dyn`, this is where we finally have recursion.
Anyway, let us do this step by step. 
Well a `⊃` (`fork`) is a bit special.
Essentially it copies the inputs to both the left and right sides of the bar `|`.
We will do the right side first. 
This is the "planet notation" as the Uiua tutorial calls it.
Think of the stack from above and map it to the "planets" from left to right.
`⊙` tells us that we use the argument, `⋅` tells us that we ignore the argument.
We must end these planets in either `∘` (`identity`) which keeps the last value or `◌` (`pop`), which ignores it.
All of these can be used elsewhere, but this planet notation kind of usage, especially in forks, is very nice.

Thus, what we get on the right side is "use the first", "skip the second", and "use the last", which then gets fed into `Dyn`.
On the left side it is "skip the first", "use the second", "use the last", with which we call `⊡` (`pick`).
The end result is that the left side picks the summed square by our coordinate, 
and the right side gives us the maximum value from that branch. 
Both of them are now on the stack, and we trivially add them with `+`.

The output is:
```uiua
6
```
Now we truly reduced.

Still, per my plans, we are not yet finished:
> On each step either visiting the coordinate or **skipping it**, 

We have to also skip the coordinate instead of using it.
There could be a better combination down the line that we would otherwise filter away.
Thankfully, we already have a really useful tool for considering that:
```uiua
Dyn ← |2 memo⍣(
  ↥⊃(
    Dyn↘1
  | ⌵˜◡≡₁-⊸⊢
    ▽≠2≡/++=1⤙=0
    +⊃(⊡⋅⊙∘|Dyn⊙⋅∘))
| 0)
```
You can see the part we went over, now it is in the right side of the fork. 
This fork copies the original input to the function `Dyn`. 
New addition is the left side, where we do `Dyn↘1`, 
that is we skip the top coordinate and call `Dyn` again. 
This gives the maximum value of that branch. 
Afterwards, we just take the maximum of the two `↥`. 
And the function is complete.

It would be nice to combine `Prep` and `Dyn`, so we can use them more easily too.
```uiua
F ← Dyn Prep
```
Thus, calling `F` on the original example from the problem description gives us:
```
6
```
Well, in this case taking the first branch every time was the correct answer.
But, we did the work to make sure of it.

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
You can see, run, and play with the code and the example [here in the Uiua online interpreter](https://uiua.org/pad?src=0_19_0-dev_4__eJxzLEpNDM7MLchJVXjUNkHhUdt6BeN4E4VoAwVDBUMFAy4FLAAkY6hgiFUOqi-WK6AotQBi5PQtj7p2POqa9ahz4aOmJg19bX1tzUfLOx51zDCKN-JyqcwDK6sxUshNzc1XfNS7WINLQeFR29JHXc0gloKCS2Xeo7YZIPtqFB71bD0959H0hWDDGnXBJi8Cq3o0be-jzgVGjzoX6mtr2xo-WjLTFuJ8bZBBj7oWPupufdQ181HHjBqQgV0zQfyOGZqaXDUKBppcbmBXgFwDcjkXl5uCIzxouAAkKm33).

Now some might realize that I originally stated:
> we must return the optimal packing of non-overlapping 2×2 squares

While this code would only return the optimal value of that packing.
However, since we already track coordinates, it is a trivial change to fix it.
I shall leave it as an exercise for the reader (after some Uiua tutorial though).

One bigger problem though, Mindustry has belts. 

That means adding routing on top of our NP-hard problem.
This, however, will have to wait until the next part.
