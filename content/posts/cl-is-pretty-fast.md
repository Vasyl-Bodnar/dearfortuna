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

This is, of course, an important positive of a language, but it will make benchmarks very unfair. 
C with SIMD and OpenMP can easily achieve 10x improvement (if not more) over a language that does not in an optimized benchmark. 
Whereas in more "usual" coding you typically don't reach for SIMD and OpenMP every day. 
You might not even be able to use them on many real problems.
Auto-vectorization can be helpful in those cases, but again, it is still a little unfair to compare SIMD time to non-SIMD time in my opinion.

As an example, [n-body problem](https://benchmarksgame-team.pages.debian.net/benchmarksgame/performance/nbody.html) is dominated by the likes of C, C++, and Rust.
Those are generally the fastest languages, so this should be a given.
Yet, if we are more specific, it is dominated by SIMD. 
N-body does lend itself to a simple SIMD speed-up.
The top 5 solutions all have explicit SIMD, where most SIMD instructions are directly specified.
Note the *explicit SIMD* part. 
The other fast solutions of Fortran, Rust, Chapel, and Julia are not SIMD-free, since auto-vectorization is not disabled.
Most compilers even directly specify ivybridge arch, which implies AVX2 support. 
Though the auto-vectorized SIMD will indeed be lesser[^2], hence the fastest solutions are still explicit.

The differences between language solutions are interesting. 
Fastest Rust with SIMD is 60% faster than fastest non-explicit-SIMD Rust. 
Fastest SIMD C# (native AOT compiled while looking arcane in some parts) is 2x faster than fastest non-explicit-SIMD C# (also native AOT).
Judging either by SIMD is not going to tell you much about how fast the usual language is, especially with native C# which is not always a given.

More importantly, what about languages without SIMD, or more exactly without SIMD solutions.
One would imagine that Go is at least in the same league as C#, and indeed the fastest Go solution is faster than the non-explicit-SIMD native C#.
But Go did not have SIMD intrinsics until recently, and even then it is still experimental, so there is no SIMD Go solution to compare against.
Maybe Go can be optimized to same heavy degree as that top 4 C# solution, maybe not, who knows.
Maybe that non-explicit-SIMD solution for C# is bad, given that it is also slower than fastest solutions for Java and Haskell.
Is Haskell faster than native C#? Node.js is slower, but only 20% slower. Is native C# that bad?

To put a proper test to it, I would want to take a language that has a pretty "slow" non-SIMD solution and make it pretty fast with SIMD.
And you might already know what language this will be.

## A Lisp with a Dream
Our friend here, "Lisp SBCL #2", is 3.7 times slower than baseline SIMD C solution, took 7.83 seconds where C took 2.10, with 10 times more memory usage.
It has been optimized over time with all the common tricks of using vectors where better, providing types, and asking kindly to optimize for speed.
But it does not use SIMD, and since SBCL does not auto-vectorize, it might not even use any SIMD besides basic floating point calculations.
Here is the core function:
```lisp
(defun nbody (n)
  (declare (fixnum n))
  (let ((system (list *sun* *jupiter* *saturn* *uranus* *neptune*)))
    (offset-momentum system)
    (format t "~,9F~%" (energy system))
    (dotimes (i n)
      (advance system 0.01d0))
    (format t "~,9F~%" (energy system))))
```
Elegant and simple, all planets are defined and the major functions of `offset-momentum`, `energy`, and `advance` are not that much larger.
To add SIMD, we can simply tweak those functions (and the one below them `applyforces`).

[^1]: Some of those links seem to be dead at the momemnt, but you can download the zipped archive. 
This is not the "true" spec, but closest you can get without a purchase or Big AI strategies.
[^2]: When anyone tells you that a compiler is smarter than you, kindly remind them of SIMD and multi-threading.
Just because a compiler can replace a division with reciprocal multiplication or 
[a pop-count trick with an actual pop-count instruction](http://localhost:1313/posts/counting-bits-speedily) does not mean it is omniscient and omnipotent in nearly all cases.
