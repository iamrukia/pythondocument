原文：[Python, the GIL, and Pyston](http://blog.kevmod.com/2014/06/python-the-
gil-and-pyston/ "Python, the GIL, and Pyston" )

---

Lately, I've been thinking a bit about supporting parallelism in Pyston --
this has been on my "wish list" for a long time.  The state of parallelism in
CPython is a bit of a sore subject, since the GIL ("global interpreter lock")
essentially enforces single-threaded execution.  It should be noted that a GIL
is not specific to CPython: other implementations such as PyPy have one
(though PyPy have their STM efforts to get rid of theirs), and runtimes for
other languages also have them.  Technically, a GIL is a feature of an
implementation, not of a language, so it seems like implementations should be
free to use non-GIL-based strategies.

The tricky part with "using non-GIL-based strategies" is that we still have to
provide the correct semantics for the language.  And, as I'll go into in more
detail, there are a number of GIL-derived semantics that have become part of
the Python language, and must be respected by compatible implementations
whether or not they actually use a GIL.  Here are a couple of the issues that
I've been thinking about:

### Issue #1: data structure thread-safety

Imagine you have two Python threads, which both try to append an item onto a
list.  Let's say the list starts empty, and the threads try to append "1" and
"2", respectively:

```python

    l = []

    def thread1():

        l.append(1)

    def thread2():

        l.append(2)

    
```

What are the allowable contents of the list afterwards?  Clearly "[1, 2]" and
"[2, 1]" are allowed.  Is "[1]" allowed?  Is "[1, 1]" allowed?  And what about
"[1, &lt;garbarge&gt;]"? I think the verdict would be that none of those,
other than "[1, 2]" and "[2, 1]" would be allowed, and in particular not the
last one.  Data structures in Python are currently guaranteed to be thread-
safe, and most basic operations such as "append" are currently guaranteed to
be atomic.  Even if we could somehow convince everyone that the builtin list
should not be a thread-safe data structure, it's certainly not ok to
completely throw all synchronization out the window: we may end up with an
inconsistent data structure with garbage in the list, breaking the memory
safety of the language. So no matter what, there needs to be some amount of
thread-safety for all the builtin types.

People have been building thread-safe datastructures for as long as there have
been threads, so addressing this point doesn't require any radical new ideas.
The issue, though, is that since this could apply to potentially all
operations that a Python program takes, there may be a very large amount of
locking/synchronization overhead.  A GIL, while somewhat distasteful,
certainly does a good job of providing thread safety while keeping lock
overheads low.

### Issue #2: memory model

This is something that most Python programmers don't think about because we
don't have to, but the "memory model" specifies the potential ways one thread
is allowed to observe the effects of another thread.  Let's say we have one
thread that runs:

```python

    a = b = 0

    def thread1():

        global a, b

        a = 1

        b = 2

    
```

And then we have a second thread:

```python

    def thread2()

        print b

        print a

    
```

What is thread2 allowed to print out?  Since there is no synchronization, it
could clearly print "0, 0", "0, 1", or "2, 1".  In many programming languages,
though, it would be acceptable for thread2 to print "2, 0",  in what seems
like a contradiction: how can b get set if a hasn't been?  The answer is that
the memory model typically says that the threads are not guaranteed to see
each others' modifications in any order, unless there is some sort of
synchronization going on.  (In this particular case, I think the x86 memory
model says that this won't happen, but that's another story.)  Getting back to
CPython, the GIL provides that "some sort of synchronization" that we needed
(the GIL-release-then-GIL-acquire will force all updates to be seen), so we
are guaranteed to not see any reordering funny-business: CPython has a strong
memory model called "sequential consistency".  While this is technically could
be considered just a feature of CPython, there seems to be consensus that this
is actually part of the language specification.  While there can and should be
a debate about whether or not this should be the specified memory model, I
think the fact of the matter is that there has to be code out there that
relies on a sequential consistency model, and Pyston will have to provide
that.

There's some precedent for changing language guarantees -- we had to wean
ourselves off immediate-deallocation when GC'd implementations started coming
around.  I feel like the memory model, though, is more entrenched and harder
to change, and that's not to say we even should.

### Issue #3: C extensions

