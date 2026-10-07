+++
author = "V"
title = "Common Lisp is pretty Fast"
date = "2026-10-05T18:00:00-04:00"
description = "Optimizing Common Lisp for a benchmark"
tags = ["CL", "SIMD", "Benchmark"]
draft = true
+++

## Intro
Recently I started playing with Common Lisp again, specifically [SBCL](https://www.sbcl.org) (arguably the best one).
Though it is an older LISP, per its [frozen ANSI standard](https://www.cs.cmu.edu/Groups/AI/html/hyperspec/clspec.html)[^1], it is still a joy to work with. 
Many features, many libraries, at a cost of symbols being all-caps (and a few other sins).
I have two projects that I am planning to finish with it, and might even talk about them if they turn out well.

Anyway, for one of the projects I was writing a matrix library, think of a BLAS, but neither comprehensive nor fast.
I did need it to be at least somewhat fast, so I was trying to optimize my CL code as much as I could.
SBCL won't match real BLAS, even veteran C can't do that. 
Still, you can get far with declarations for optimization, types, and of course using arrays and fixnums instead of linked lists and bignums.
These, thankfully, are mostly given when writing a matrix library, so I did that much.

Yet, the performance was still subpar. Besides more extreme or arcane optimizations, the only two big ones were SIMD and multi-threading. 
Multi-threading is tricky for my usecase, since I run these very often and the matrices should have a good size to make the overhead worth it.
But SIMD would be an easy win, if only we had it in the dynamic lisp world. 

Oh, right, there is a library for that, [sb-simd](https://github.com/sbcl/sbcl/tree/master/contrib/sb-simd), included with SBCL, so might as well call it official.
It provides a pretty nice API for using different versions of SSE, FMA, AVX, and NEON instructions.
There are limitations with boxing in some cases e.g. passing to functions, but it does not apply in my usecase.
Using AVX2 SIMD gave me an immediate huge speedup, theoretically 8x, though practically not quite that much. 
SIMD was probably the strongest optimization I had to use so far regardless.

The fact that SBCL had SIMD was a surprise for me, since I did not really think it would. 
I don't believe SBCL has auto-vectorization yet, certainly not as developed as GCC and Clang.
That does give me hope that one day SBCL will be able to auto-vectorize too.
Regardless, this fact did remind me of something: language benchmarks, specifically [this site](https://benchmarksgame-team.pages.debian.net/benchmarksgame).

## Onto Benchmarking
I actually talked about language benchmarks in an older post of mine, though I did not like how it turned out.
One of my major points was the inherent unfairness of SIMD and multi-threading support. 
Languages like C have intrinsics, which generally map to good SIMD instructions, 
and built-in libraries like OpenMP, which allows you to parallize some parts of your program quite easily.

This is, of course, a positive of a language, but it will make benchmarks very unfair. 
C with SIMD and OpenMP can easily achieve 10x improvement (if not more) over a language that does not in an optimized benchmark. 
Whereas in more "usual" coding you typically don't reach for SIMD and OpenMP every day. 
You might not even be able to use on many problems that are not just clean array operations.

As an example, [n-body problem](https://benchmarksgame-team.pages.debian.net/benchmarksgame/performance/nbody.html) is dominated by the likes of C, C++, and Rust.
Those are the fastest languages, so this should be a given.
Yet, if we are more specific, it is dominated by SIMD. 
8 of 13 top solutions within 2x of the fastest one (SIMD C), as well as 5 out of top 5, have explicit SIMD.
*Note that I said explicit SIMD.* 
The other fast solutions of Fortran, Rust, Chapel, and Julia are not SIMD-free, since auto-vectorization is not disabled.
All compiler flags beside Chapel include ivybridge arch, which implies AVX2.
Chapel is a parallel-focused language, which compiles to C, so it too uses auto-vectorization.

[^1]: Some of those links seem to be dead at the momemnt, but you can download the zipped archive. 
This is not the "true" spec, but closest you can get without a purchase or Big AI strategies.
