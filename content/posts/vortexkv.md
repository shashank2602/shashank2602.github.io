---
title: "VortexKV: a multithreaded key-value store in C++20"
date: 2026-05-26
summary: "How a specialized, shared-nothing KV store hits 9.9M pipelined GETs/sec — 3.7× faster than Dragonfly — and the engineering decisions behind it."
tags: ["cpp", "systems", "performance", "databases"]
ShowToc: true
---


On a 64-vCPU EPYC, my key-value store does 9.9 million pipelined GETs per second — 3.7× more than Dragonfly, the fastest multi-threaded Redis alternative on the market. Dragonfly is built by a seasoned team and backed by years of work. So how does a single person's side project beat it on this benchmark? The short answer is specialization — and the long answer is the rest of this post.

## Why I built it

Last December, I was itching to build something from scratch. I had no idea what it would be, just that it had to be different, at least in some ways. After some research, I decided to make a KV store. I'd used Redis before, so I wasn't a stranger to in-memory caches — but using something is quite different from building it. I didn't want to follow any guide, of which there are plenty, since I felt it might cause me to fixate on the design decisions from the tutorial. So I decided to do my own research. I did settle on the RESP protocol early, for Redis client compatibility.

Now the big question: how to make it different? Well, since I'm a C++ developer, why not try making it faster than the competition? That's not so easy either — the most popular KVs (Redis, memcached, Dragonfly) are written in C or C++, so language alone doesn't give me any performance boost. There's one thing I *can* do to beat industry leaders: make it more performant through **specialization**. Popular products, even though developed over years or even decades by a group of very smart people, have to support a wide variety of features, and these come at a cost.

## The single-threaded foundation

