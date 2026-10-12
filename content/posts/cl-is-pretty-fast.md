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
Yet, if we are more specific, it is dominated by SIMD, which n-body does lend itself to. 
The top 4 solution is actually native C#, which you can make fast,
but it is not fast on its own due to GC and various VM language weirdnesses.
Especially with how "extra" optimized it is, even in comparison to C, C++, and Rust.
Overall, the top 5 solutions all have explicit SIMD, where most SIMD instructions are directly specified.

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

There is a good reason why even the website itself states multiple times to always look at the implementation. 
The website even has stars to denote unsafe and (explicit) SIMD usage on solutions. 
Still, the graphs and leaderboards will be misleading even with such measures in my opinion. 
To quote the site, or at least whoever the site quotes:

> ... a pretty solid study on the boredom of performance-oriented software engineers grouped by programming language.

Regardless of the quote, to put a proper test to it, 
I would want to take a language that has a pretty "slow" non-SIMD solution and make it pretty fast with SIMD (and extra).
And you might already know what language this will be.

## A Lisp with a Dream
Our friend here, "Lisp SBCL #2", is 3.7 times slower than baseline SIMD C solution, 
took 7.83 seconds where C took 2.10, with 10 times more memory usage.
It has been optimized over time with all the common tricks of using vectors, providing types, and asking kindly to optimize for speed.
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
To add SIMD, we can simply tweak those functions (and the one below them `applyforces`) and any necessary structures. 
I will utilize a similar method to the top SIMD solutions.

