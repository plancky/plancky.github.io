---
title: "Py_INCREF Got Three Fields: Inside CPython's Free-Threaded Build"
date: 2026-10-13
description: "Reading through the CPython source to see how a rewritten reference count let Python run without its Global Interpreter Lock, and why the GIL's days are numbered."
draft: false
hideToc: true
enableToc: false
categories:
    - engineering
---

Open up an object in a free-threaded CPython build and its reference count is gone. In its place are three fields: the id of the thread that owns the object, a local count, and a shared count. `Py_INCREF`, which runs more often than almost anything else in the interpreter, now checks which thread is calling before it decides how to count. That one change is the reason Python can finally run without its Global Interpreter Lock.

**And the GIL is going to die.** CPython already ships **two interpreters built from one codebase**: the familiar build with the GIL, and a *free-threaded* build without it. The free-threaded build is optional for now, but the Steering Council's stated long-term goal is to make it the default and to "remove any vestiges of the GIL." Every release since 3.13 has moved further that way: experimental, then officially supported, then given its own stable ABI. Nobody has set a date yet, but the direction is settled. Getting here took almost 30 years of failed attempts, one very determined engineer, and a set of clever tricks around reference counting.

This post covers why the core team wanted the GIL gone, why earlier attempts failed, and where the new code lives in the CPython source, starting with those three fields. All code links point to a fixed commit on `main` ([`c04ad8c`](https://github.com/python/cpython/tree/c04ad8ce36e35fba85468eec8a54f1688fa16972), October 2026, which is 3.16.0 alpha).

## What the GIL actually is

The GIL is one mutex per interpreter. A thread must hold it to execute Python bytecode. The implementation is in [`Python/ceval_gil.c`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/ceval_gil.c):

- [`take_gil()`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/ceval_gil.c#L286) and [`drop_gil()`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/ceval_gil.c#L217) acquire and release it.
- A thread waiting for the GIL waits on a condition variable for `interval` microseconds (default 5 ms, settable with `sys.setswitchinterval()`). If it times out, it sets `gil_drop_request`.
- The running thread checks for that request inside the eval loop, in [`_Py_HandlePending()`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/ceval_gil.c#L1358), and gives up the lock.

The GIL is what made CPython's memory management simple. Every object has a reference count, and the GIL means `ob_refcnt++` never needs to be atomic. It's also what keeps a `dict` from being corrupted by two threads writing to it at once. The cost is that a 64-core machine runs your threaded Python code on effectively one core.

## Why remove it?

The case is set out in [PEP 703 – Making the Global Interpreter Lock Optional in CPython](https://peps.python.org/pep-0703/), by Sam Gross of Meta. The motivation section is mostly real user reports:

- **Reinforcement learning:** a DeepMind engineer reported GIL bottlenecks with fewer than 10 threads, which pushed teams to rewrite Python code in C++.
- **Bioinformatics:** a sequence-alignment tool stopped scaling at 8 threads, and communicating between subprocesses cost too much.
- **Medical hardware:** a radiotherapy data-acquisition project spent most of its CPU time moving data between processes. Moving to the nogil fork took less than half a day and gave large speedups.
- **GPU orchestration:** even when the heavy maths runs on a GPU, the Python code driving it needs CPU parallelism. PyTorch sometimes ran 72 processes where one would do.

**Why not just use `multiprocessing`?** The PEP lists the costs:

| | Threads | Processes |
|---|---|---|
| Startup | ~100 µs | ~50 ms |
| Sharing data | shared memory | serialize + copy |
| CUDA context | shared | one per process |

Workarounds like PyTorch's fork-based `DataLoader` and scikit-learn's `joblib`/`loky` exist mainly to work around the GIL, and the PEP describes them as complex and brittle.

## Why it took 30 years

Removing the GIL was never the hard part. The hard part was removing it **without making single-threaded Python slower**. As [LWN's coverage of Thomas Wouters' PyCon US 2026 talk](https://lwn.net/SubscriberLink/1078367/5800cc051d01e516/) recounts:

| Year | Attempt | Fate |
|---|---|---|
| 1996 | Greg Stein's free-threading patch (Python 1.4) | Fine-grained locks everywhere; single-threaded code got too slow |
| 2013 | PyParallel (Trent Nelson) | Didn't support ordinary OS threads |
| 2015 | Gilectomy (Larry Hastings) | Abandoned because of reference-counting scalability problems |
| 2021 | [nogil](https://github.com/colesbury/nogil) (Sam Gross, fork of 3.9) | Worked, and became PEP 703 |

The core problem: if every `Py_INCREF` and `Py_DECREF` becomes an atomic operation, *every* Python program pays for it, even one that never starts a thread.

## How it works, in the code

### 1. Biased reference counting

Most objects are only ever touched by the thread that created them. PEP 703 takes advantage of that with *biased reference counting* (Choi, Shull & Torrellas, PACT 2018). In the free-threaded build, the object header gets three fields instead of one `ob_refcnt` ([`Include/object.h#L157-L165`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Include/object.h#L157-L165)):

```c
uintptr_t ob_tid;           // owning thread id
uint32_t ob_ref_local;      // local reference count
Py_ssize_t ob_ref_shared;   // shared (atomic) reference count
```

And here's `Py_INCREF` itself ([`Include/refcount.h#L255`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Include/refcount.h#L255)):

```c
#if defined(Py_GIL_DISABLED)
    uint32_t local = _Py_atomic_load_uint32_relaxed(&op->ob_ref_local);
    uint32_t new_local = local + 1;
    if (new_local == 0) {
        // local is equal to _Py_IMMORTAL_REFCNT_LOCAL: do nothing
        return;
    }
    if (_Py_IsOwnedByCurrentThread(op)) {
        _Py_atomic_store_uint32_relaxed(&op->ob_ref_local, new_local);
    }
    else {
        _Py_atomic_add_ssize(&op->ob_ref_shared, (1 << _Py_REF_SHARED_SHIFT));
    }
#elif ...
```

The owning thread does a cheap non-atomic store, and other threads pay for an atomic add. When the shared count of an object owned by another thread goes negative, the object is queued so its owner can merge the two counts. That code is in [`Python/brc.c`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/brc.c), in [`_Py_brc_queue_object()`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/brc.c#L76) and [`merge_queued_objects()`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/brc.c#L141).

### 2. Immortal objects

`None`, `True`, small integers, interned strings and static types are used by every thread all the time. If every thread updated their refcounts, those objects' cache lines would be constantly contended. Instead they're made *immortal*: their refcount is set to a sentinel value and never changes. You can see the `new_local == 0` early return above. Immortality first arrived in the default build too, through [PEP 683](https://peps.python.org/pep-0683/) in 3.12. The constants are in [`Include/refcount.h#L47`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Include/refcount.h#L47).

### 3. Deferred reference counting

Functions, code objects, modules and type objects are shared widely but are not immortal. For these objects, references held on the interpreter's stack skip refcount updates altogether, and the garbage collector frees such objects only when it runs. See the free-threaded GC, [`Python/gc_free_threading.c`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/gc_free_threading.c), and [`InternalDocs/stackrefs.md`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/InternalDocs/stackrefs.md).

### 4. Per-object locks via "critical sections"

Without the GIL, `list.append` racing with `list.pop` would corrupt memory. Each `list`, `dict` and `set` gets a small `PyMutex`, defined in [`Python/lock.c`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/lock.c). They're locked through a *critical section* API ([`Include/critical_section.h#L74`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Include/critical_section.h#L74)):

```c
# define Py_BEGIN_CRITICAL_SECTION(op)                                  \
    {                                                                   \
        PyCriticalSection _py_cs;                                       \
        PyCriticalSection_Begin(&_py_cs, _PyObject_CAST(op))
```

The key design point is that when a thread blocks or enters a nested critical section, its outer locks are *suspended* and later resumed by [`_PyCriticalSection_Resume()`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/critical_section.c#L133). This prevents deadlocks the way the GIL used to: you can never hold one lock while waiting for another. In the GIL build, these macros [compile to no-ops](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Include/cpython/critical_section.h).

### 5. Lock-free reads + QSBR

Taking a lock for every `d[key]` would be too slow. So some `list` and `dict` reads are *optimistic*: they read without a lock and then check that nothing changed underneath them. That's only safe if memory a reader might be looking at isn't freed while it's looking. **Quiescent-state-based reclamation** handles this: freed memory is reused only after every thread has passed a "quiescent" point. See [`Python/qsbr.c`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/qsbr.c) ([`_Py_qsbr_advance`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/qsbr.c#L115), [`_Py_qsbr_poll`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/qsbr.c#L161)) and [`InternalDocs/qsbr.md`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/InternalDocs/qsbr.md).

### 6. mimalloc

`pymalloc` assumed the GIL. The free-threaded build uses Microsoft's thread-safe [mimalloc](https://github.com/microsoft/mimalloc), vendored in [`Objects/mimalloc/`](https://github.com/python/cpython/tree/c04ad8ce36e35fba85468eec8a54f1688fa16972/Objects/mimalloc). Its per-type heaps also let the GC find every object by walking the heaps.

### 7. A stop-the-world cycle collector

The default collector, [`Python/gc.c`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/gc.c), relied on the GIL to keep the heap from changing while it ran. The free-threaded collector, [`Python/gc_free_threading.c`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/gc_free_threading.c), instead pauses every thread using [`_PyEval_StopTheWorld()`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/pystate.c#L2618) and is non-generational, so these pauses happen less often. More detail is in [`InternalDocs/garbage_collector.md`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/InternalDocs/garbage_collector.md).

### 8. One codebase, two builds

All of this sits behind one preprocessor flag, `Py_GIL_DISABLED`, which is set by a configure option that is **off by default** ([`configure.ac#L1789`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/configure.ac#L1789)):

```
AS_HELP_STRING([--disable-gil], [enable support for running without the GIL (default is no)])
```

### 9. The safety net: the GIL comes back

C extensions written before free-threading may depend on the GIL. So they have to opt in with the [`Py_mod_gil`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Include/moduleobject.h#L85) slot. If a module doesn't, the interpreter turns the GIL **back on** at import time ([`Python/import.c#L1667`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/import.c#L1667)):

```c
"The global interpreter lock (GIL) has been enabled to load "
"module '%U', which has not declared that it can run safely "
"without the GIL. To override this behavior and keep the GIL "
"disabled (at your own risk), run with PYTHON_GIL=0 or -Xgil=0."
```

You can check at runtime with [`sys._is_gil_enabled()`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Python/sysmodule.c#L2633).

## What the core team officially committed to

On **28 July 2023**, Thomas Wouters, writing for the Steering Council, posted [*A Steering Council notice about PEP 703*](https://discuss.python.org/t/a-steering-council-notice-about-pep-703-making-the-global-interpreter-lock-optional-in-cpython/30474) on discuss.python.org. It set out three phases:

> **Short term:** "we add the no-GIL build as an experimental build mode, presumably in 3.13"
>
> **Mid-term:** supported but not the default, which will "take at least a year or two, possibly more"
>
> **Long term:** "we want no-GIL to be the default, and to remove any vestiges of the GIL", which could take "as much as five years"

The notice also included two important conditions:

> "We do not want another Python 3 situation."

> "We want to be able to change our mind if it turns out, any time before we make no-GIL the default" — which "could mean rolling back all of the work."

### The timeline so far

| Release | Phase | Official reference | Single-thread overhead |
|---|---|---|---|
| **3.13** (Oct 2024) | I – experimental | [What's New in 3.13](https://docs.python.org/3.13/whatsnew/3.13.html#free-threaded-cpython) | ~20–40% |
| **3.14** (Oct 2025) | II – supported, optional | [PEP 779](https://peps.python.org/pep-0779/) (accepted 16 June 2025) | ~0–10%, typically ~5% |
| **3.15** (Oct 2026) | II, ABI stabilized | [PEP 803 – "abi3t"](https://peps.python.org/pep-0803/) (accepted 30 March 2026) | — |
| **3.16+** | III – default? | not decided; needs a new PEP | — |

PEP 779 set the bar for "supported": no more than **15% slower** and no more than **20% more memory** than the GIL build on pyperformance. At the time of writing, the slowdown was about 10% and the memory overhead about 15–20%.

PEP 803 is the change most relevant to package maintainers. It adds a Stable ABI for free-threaded builds, so an extension can be compiled once and tagged `abi3.abi3t` instead of being rebuilt for every `cp3XXt` release. The 3.15 What's New ([`Doc/whatsnew/3.15.rst`](https://github.com/python/cpython/blob/c04ad8ce36e35fba85468eec8a54f1688fa16972/Doc/whatsnew/3.15.rst)) also notes that the official macOS installers now install free-threading support by default.

### When will it become the default?

Nobody has officially committed to a date. In the PyCon US 2026 talk, Wouters stressed that this was a *personal* prediction, not a Steering Council position. The prediction was that free-threading becomes the default sometime **between 3.16 and 3.20**, and that the GIL build is removed entirely in the decade after that. Wouters also expects Linux distributions and large companies to switch before CPython does ([LWN](https://lwn.net/SubscriberLink/1078367/5800cc051d01e516/)).

The ecosystem is the main thing to watch. More than half of the top 360 PyPI packages that ship compiled wheels now support free-threading. The community-maintained [Python Free-Threading Guide](https://py-free-threading.github.io/), largely written by Quansight, tracks progress and explains how to port extensions.

## Try it yourself

```bash
git clone --depth 1 https://github.com/python/cpython.git
cd cpython
./configure --disable-gil && make -j
./python -c "import sys; print(sys._is_gil_enabled())"   # False
```

Or install a `python3.14t` / `python3.15t` build from python.org or `uv python install 3.15t`.

## Sources

- [PEP 703 – Making the GIL Optional in CPython](https://peps.python.org/pep-0703/)
- [PEP 779 – Criteria for supported status for free-threaded Python](https://peps.python.org/pep-0779/)
- [PEP 803 – "abi3t": Stable ABI for Free-Threaded Builds](https://peps.python.org/pep-0803/)
- [PEP 683 – Immortal Objects](https://peps.python.org/pep-0683/)
- [Steering Council notice on PEP 703 (July 2023)](https://discuss.python.org/t/a-steering-council-notice-about-pep-703-making-the-global-interpreter-lock-optional-in-cpython/30474)
- [Python HOWTO: Python support for free threading](https://docs.python.org/3/howto/free-threading-python.html)
- [Python HOWTO: C API extension support for free threading](https://docs.python.org/3/howto/free-threading-extensions.html)
- [`InternalDocs/`](https://github.com/python/cpython/tree/c04ad8ce36e35fba85468eec8a54f1688fa16972/InternalDocs): `garbage_collector.md`, `qsbr.md`, `stackrefs.md`, `interpreter.md`
- [Python Free-Threading Guide](https://py-free-threading.github.io/)
- [LWN: Free-threaded Python: past, present, and future](https://lwn.net/SubscriberLink/1078367/5800cc051d01e516/)
- [Sam Gross's original nogil fork](https://github.com/colesbury/nogil)