My initial plan was to make a single-threaded version. Why? Because of relative simplicity. I didn't want multithreading to prevent me from creating a solid foundation. And that's exactly what I did with RapidKV — a single-threaded store that already beat Redis by 4.2×. (It's still up on GitHub if you want the single-threaded story in isolation, but everything that matters carries forward into what came next.)

For the first iteration of this project, everything ran on a single thread: the network IO, the parsing, the memory reads/writes — similar to Redis (though they've recently introduced separate IO threads). I wrote the code in such a way that it would make it easier to transition to a multithreaded system.

## The core: storage engine

It was essential that I create a fast storage engine, since that would be the backbone of the system. Modern systems rely heavily on CPU caches for their speed, so if I could choose data structures that take full advantage of these, I could see a performance improvement. The core of this storage engine is a hash table, which stores the actual KV entries.

Most implementations of hash tables (like C++'s `std::unordered_map`) use separate chaining. While this has its advantages, it also comes at the cost of poor cache locality. Since each entry is its own separate node, each pointer jump risks a cache miss. So if you need to probe, say, 4 times for an entry, the CPU will read 4 separate 64-byte cache lines that can be scattered across memory. This leads to poor performance. This is what Redis uses. The other end of the spectrum is open addressing, where everything is laid out sequentially, and CPUs love sequential data because of the excellent cache usage. This does come with its own disadvantages, but if you want speed, it's the right choice. Dragonfly, by comparison, uses a hybrid approach: their hash table uses separate nodes, but each node is a mini open-addressing hash table itself. So they created an excellent data structure that's faster than the Redis dict while also using less memory. I, on the other hand, went all-in on an open-addressing implementation (Robin Hood), since that should still give the best performance — but comes at the cost of higher memory usage.

The open-addressing implementation I went with is called Robin Hood hashing (my implementation is called FlatMap). After my research, this seemed to serve my purpose best: excellent cache performance with great average seek times. Here's how it works — on a collision, we linearly probe and start checking consecutive slots. Either we find an empty slot, or in the case of an occupied slot we check if the new key is farther away from its home than the current key occupying the slot. If it is, we "rob" the slot from that key and displace it forward.

```cpp
if (newEntryMetadata.psl > m_table.metadata[keyPos].psl)
{
    std::swap(key, m_table.keys[keyPos]);
    std::swap(value, m_table.values[keyPos]);
    std::swap(newEntryMetadata, m_table.metadata[keyPos]);
}
```

And we keep doing this until we reach an empty slot.

What this does, compared to simple linear probing, is dramatically reduce the variance in probe length. In my testing, even with millions of keys, the average probe length turned out to be around 1 (0 being its home) even at a high load factor of 0.85. These are excellent results. Even the max probe length (worst-case scenario), over tens of iterations with millions of keys, the longest I got was in the 20s.

Because of the Structure-of-Arrays (SoA) design, the metadata is packed into just 2 bytes per entry. This allows up to 32 entries to reside within a single 64-byte cache line. Consequently, worst-case probes require fetching at most two cache lines to scan the metadata, filtering out 99% of unnecessary key comparisons before the heavier key/value arrays are ever accessed.

Resizing is handled incrementally rather than stop-the-world, to avoid the latency spikes a bulk rehash would cause — there's enough there for its own post.

Now let's come to what the FlatMap stores in its keys and values. While my initial plan was to support many more data types, I focused on getting two types right — integers and strings — rather than many types half-done. Integers are stored as-is, but the big question was: can I optimize the strings somehow? There are three ways I could think of: make the size of the object as small as possible for better cache performance and move-ability (remember, we need to move objects around in Robin Hood); some level of inlining of data, called Small String Optimization (SSO), to reduce pointer jumps and therefore improve cache performance; and lastly a custom allocator that doesn't fragment the heap for every new string allocation. C++'s default `std::string` isn't bad, but object size varies between implementations from 24–32 bytes, and so does the SSO length. For full control I needed to write something from scratch. For comparison, both Redis and Dragonfly employ some form of SSO — for Redis this SSO still requires a pointer dereference, and for Dragonfly it's completely inline.

This is what I decided on: a custom 16-byte string called CompactString that inlines data for size ≤ 15 and falls back to a custom slab allocator for larger sizes. Since the size is exactly 16 bytes and it's aligned to 16 bytes, 4 of these can fit in a single cache line. That means in the best case, for small strings in our table, a single read has 4 strings in contiguous memory — no pointer jump needed. And even if the size is larger and heap allocation is required, the slab allocator can allocate and deallocate memory in O(1) time, making for an extremely fast string type. This custom string type is 3.6× faster than `std::string` for construction, 1.8× faster for access, and a blazing 5.8× faster for move operations.

So what happens when we combine this new CompactString with our FlatMap? We get 2.5× faster insertions than `std::unordered_map` with `std::string`, 2× faster lookups, and 2.6× faster deletions.

> *These micro-benchmarks are part of the repo.*

This wraps up the core storage engine. Now let's see the overall single-threaded architecture and get a better understanding of the other components.

![Single-threaded architecture](/images/single_threaded_architecture.svg)

Let's start with Asio. This is the part of the system responsible for managing TCP connections and IO; it fires when data is read and when data has been written to the network buffer. This is managed by the `io_context` event loop. One thread here is doing all the work — whenever a connection receives a request, the event loop wakes up the thread. It then does the reading, parsing, executing, and writing. Reading and writing operations are both async. Once the writing completes, we start all over again. All the connections reside in the server class.

The crucial part of the system is IO. A system handicapped by IO cannot be scaled properly, no matter how many cores you throw at it. When you're doing millions of operations per second, it's necessary to reduce as many copies and allocations in the pipeline as you can, since these dirty the CPU caches and use up memory bandwidth — other than, of course, just wasting CPU cycles. And that's exactly what I did: the input data that's read into the network buffer is used as-is to create commands using `string_view`, and passed directly to the storage layer to be stored in the hash tables. Similarly, when writing output, it's written directly to the network buffer; any formatting required (like integer-to-string) is done using preallocated stack buffers. So the protocol (RESP) parsing is zero-copy, and protocol writing is zero-allocation.

Both the input and output buffers use a custom linear buffer with compaction, with the output buffer actually consisting of two buffers — one used to write to the network while the other collects responses. This means writes never stall. (When I moved onto the multithreaded architecture, there was an extra copy step that needed to be added — but I'll explain that later.)

A micro-optimization in the IO path is an integer-packed command dispatcher. Command dispatch avoids string comparison entirely: each command name (up to 8 bytes) is folded to uppercase and packed into a single `uint64_t`, so matching an incoming command against the table is a handful of integer comparisons instead of repeated string compares on the hot path.

This results in an IO path that scales incredibly well even on a single thread with pipelining, leaving Redis far behind as the pipelining depth goes up. (Pipelining here means sending N requests at a time.)

At pipelining = 500, this version achieves a whopping 8 million ops/sec, compared to Redis's 1.9 million — and that's while having 1/5th the p99 latency of Redis (tested on a Ryzen 4800H). Now, I know a pipeline depth of 500 is very high and at the extreme end of realistic values, but it's meant to show the maximum throughput you could get out of this system. And yes, pipelining this deep is used in some real workloads.

Some design choices made early on, with the multithreaded future of this project in mind, include the thread-local slab allocator, no global shared resource/mutex, and a self-contained connection object.

## Hitting the wall

Hitting high throughput with pipelining has its use cases, but without pipelining you're still severely limited on throughput compared to multithreaded counterparts. Those extra cores on the machine are sitting idle. To use them, we have to somehow divide the incoming work across multiple threads. One basic solution is to create N worker threads that access the hash table protected by a lock. While this may perform better than the single-threaded version, it'll scale very poorly — since all the threads are accessing the same data structure, there's going to be heavy lock contention.

A better approach is fine-grained locking, something memcached uses. Essentially, instead of one global lock, you create many small specific locks that each protect separate segments of the hash table. This minimizes lock contention and can result in good throughput scaling — but only up to a point. As the thread count increases, you start running into the same problem again: a higher thread count means higher chances of multiple threads hitting the same locked segment, and so we have the same issue. It goes from near-linear scaling in the beginning to flatlining at higher thread counts.

So what's the answer? It's called a shared-nothing design, and it's used by Dragonfly — for good reason, since it allows for maximum throughput. Let's get into the details.

## The shared-nothing design

![Shared-nothing multithreaded architecture](/images/multithreaded_architecture.svg)

In simple terms, we take the one hash table we had and split it into N tables, where N is the number of threads. Each of these N tables, enclosed in an independent entity called a shard, owns a different part of the key space. The distribution is decided using a hash function, with uniform keys resulting in uniform distribution across the N shards. These shards run their own separate event loop in their own separate thread. The shards are cache-line aligned to prevent false sharing, and each shard's thread is pinned to a dedicated CPU core for optimal cache locality.

But now the question is: who takes care of the IO — the main thread, or separate threads? It's the shards themselves. Whenever a new connection is made, it's assigned to one of the shards in round-robin fashion.

```cpp
// hash the key, then a fast modulo (via libdivide) gives the owning shard
uint64_t hash     = rapidhash_withSeed(request.arguments[0].data(),
                                       request.arguments[0].size(),
                                       m_routingHashSeed);
uint64_t quotient = hash / m_fastModDivisor;
targetShardId     = hash - (quotient * m_shardPool.size());

if (targetShardId == m_shardId)
    m_dispatcher.dispatch(request, m_database, response);          // local: inline fast path
else
    m_shardPool[targetShardId]->ExecuteRemote(std::move(request), /* ... */);  // remote: post to owner
```

So whenever a connection gets a request, we calculate the hash of the key using rapidhash (one of the fastest non-cryptographic hashes), then calculate the fast modulo using libdivide. This gives us the shard ID where the key resides.

Now there are two possibilities: either the key it requests is in the same shard, or in a different shard. If it's the same shard, the command is executed inline in a fast path. If it's in a different shard, we post this request to the target shard. Once the target shard executes the command asynchronously, it notifies the local shard, and we write to the network buffer.

Now the problem: what happens with pipelined requests? Since these requests can be spread across the shards, they can complete in a random order. Since a client expects pipelined responses back in the same order it sent them, we need to enforce strict ordering of the responses. This is done using request indexes — each request's response is written by the local or remote shard into its own buffer, and each shard posts its completion to the local shard. Once all responses are accounted for, they're copied to the network buffer in their respective order. And because of this ordering requirement, there's an extra copy in the multithreaded architecture.

So how are cross-shard requests handled internally? When we want to access data in a remote shard, we post the request to the target shard. The target shard stores this request in its own task queue (managed by Asio's `io_context`), and all the requests coming from different shards are queued up here and executed one by one. The hash table itself doesn't have any form of locking mechanism on it. Similarly, when a target shard has executed a caller's request, it posts the completion notification to the caller shard, which again means a completion callback is inserted into the caller's queue.

So while there's no locking mechanism on the hash table itself, the task queue still needs to be protected from corruption by multiple threads. Asio's `io_context` internally uses locks to ensure this thread safety.

There's one additional challenge with cross-shard ops. Each cross-shard op requires a small heap allocation to store the callback — and here's the nasty part: the allocation and deallocation happen on different threads. Shard A allocates the callback, shard B frees it. Then B allocates the completion callback, and A frees it. This is the textbook worst case for a thread-local allocator. My slab allocator assumes alloc and free happen on the same thread — it has no safe answer for a cross-thread free. So I couldn't use it on this path. That leaves two additional heap allocations per cross-shard op (and as shard count increases, so do the cross-shard ops) routed through the generic allocator. That's exactly why I chose mimalloc: it's designed so that freeing memory on a different thread than it was allocated on is cheap. Using it gave me roughly a 20% boost in throughput.

So what does all this achieve? A system that keeps on scaling even at very high thread counts. Let's see the results.

## Benchmarks

### Test environment

| Component | Details |
|---|---|
| **CPU** | AMD EPYC 9554P (64C / 128T) |
| **Server cores** | 64 threads pinned to cores 0–63 |
| **Benchmark cores** | 64 threads pinned to cores 64–127 via `taskset -c 64-127` |
| **Benchmark tool** | `memtier_benchmark` |
| **Value size** | 256 bytes |
| **Key range** | 1–10,000,000 |

### Server configurations

| Server | Launch command |
|---|---|
| **Redis** | `redis-server --save "" --appendonly no` |
| **Dragonfly** | `dragonfly --dbfilename "" --snapshot_cron "" --cache_mode=true --version_check=false --proactor_threads=64` |
| **VortexKV** | `./VortexKV VortexKV.config` (64 shards) |

> All three servers ran on the same bare-metal machine. Redis had all persistence disabled; Dragonfly ran in cache mode with snapshots disabled. Servers were restarted between runs for a clean state.

### Without pipelining

**SET throughput (no pipeline)** — 64 threads × 10 connections · ratio 1:0 (SET only) · 256B values · 100 seconds

| Metric | Redis | Dragonfly | VortexKV | vs Redis | vs Dragonfly |
|---|---|---|---|---|---|
| **Ops/sec** | 66,631 | 2,489,825 | **2,575,169** | **38.6×** | **+3.4%** |
| **Avg latency** | 9.508 ms | 0.254 ms | **0.248 ms** | **−97.4%** | **−2.4%** |
| **p99 latency** | 11.967 ms | 0.351 ms | **0.343 ms** | **−97.1%** | **−2.3%** |

**GET throughput (no pipeline, pre-populated)** — 64 threads × 10 connections · ratio 0:1 (GET only) · 256B values · 100 seconds · 10M keys pre-populated

| Metric | Redis | Dragonfly | VortexKV | vs Redis | vs Dragonfly |
|---|---|---|---|---|---|
| **Ops/sec** | 70,780 | 2,552,889 | 2,541,428 | **35.9×** | −0.4% |
| **Avg latency** | 9.131 ms | 0.253 ms | **0.249 ms** | **−97.3%** | **−1.6%** |
| **p99 latency** | 9.407 ms | 0.351 ms | **0.343 ms** | **−96.4%** | **−2.3%** |

> **Takeaway:** Without pipelining, VortexKV and Dragonfly are effectively neck-and-neck at ~2.5M ops/sec — both ~36× faster than single-threaded Redis. VortexKV holds a slight latency edge, while Dragonfly is fractionally ahead on raw GET throughput (−0.4%).

### With pipelining

**Pipelined SET (pipeline = 30)** — 64 threads × 10 connections · pipeline = 30 · ratio 1:0 · 256B values · 200K requests/client

| Metric | Redis † | Dragonfly | VortexKV | vs Redis | vs Dragonfly |
|---|---|---|---|---|---|
| **Ops/sec** | 567,642 | 7,417,396 | **11,880,297** | **20.9×** | **+60.2%** |
| **Avg latency** | 8.298 ms | 2.358 ms | **1.769 ms** | **−78.7%** | **−25.0%** |
| **p99 latency** | 15.743 ms | 4.047 ms | **2.319 ms** | **−85.3%** | **−42.7%** |

**Pipelined GET (pipeline = 30, pre-populated)** — 64 threads × 10 connections · pipeline = 30 · ratio 0:1 · 256B values · 100 seconds · 10M keys pre-populated

| Metric | Redis † | Dragonfly | VortexKV | vs Redis | vs Dragonfly |
|---|---|---|---|---|---|
| **Ops/sec** | 558,308 | 2,688,873 | **9,889,346** | **17.7×** | **3.68×** |
| **Avg latency** | 8.592 ms | 7.140 ms | **1.918 ms** | **−77.7%** | **−73.1%** |
| **p99 latency** | 17.023 ms | 7.807 ms | **2.655 ms** | **−84.4%** | **−66.0%** |

> † Redis was run at its optimal 16-thread × 10-connection config for the pipelined tests. Redis executes commands on a single thread, so adding more threads doesn't improve its execution throughput — this gives Redis its best showing rather than handicapping it.

> **Takeaway:** Under pipelining, VortexKV's shared-nothing design with per-shard databases pulls clearly ahead. Pipelined SET is **60% faster** than Dragonfly; pipelined GET is **3.7× faster** — the largest gap in the entire suite.

### An anomaly worth chasing: why were writes outpacing reads?

One interesting thing to note here is the difference in SET vs GET numbers. Both VortexKV and Dragonfly achieve higher SET ops, which feels like an anomaly — it should be the exact opposite, especially for Dragonfly. I thought something was wrong, so I ran the benchmarks again, not just on this EPYC server but on two other machines as well: a C3D GCP instance and my own Ryzen laptop. The trend was similar.

Knowing my own architecture, I had a suspicion that might be true for Dragonfly as well — and that was the extra copy required in the response pipeline I mentioned earlier. For any SET request, the incoming data is passed directly using `string_view` to the target shard, where it's copied once into the hash table entry, and we respond with a small `+OK\r\n` (5 bytes). That response data needs to be passed to the caller's shard, for which we first copy into a temporary buffer, then copy to the network to preserve ordering. For a GET response, we again send the data directly, but of course there's no copying into the hash table entry — what's different is that the response data is now much bigger (256 bytes in our benchmarks).

Maybe Dragonfly was experiencing something similar, but worse. So I simply changed the data size in the benchmarks from 256 bytes to 8 bytes, and — boom — now Dragonfly's GETs were as fast as its SETs. To make sure, I read some of Dragonfly's code to see what was going on: for a GET response, Dragonfly first creates a temporary string on the target shard (so a heap alloc plus a copy), then this string object is passed to the caller, where it's read again (from the target's memory, which is more costly for multi-CCD CPUs) and copied to the output buffer. If we reduce the data size, this string is now inlined — so no heap alloc, and the copy is significantly cheaper — resulting in better numbers.

### Scaling

**Shard scaling (1:1 SET:GET, no pipeline)** — 256B values · 25 seconds · client threads scaled proportionally to server shards

| Shards | Dragonfly (ops/sec) | VortexKV (ops/sec) | Δ | Dragonfly p99 | VortexKV p99 |
|---|---|---|---|---|---|
| 1 | **83,664** | 73,510 | −12.1% | **0.287 ms** | 0.295 ms |
| 4 | **311,311** | 287,612 | −7.6% | **0.367 ms** | 0.415 ms |
| 16 | **1,064,231** | 959,061 | −9.9% | **0.415 ms** | 0.447 ms |
| 32 | **1,736,663** | 1,572,802 | −9.4% | **0.559 ms** | 0.575 ms |
| 64 | 2,403,368 | **2,551,297** | **+6.2%** | 0.351 ms | 0.351 ms |

![Shard scaling: VortexKV vs Dragonfly](/images/shard_scaling.svg)

At 1–32 shards, Dragonfly leads by 8–12%. I don't have a precise breakdown of where that comes from, but it's likely from a more mature implementation. The 1-shard count is an especially interesting case, since this configuration makes both systems act like a single-threaded cache (no cross-shard ops). This lead might be coming from their helio I/O framework, plus years of profiling-driven micro-optimization on a mature codebase.

At 64 shards the relationship inverts. Dragonfly wraps every command — even single-key ops — in a Transaction object, the machinery that gives them lock-free atomic multi-key operations (something VortexKV deliberately doesn't support). That per-command cost is a fair price for a feature I skipped, and skipping it is likely part of why my lighter path scales better at full saturation.

> **Note:** This scaling test is unpipelined on purpose, to isolate how cleanly each architecture scales with core count rather than to measure peak throughput. The pipelined numbers earlier are the throughput *ceiling*; this is the scaling *shape*. The +6.2% here isn't the same claim as the 3.7× there — one is "scales better at saturation," the other is "higher peak under load."

## Limitations

VortexKV is specialized, and not complete. A few honest categories of what's missing and why:

**Multi-key DEL/EXISTS.** These existed in the single-threaded version — since everything was in a single table, they were easy. As soon as we move to the sharded architecture, a single multi-key request can span multiple shards. This makes these ops non-trivial: every multi-key operation now needs to account for results from multiple shards, something similar to how we handle pipelined requests. The implementation is not only genuinely hard, it also introduces extra work that would hurt throughput — counterintuitive to what I'm trying to build. So I gave up a feature I'd already had working.

**Persistence, replication, AUTH, and TLS** aren't here either — but those are production-completeness features, and VortexKV is a study of the hot path, not a production-ready Redis replacement. They were out of scope by design.

**Data types beyond strings and integers** were left out because I refuse to ship a type that isn't optimized. I initially did have plans to support more types — you can actually see the remnants of this in the hash table's value type, which uses a `std::variant` (for just string and int, a custom union would be better). Take lists, for example: I could have added the feature backed internally by a `std::vector`; it was trivial enough. But shipping a slow data structure in a project that's *about* data-structure performance would contradict the entire point. I'd rather ship two data types I'm proud of than ten I'm not.

## Try it yourself

And that's how a single-person side project ends up at 9.9M pipelined GETs/sec — not by out-engineering a seasoned team, but by refusing to pay for features I didn't need and optimizing wherever I could. VortexKV, the full source, and reproducible benchmarks are all on GitHub: **[github.com/shashank2602/VortexKV](https://github.com/shashank2602/VortexKV)**.

I'm currently open to C++ roles. The best way to reach me is by email at shashankjoshi2602@gmail.com or on [LinkedIn](https://www.linkedin.com/in/shashank26/).