Before that I do have to mention the [Idiomatic Lisp and the nbody benchmark](https://www.stylewarning.com/posts/nbody), 
by the author of [Coalton](https://coalton-lang.github.io).
Now this does show SBCL with a small kernel macro lib matching C in n-body while looking quite lean, though it is a non-explicitly-SIMD C.
This is different from what I will be doing, since I will be writing straight SIMD intrinsics inline and comparing to a C solution that does the same.
With this cleared up, we shall proceed.

Firstly, we do need to setup the environment. 
My SBCL version is 2.5.2, theirs is 2.4.8. 
This difference does not matter since all my comparisons will be relative to my numbers on my machine anyway.
They do use a setup with extra files to compile and turn the program into a core which then will be used with a couple of compiler flags.
This setup was not significant in testing, at best a 1% time difference from just evaling `(time (nbody n))` in my tests, which is very much in error margins.
Still, I will use that for the final time.

After running the original solution, we have the minimum time of `14.5 seconds`. 
This is double the number on the website, but that's the difference in CPUs, environment, etc., and is not important.
This time will be our baseline. 
I will also compare to the time of the solutions in SIMD and non-explicit-SIMD C[^3] to establish the factor difference without using their entire benchmarking suite.

### Waste is Good, Actually
A fun thing to see in the top SIMD solutions is that the position and velocity have a wasted dimension:
```c
// n-body C gcc #9
// jupiter
m[1] = 9.54791938424326609e-04 * SOLAR_MASS;
    p[1] = _mm256_setr_pd(0.0,
         4.84143144246472090e+00,
        -1.16032004402742839e+00,
        -1.03622044471123109e-01);
    v[1] = _mm256_setr_pd(0.0,
         1.66007664274403694e-03 * DAYS_PER_YEAR,
         7.69901118419740425e-03 * DAYS_PER_YEAR,
        -6.90460016972063023e-05 * DAYS_PER_YEAR);
```
That zero does not represent anything, it is just waste.
This waste does allow us to use simpler SIMD arithmatic without having to juggle values however. 
Now I could pretty much do the same thing for Lisp, but I am afraid of triggering boxing by accident, 
so the better solution is to just store all these values as a vector and then access them by index.

### Setup the Vectors
The current solution already uses a vector, though through a struct and without waste:
```lisp
(defstruct (body (:type (vector double-float))
                 (:conc-name nil)
                 (:constructor make-body (x y z vx vy vz mass)))
  x y z
  vx vy vz
  mass)
(deftype body () '(vector double-float 7))
```
Even though it does not impact performance or vector shape, we do not really need this extra setup. 
We can just use a straight vector (though I now use simple-array) with 9 values (2 wasted) throughout:
```lisp
(deftype body () '(simple-array double-float (9)))

(defmacro make-body (x y z vx vy vz mass)
  `(make-array 9 :element-type 'double-float
                 :initial-contents (list ,x ,y ,z 0.0d0 ,vx ,vy ,vz 0.0d0 ,mass)))
```
Make-x is best reserved for actual structs, but it will do. 
Since we have yet to add vector instructions, we should make sure the code will still work:
```lisp
(defmacro x (body)
  `(aref (the body ,body) 0))
(defmacro y (body)
  `(aref (the body ,body) 1))
;; etc.
```
The forced type is necessary since otherwise SBCL cannot guess the type.
Without the types, program time is more than doubled to 33 seconds (the benefit of a typed struct, and types in general).
With the types provided as I did, we keep our performance while using a straight-forward vector under. 

### Slow and Steady
I will update the functions one by one, so we will start with the `applyforces`:
```lisp
(declaim (inline applyforces))
(defun applyforces (a b dt)
  (declare (type body a b) (type double-float dt))
  (let* ((dx (- (x a) (x b)))
         (dy (- (y a) (y b)))
         (dz (- (z a) (z b)))
         (distance (sqrt (+ (* dx dx) (* dy dy) (* dz dz))))
         (mag (/ dt (* distance distance distance)))
         (dxmag (* dx mag))
         (dymag (* dy mag))
         (dzmag (* dz mag)))
    (decf (vx a) (* dxmag (mass b)))
    (decf (vy a) (* dymag (mass b)))
    (decf (vz a) (* dzmag (mass b)))
    (incf (vx b) (* dxmag (mass a)))
    (incf (vy b) (* dymag (mass a)))
    (incf (vz b) (* dzmag (mass a))))
  nil)
```
It is already inlined and has the key types. 
It looks very vectorizable as it is too. 
So let us do a simple SIMD conversation on these then:
```lisp
(declaim (inline applyforces))
(defun applyforces (a b dt)
  (declare (type body a b) (type double-float dt))
  (let* ((dp (f64.4- (vec-p a) (vec-p b)))
         (distance (sqrt (f64.4-horizontal+ (f64.4* dp dp))))
         (mag (/ dt (* distance distance distance))))
    (declare (type double-float distance mag))
    (setf (vec-v a) (f64.4- (vec-v a) (f64.4* dp mag (mass b))))
    (setf (vec-v b) (f64.4+ (vec-v b) (f64.4* dp mag (mass a)))))
  nil)
```
I had to add the extra type declaration for `distance` and `mag`, since they needed it.
It is certainly more compact since we are now working directly on xyz and vxvyvz together.
I also had the liberty to replace what would be `f64.4-incf/f64.4-decf` with equivalent `setf`. 
The reason for this change is that it actually is a significant performance improvement[^4]. 
The same is not true for the original solution, so they did not make a mistake, just a defficiency in vector incf and decf.

Anyway, what's the time? Roughly `5.1 seconds`.
Indeed, nearly 3 times faster than the original solution.
I have not changed any other function yet, only this one, though it is a key one.

Next towards `advance`:
```lisp
(defun advance (system dt)
  (declare (double-float dt))
  (loop for (a . rest) on system do
    (dolist (b rest)
      (applyforces a b dt)))
  (dolist (b system)
    (incf (x b) (* dt (vx b)))
    (incf (y b) (* dt (vy b)))
    (incf (z b) (* dt (vz b))))
  nil)
```
There is only one obvious vectorization here, so we shall take care of that:
```lisp
(defun advance (system dt)
  (declare (double-float dt))
  (loop for (a . rest) on system do
    (dolist (b rest)
      (applyforces a b dt)))
  (dolist (b system)
    (setf (vec-p b) (f64.4+ (vec-p b) (f64.4* dt (vec-v b)))))
  nil)
```
Now our time is further dropped to `4.2 seconds`. 
In case you are curious, using `f64.4-incf` does not seem to impact performance here, but I will be consistent.

The next function, `energy`, is a lot less boring, since it has a couple of things going on:
```lisp
(defun energy (system)
  (let ((e 0.0d0))
    (declare (double-float e))
    (loop for (a . rest) on system do
      (incf e (* 0.5d0
                 (mass a)
                 (+ (* (vx a) (vx a))
                    (* (vy a) (vy a))
                    (* (vz a) (vz a)))))
      (dolist (b rest)
        (let* ((dx (- (x a) (x b)))
               (dy (- (y a) (y b)))
               (dz (- (z a) (z b)))
               (dist (sqrt (+ (* dx dx) (* dy dy) (* dz dz)))))
          (decf e (/ (* (mass a) (mass b)) dist)))))
    e))
```
However, SIMD usage itself is limited as we only need `e` without changing everything more severely.
Still a speedup is a speedup:
```lisp
(defun energy (system)
  (let ((e 0.0d0))
    (declare (double-float e))
    (loop for (a . rest) on system do
      (incf e (* 0.5d0
                 (mass a)
                 (f64.4-horizontal+ (f64.4* (vec-v a) (vec-v a)))))
      (dolist (b rest)
        (let* ((dp (f64.4- (vec-p a) (vec-p b)))
               (dist (sqrt (f64.4-horizontal+ (f64.4* dp dp)))))
          (decf e (/ (* (mass a) (mass b)) dist)))))
    e))
```
Nice, but the time is still `4.2 seconds`. 
The energy is run twice, so while it is sped-up, this is not actually significant to our time.
Interestingly, while you can say that this was a waste of time, it took me like 30 seconds. 
If we imagine a future where `energy` is used more often, then this is an easy win. 
If you don't imagine that future, a speed-up is a speed-up. 
Don't even dare to mention that one anti-optimization quote if you don't know its history.

Well, we are left with the last one, `offset-momentum`:
```lisp
(defun offset-momentum (system)
  (let ((px 0.0d0)
        (py 0.0d0)
        (pz 0.0d0))
    (dolist (p system)
      (incf px (* (vx p) (mass p)))
      (incf py (* (vy p) (mass p)))
      (incf pz (* (vz p) (mass p))))
    (setf (vx (car system)) (/ (- px) +solar-mass+)
          (vy (car system)) (/ (- py) +solar-mass+)
          (vz (car system)) (/ (- pz) +solar-mass+))
    nil))
```
I will prepare you though, this one is only called once, so it matters even less to vectorize. 
But I shall do so anyway to show how easy it is to do:
```lisp
(defun offset-momentum (system)
  (let ((pp (make-f64.4 0.0d0 0.0d0 0.0d0 0.0d0)))
    (dolist (p system)
      (setf pp (f64.4+ pp (f64.4* (vec-v p) (mass p)))))
    (setf (vec-v (car system)) (f64.4/ (f64.4- pp) +solar-mass+))
    nil))
```
Looks so small and feels so optimized.
The time of course did not change.

So on this somber note, we have the final time of `4.2 seconds`.
To compare, quick runs of the fastest SIMD C and non-explicit-SIMD C[^5] are `1.7 seconds` and `2.8 seconds` respectively.

### My Machine's Unfairness
An interesting side note, but my C runs are faster than the website (where it was closer to 2.1 and 5 seconds), 
while my SBCL run is slower (7 seconds even without SIMD versus my 14 seconds). 
This means that while we undoubtedly made it much faster, the ratio has not improved as much. 
Indeed on the website, the ratios for SIMD C, non-explicit-SIMD C, and SBCL were 1, 2.4, and 3.7.
On my machine they were actually 1, 1.7, 8.5, with the exact same compiler flags, and even nearly the same C compiler. 
Unless there was an SBCL regression between theirs and my version, hard to explain how such a ratio could occur.

In the end we were able to get the C-SBCL ratio down to 2.5. 
This result is still pretty good given that this is a dynamic Lisp language and not the portable assembler (even if for PDP-11).
But quite a bit dissapointing that my solution for SBCL was still that much slower.

## Not Yet
I would say that, except that we have yet to actually change the code all that much. 
Yes, I replaced a lot of operations with SIMD and even changed the struct to a pure array.
However, these might as well be cosmetic, the underlying operations and loops have not changed at all.

One core issue, though still closer to a cosmetic one, is that we use linked lists for storing bodies.
The original SBCL solution has a change comment where apparently vectors were done once but got removed for cleaner code purposes.
Whether this implies that the performance difference was not significant I am not sure, but we can test it. 
Though with a linked list this small, it will not be a huge gain, even inside a loop.

The reason for not doing this from the start is that unlike my SIMD which involves combining operations, 
this would involve rewriting a lot of parts that rely on `dolist` and other list-unique features.
You can see this easily in the functions that I rewrote.

Regardless, after changing the system to an array, what is the result? 8 seconds. 
Ah sorry, forgot to add the types. The result is `4.2 seconds`. 
Yeah, it might have had a slight impact since I do average and round these, but we are just not breaking through that number.

I will not give up yet, let's see if there are any obvious optimizations in the `applyforces` and `advance` then.
Well, turns out there are.
```lisp
(declaim (inline applyforces))
(defun applyforces (a b dt)
  (declare (type body a b) (type double-float dt))
  (let* ((dp (f64.4- (vec-p a) (vec-p b)))
         (distance (sqrt (f64.4-horizontal+ (f64.4* dp dp))))
         (mag (/ dt (* distance distance distance))))
    (declare (type double-float distance mag))
    (let ((dpmag (f64.4* dp mag)))
      (setf (vec-v a) (f64.4- (vec-v a) (f64.4* dpmag (mass b))))
      (setf (vec-v b) (f64.4+ (vec-v b) (f64.4* dpmag (mass a))))))
  nil)
```
In `applyforces`, reusing dp * mag calculation saves us a 0.1. 
That is, we now have `4.1 seconds`. 
That's great, but we definitely need more time savings.
    
### Don't Optimize?
Thankfully, I found one more optimization that got my time down to impressive `3.7 seconds`.
That's this line:
```lisp
(declaim (optimize (speed 3)(safety 0)(space 0)(debug 0)))
```
Now it already exists in the original code. 
So what we do is actually just remove it. 
This is quite weird.

Note that this is the standard optimization command, in some ways it is similar to adding `-O3` as your compiler flag in GCC.
I don't have much idea on how adding optimizations can hurt the time. I can only assume it might have to do with `sb-simd` or my REPL setup.

On further testing, it seems that restarting lisp to a fresh start without optimizations gives me 8+ seconds, 
and I can then add these optimize flags to get it down to 3.7. 
As I keep running it goes up to 4.1. Then as I remove it goes back down to 3.7 and stays there on average. 
This is only the case when evaling and not when I compile-and-load (that is produce a .fasl and then load it).
Quite weird indeed. 

I have yet to compile this SIMD version through their method, and I will do so, 
but I don't believe it will much different from a compile-and-load really.

Anyway, the time is thus `3.7 seconds`, so the ratio is actually 2.2, a lot nicer.
I will use compile-and-load to report these now. 
There is just one more optimization I want to try.

### Less Vectors
I have converted the bodies from structs, though still vectors underneath, to pure vectors. 
I have converted the system from a list of vectors to a vector of vectors, to no performance benefit either.
There is however one more trick, combining them, i.e. a single vector containing all the body values. 

This change should improve caching and avoid an extra dereference from having a vector of vectors.
Well, at least this is my hope. We have to actually see whether this has any impact. 
The extra dereference happens in the `advance` which is not called as often as `applyforces`, though often enough to matter.
For this I adapted a macro to simplyfy it. 

Thus, we went from this:
```lisp
(defparameter *jupiter*
  (make-body 4.84143144246472090d0
             -1.16032004402742839d0
             -1.03622044471123109d-1
             (* 1.66007664274403694d-3 +days-per-year+)
             (* 7.69901118419740425d-3 +days-per-year+)
             (* -6.90460016972063023d-5  +days-per-year+)
             (* 9.54791938424326609d-4 +solar-mass+)))
;; etc.
;; (system (list *sun* *jupiter* etc.))
```

To this:

```lisp
(defparameter *system*
  (make-system
   (sun 0.0d0 0.0d0 0.0d0 0.0d0 0.0d0 0.0d0 +solar-mass+) ;; Sun is first in order
   (jupiter 4.84143144246472090d0
            -1.16032004402742839d0
            -1.03622044471123109d-1
            (* 1.66007664274403694d-3 +days-per-year+)
            (* 7.69901118419740425d-3 +days-per-year+)
            (* -6.90460016972063023d-5 +days-per-year+)
            (* 9.54791938424326609d-4 +solar-mass+))
;; etc.
```
Where names are purely cosmetic. 
After that it was changing all cases of `(aref body j)` to `(aref system i j)` with some code changes needed.
Support for multidimensional arrays is quite nice in SBCL when you need it.

Unfortunately, as noble as this idea was, a failure it ended up. 
Implementing this seems to have caused type issues or some other optimization hurdle as it ended up being slower.
I have tried to fix this, but it does not seem to be that simple. 
Maybe this is just theory vs practice. 
I believe there should be a way to fix this, but even with this I don't see us getting that much faster.

I tried everything I could think of for this, and ran out of ideas for further optimization tricks. 
But, there is yet another path to take.

## "Inspiration"
So far I have simply been adapting the original SBCL solution to SIMD and more vectors.
This has provided us with large performance increases to the point where SBCL was only away from the best C solution by a factor of 2.2.
Very impressive, especially since the most important optimizations were just replacing a few instructions with the vector ones.
Of course I did look at the best C solution for an inspiration of how best to use SIMD. 
However, it was just that, an inspiration, only for adding a waste slot really. 
This solution is not a Lisp copy of the C one. 
But, what if we actually did that, learn from the enemy?

Firstly, what does the C solution do differently, after all if it's already identical just in C, it would not help much.
Well, it actually does a couple of things. 
To begin, it does some manual loop unrolling to optimize some SIMD operations better.
They actually use a `rsqrt`, and thus arrange some things differently.
Typically, this would be an issue since there is no SIMD `rsqrt` for double floats. 
However, they implemented one themselves by downcasting and then upcasting with fixes to keep accuracy.
The C solution also makes sure to use stack memory at all time and reuse as much as possible, e.g. with pairs.
It really is no wonder this is the fastest solution.

Loop unrolling was an obvious one that I missed. 
`rsqrt` is something I thought of, but failed to get good results.
Though, the way they were able to do it feels a little bit like cheating[^6].
The stack allocations are nice, though slightly more annoying to repeat.

### Rolling
This is the big one, so might as well start with it.
Loop unrolling is typically done for two cases.
Either you want more instructions between your branches to improve throughput, 
or you have a way to combine instructions together, e.g. auto-vectorization.
In our case, this provides SIMD over parts that could not be SIMD before, like `sqrt` (we are not using `rsqrt` yet).

[^1]: Some of those links seem to be dead at the moment, but you can download the zipped archive. 
This is not the "true" spec, but closest you can get without a purchase or Big AI strategies.
I swear I had a working online one, might have been a different link.
[^2]: When anyone tells you that a compiler is smarter than you, kindly remind them that being wrong is not ok.
Just because a compiler can turn a seemingly complex loop into SIMD, or replace a division with reciprocal multiplication 
does not mean it is omniscient and omnipotent in nearly all cases. 
[See my article for when even a cool optimization is a failure](http://localhost:1313/posts/counting-bits-speedily).
[^3]: Funny enough, my GCC version is identical 14.2.0, though possibly some different patches since this is WSL Debian whereas they use Ubuntu.
[^4]: The function itself actually compiles to identical assembly, so this difference in performance likely has to do with changes that inlining brings.
[^5]: In case you are wondering what a technical no-SIMD C would be like, it would take around 3.6 seconds by removing auto-vectorization and disabling AVX2. 
Note that floating point operations are done using SIMD.
One could disable SIMD completely, but generally any compiled language will use SIMD for floating point, often for SISD. 
This includes SBCL, which generates SSE instructions for all floating point operations, even though it does not auto-vectorize.
[^6]: There is no law that SIMD `rsqrt` for double-floats is forbidden, but using single-floats like this feels a little wrong, even if accuracy is fixed in post.
