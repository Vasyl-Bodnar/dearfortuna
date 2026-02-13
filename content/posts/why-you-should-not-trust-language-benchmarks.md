+++
author = "V"
title = "Why You Should Not Trust Language Benchmarks"
date = "2026-02-11T14:00:00-04:00"
description = "Why you should not trust language benchmarks."
tags = ["C", "C++", "OCaml", "Benchmark"]
draft = false
+++

## Intro
Recently I felt drawn back to OCaml.
It was always one of my favorite languages, but it did have its issues with the standard library (There is Base, but I'd rather not).
And, OCaml always lacked some optimization powers of a lower language like C.
However, this is 2026, the Year of OCaml, maybe.

Plenty of QoL and new cool things like effects are added to the language and the actual standard library.
There is also OxCaml, a bunch of extensions for more QoL, better control over allocation, layouts, etc..
Some of those extensions are even being added to OCaml itself!
A dream come true for me certainly.

With all these thoughts, though, I have made a mistake in trying to look at OCaml benchmarks.
It is well known that there are lies, damn lies, and (language) benchmarks.
Comparing languages and their implementations is often a futile effort. 
At best you could argue for tiers, e.g. system language vs GCed language vs Python.
But with things like OxCaml and Numpy Python, even that can be unreliable.
However, it is an innocent idea in theory, just search for "OCaml performance benchmarks".

On Google, 
the first few results are two 2020 `discuss.ocaml.org` posts, 
one informationless blog post that I hope was AI-generated, 
and a proudly vercel benchmarking site, [Programming-Language-Benchmarks](https://programming-language-benchmarks.vercel.app/ocaml-vs-go).
Lower you can also see [Benchmarks Game](https://benchmarksgame-team.pages.debian.net/benchmarksgame/measurements/ocaml.html),
Rust vs OCaml Medium article and some resources for benchmarking actual OCaml code.
The article requires an account, so I can't see the actual benchmark, 
but it seems to focus on actual language differences like GC in the intro.
We will be looking at the benchmark aggregates in PLB and BG though.

But first, a warmup example.

### C++ is slower than JavaScript
There are plenty of posts on every kind of tech and tech-adjacent site about how JavaScript somehow outperforms C or C++.
e.g. 

[Why does JavaScript appear to be 4 times faster than C++?](https://stackoverflow.com/questions/17036059/why-does-javascript-appear-to-be-4-times-faster-than-c)

The provided sources are simple and feel equivalent (well maybe Date vs clock()):
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

So why is JS faster? Some could say V8 magic.
All kinds of tricks are used like pointer tagging, NaN-boxing, type predictions, not to mention just having a great GC. 
What should have been interpreted floats in theory magically turn to assembly bits. 
Maybe JIT has finally gotten to the point where it makes AOT obsolete.
But it really is just microbenchmarks. 
JIT are great at optimizing hot loops. 
The real performance though tends to fall off, especially with all the objects and type messes. 

Use of the language in the real software, 
rather than specialized algorithms or data structures, can be very different.
Hence the term "microbenchmarks". 
But even benchmarks of real software can hide that the difference is not in language, but in implementation of software itself.
Oftentimes there are differences in architecture, design, usecases. 
Not to mention that newer works benefit from mistakes of previous works.
Sometimes rewriting a C codebase in Rust (within reason) is beneficial for the sole reason that you now know what to do.

Of course, in this case, the OP just forgot to use optimizations for C++ whereas Node did it for him in JS.
I am not even talking about any advanced performance sensitive flags, just good old `-O2`.

Another common issue is non-equivalent programs.
E.g. a C program that is forced to convert a long to float every iteration vs 
JS one that simply used floats from the start (in theory everywhere, but reality is different).
There is also a subissue there with "idiomatic code". 
Idiomatic is not necessarily the most performant, but it is often the expected one for the language.
Thus, there is the issue of when to forsake idiomatic code for performant code

## The Numerous Problems
### SIMD, Multi-threading?
When you use lower languages like C and C++, is it fair to use SIMD?
That is a good question as SIMD can more accurately represent the ceiling of the language. 
At the same time, not all languages even provide SIMD intrinsics. 

No need to mention Python or Ruby, when even Go required you to just write straight assembly until Go 1.26 (released yesterday) and it is still under an experimental flag!
OCaml by virtue of OxCaml extensions now has SIMD intrinsics, but it did not have them before in any non-C way.
Given that Go and OCaml theoretically have SIMD support, 
it is harder to say that comparing them without SIMD to C with SIMD is fair.

You could call C for many of these other languages.
However, at that point, you can lose a lot of performance due to FFI overhead. 
The overhead itself also depends on where and how SIMD was used. 
And if we are talking about using C FFI, all bets are off anyway. 
What is the difference between implementing a small part in C vs just the entire algorithm.
Not ideal.

There is also the question of whether it is realistic. Even in C you don't just reach for SIMD whenever due to lack of abstractions. 
But then what should you do if the languages do provide those nice abstractions. 
They can make the SIMD a lot easier, portable, and simply accessible.
Zig has fine abstractions for SIMD in the core language. 
Rust has them in its nightly builds. 
Similar abstractions also seem to be the goal for Go once they are finished with implementing intrinsics.

Using multiple CPU threads is another issue. 
Many languages support POSIX threads, but ergonomics can be awful.
Some languages provide better abstractions, many don't. 
C and C++ suprisingly make it easy with OpenMP, at least for more embarassing problems.

### Implementation?
Furthermore, different languages may optimize for different purposes. 
Scheme, e.g., must be tail-recursive. 
It is even specified in exactly what ways tail-recursion must be handled in the standard. 
JavaScript implementations (besides Safari somehow) do not support it. 
Yet, it is actually in the ES6 spec (from *2015*). 
C does it, but there is no requirement nor guarantee generally.
Tail-recursion optimization can turn recursion into a while loop, 
so it is a very important feature of the language implementation.

These problems relate to issue of equivalence too. 
Should a C program using OpenMP, tail-recursion, and SIMD even be considered similar to any program that does not or can not. 
Hell, even if you are not using SIMD yourself, GCC automatically enables SSE2 for x86_64 programs,.
SSE2 matters for libc (e.g. memcpy or memset), not to mention potential auto-vectorization.
To go even further, what if a C program uses `--ffast-math` to optimize floating point? 
If the results are the same, does it even matter if that C program ignores IEEE compliance.
Means do matter.

There is an argument for separating language spec and implementation. 
In majority of cases there is already one least unpopular choice, 
even if it is questionable in performance, like CPython. 
The issue is that some languages don't *really* have a spec or common implementation, see Scheme for latter. 
Or, the spec might not be respected in all cases like the tail-recursion in ES6 for example.
As another example, OxCaml provides many performance-related extensions 
that vanilla OCaml does not have yet.

Different implementations may suffer from lack of support. 
E.g. Pypy is much faster than CPython, benchmarks will show as much. 
However, there is a reason why PyPy has not replaced CPython with their incompatibilities.
Another example is tinygo, which uses LLVM for backend. 
It can outperform go in some benchmarks, but it lacks certain features and has a different focus from go for real use.

Talking about LLVM, far too many languages are just LLVM frontends. 
This is for pragmatic reasons, however, it does make comparing them a bit difficult.
An extra annoyance is that some languages may indeed optimize above LLVM (e.g. Rust), others may not at all.

Note that the issue is not only in that these questions are often unanswered, 
but in that answers to them are often controversial. 
It is easy to say or enforce one way, like banning SIMD, 
but it is hard to convince those who disagree.

### Exhibit #111
There are plenty of sites and blog posts for ranking languages by performance (as most other metrics are even more pointless).
[Programming-Language-Benchmarks](https://programming-language-benchmarks.vercel.app) (I will use PLB for short)
is certainly one of the better ones. 
It states that it is influenced by [Benchmarks Game](https://benchmarksgame-team.pages.debian.net/benchmarksgame/index.html) (BG),
and you can see the similarities. 
Their primary purpose is to gather the fastest solutions for different language.

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
And this is the same language and compiler.
Not to mention all the different implementations inbetween in also optimized Rust and C++ code. 

There are still plenty of more naive solutions as well, which in some cases can make C slower than (optimized) Java.

Regardless, from these problems it is very hard to tell what language is "better". 
It is possible that someone wrote a very fast C++ solution with AVX against a poor SSE C solution.
Both have SIMD, but different tiers. 

BG does provide some box plots which show you the general floor, ceiling, mean, as well as outliers.
This in theory provides a better description of the language capability. 
In reality, there really isn't enough and all the problems I mentioned can even be compounded.

Another issue was, as I mentioned, potentially misleading remarks. 
The original PLB page that I shared was to OCaml vs Go comparison (interesting that this was the top result and not OCaml's page).
The first problem for me was that `1-m.go`, multi-threaded version, was slower than `1.go`, the base one. 
Granted `1.go` was tinygo and `1-m.go` was go, but unless go is an order of magnitude slower to cancel out multi-threading, something was off. 
Indeed, if you just hover the links you will realize that `1-m.go` link points to `1.go`. 
Interestingly, all `-m.go` problems except for spectral norm point to non-m variants, which does track with what numbers we get.
This is in the benchmark yaml file for go, so I am not sure why the website values don't match.

If you look over all the problems you will indeed see that tinygo and go are trading blows in speed and memory use. 
OCaml is behind in speed and memory, but usually not too far off, even winning in a few select cases. 
If we use C solution `2.c` (no SIMD) for nbody problem as a baseline, then go's speed is 11% slower, OCaml's is 17% slower.
So now that we have such a comparison, what does it really tell us? Nothing. 

C is a good baseline as it is typically the fastest with lowest overhead. 
But realistically 11% and 17% are pretty close. 
The difference is essentially ~300 ms vs ~350 ms, not something you will even tell apart.
The larger difference is memory usage, being 2MB vs 3.5MB vs 5MB, but you would expected that from GCs. 
We are not using floppys as our RAM, so that is extremely negligible.
So C is not fast, Go and OCaml are somewhat close. But that was just one nbody problem.

I could instead pick nsieve, where C is ~250 ms, Go is ~300ms, and OCaml is ~900ms. 
Huh, suddenly OCaml looks like a grade below those two. 
Now you can suddenly make an argument that OCaml is much worse. 
Indeed on most of these problems, OCaml is trailing lightly behind Go, but on others it is nearly an order away.
You could just chuck it up to GC being GC, maybe some optimization differences, maybe boxes since people like `ref` too much.
Still, how can you make an argument that OCaml is close or far from Go, when the results are so varied.
I guess you could say that because OCaml is varying so much, it is strictly worse. 
But what if your usecases align perfectly. 
What if you don't pick a language based on a language benchmark.

In the end, the reality is that to make a conclusion you would need to read the source codes, 
check the compile flags, etc. to understand the differences, trade-offs, benefits. 
This inadvertedly requires at least passing understanding of the languages you look at. 
But at that point you could already be familiar with a language enough to gauge its speed.
A beginner, on the other hand, might misinterpret the results completely without any knowledge to guide.

Note that these sites are likely made in jest, 
or mostly to collect optimized solutions to problems in a way that is gradable. 
I doubt (as in hope this is not the case) the authors genuinely believe that they are fairly comparing languages.
But these sites can be misused, misunderstood, or twisted in ways that are orthogonal to their purpose. 
At the same time, a much worse example would be someone with something to prove.

### Intent matters
As for any statistic, often it is not the numbers but the flags. 
PLB and BG don't seem to own any horses in the race, 
though, it is not impossible for even personal bias to show up.
In a lot of cases, it is on the author of the benchmark to tweak algorithms to make their predetermined winner look better.
Here is a [cool guide](https://leveluppp.ghost.io/how-to-lie-with-benchmarks/) if you want to lie better.

It is not rare in this day and age for an innocent blog to be a thinly veiled advertisement for something. 
Many Github repos these days are faces of a product to sell or popularize.
Benchmarks far too often are convenient for this, as people look at faster speed and better resource usage in awe.
The same benchmarks that can easily be misleading, manipulated, cherry-picked, and abused.

## The End
Don't do language benchmarks, kids. 
It is a waste of time, memory, and gzipped bytes.
