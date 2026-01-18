+++
author = "V"
title = "Finding out My Hashtable is Awful"
date = "2026-01-18T16:00:00-04:00"
description = "How to not write a hashtable"
tags = ["C", "C++", "Hashtable", "Leetcode"]
draft = false
+++

## Intro
I once found myself bored, though, not quite the useful kind of boredom. 
I did not want to do my projects or something nice. 
At the same time, I did not want to just spend it watching youtube or similar.
Thus, I thought, might as well do some leetcode.

I am not particularly fond of leetcode generally. 
Some algorithms are nice, most are not, and I rarely learn as opposed to "memorize" patterns.
Doing union-find on leetcode rarely feels as nice as using it for constant-folding optimizations in a compiler.
Still, leetcode is necessary for a lot of interviews (for now, given AI and grindflation).

I went through a couple of algorithms, doing all of them in C to provide a modicum of joy.
Well as close as you can get to joy given extra annoyances. 
E.g. leetcode C compiler setup fails on signed integer overflow.
This requires `-fsanitize=signed-integer-overflow` on my gcc setup.
This setting has its uses, but not when I just wanted to do a quick fnv1a.

Anyway, some problems went well, usually those I knew or those that are obvious to me. 
Some did not, and I had to give up and look up the solution and try to understand it.
There were a few that relied on stuff that would take me a while to implement in C too.

