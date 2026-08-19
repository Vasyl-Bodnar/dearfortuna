+++
author = "V"
title = "Counting Bits done Speedily"
date = "2026-08-18T14:00:00-04:00"
description = "Comparing different solutions to NeetCode problem 'Counting Bits'"
tags = ["Leetcode", "C"]
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
It will work and will even pass, but it is not "optimal".
Indeed, the optimal O(n) solution is everyone's favorite misnomer (1D) Dynamic Programming[^1].
I did foresee this possibility given the recurrence pattern consecutive numbers have. 
But I did not want to overthink the problem.
Still, it did leave me with something else.

## On Overdoing
To me that was just a little counter-intuitive.
DP makes sense when you otherwise have to do serious work, so going from O(n) to O(n^2) or even O(n^3) otherwise.
I can consider removing some huge constansts too.
But counting bits is not (relatively) serious work, it feels wasteful to use DP there.
You have O(bits * n) even in the naive solutions, which uses fast instructions in typical O(64) or O(32) per iteration.
There is even a trick to count bits with less iterations.
Thus, I wondered how would they compare to the optimal.

The implementation for the normal (naive) and human solution is as expected:
```c
for (unsigned int i = 1; i <= n; i++) {
    for (unsigned int j = 0; j < 32; j++) {
        out[i] += (i >> j) & 1;
    }
}
```
We do assume that `out` is zero-initialized, hence `i = 1` and not `i = 0`. This does not really matter anyway.

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

| Name   | Average Time (ms) |
|--------|-------------------|
| Normal | 23.88             |
| Trick  | 2.32              |
| DP     | 0.55              |

The normal version is an order or even two worse than others. 
The trick is a good speedup, though note that we are only going to n=1000 here. 
Trick is very efficient in these territories since it is based on number of bits.
This speed is not a given for larger numbers covering more bits.
Therefore, the DP is the "true optimal way" here.

## The Light
Well, there is one more way though, `popcnt`.
Introduced with SSE4.2 and SSE4a, and available on many CPUs, `popcnt` is able to take a 32 or 64 bit register and give you a count of 1s in a single instruction.
Technically `popcnt` would give us real O(1), which is even better than DP with its few operations.
However, `popcnt` is not free, and its implementation is not instant. 
Still, it is likely to be very optimized and fast in hardware, so let's see the implementation:
```c
for (unsigned int i = 1; i <= n; i++) {
    out[i] = __builtin_popcount(i);
}
```
As simple as it gets. Let's benchmark and add to the table:

| Name   | Average Time (ms) |
|--------|-------------------|
| Normal | 23.88             |
| Trick  | 2.32              |
| DP     | 0.55              |
| PopCnt | 2.28              |

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
Quite good, remember that the trick solution benefits from small numbers, so it lost even with advantage.
Unfortunately, it is not enough to beat the proper instruction.

So why does my CPU not support it? 
Well, my CPU does, it is a 10th gen i5 with AVX2 support. 
I doubt Intel considered `popcnt` support too expensive there.
I did not use `-march=native` or similar, only `-O3`, so it is possible that gcc failed to detect `popcnt` support.
I also use WSL on this machine, so that could be a wrench in the gears.
On the plus side, this does mean that I did not need to worry about the compiler converting my naive solutions into `popcnt`.
I can manually specify `-mpopcnt` anyway.

Hmm, still getting the fallback function, even with `-march=native`. It might be WSL, might be something else.
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
| Name          | Average Time (ms) |
|---------------|-------------------|
| Normal        | 23.88             |
| Trick         | 2.32              |
| DP            | 0.55              |
| PopCnt        | 2.28              |
| PopCnt (Real) | 0.84              |

Pretty good, we are at a point, where you can consider using this over the DP for simplicity reasons. 
DP is not complex, but it is not a single instruction either. 
It did require assembly on my end, but this is not the case generally.
Still, a shame I could not improve upon DP by abusing hardware (DP *is* abusing cache though).

Actually, there is still a way.

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
Relatively simple and clear. The plus block seems bad, but compiler will take care of that. As for the speed?

| Name             | Average Time (ms) |
|------------------|-------------------|
| Normal           | 23.88             |
| Trick            | 2.32              |
| DP               | 0.55              |
| PopCnt           | 2.28              |
| PopCnt (Real)    | 0.84              |
| Normal (SWAR 32) | 0.54              |

It beats `popcnt` and even DP by a slight margin. 
Hmm, that's weird, but it works and it matches the DP solution in hash of the output.
Looking at the assembly, it seems that my loop got unrolled, but otherwise nothing obvious.
But, as I was scrolling higher I saw that the compiler autovectorized this function.
I guess that is the solution, and I beat DP, even if it is a slight margin. 
Still, I want to go further by myself, how about without av?

| Name                | Average Time (ms) |
|---------------------|-------------------|
| Normal              | 23.88             |
| Trick               | 2.32              |
| DP                  | 0.55              |
| PopCnt              | 2.28              |
| PopCnt (Real)       | 0.84              |
| Normal (AV SWAR 32) | 0.54              |
| Normal (SWAR 32)    | 1.86              |

That's much more reasonable, the fact that my SWAR was that much faster than the fallback SWAR was a good tell alone.
Anyway, the improvement is great, so we can switch to 64 bit SWAR now to push further.
The code is much the same:
```c
for (uint64_t i = n & 1, res; i <= n; i += 2, res = 0) {
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
```
We go over two elements at once, using the fact that zero does not matter to make up for the uneven pairs.
The summation part is slightly optimized since compiler seems to need help here from my testing, whereas it works fine on 32-bit (i.e. one output).
We got a decent speed-up in exchange:

| Name                | Average Time (ms) |
|---------------------|-------------------|
| Normal              | 23.88             |
| Trick               | 2.32              |
| DP                  | 0.55              |
| PopCnt              | 2.28              |
| PopCnt (Real)       | 0.84              |
| Normal (AV SWAR 32) | 0.54              |
| Normal (SWAR 32)    | 1.86              |
| Normal (SWAR 64)    | 1.15              |

Looking good, but not quite the limit of my CPU.

[^1]: Yes, the name has a fun story with research on the wonderful Bellman equations. Still a horrible name.
