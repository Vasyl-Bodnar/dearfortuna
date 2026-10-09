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

Regardless of the quote, to put a proper test to it, I would want to take a language that has a pretty "slow" non-SIMD solution and make it pretty fast with SIMD.
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
To compare, quick runs of the fastest SIMD C and non-explicit-SIMD C are `1.7 seconds` and `2.8 seconds` respectively.

### My Machine's Unfairness
An interesting side note, but my C runs are faster than the website (where it was closer to 2.1 and 5 seconds), 
while my SBCL run is slower (7 seconds even without SIMD versus my 14 seconds). 
This means that while we undoubtedly made it much faster, the ratio has not improved as much. 
Indeed on the website, the ratios for SIMD C, non-explicit-SIMD C, and SBCL were 1, 2.4, and 3.7.
On my machine they were actually 1, 1.7, 8.5, with the exact same compiler flags, and even nearly the same C compiler. 
Unless there was an SBCL regression between theirs and my version, hard to explain how such a ratio could occur.

In the end we were able to get the C-SBCL ratio down to 2.5. 
This result is still pretty good given that this is a dynamic Lisp language and not the portable assembler (for PDP-11).
But quite a bit dissapointing that my solution for SBCL was still that much slower.

[^1]: Some of those links seem to be dead at the momemnt, but you can download the zipped archive. 
This is not the "true" spec, but closest you can get without a purchase or Big AI strategies.
[^2]: When anyone tells you that a compiler is smarter than you, kindly remind them of SIMD and multi-threading.
Just because a compiler can turn a seemingly complex loop into SIMD, replace a division with reciprocal multiplication or 
[a pop-count trick with an actual pop-count instruction](http://localhost:1313/posts/counting-bits-speedily) does not mean it is omniscient and omnipotent in nearly all cases (just like that cool pop-count replacement was a mistake in the article).
[^3]: Funny enough, my GCC version is identical 14.2.0, though possibly some different patches since this is WSL Debian whereas they use Ubuntu.
[^4]: The function itself actually compiles to identical assembly, so this difference in performance likely has to do with changes that inlining brings.
