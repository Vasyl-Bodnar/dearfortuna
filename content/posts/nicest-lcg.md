+++
author = "V"
title = "Nicest (small) LCG numbers"
date = "2026-02-09T22:00:00-04:00"
description = "Finding the nicest small LCG numbers through brute-force"
tags = ["C", "LCG", "PRNG"]
draft = false
+++

## Intro
Recently I wanted to give a nice small example of a [Linear Congruential Generator](https://en.wikipedia.org/wiki/Linear_congruential_generator) (LCG)
to show how easy and simple [PseudoRandom Number Generators](https://en.wikipedia.org/wiki/Pseudorandom_number_generator) (PRNG) are.

An LCG is typically `x[i] = a * x[i - 1] + c (mod m)`, where `x[0]` is the initial seed, `a` is multiplier, `c` is additive, `m` is our space.

There are many good numbers online, and even plenty of bad numbers have a decent enough period, i.e. when the sequence starts repeating. 
You will not notice the issue immediately in many of those cases, even if a computer or a bad actor would.
LCGs are simple and efficient, but are one of the least cryptographically secure after all.

The issue is that LCG numbers are frequently large. 
They have to be for good randomness over a large space, e.g. 2^32, the typical size of an `int`.
This does form a good question though, what would be the nicest LCG numbers, i.e. still looks "random", yet small (under a 100 preferably).

## Brute Forcing into N
Now I am certain that there are analytical methods to find good numbers with the properties that I want. 
I would not be surprised if someone did that already either. 
But, these numbers are small, I can quite literally just loop over the entire input space and pick the best ones.
We do need a scoring function to check how good the inputs are though.

Let's start with uniqueness as our scoring function. 
That is, we just check whether a number occurs for each possible number.
This rewards long periods and using all of our numbers as any future period or identical numbers will not impact the score.

Let us check all numbers up to (but excluding) 12.

The best result we get is `m = 11`, `c = 1`, `a = 1`, `x[0] = 1` or:
```
1 2 3 4 5 6 7 8 9 10 0 1 2 3 4 ...
```

Oh how great and random it is, most would almost certainly disagree.
The sequence is not ideal in what we usually think of as "random".

Yeah, I should have expected that. 
Despite not looking random, it has the best period of 11, known as full period (`= m`). 
It even uses all of our numbers!

What about second or third or fifteenth best? 
Just shifted, you get tweaks to `c` or `x[0]`, but it is practically the same obvious ordered sequence.

The flaw is in the scoring system as it does not care about anything except uniqueness. 
Indeed, to it `1 2 3 4 5` and `4 2 5 3 1` are identical as long as same numbers are involved, 
even if to us the second is more "random".

## Discouraging Unfair Play
Well the core issue is that the algorithm can set `a = 1`, `m = 11` and 
tweak `c` and `x[0]` with a simple ordered sequence.
One way to deal with it somewhat is to check whether the difference between numbers changes.
If it does not, reduce the score for each violation.

This system can have trouble tracking changes modulo. 
E.g. `10 0 1`, has differences of `10` and `1`, rather than `1 mod 11` for both.
But we can just ignore those and look at other options. I call this technology Human-In-The-Loop™.

The results look much better:
```
x[0] = 5, a = 1, c = 5, m = 11 :: 5 10 4 9 3 8 2 7 1 6 0 5 ...
x[0] = 5, a = 1, c = 6, m = 11 :: 5 0 6 1 7 2 8 3 9 4 10 5 ...
x[0] = 1, a = 2, c = 0, m = 11 :: 1 2 4 8 5 10 9 7 3 6 1 2 ...
x[0] = 1, a = 6, c = 0, m = 11 :: 1 6 3 7 9 10 5 8 4 2 1 6 ...
x[0] = 1, a = 7, c = 0, m = 11 :: 1 7 5 2 3 10 4 6 9 8 1 7 ...
x[0] = 1, a = 8, c = 0, m = 11 :: 1 8 9 6 4 10 3 2 5 7 1 8 ...
x[0] = 1, a = 2, c = 1, m = 11 :: 1 3 7 4 9 8 6 2 5 0 1 3  ...
x[0] = 1, a = 6, c = 1, m = 11 :: 1 7 10 6 4 3 8 5 9 0 1 7 ...
x[0] = 1, a = 7, c = 1, m = 11 :: 1 8 2 4 7 6 10 5 3 0 1 8 ...
x[0] = 1, a = 8, c = 1, m = 11 :: 1 9 7 2 6 5 8 10 4 0 1 9 ...
```

The first two do only tweak `c` and `x[0]`, but they don't look too bad in comparison to `1 2 3 4 ...`.
Also `x[0]` is set to `1`, but this is more of a first mover advantage (my brute force loop starts with `x[0] = 1`), and this is a seed to begin with.
I wouldn't mind using these, but it would be useful to test it on more numbers.

## Larger numbers
Let us then try up to (including) 50, this takes a second, but does find me some interesting results:
```
x[0] = 1, a = 11, c = 1, m = 50 :: 1 12 33 14 5 6 17 38 19 10 11 22 ...
x[0] = 1, a = 21, c = 1, m = 50 :: 1 22 13 24 5 6 27 18 29 10 11 32 ...
x[0] = 1, a = 31, c = 1, m = 50 :: 1 32 43 34 5 6 37 48 39 10 11 42 ...
x[0] = 1, a = 41, c = 1, m = 50 :: 1 42 23 44 5 6 47 28 49 10 11 2  ...
x[0] = 1, a = 11, c = 3, m = 50 :: 1 14 7 30 33 16 29 22 45 48 31   ...
x[0] = 1, a = 21, c = 3, m = 50 :: 1 24 7 0 3 16 39 22 15 18 31 4   ...
x[0] = 1, a = 31, c = 3, m = 50 :: 1 34 7 20 23 16 49 22 35 38 31   ...
x[0] = 1, a = 41, c = 3, m = 50 :: 1 44 7 40 43 16 9 22 5 8 31 24   ...
x[0] = 1, a = 11, c = 7, m = 50 :: 1 18 5 12 39 36 3 40 47 24 21    ...
x[0] = 1, a = 21, c = 7, m = 50 :: 1 28 45 2 49 36 13 30 37 34 21   ...
```
It of course prefers `50` as `m`, since that provides the largest period.
The biggest issue is that `a = 11` and `+10` cousins have their issues, especially for `c = 1`.
It is a little off to see so many multiples of `11`. Likely due to `m = 50`.
But with `c = 7` or even `c = 3`, it looks decent enough. 
I will have to look into a way to improve it by adding more to the score.

I feel that while `m = 50` is great for long periods, it is not good for "randomness".
So that is another avenue for improvement.

We can also look at `m = 61` instead of `m = 50` as it might be nicer:
```
x[0] = 1, a = 26, c = 0, m = 61 :: 1 26 5 8 25 40 3 17 15 24 14 59 9  ...
x[0] = 1, a = 30, c = 0, m = 61 :: 1 30 46 38 42 40 41 10 56 33 14 54 ...
x[0] = 1, a = 31, c = 0, m = 61 :: 1 31 46 23 42 21 41 51 56 28 14 7  ...
x[0] = 1, a = 35, c = 0, m = 61 :: 1 35 5 53 25 21 3 44 15 37 14 2 9  ...
x[0] = 1, a = 43, c = 0, m = 61 :: 1 43 19 24 56 29 27 2 25 38 48 51  ...
x[0] = 1, a = 44, c = 0, m = 61 :: 1 44 45 28 12 40 52 31 22 53 14 6  ...
x[0] = 1, a = 51, c = 0, m = 61 :: 1 51 39 37 57 40 27 35 16 23 14 43 ...
x[0] = 1, a = 54, c = 0, m = 61 :: 1 54 49 23 22 29 41 18 57 28 48 30 ...
x[0] = 1, a = 55, c = 0, m = 61 :: 1 55 36 28 15 32 52 54 42 53 48 17 ...
x[0] = 1, a = 59, c = 0, m = 61 :: 1 59 4 53 16 29 3 55 12 37 48 26 9 ...
```
I look at bigger `a`s as small ones give obvious powers. 
These look alright, but have their issues too.

## Final Results
My quest continues for the better numbers. 
There is also a limit to how many inputs I can handle, 
so maybe I should consider using some better state exploration than "loop over everything".
Naturally more work required for a better scoring system.
