+++
author = "V"
title = "Nicest (small) PRNG"
date = "2026-08-16T20:00:00-04:00" 
description = "Finding the nicest yet small PRNG for non-crypto uses"
tags = ["PRNG", "C"]
draft = true
+++

## Intro
Some moons ago I made a post where I was trying to find the nicest LCG, [Linear Congruential Generator](https://en.wikipedia.org/wiki/Linear_congruential_generator),
using small numbers. That is, the numbers that would be easy to calculate LCG for, even by human hands.
The experiment was not super successful. Could have used bigger and nicer numbers, could have had better tests.
However, all of it is but a mere pebble to my holy grail: 
a PRNG that is small and easy to remember, yet sufficient for "real-world" tasks.

Now, of course, the LCG itself is an option, it is small and simple.
The numbers should not be small for "real-world", 
but finding the easy-to-remember ones is not hard.
Still, I wanted to explore a couple more options.
There is plenty of fish in this sea after all.

## Options
To be more exact, this algorithm should be memorizable, so as few lines and magic numbers as possible.
This algorithm should still be good, cryptographically secure would be great
but we just don't want anything too predictable. 
This algorithm should still be decently fast, otherwise we would just be using the best crypto ones all the time.
Ideally, this is the algorithm you could use for nearly any usecase except for the extremes (e.g. max speed or max security).


You can glean that I would prefer a single algorithm. 

Here are some algorithms I thought would be interesting to cover:
- LCG
- Hash with a Counter
- PCG
- MWC
- wyrand
- xorshift
- xoshiro
- SplitMix64
- Mersene Twister
- ChaCha20

Note that this is very far from exhaustive and I have excluded some good ones.

## Considerations
Now, some of these are already out based on how they are not "memorizable" or "good".

Mersene Twister is complex in its setup and use, with lots of state to keep. 
Yet it does not even pass BigCrush and fails early on PractRand, the absolute minimum for a half-decent PRNG.
MT does have its uses, especially in uniformity of its output thanks to that huge state, 
but it is not a good back-pocket solution.

SplitMix64 is fun in that it is relatively simple yet still good. 
64 bit of state, a few multiplies, shifts and xors. 
I learned about it as it was the recommendation of xoshiro author.
It is a bit of a problem in that you would have to memorize a couple of 64bit magic numbers.
There is also the 32bit solution, but it is quite worse statistically

## Conclusions
ChaCha is great