One of the goals of Pyston is to support unmodified CPython C extensions;
unfortunately, this poses a pretty big parallelism problem.  For Python code,
we are only given the guarantee that each individual bytecode is atomic, and
that the GIL could be released between any two bytecodes.  For C extension
code, a far bigger promise is made: that the GIL will not be released unless
explicitly requested by the C code.  This means that C extensions are free to
be as thread-unsafe as they want, since they will never run in parallel unless
requested.  So while I'd guess that not many extensions explicitly make use of
the fact that the GIL exists, I would highly doubt that all the C extension
code, written without thread-safety in mind, would miraculously end up being
thread safe. So no matter how Python-level code is handled, we'll have to (by
default) run C extension code sequentially.



### Potential implementation strategy: GRWL

So there's certainly quite a few constraints that have to be met by any
threading implementation, which would easily and naturally be met by using a
GIL.  As I've mentioned, it's not like any of these problems are particularly
novel; there are well-established (though maybe tricky-to-implement) ways of
solving them.  The problem, though, is the fact that since we have to do this
at the language runtime level, we will incur these synchronization costs for
all code, and it's not clear if that will end up giving a better performance
tradeoff than using a GIL.  You can potentially get better parallelism,
limited though by the memory model and the fact that C extensions have to be
sequential, but you will most likely have to sacrifice some amount of single-
threaded performance.

I'm currently thinking about implementing these features using a Global Read-
Write Lock, or GRWL.  The idea is that we typically allow threads to run in
parallel, except for certain situations (C extension code, GC collector code)
where we force sequential execution.  This is naturally expressed as a read-
write lock: normal Python code holds a read lock on the GRWL, and sequential
code has to obtain a write lock.  (There is also code that is allowed to not
hold the lock at all, such as when doing IO.)  This seems like a pretty
straightforward mapping from language semantics to synchronization primitives,
so I feel like it's a good API.

I have a prototype implementation in Pyston; it's nice because the GRWL API is
a superset of the GIL API, which means that the codebase can be switched
between them by simply changing some compile-time flags.  So far the results
aren't that impressive: the GRWL has worse single-threaded performance than
the GIL implementation, and worse parallelism -- two threads run at 45% of the
total throughput of one thread, whereas the GIL implementation manages 75% [ok
clearly there's some improvement for both implementations].  But it works!
(As long as you only use lists, since I haven't added locking to the other
types.)  It just goes to show that simply removing the GIL isn't hard --
what's hard is making the replacement faster.  I'm going to spend a little bit
of time profiling why the performance is worse than I think it should be,
since right now it seems a bit ridiculous.  Hopefully I'll have something more
encouraging to report soon, but then again I wouldn't be surprised if the
conclusion is that a GIL provides an unbeatable effort-reward tradeoff.



### Update: benchmarks

So I spent some time tweaking some things; the first change was that I
replaced the choice of mutex implementation.  The default glibc pthread mutex
is PTHREAD_MUTEX_TIMED_NP, which apparently has to sacrifice throughput in
order to provide the features of the POSIX spec.  When I did some profiling, I
noticed that we were spending all our time in the kernel doing futex
operations, so I switched to PTHREAD_MUTEX_ADAPTIVE_NP which does some
spinning in user space before deferring to the kernel for arbitration.  The
performance boost was pretty good (about 50% faster), though I guess we lose
some scheduling fairness.

The second thing I changed was reducing the frequency with which we check
whether or not we should release the GRWL.  I'm not quite sure why this
helped, since there rarely are any waiters, and it should be very quick to
check if there are other waiters (doesn't need an atomic operation).  But that
made it another 100% faster.



Here are some results that I whipped up quickly.  There are three versions of
Pyston being tested here: an "unsafe" version which has no GIL or GRWL as a
baseline, a GIL version, and a GRWL version.  I ran it on a couple different
microbenchmarks:

```python

                                     unsafe  GIL    GRWL

    raytrace.py [single threaded]    12.3s   12.3s  12.8s

    contention_test.py, 1 thread     N/A     3.4s   4.0s

    contention_test.py, 2 threads    N/A     3.4s   4.3s

    uncontended_test.py, 1 thread    N/A     3.0s   3.1s

    uncontended_test.py, 2 threads   N/A     3.0s   3.6s

    
```

So... things are getting better, but even on the uncontended test, which is
where the GRWL should come out ahead, it still scales worse than the GIL.  I
think it's GC related; time to brush up multithreaded performance debugging.
