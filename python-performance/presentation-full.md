# A Practical Guide to Fast, Concurrent Python

---

## 1. Performance Problems: CPU-bound vs I/O-bound

- CPU-bound: image resizing, number crunching, parsing/serialization
- I/O-bound: API calls, DB queries, file/network reads

---

## 2. Python's Object Model: Everything Lives on the Heap

- Every Python object — even a small int or a bool — is heap-allocated, never stack-allocated.
- Variables are just names bound to references (pointers) to these heap objects, which is why Python's types are all reference types rather than value types.
- Each object carries a **reference count** that CPython increments/decrements as references are created and dropped; when it hits zero, the object is freed immediately.
- This refcounting is CPython's primary GC mechanism, backed by a secondary cyclic garbage collector that periodically sweeps for reference cycles (e.g. objects referencing each other) that refcounting alone can't clean up.
- **Limitation:** because *everything* is heap-allocated, Python can't do the allocation-free, stack-only style of code that's routine in languages with real value types (C, C++, Rust, C# `struct`) — even a tiny local variable costs a heap allocation plus later refcount/GC bookkeeping, which is part of why tight numeric loops are so much slower in Python.

![Python reference counting mechanics](python_refcount_mechanics.svg)

![Reference counting race condition causing a double free](refcount_race_double_free.svg)

---

## 3. GIL (Global interpreter lock)

- A thread must acquire the GIL before running any bytecode; if another thread holds it, the requesting thread waits.
- The GIL is released in two scenarios:
  - when a thread executes a blocking I/O operation (network requests, file reads), letting other threads proceed
  - after a fixed switching interval (default 5ms), which forces a context switch to the next waiting thread

*Further reading: [What is the GIL in Python and why should you care?](https://dev.to/imsushant12/what-is-the-gil-in-python-and-why-should-you-care-1cai)*

![Global Interpreter Lock sequence diagram](gil.svg)

**However, multiprocessing sidesteps the GIL entirely:** each process is a separate Python interpreter with its own GIL, so CPU-bound tasks can run in true parallel across multiple cores:

![Each process has its own GIL, enabling true parallelism](processes-gil.svg)

---

## 4. How Other Languages Solve It

Python's refcounting-under-a-GIL is one point in a wider design space. Most other managed languages instead use a **tracing garbage collector**, which sidesteps the refcount-race problem entirely by never mutating a per-object counter on every reference change — instead it periodically walks the object graph from scratch.

**C# / .NET example** — the mark phase works in three steps:

1. **Find the roots** — local variables currently on any thread's stack, static fields, CPU registers, anything an executing method could reach right now.
2. **Trace outward** — starting from each root, walk every reference: "this variable points to this `Customer`, which has a field pointing to this `Address`," and so on. Everything reachable this way gets marked "alive."
3. **Everything unmarked is garbage** — by definition, if nothing traces to it, nothing in the running program can ever use it again.

Because collection happens in occasional passes over the whole graph rather than on every single reference assignment, there's no per-object counter to race on — multiple threads can freely read/write references without corrupting GC state the way an un-synchronized refcount would.

- **The payoff:** because there's no per-object refcount to race on, tracing-GC languages don't need a GIL — threads can run on multiple cores simultaneously without corrupting garbage-collector state.

![Tracing garbage collector mark phase](tracing-gc-mark-phase.svg)

---

## 5. Optimizing CPU-Bound Work (Without Parallelization)

Before reaching for threads or processes, it's often cheaper — and simpler — to just make the single-threaded code faster:

- **Swap list membership checks for set/dict lookups** — `x in my_list` is O(n): every check can scan the whole list. `x in my_set` (or `x in my_dict`) is O(1) via hashing.

```python
needle = "z"  # the value we're checking for membership

# O(n) per check — scans the whole list every time
allowed = ["a", "b", "c", "..."]
if needle in allowed:
    ...

# O(1) per check — same logic, hash lookup instead of a scan
allowed = {"a", "b", "c", "..."}
if needle in allowed:
    ...
```

- **`io.StringIO` for building strings in a loop** — strings are immutable, so repeated `+=` allocates a brand-new string and copies the old contents in every time (quadratic). `io.StringIO` gives you a mutable buffer to write into instead.

```python
import io

# Quadratic — each += allocates a new string and copies everything so far into it
result = ""
for chunk in chunks:
    result += chunk

# Linear — StringIO writes into a buffer; one final allocation via getvalue()
buf = io.StringIO()
for chunk in chunks:
    buf.write(chunk)
result = buf.getvalue()
```

- **Reach for libraries that drop into C under the hood** — NumPy (and similarly Pandas, Cython-based libraries) do the actual number crunching in compiled C over contiguous memory, sidestepping the per-element bytecode dispatch and refcounting overhead of a Python loop entirely.

```python
import numpy as np

# Pure Python — one bytecode dispatch, refcount bump, etc. per element
squares = [x * x for x in range(1_000_000)]

# NumPy — the loop runs once, in C, over a contiguous buffer
squares = np.arange(1_000_000) ** 2
```

---

## 6. Threads vs Processes

**Threads** are lightweight units of execution within a single process that share the same memory space. They are ideal for I/O-bound tasks (like web scraping, file reading, or API calls) because they allow the program to perform other work while waiting for external operations to complete. However, due to the GIL, threads in standard CPython cannot execute Python bytecode in true parallel; they only achieve concurrency by switching contexts during I/O waits. They're especially useful when the external service's API has no "batch" endpoint — if it only accepts one resource per call, threads are how you get concurrency without a slow one-request-at-a-time loop.

```python
from concurrent.futures import ThreadPoolExecutor
import requests

def fetch_url(url):
    response = requests.get(url)  # blocks waiting on the network — GIL is released here
    return response.status_code

urls = get_urls_to_fetch()  # e.g. this returns 37 URLs

with ThreadPoolExecutor(max_workers=len(urls)) as executor:
    executor.map(fetch_url, urls)
```

**Processes** are independent instances of the Python interpreter, each with its own memory space and its own GIL. They are necessary for CPU-bound tasks (like heavy number crunching or data processing) because they can run in true parallel across multiple CPU cores, bypassing the GIL limitation. While processes offer better isolation and robustness, they have higher overhead, slower startup times, and more complex inter-process communication compared to threads.

![Each process has its own GIL, so CPU-heavy tasks run in true parallel across cores](processes-gil.svg)

```python
from concurrent.futures import ProcessPoolExecutor

def crunch_numbers(n):
    return sum(i * i for i in range(n))  # simulate CPU-bound work

with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(crunch_numbers, [10_000_000] * 10))
```

| Feature | Threads | Processes |
|---|---|---|
| Memory | Shared within the same process | Separate, isolated memory spaces |
| Best For | I/O-bound tasks | CPU-bound tasks |
| Execution | Concurrency (context switching) | Parallelism (true simultaneous execution) |
| GIL Impact | Limited by single GIL per process | Each process has its own GIL |
| Overhead | Low (lightweight) | High (heavier resource usage) |
| Communication | Direct memory access (easier) | Pipes/queues (more complex) |
| Robustness | Lower (crash affects whole process) | Higher (isolation prevents cascade failure) |

### Sizing the Pool: How Many Threads / Processes?

**Threads** — since threads are cheap and I/O-bound work spends most of its time *waiting*, not computing, the pool size doesn't need to be fixed in advance. A common pattern is dynamic sizing: compute how much work there is (e.g. the number of API calls or URLs to fetch) and size the pool to match, up to a sane ceiling so you don't overwhelm the remote service, exhaust file descriptors, or flood a rate limit:

```python
urls = get_urls_to_fetch()  # say this returns 37 URLs

# Match the pool to the workload, capped so we don't hammer the server
MAX_WORKERS_CAP = 20
worker_count = min(len(urls), MAX_WORKERS_CAP)

with ThreadPoolExecutor(max_workers=worker_count) as executor:
    executor.map(fetch_url, urls)
```

**Processes** — bounded first by available CPU cores, since a CPU-bound worker beyond that count just competes for the same cores instead of adding parallelism:

```python
import os
from concurrent.futures import ProcessPoolExecutor

def crunch_numbers(n):
    return sum(i * i for i in range(n))  # simulate CPU-bound work

# Unlike threads, more processes than CPU cores doesn't buy more
# parallelism — they'd just compete for the same cores.
# os.process_cpu_count() (Python 3.13+) respects CPU affinity (taskset,
# sched_setaffinity); os.cpu_count() always reports the host's total cores.
# Neither one sees a cgroup CPU quota — see the Kubernetes note below.
available_cores = getattr(os, "process_cpu_count", os.cpu_count)() or 1
worker_count = max(1, available_cores - 1)  # leave a core for the OS/main process

with ProcessPoolExecutor(max_workers=worker_count) as executor:
    results = list(executor.map(crunch_numbers, [10_000_000] * 10))
```

- **Core count is a ceiling, not a target** — going past `available_cores` just adds scheduling overhead (context switches between more processes than there are cores to run them on) with no extra throughput.
- **Leave headroom** — `available_cores - 1` keeps a core free for the main/parent process, the OS scheduler, and any other work on the machine, so the pool doesn't starve everything else.
- **On Kubernetes, both of those calls can lie** — a pod's `resources.limits.cpu` is enforced via a cgroup CFS quota, not by restricting which cores the process can see, so `os.process_cpu_count()` still reports the *node's* cores unless the cluster also pins CPUs (Guaranteed QoS + static CPU Manager policy). Read the cgroup quota directly (`cpu.max` on cgroup v2, `cpu.cfs_quota_us` / `cpu.cfs_period_us` on v1) to get the pod's real limit.
- **Core count is only half the budget** — each process is a full interpreter with its own memory space, so also account for available RAM: `min(available_cores, available_ram // estimated_ram_per_worker)`.
- **Spawning more processes than RAM supports** causes swapping or OOM kills, which is far worse than the parallelism you gained by adding them.
- **Rule of thumb**: profile one worker's peak RSS first, then size the pool against both constraints — don't just default to `cpu_count()`.

---

## 7. Locks

- The race condition it solves — tiny broken-counter demo
- `acquire()` / `release()`, context manager usage (`with lock:`)
- Deadlock risk (inconsistent lock ordering)

```python
import threading

counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(100_000):
        with lock:  # only one thread can be inside this block at a time
            counter += 1

threads = [threading.Thread(target=increment) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(counter)  # always 400_000 — without the lock, this is racy and comes out lower
```

---

## 8. Semaphores

- Lock generalized to N permits — bounding concurrency (e.g. "only 5 requests at once")

Say we're fanning out two kinds of calls to the same Bedrock model — "describe this image" and "summarize this text" — across threads. Different operations, same endpoint, same provider-enforced rate limit shared across both:

```python
import boto3

bedrock = boto3.client("bedrock-runtime")

# Region-versioned cross-region inference profile. The bare base model ID
# ("anthropic.claude-sonnet-4-5-20250929-v1:0") returns HTTP 400 on current
# Claude models — Bedrock requires an inference profile. Swap the "us." prefix
# for "eu." / "apac." / "global." to change the routing scope.
MODEL_ID = "us.anthropic.claude-sonnet-4-5-20250929-v1:0"

def describe_image(payload):
    return bedrock.invoke_model(modelId=MODEL_ID, body=build_body(f"Describe this image: {payload}"))

def summarize_text(payload):
    return bedrock.invoke_model(modelId=MODEL_ID, body=build_body(f"Summarize: {payload}"))

# One "node" per operation — same shape as a LangGraph node registry
NODES = {
    "describe_image": describe_image,
    "summarize_text": summarize_text,
}

tasks = (
    [("describe_image", img) for img in images]
    + [("summarize_text", txt) for txt in texts]
)

def call_bedrock(task):
    operation, payload = task
    return NODES[operation](payload)
```

**Scenario 1 — as many threads as possible:**

```python
from concurrent.futures import ThreadPoolExecutor

# One thread per task, no cap — blasts the endpoint with every
# request at once regardless of what the provider can actually handle
with ThreadPoolExecutor(max_workers=len(tasks)) as executor:
    results = list(executor.map(call_bedrock, tasks))
    # botocore.exceptions.ClientError: An error occurred (ThrottlingException)
    # — the account-level rate limit gets blown through almost immediately
```

**Scenario 2 — add retries with backoff + jitter:**

```python
import random
import time
from botocore.exceptions import ClientError

def call_bedrock_with_retry(task, max_retries=6):
    for attempt in range(max_retries):
        try:
            return call_bedrock(task)
        except ClientError as e:
            if e.response["Error"]["Code"] != "ThrottlingException":
                raise
            # exponential backoff + jitter so retries from different threads
            # don't all land on the server at the same instant
            sleep_s = (2 ** attempt) * 0.5 + random.uniform(0, 0.5)
            time.sleep(sleep_s)
    raise RuntimeError(f"Gave up on {task} after {max_retries} retries")

with ThreadPoolExecutor(max_workers=len(tasks)) as executor:
    results = list(executor.map(call_bedrock_with_retry, tasks))
    # Correct now, but slower — every throttled call still burns a round
    # trip, and the backoff sleeps stack up under sustained load
```

**Scenario 3 — size a semaphore to the provider's actual limit:**

```python
import threading

# Say Bedrock allows 10 concurrent requests against this model for our account
RATE_LIMIT = 10
semaphore = threading.Semaphore(RATE_LIMIT)

def call_bedrock_limited(task):
    with semaphore:  # the 11th caller blocks here until a permit frees up
        return call_bedrock(task)

with ThreadPoolExecutor(max_workers=len(tasks)) as executor:
    results = list(executor.map(call_bedrock_limited, tasks))
    # Never exceeds what the provider allows, so we stop paying for
    # throttled requests and retries entirely — just steady, bounded throughput
```

---

## 9. asyncio

Asyncio runs an **event loop** that schedules tasks. Tasks voluntarily "pause" themselves when waiting for something — a network response, a file read — and while a task is paused the event loop switches to another one, so no time is wasted sitting idle. Everything runs on a **single thread**, which means asyncio avoids the overhead and complexity of thread switching entirely.

That makes it ideal for workloads made of many small tasks that spend most of their time waiting: thousands of concurrent web requests, database queries, message-queue consumers.

### Cooperative vs Preemptive

The key difference between asyncio and multithreading is *who decides* when a waiting task gets switched out:

- **Multithreading — preemptive context switching.** The OS decides. When a thread blocks on a syscall, the kernel switches to another thread automatically; the code doesn't have to be written to cooperate.
- **Asyncio — cooperative multitasking.** The tasks decide. A coroutine only yields control at an `await`, so a task that never awaits (or that awaits something blocking) holds the whole loop hostage.

```python
import asyncio
import httpx

async def fetch_url(client, url):
    response = await client.get(url)  # yields control back to the loop while waiting
    return response.status_code

async def main():
    urls = get_urls_to_fetch()  # e.g. this returns 37 URLs
    async with httpx.AsyncClient() as client:
        # all 37 requests are in flight concurrently — on one thread
        return await asyncio.gather(*(fetch_url(client, u) for u in urls))

asyncio.run(main())
```

### What Multithreading Actually Costs You

When you spin up a thread per request, each thread blocks on a syscall like `recv()` while waiting for a response. That's fine in principle — but every thread carries real overhead:

- **Memory** — each OS thread gets its own stack (megabytes by default on most platforms), even though it's just sitting idle waiting on a socket.
- **Context switching** — when a socket becomes ready, the kernel has to switch back to that thread: save/restore registers, flush CPU caches/TLB, run the scheduler. That's a real syscall-level operation, not free.
- **GIL contention** — in CPython only one thread runs bytecode at a time. The GIL *is* released during blocking I/O, so threads can overlap their waits — but as thread count grows you pay contention reacquiring it on top of the scheduling overhead above.
- **Scalability ceiling** — a few hundred to low thousands of threads is usually the practical limit before memory and scheduler overhead start to dominate.

An asyncio task, by contrast, is just a Python object with a small heap footprint and no kernel stack. Switching between tasks is a function return inside one thread — no syscall, no scheduler, no GIL handoff. Tens of thousands of concurrent tasks is routine.

### Bounding Concurrency: `asyncio.Semaphore`

The rate-limit problem from [section 8](#8-semaphores) doesn't go away just because tasks are cheap — if anything it gets worse, because it's now trivial to have 10,000 requests in flight. Same fix, async flavour:

```python
RATE_LIMIT = 10
semaphore = asyncio.Semaphore(RATE_LIMIT)  # create inside the running loop

async def fetch_limited(client, url):
    async with semaphore:  # the 11th caller awaits here until a permit frees up
        return await fetch_url(client, url)
```

Note there's no pool size to tune here. With threads, `max_workers` served double duty as both the concurrency cap and the resource budget; with asyncio the semaphore is the *only* cap you need, because tasks themselves are nearly free.


### When to Reach for Which

| | Threads | asyncio |
|---|---|---|
| Concurrency unit | OS thread (MBs of stack) | coroutine (bytes on the heap) |
| Switching | preemptive, by the kernel | cooperative, at `await` |
| Practical ceiling | hundreds → low thousands | tens of thousands+ |
| Works with sync libraries | yes, unchanged | no — needs async-native clients |
| Blocking call impact | that thread only | stalls *every* task |
| CPU-bound work | still GIL-limited | still GIL-limited (use processes) |

- **asyncio wins** at high connection counts — servers, crawlers, fan-out to many endpoints, long-lived websocket/streaming connections — and when you're writing the whole stack anyway (FastAPI, aiohttp, `asyncpg`).
- **asyncio is overkill** for a few dozen calls in an otherwise synchronous codebase. A `ThreadPoolExecutor` gets the same wall-clock win with existing `requests`/`boto3` code and no rewrite. Section 6's thread pool is the pragmatic default; asyncio is what you graduate to when thread count itself becomes the bottleneck.
- **Neither helps CPU-bound work.** asyncio is still one thread under one GIL — a tight numeric loop blocks the loop just like `time.sleep` does. That's what section 6's processes are for.

https://medium.com/data-science/deep-dive-into-multithreading-multiprocessing-and-asyncio-94fdbe0c91f0

https://www.youtube.com/watch?v=Ww2HBNpu98g
Demystifying AsyncIO: Building Your Own Event Loop in Python — Arthur Pastel

---

## 10. Free-Threaded Python (3.14)

Everything above treats the GIL as a fact of life. As of **Python 3.14**, the free-threaded build ([PEP 703](https://peps.python.org/pep-0703/)) is an officially supported configuration rather than an experiment — a separate interpreter (`python3.14t` on most platforms) with the GIL compiled out. Threads in that build execute Python bytecode on multiple cores *simultaneously*, which means threads finally work for CPU-bound code, not just I/O.

```python
import sys

sys._is_gil_enabled()  # False on a free-threaded build with the GIL actually off
```

### How It Avoids the Refcount Race

Section 2's race — two threads mutating one object's refcount, ending in a double free — is exactly what the GIL was protecting. The free-threaded build solves it without a global lock:

- **Two refcount fields instead of one, plus an owning-thread id.** The thread that owns an object updates its *local* count with plain, non-atomic writes; other threads touch a separate *shared* count. Because only one thread ever writes the local field, the common case needs **no locking and no atomic instructions** at all — the true refcount is the sum of the two.
- **Containers get per-object locks, not a global one.** Each `dict`, `list`, and `set` carries its own fine-grained lock, so two threads mutating two different dicts never contend with each other. Contention is now scoped to the specific object being shared, not to the entire interpreter.
- **GC pauses.** Cyclic garbage collection can no longer rely on the GIL to hold the object graph still, so it now does **two brief stop-the-world pauses** to get a consistent view. Short, but they exist — unlike the classic build, where the collector simply ran under the GIL.

### The Catch: Your Dependency Stack Decides

Any C extension that isn't explicitly marked thread-safe — via the `Py_mod_gil = Py_MOD_GIL_NOT_USED` slot — **re-enables the GIL for the whole process** when it's imported. CPython prints a warning when this happens, but it's easy to lose in startup noise, so assert it rather than trusting the log:

```python
import sys

# Run this *after* your imports — the GIL can be switched back on by any of them
assert not sys._is_gil_enabled(), "a C extension re-enabled the GIL"
```

One unmarked module anywhere in your dependency tree and you're running a slower interpreter with none of the parallelism you switched builds for.

So the real gains depend entirely on whether your stack — NumPy, Pandas, FastAPI's `pydantic-core`, database drivers, anything compiled — ships **free-threaded wheels**. The major projects increasingly do, but coverage is uneven and it only takes one holdout.

- **Also worth knowing:** the free-threaded build gives up some single-threaded speed (specialising adaptive interpreter optimisations are harder without a GIL), so a single-threaded script can run measurably slower — the gap narrowed a lot in 3.14 but hasn't closed. It's a trade, not a free upgrade.
- **Practical read for now:** treat it as the direction of travel rather than the default. Processes remain the boring, portable answer for CPU-bound work today; free-threading is what eventually makes `ThreadPoolExecutor` the right answer for *both* kinds of workload.

---

### Notes to self
- [x] free-threaded Python
- [x] asyncio as dedicated section
