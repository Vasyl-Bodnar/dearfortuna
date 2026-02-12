+++
author = "V"
title = "Why You Should Not Trust Language Benchmarks"
date = "2026-02-11T14:00:00-04:00"
description = "Why you should not trust language benchmarks."
tags = ["C", "C++", "OCaml", "Benchmark"]
draft = true
+++

## Intro
Recently I felt drawn back to OCaml.
It was always one of my favorite languages, but it did have its issues with standard library (There is Base, but I'd rather not).
And, OCaml always lacked some optimization powers of a lower language like C.
However, this is 2026, the Year of OCaml, maybe.

Plenty of QoL and new cool things like effects are added to the language and the actual standard library.
There is also OxCaml, a bunch of extensions for more QoL, better control over allocation, layouts, etc..
Some extensions are even being added to OCaml itself!
A dream come true for me certainly.

With all these thoughts, though, I have made a mistake in trying to look at OCaml benchmarks.
It is well known that there are Lies, damn Lies, and (Language) Benchmarks.
However, it is an innocent idea in theory, just search for "OCaml performance benchmarks".

On Google, 
the first few results are two 2020 `discuss.ocaml.org` posts, 
one informationless blog post that I hope was AI-generated, 
and a proudly vercel benchmarking site, [Programming-Language-Benchmarks](https://programming-language-benchmarks.vercel.app/ocaml-vs-go).
Lower you can also see [Benchmarks Game](https://benchmarksgame-team.pages.debian.net/benchmarksgame/measurements/ocaml.html),
Rust vs OCaml Medium article and some resources for benchmarking actual OCaml code.
The article requires an account, so I can't see the actual benchmark, 
but it seems to focus on actual language differences like GC in the intro.
We will be looking at the benchmark aggregates in PLB and BG.

But first, a warmup example.

### C++ is slower than JavaScript
There are plenty of posts on every kind of tech and tech-adjacent site about how JavaScript somehow outperforms C or C++.
e.g. 

[Why does JavaScript appear to be 4 times faster than C++?](https://stackoverflow.com/questions/17036059/why-does-javascript-appear-to-be-4-times-faster-than-c)

The provides sources are simple and feel equivalent (well maybe Date vs clock()):
```js
(function() {
    var a = 3.1415926, b = 2.718;
    var i, j, d1, d2;
    for(j=0; j<10; j++) {
        d1 = new Date();
        for(i=0; i<100000000; i++) {
            a = a + b;
        }
        d2 = new Date();
        console.log("Time Cost:" + (d2.getTime() - d1.getTime()) + "ms");
    }
    console.log("a = " + a);
})();
```
```c
int main() {
    double a = 3.1415926, b = 2.718;
    int i, j;
    clock_t start, end;
    for(j=0; j<10; j++) {
        start = clock();
        for(i=0; i<100000000; i++) {
            a = a + b;
        }
        end = clock();
        printf("Time Cost: %dms\n", (end - start) * 1000 / CLOCKS_PER_SEC);
    }
    printf("a = %lf\n", a);
    return 0;
}
```

Some could say V8 magic.
All kinds of tricks are used like pointer tagging, NaN-boxing, type predictions, not to mention just having a great GC. 
Maybe JIT finally gotten to the point where it makes AOT obsolete.
But it really is just microbenchmarks. 
JIT are great at optimizing hot loops. 
What should have been interpreted floats magically turn to assembly bits. 
The real performance though tends to fall off, especially with all the objects and type messes. 

Of course, in this case, the OP just forgot to use optimizations for C++ whereas Node did it for him in JS.
I am not even talking about any advanced performance sensitive flags, just good old `-O2`.

Another common issue is non-equivalent programs.
E.g. a C program that is forced to convert a long to float every iteration vs 
JS one that simply used floats from the start (in theory everywhere, but reality is different).
There is also a subissue there with "idiomatic code". 
Idiomatic is not necessarily the most performant, but it is often the expected one for the language.
Thus, there is the issue of when to forsake idiomatic code for performant code

## The Numerious Problems
### SIMD, Multi-threading?
When you use lower languages like C and C++, is it fair to use SIMD?
That is a good question as SIMD can more accurately represent the ceiling of the language. 
At the same time, not all languages even provide SIMD intrinsics. 

No need to mention Python or Ruby, when even Go required you to just write straight assembly until Go 1.26 (released yesterday) and it is still under an experimental flag!
OCaml by virtue of OxCaml extensions now has SIMD intrinsics, but it did not have them before in any non-C way.
You could call C for many of these other languages.
However, at that point, you can lose a lot of performance due to overhead depending on where and how SIMD was used. 
Not ideal.

There is also a question of whether it is realistic. Even in C you don't just reach for SIMD whenever due to lack of abstractions. 
But then what should you do if the languages do provide those abstractions. They make the SIMD a lot easier, portable, and simply accessible.
Zig famously has abstractions for SIMD. 
Rust has similar in its nightly builds. 
These abstractions also seem to be the goal for Go once they are finished with the intrinsics.

Using multiple CPU threads is another issue. 
Many languages support POSIX threads, but ergonomics can be awful.
Some languages provide better abstractions, many don't. 
C and C++ suprisingly make it easy in a lot of cases with OpenMP.

### Implementation?
Furthermore different languages may optimize for different purposes. 
Scheme, e.g., must be tail-recursive. 
It is even specified in exactly what ways tail-recursion must be handled in the standard. 
JavaScript implementations, besides Safari somehow, do not support it. Yet, it is actually in the ES6 spec (from *2015*). 
C does it, but there is no requirement nor guarantee generally.
Tail-recursion optimization can turn recursion into a while loop, 
so it is a very important feature of the language implementation.

These problems relate to issue of equivalence too. 
Should a C program using OpenMP, tail-recursion, and SIMD even be considered similar to any program that does not. 
Even if you are not using SIMD yourself, GCC automatically enables SSE2 for x86_64 programs, 
which matters for libc (e.g. memcpy or memset), not to mention potential auto-vectorization.
To go even further, what if a C program uses `--ffast-math` to optimize floating point? 
If the results are the same, does it even matter if that C program ignores IEEE compliance.
Means do matter.

There is an argument for separating language spec and implementation. 
It does not help that a lot of languages wrap LLVM for backend, even if they do other optimizations themselves.
And in majority of cases there is already one least unpopular choice, 
even if it is questionable in performance, like CPython. 
The issue is that some languages don't *really* have a spec. 
Or, it might not be respected in all cases like the tail-recursion in ES6 for example.
As another example, OxCaml provides many performance-related extensions 
that vanilla OCaml does not have yet.

Note that the issue is not only in that these questions are often unanswered, 
but in that answers to them are often controversial. 
It is easy to say or enforce one way, like banning SIMD, 
but it is hard to convince those who disagree.

### Exhibit #01
There are plenty of sites and blog posts for ranking languages by performance (as most other metrics are even more pointless).
[Programming-Language-Benchmarks](https://programming-language-benchmarks.vercel.app) (I will use PLB for short)
is certainly one of the better ones. 
It states that it is influenced by [Benchmarks Game](https://benchmarksgame-team.pages.debian.net/benchmarksgame/index.html) (BG),
and you can see the similarities.

PLB and BG both provide a set of problems and solutions in different languages. 
Some of these problems are very basic like "helloworld", 
others are more complicated (relatively) data structures and algorithms like LRU, "Least Recently Used".
Unlike many other benchmarks, these sites do provide the source code, the flags, and everything that they did.
Though you would have to trust them on properly benchmarking these with warmup and whatnot.

PLB and BG do good in mentioning if the code uses multi-threading, SIMD, or in BG's case some other forms of unsafe/horrid optimizations.
PLB has a different name (e.g. `1.c` vs `1-i.c`), BG has a star, which I would prefer (as in all bets are off). 
The big benefit of BG is that it has a lot more implementations and problems, but PLB has plenty itself.
And these do matter, as some implementation are significantly worse. 
For example, n-body problem on BG has `* C gcc #9` on the top when sorted by time, optimized with 256bit AVX SIMD intrinsics.
On the other hand, `* C gcc #4` only uses 128bit SSE, so unless its algorithm is a direct upgrade, it simply has to be worse.
Not to mention all the different implementations in also optimized Rust and C++ code. 

Regardless, from these problems it is very hard to tell what language is "better". 
It is possible that someone wrote a very fast C++ solution with AVX against a poor SSE C solution.
Both have SIMD, but different tiers. 
BG does provide some box plots which show you the general floor, ceiling, mean, as well as outliers.

In the end, the reality is that to make a conclusion you would need to read the source codes, 
check the compile flags, etc.

...

### Intent matters
As for any statistic, sometimes it is not the numbers but the flags. 
PLB and BG don't seem to own any horses in the race, 
but it is not impossible for even personal bias to show up. 
In a lot of cases, it is on the author of the benchmark to tweak algorithms to make their predetermined winner look better.
Here is a [cool guide](https://leveluppp.ghost.io/how-to-lie-with-benchmarks/) if you want to lie better.

...

## The End
Don't do language benchmarks kids.
