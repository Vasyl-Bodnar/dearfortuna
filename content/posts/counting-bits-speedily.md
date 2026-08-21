+++
author = "V"
title = "Counting Bits done Speedily"
date = "2026-08-18T14:00:00-04:00"
description = "Comparing different solutions to NeetCode problem 'Counting Bits'"
tags = ["Leetcode", "C", "SIMD", "Benchmark"]
draft = true
+++

## Intro
Recently I decided to try solving some NeetCode problems. 
Specifically, I was starting with bit manipulation problems as I can do those in C without issues.
Disappointingly, NeetCode is essentially LeetCode, so there was nothing new or interesting. 
NeetCode does not even have C.
Thankfully, that did not matter for these problems, though I had to settle with C++. 

Anyway, I was doing the easy problems, and got to ["Counting Bits"](https://neetcode.io/problems/counting-bits/question).
Note that the previous problem was doing the same for only one element.
The simplest solution is to then apply that previous problem in a loop. 
It will work and will even pass, I did it that way myself, but it is not "optimal".
Indeed, the optimal O(n) solution is everyone's favorite misnomer (1D) Dynamic Programming[^1].
I did foresee this possibility given the recurrence pattern consecutive numbers have. 
But I did not want to overthink the problem.
Still, it did leave me with something else.

## On Overdoing
DP was just a little counter-intuitive for this problem.
DP makes sense when you otherwise have to do serious work, so, without DP, going from O(n) to O(n^2) or even O(n^3) otherwise.
I can consider removing some huge constansts too.
But counting bits is not (relatively) serious work, it feels wasteful to use DP there.
You have O(bits * n) even in the naive solutions, which uses fast instructions in typical O(64) or O(32) per iteration.
There is even a trick to count bits with less iterations.
Thus, I wondered how would they compare and whether it is possible to go further than this "optimal" solution.

The implementation for the normal (naive) and human solution is as expected:
```c
for (unsigned int i = 1; i <= n; i++) {
    for (unsigned int j = 0; j < 32; j++) {
        out[i] += (i >> j) & 1;
    }
}
```
We do assume that `out` is zero-initialized[^2], hence `i = 1` and not `i = 0`. This does not really matter anyway.

For the trick solution, there is a handy method to run exactly `out[i]` times:
```c
for (unsigned int i = 1, j; i <= n; i++) {
    j = i;
    while (j) {
        out[i] += 1;
        j &= j - 1;
    }
}
```

And the DP solution is elegant in its own way:
```c
for (unsigned int i = 1; i <= n; i++) {
    if (i & 1) {
        out[i] = out[i - 1] + 1;
    } else {
        out[i] = out[i / 2];
    }
}
```

Looping over it a 1000 times (and a 1000 times over that to get average) gave us an expected result:

| Optimization             | Average Time (ms) |
|--------------------------|-------------------|
| None                     | 23.88             |
| Trick                    | 2.32              |
| DP                       | 0.55              |

The normal version is an order or even two worse than others. 
The trick is a good speedup, though note that we are only going to n=1000 here. 
Trick is very efficient in these territories since it is based on number of bits.
This speed is not a given for larger numbers covering more bits.
Regardless, the DP is the "true optimal way" here.

## The Light
Well, there is one more way though, `popcnt`.
Introduced with SSE4.2 and SSE4a, and available on many CPUs, `popcnt` is able to take a 32 or 64 bit register and give you a count of 1s in a single instruction.
Technically `popcnt` would give us real O(1), which is even better than DP with its few operations.
However, `popcnt` is not a bit shift or addition. Its implementation might take many cycles and fail to overcome DP.
Still, it is should be very optimized and fast in hardware, so let's see the solution:
```c
for (unsigned int i = 1; i <= n; i++) {
    out[i] = __builtin_popcount(i);
}
```
As simple as it gets. Let's benchmark and add to the table:

| Optimization | Average Time (ms) |
|--------------|-------------------|
| None         | 23.88             |
| Trick        | 2.32              |
| `popcnt`     | 2.28              |
| DP           | 0.55              |

Uhh, interesting. It is ever so slightly faster than the trick solution, but this is embarrassing.
Thankfully, by looking at the assembly, the reason is quite obvious.
A little evil, but fair:
```text
call   1700 <__popcountdi2>
```
Yes, instead of a beautiful instruction, we get a call to this wonder:
```text
0000000000001700 <__popcountdi2>:
    1700:   f3 0f 1e fa             endbr64
    1704:   ba 55 55 55 55 55       movabs $0x5555555555555555,%rdx
    170b:   55 55 55 
    170e:   48 89 f8                mov    %rdi,%rax
    1711:   48 d1 e8                shr    $1,%rax
    1714:   48 21 d0                and    %rdx,%rax
    1717:   48 29 c7                sub    %rax,%rdi
    171a:   48 b8 33 33 33 33 33    movabs $0x3333333333333333,%rax
    1721:   33 33 33 
    1724:   48 89 fa                mov    %rdi,%rdx
    1727:   48 c1 ef 02             shr    $0x2,%rdi
    172b:   48 21 c2                and    %rax,%rdx
    172e:   48 21 c7                and    %rax,%rdi
    1731:   48 01 fa                add    %rdi,%rdx
    1734:   48 89 d0                mov    %rdx,%rax
    1737:   48 c1 e8 04             shr    $0x4,%rax
    173b:   48 01 d0                add    %rdx,%rax
    173e:   48 ba 0f 0f 0f 0f 0f    movabs $0xf0f0f0f0f0f0f0f,%rdx
    1745:   0f 0f 0f 
    1748:   48 21 d0                and    %rdx,%rax
    174b:   48 ba 01 01 01 01 01    movabs $0x101010101010101,%rdx
    1752:   01 01 01 
    1755:   48 0f af c2             imul   %rdx,%rax
    1759:   48 c1 e8 38             shr    $0x38,%rax
    175d:   c3                      ret
```
Just from looking at the constants, you can guess that this is a SWAR (SIMD Withing A Register) solution to popcount.
This is likely a fallback in the standard library if your hardware does not support `popcnt`.
Quite good, and remember that the trick solution benefits from small numbers, so it lost even with an advantage.
Unfortunately, it is not enough to beat the proper instruction.

So why does my CPU not support it? 
Well, my CPU does, it is a 10th gen i5 with AVX2 support. 
I doubt Intel considered `popcnt` support too expensive there.
I did not use `-march=native` or similar, only `-O3`, so it is possible that gcc failed to detect `popcnt` support.
I also use WSL on this machine, so that could be a wrench in the gears.
On the plus side, this does mean that I did not need to worry about the compiler converting my naive solutions into `popcnt`.
I can manually specify `-mpopcnt` anyway.

Hmm, still getting the fallback function, even with `-march=native`. It might be WSL, might be something else (like my custom build system :).
Tricky problem, but as they say, if you want it done right, do it yourself:
```c
for (unsigned int i = 1, tmp; i <= n; i++) {
    asm volatile("popcnt %1, %0" : "=r"(tmp) : "r"(i));
    out[i] = tmp;
}
```

With this, we get `popcnt` in generated assembly as we wanted:
```
popcnt %eax,%ecx
```

Anyway, how good is it?

| Optimization        | Average Time (ms) |
|---------------------|-------------------|
| None                | 23.88             |
| Trick               | 2.32              |
| `__popcountdi2`     | 2.28              |
| Inline asm `popcnt` | 0.84              |
| DP                  | 0.55              |

Pretty good, we are at a point, where you can consider using this over the DP for simplicity reasons. 
DP is not complex, but it is not a single instruction either. 
It did require assembly on my end, though this is not the case generally.
Still, a shame I could not improve upon DP by utilizing hardware (DP *is* abusing cache though).

Actually, looking at the `__popcountdi2`, there is still a way.

## Multiple Lights
`__popcountdi2` used a fancy SWAR, and no one forbid me from using it too.
Indeed, `__popcountdi2` is even a little flawed in that it uses 64bit SWAR, 
while its input is always 32bit. 
This should not have a significant effect, but inefficient nonetheless.

The only function that I can SWAR easily would be the naive solution.
So let's do that, first with 32-bit:
```c
for (unsigned int i = 1, res = 0; i <= n; i++, res = 0) {
    for (unsigned int j = 0; j < 4; j++) {
        res += (i >> j) & 0x11111111;
    }
    out[i] = res & 0xf;
    out[i] += (res >> 4) & 0xf;
    out[i] += (res >> 8) & 0xf;
    out[i] += (res >> 12) & 0xf;
    out[i] += (res >> 16) & 0xf;
    out[i] += (res >> 20) & 0xf;
    out[i] += (res >> 24) & 0xf;
    out[i] += (res >> 28) & 0xf;
}
```
Relatively simple and clear. The plus block seems bad, but compiler will take care of that. 
There could be some other tricks or optimizations I missed, but I believe this is a good start.
As for the speed?

| Optimization        | Average Time (ms) |
|---------------------|-------------------|
| None                | 23.88             |
| Trick               | 2.32              |
| `__popcountdi2`     | 2.28              |
| Inline asm `popcnt` | 0.84              |
| DP                  | 0.55              |
| SWAR 32bit          | 0.54              |

It beats `popcnt` and even DP by a slight margin. 
It works and matches the DP solution in the output hash.

Hmm, that's weird.
Looking at the assembly, it seems that my loop got unrolled, but otherwise nothing obvious.
Ah, as I was scrolling higher I saw that the compiler auto-vectorized this function with SSE2.
I guess this is the solution, and I beat DP, even if it is a slight margin. 
In some other test runs, the difference was even larger too.

Still, I want to go further by myself, how about without auto-vectorization?

| Optimization         | Average Time (ms) |
|----------------------|-------------------|
| None                 | 23.88             |
| Trick                | 2.32              |
| `__popcountdi2`      | 2.28              |
| SWAR 32bit           | 1.86              |
| Inline asm `popcnt`  | 0.84              |
| DP                   | 0.55              |
| SWAR 32bit + SSE2 AV | 0.54              |

That's much more reasonable, now it is in line with `__popcountdi2`.
Anyway, the improvement over the naive is great, so we can switch to 64 bit SWAR now to push further.
The code is much the same:
```c
for (uint64_t i = 0, res; i < n; i += 2, res = 0) {
    uint64_t ii = i | ((i + 1) << 32);
    for (unsigned int j = 0; j < 4; j++) {
        res += (ii >> j) & 0x1111111111111111;
    }
    res = (res & 0x0f0f0f0f0f0f0f0f) + ((res >> 4) & 0x0f0f0f0f0f0f0f0f);
    res = (res & 0x00ff00ff00ff00ff) + ((res >> 8) & 0x00ff00ff00ff00ff);
    res = (res & 0x0000ffff0000ffff) + ((res >> 16) & 0x0000ffff0000ffff);
    out[i] = res & 0xffffffff;
    out[i + 1] = (res >> 32) & 0xffffffff;
}
out[n] = __builtin_popcount(n);
```
We go over two elements at once.
The summation part is slightly optimized since compiler seems to need help here from my testing, whereas it works fine on 32-bit.
Also note that 64bit SWAR has a tendency to auto-vectorize as well, though the performance is a good bit worse than 32bit SWAR. 
It is as though they swap places, but it does make sense that compiler has a better shot with a simpler 32bit loop.
We got a decent speed-up all the same:

| Optimization         | Average Time (ms) |
|----------------------|-------------------|
| None                 | 23.88             |
| Trick                | 2.32              |
| `__popcountdi2`      | 2.28              |
| SWAR 32bit           | 1.86              |
| SWAR 64bit           | 1.15              |
| Inline asm `popcnt`  | 0.84              |
| DP                   | 0.55              |
| SWAR 32bit + SSE2 AV | 0.54              |

Looking good, but not quite the limit of my CPU. 
Let's double these efforts with an SSE2 solution. 
SSE2 is enabled by default on x86_64, so this is still applicable to most PC CPUs too.
```c
if (n < 4) {
    countBitsNormalSWAR32(out, n);
} else {
    __m128i ii = _mm_set_epi32(3, 2, 1, 0);
    for (unsigned int i = 0; i < n; i += 4) {
        __m128i ones = _mm_set1_epi8(0x11);
        __m128i res = _mm_and_si128(ii, ones);
        res = _mm_add_epi8(res, _mm_and_si128(_mm_srli_epi32(ii, 1), ones));
        res = _mm_add_epi8(res, _mm_and_si128(_mm_srli_epi32(ii, 2), ones));
        res = _mm_add_epi8(res, _mm_and_si128(_mm_srli_epi32(ii, 3), ones));
        __m128i nibbles = _mm_set1_epi8(0x0f);
        res = _mm_add_epi8(_mm_and_si128(res, nibbles),
                           _mm_and_si128(_mm_srli_epi32(res, 4), nibbles));
        __m128i bytes = _mm_set1_epi16(0x00ff);
        res = _mm_add_epi16(_mm_and_si128(res, bytes),
                            _mm_and_si128(_mm_srli_epi32(res, 8), bytes));
        __m128i halves = _mm_set1_epi32(0x0000ffff);
        res = _mm_add_epi32(_mm_and_si128(res, halves),
                            _mm_and_si128(_mm_srli_epi32(res, 16), halves));
        ii = _mm_add_epi32(ii, _mm_set1_epi32(4));
        _mm_storeu_si128(((__m128i *)(out + i)), res);
    }
    for (unsigned int i = n - (n & 3); i <= n; i++) {
        out[i] = __builtin_popcount(i);
    }
}
```
Now, that's a lot of intrinsics. 
Still, you might be able to recognize that this is essentially the same solution as 64bit, just extended to four `i`s rather than two.
It even uses the same summing mechanism.
The only large differences is the SWAR32 and popcount that are used for too small N/leftovers.
My testing does not actually use N that small, but this is to make the function more complete.

A fun sidenote, but the compiler could not optimize `_mm_set_epi32(i + 3, i + 2, i + 1, i)`.
This slack I had to fix by separating them into `set1` and `add`. 
This optimization might seem minor, but it was a change of 25% in average time.
I ended up moving it partially outside the loop, but that change is less impactful.

Anyway, SSE2 was a good change, but how will it do in comparison?

| Optimization         | Average Time (ms) |
|----------------------|-------------------|
| None                 | 23.88             |
| Trick                | 2.32              |
| `__popcountdi2`      | 2.28              |
| SWAR 32bit           | 1.86              |
| SWAR 64bit           | 1.15              |
| Inline asm `popcnt`  | 0.84              |
| SSE2                 | 0.67              |
| DP                   | 0.55              |
| SWAR 32bit + SSE2 AV | 0.54              |

As you can see I have yet to breach the DP/SWAR32 auto-vectorization territory.
The only benefit of my SSE2 solution over compiler's SSE2 solution is that compiler uses double the bytes.

Now I could try to find further and better optimizations to my SWAR or SSE2 solution until I can reach the compiler's level.
Or, we can go further, to AVX2, the best my CPU can offer. But first, I do need `-mavx2` to work when even `-mpopcnt` does not.

## Mid-season Patch
So, for a funny reason[^3], I indeed made a mistake and WSL's gcc does support `-mpopcnt`, `-march=native`, and `-mavx2` as one would expect.
This also enables better auto-vectorization, so I shall add those to the table as well.

Importantly, `__builtint_popcount` will now use actual `popcnt`, and it does make a difference.
In fact, This `popcnt` is *faster* than DP. 
The difference from my assembly `popcnt` is that the compiler's `popcnt` is no longer a black box.
This visibility likely allowed for better optimization of the loop or how values are used.

The table is updated correspondingly with the new solution:

| Optimization         | Average Time (ms) |
|----------------------|-------------------|
| None                 | 23.88             |
| Trick                | 2.32              |
| `__popcountdi2`      | 2.28              |
| SWAR 32bit           | 1.86              |
| SWAR 64bit           | 1.15              |
| Inline asm `popcnt`  | 0.84              |
| SSE2                 | 0.67              |
| DP                   | 0.55              |
| SWAR 32bit + SSE2 AV | 0.54              |
| `-mpopcnt`           | 0.42              |

Haha, using `__builtin_popcount` is indeed the proper fastest solution to this problem.
DP is only "optimal" if you are running an ancient machine by today's standards, even in simplicity.

Now compiler did try to auto-vectorize some problems with SSE2, 
so I wondered what changed with AVX2 enabled. 
Well, not only do we get faster SWAR solutions, even the naive and trick solutions are optimized.

Here is all the important ones, marked by "+ AVX2 AV" and "+ `-mpopcnt`" (`-mavx2` implies `-mpopcnt`):

| Optimization         | Average Time (ms) |
|----------------------|-------------------|
| None                 | 23.88             |
| None + AVX2 AV       | 3.00              |
| Trick                | 2.32              |
| `__popcountdi2`      | 2.28              |
| SWAR 32bit           | 1.86              |
| SWAR 64bit           | 1.15              |
| Inline asm `popcnt`  | 0.84              |
| SSE2                 | 0.67              |
| Trick + `-mpopcnt`   | 0.57              |
| DP                   | 0.55              |
| SWAR 32bit + SSE2 AV | 0.54              |
| `-mpopcnt`           | 0.42              |
| SWAR 32bit + AVX2 AV | 0.24              |

The compiler seemed to have optimized the inner loop in the naive solution. 
The speedup is very large. 
Yet, I am dissapointed that the compiler still fails to detect a popcnt here.

The trick solution was something I did not expect. 
It is not complex, but not exactly a nice straight loop, so I wondered what would auto-vectorization do.
But, the compiler replaced it with a `popcnt`, detecting it accurately. 
Interestingly, this result is worse than just using `__builtin_popcount`, but still on par with DP.

Now, given that SWARs auto-vectorize, I was not suprised to see AVX2 instructions with 32bit SWAR.
The 2x speedup is also very much realistic given that we go from 128bit to 256bit instructions.
This makes it the absolute best option, while not even being that complex (this is still just 32bit SWAR solution!).

Now AVX2 does exist only from this being a straightforward loop over an array. But so does DP, so it is fair.
The fact that `popcnt` can even approach this speed while working on a single number at a time is impressive.

## Outdone
Well, let us finish this with my own AVX2 solution:
```c
if (n < 8) {
    // Do it the basic way
    countBitsNormalSWAR32(out, n);
} else {
    __m256i ii = _mm256_set_epi32(7, 6, 5, 4, 3, 2, 1, 0);
    for (unsigned int i = 0; i < n; i += 8) {
        __m256i ones = _mm256_set1_epi8(0x11);
        __m256i res = _mm256_and_si256(ii, ones);
        res = _mm256_add_epi8(
            res, _mm256_and_si256(_mm256_srli_epi32(ii, 1), ones));
        res = _mm256_add_epi8(
            res, _mm256_and_si256(_mm256_srli_epi32(ii, 2), ones));
        res = _mm256_add_epi8(
            res, _mm256_and_si256(_mm256_srli_epi32(ii, 3), ones));
        __m256i nibbles = _mm256_set1_epi8(0x0f);
        res = _mm256_add_epi8(
            _mm256_and_si256(res, nibbles),
            _mm256_and_si256(_mm256_srli_epi32(res, 4), nibbles));
        __m256i bytes = _mm256_set1_epi16(0x00ff);
        res = _mm256_add_epi16(
            _mm256_and_si256(res, bytes),
            _mm256_and_si256(_mm256_srli_epi32(res, 8), bytes));
        __m256i halves = _mm256_set1_epi32(0x0000ffff);
        res = _mm256_add_epi32(
            _mm256_and_si256(res, halves),
            _mm256_and_si256(_mm256_srli_epi32(res, 16), halves));
        ii = _mm256_add_epi32(ii, _mm256_set1_epi32(8));
        _mm256_storeu_si256(((__m256i *)(out + i)), res);
    }
    for (unsigned int i = n - (n & 7); i <= n; i++) {
        out[i] = __builtin_popcount(i);
    }
}
```
No surprises. It really is just an extension of SSE2 based on the original 64bit SWAR.
Now, for how fast it is:

| Optimization         | Average Time (ms) |
|----------------------|-------------------|
| None                 | 23.88             |
| None + AVX2 AV       | 3.00              |
| Trick                | 2.32              |
| `__popcountdi2`      | 2.28              |
| SWAR 32bit           | 1.86              |
| SWAR 64bit           | 1.15              |
| Inline asm `popcnt`  | 0.84              |
| SSE2                 | 0.67              |
| Trick + `-mpopcnt`   | 0.57              |
| DP                   | 0.55              |
| SWAR 32bit + SSE2 AV | 0.54              |
| `-mpopcnt`           | 0.42              |
| AVX2                 | 0.32              |
| SWAR 32bit + AVX2 AV | 0.24              |

It is right between `popcnt` and auto-vectorized 32bit SWAR.
This does mean that I still have improvements to add, but my goal is accomplished long ago at this point:
Multiple solutions that easily outdo the original dynamic programming solution. 
And `popcnt` is even simpler than DP and does not need SIMD.

## Conclusions
This makes for a final table:

| Optimization         | Average Time (ms) |
|----------------------|-------------------|
| None                 | 23.88             |
| None + AVX2 AV       | 3.00              |
| Trick                | 2.32              |
| `__popcountdi2`      | 2.28              |
| SWAR 32bit           | 1.86              |
| SWAR 64bit           | 1.15              |
| Inline asm `popcnt`  | 0.84              |
| SSE2                 | 0.67              |
| Trick + `-mpopcnt`   | 0.57              |
| DP                   | 0.55              |
| SWAR 32bit + SSE2 AV | 0.54              |
| `-mpopcnt`           | 0.42              |
| AVX2                 | 0.32              |
| SWAR 32bit + AVX2 AV | 0.24              |

The answer was always just to use `__builtin_popcount`. 
Worst case, your system does not support it and you will use a fallback, which is pretty good SWAR still.
The common case is that you get one of the best solutions in the tiniest amount of code.

One other thing to note is that throughout all of this I used n=1000. 
This is good enough for experimenting and most solutions don't actually care about how larg or small n is.
Plus, I can't afford gigabyte arrays. 
Still, I thought to at least use n=100,000.

I will also keep the same order as above, so sorted by n=1000 time.
Due to a higher n (solutions are still linear) I will have to do less runs, so the results are not quite as "high-quality". 
This should be fine for a quick comparison though:

| Optimization            | Avg Time (ms) n=100,000 | Avg Time (ms) n=1000 |
|-------------------------|-------------------------|----------------------|
| None                    | 31.33                   | 23.88                |
| None + AVX2 AV          | 2.41                    | 3.00                 |
| Trick                   | 4.71                    | 2.32                 |
| `__popcountdi2`         | 2.26                    | 2.28                 |
| SWAR 32bit              | 2.52                    | 1.86                 |
| SWAR 64bit              | 1.33                    | 1.15                 |
| --->Inline asm `popcnt` | 0.84                    | 0.84                 |
| SSE2                    | 0.67                    | 0.67                 |
| Trick + `-mpopcnt`      | 0.57                    | 0.57                 |
| DP                      | 0.55                    | 0.55                 |
| SWAR 32bit + SSE2 AV    | 0.54                    | 0.54                 |
| `-mpopcnt`              | 0.42                    | 0.42                 |
| AVX2                    | 0.32                    | 0.32                 |
| SWAR 32bit + AVX2 AV    | 0.24                    | 0.24                 |


[^1]: Yes, the name has a fun story with research on the wonderful Bellman equations. Still a horrible name.
[^2]: In my benchmark setup, I do `memset(out, 0, N * sizeof(*out))` in each function before the loops you see.
[^3]: I was using my own build system for this and in my genius I have decided to add an option called `extra-args`.
It felt right to use it for extra compiler options not already included. However, when I read the source I realized that this is in fact extra args for the *linker*.
Thus I was passing `-march=native`, `-mavx2`, and `-mpopcnt` to my linker.