One problem I had was ["261. Contains Duplicate II"](https://leetcode.com/problems/contains-duplicate-ii/description/). 
I started with a simple naive double loop, essentially doing sliding window, but it left me wanting.
I was nearing the end of my leetcode energy, so I decided to look up the solution.
Hashtable, obviously. Very simple too, just a get and a put in a loop.

C does not have a hashtable natively. 
Leetcode apparently provides [uthash](https://support.leetcode.com/hc/en-us/articles/360011833974-What-are-the-environments-for-the-programming-languages), 
though I had never seen it in a wild (becomes obvious why when you see the ergonomics).
There is also the libc [hash table](https://linux.die.net/man/3/hsearch), but that is just one of POSIX April Fools joke.

Anyway, I implemented one recently while testing out my C build system (Guile script, nothing too fancy, though it does cache).
It even has SSE2 SIMD and 64bit SWAR (SIMD Within A Register). 
So I thought it would be good practice to do another one, and it very much was in hindsight.

## Spec matters
Got, good-old-table as I called it, based itself on Abseil SwissTable.
Except I tried to avoid just reimplementing someone's solution. I only read an overview and skimped on the details.
It sounded simple enough.

There was a point though where I wondered why Abseil seemed to use tombstones. 
But, I did not overthink it and thought I will learn it eventually.

I also thought about doing benchmarks to compare to at least C++ STL `std::unordered_map`.
But, that could have been too much work for a not too serious hashtable.
Especially since it was more useful to compare to more than one library.

Thus, I decided to do something similar for my solution. 
But doing SWAR, and especially with `int`s as opposed to 64 bit values seemed like a waste of time.
A speedup, sure, but not that serious. 
As such, I took to just doing linear probing. 
It should be good enough, that's the base for a SwissTable anyway.

## Absolute failure
So straight from memory I implemented a simple linear probing hashtable.

Here is its struct:
```c
struct HashMap {
    int capacity;
    int length;
    struct {
        int valid, key, val;
    } arr[];
};
```
Nothing too involved besides the zero length array trick. That is just used to allocate everything in a single allocation.

I originally did not include `length` for funny reasons, but it is necessary. 
Well, you can optimize it out given that leetcode tests will never get that bad, but let's not get into that.

So after finishing up and cleaning up any immediate compile-time errors, I ran the solution on basic tests.
Success, and given how simple the solution really is, that was to be expected.

Here is how the "solution function" looks:
```c
bool containsNearbyDuplicate(int *nums, int numsSize, int k) {
    struct HashMap *map = create_ht(64);

    for (int i = 0; i < numsSize; i++) {
        int *p = get_ht(map, nums[i]);
        if (p && i - *p <= k) {
            return true;
        }
        put_ht(&map, nums[i], i);
    }

    return false;
}
```
Freeing wastes cycles. Anyway,
satisfied, I hit the submit button... Timed out.

That was weird. Even if my table is not particularly fast, that was orders of magnitude too slow.
This was on a testcase of only 54500 `int`s, so it should not take that long.

I tried a couple of optimizations. E.g. valid field is not necessary since a key can never be more than 2^30.
I even tried to just increase the preallocated memory to see if that could improve runtime, even if just for this one.

All of them were bandaids at best. This was a fundamental issue.

## Evil assumptions
So what was at the core of my `put` and `get` that made this many times slower than it should be.

Well I had a simple assumption. The valid entry could be anywhere after the initial index from hash.
If you do not see the problem immediately, think about it.
What made it worse was that I do not need to delete anything.

Every entry is allocated right after the last one. 
So I was forcibly and completely needlessly going through the entire hash table. 
For every call to `get` and most calls to `put`. 
Most calls to `put` was because I had this useful thing called early return.

Ironic that the problem itself also has an early return.

The fix to `get` was adding an `else return 0;`. Immediate improvement.

Here is it so you can see the error of my ways:
```c
for (int i = init; i < map->capacity; i++) {
  if (map->arr[i].valid) {
    if (map->arr[i].key == key) {
      return &map->arr[i].val;
    }
  } else { // just this part
    return 0;
  }
}
```
This is also a case of a small optimization hiding a better one. 
The original code used `map->arr[i].valid && map->arr[i].key == key`.
This couples them when they really should have been separate.

The fix to `put` was slightly longer, but essentially the same.

I only realized this after being very annoyed by it and testing it locally.

Thus, the problem was submitted and solved in reasonable time.

## Unreasonable time
Ok, no. The time was ~800ms. 
This counts as "solved", but realistically extremely slow. This is array search in a loop speeds, if not worse.
The other solutions that leetcode presented were 100ms at worst. I was bottom 0.20%.

This prompted me to do some extra testing. Nothing too precise or involved, but enough to see the problems.
You can find the code for it [here](https://github.com/Vasyl-Bodnar/contains-duplicate2-shenanigans).
I will include the (incomplete) table here anyway to avoid spoilers:

| Name       | Time    | Time (-O2) |
|------------|---------|------------|
| fine.c     | 5 ms    | 4 ms       |
| fine.cpp   | 26 ms   | 6 ms       |
| with_got.c | 2227 ms | 691 ms     |
| awful.c    | 8292 ms | 1893 ms    |

`fine.c` is the good solution, `awful.c` is the original bad solution. 
There is also `fine.cpp` which uses `unordered_map` and `with_got.c` which uses my got library.

The only test case I used was the one I timed out on. 
Thus, this table is a little useless to compare my table and `unordered_map` in my opinion.
But it does show you the magnitudes of difference from the bad ones.

Still, it is interesting to see that my `fine.c` is better or comparable to `unordered_map`. 
Yet, the C++ solution is finished in ~80ms, mine is not.

I then decided to apply the `valid` field removal I talked about.
I was able to get the time down to ~350ms, which is still far from ideal, but more manageable.

I also tried the `uthash` that leetcode provides. 
That one gave me ~90ms. 
The ergonomics and documentation were questionable. Lots of macros, which are not friendly to leetcode.
You have to create your own entry and even include a magic field for a hash handle. 
The primary purpose of which seemed to be iteration, but could be more.
Well now that is a little sad.

I then went on to do some stronger optimizations. 
I have started using static preallocated memory.
Got rid of length for good with that.
Tested different preallocation sizes.
I found that allocating 256KB is enough to pass the tests with flying colors.
Any more slowed down, any less slowed down.

Done in 2ms, and 16MB of memory per what leetcode reports. C++ uses ~100MB and uthash uses ~60MB for comparison.
Beats ~98% and ~96% respectively.
Pride restored, technically. 

I included this as `best.c` in that [same repo](https://github.com/Vasyl-Bodnar/contains-duplicate2-shenanigans).

## What happened
Going from ~800ms to ~350ms by just removing the valid field is not too surprising.
This simply uses less memory which means we have to seek less if we hit a collision.
Better cache usage and whatnot, possibly compiler optimizations too (leetcode uses -O2).

Going from ~350ms to 2ms is a different question. 
But the trick is that by having so much capacity (256KB), 
I basically turned `get` and `put` into array access operations, not "amortized", actual O(1).
At that point you could just create buckets for every used number.

I could and probably should do some `perf` testing to see whether there is another obvious mistake. 
E.g. if anything hashes to the end of the array that would always force a resize, a potentially serious issue.
Still, I am satisfied with getting 2ms for now.

## What did we learn
I should go fix my got library. 
I call it not too serious, but I cannot allow this level of underperfomance.

The main lesson would probably be an importance of assumptions.
If you take in wrong or expensive assumptions, you may suffer.
On the other hand, if you take in correct assumptions, you can benefit a lot from it.
This often involves a tradeoff with generality like what I did for `best.c`, but other kinds exist too.
