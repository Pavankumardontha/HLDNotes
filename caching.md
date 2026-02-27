# Caching

## Table of Contents

- [Caching](#caching-1)
- [Static vs Dynamic Cache](#static-vs-dynamic-cache)
  - [Dynamic Cache](#dynamic-cache)
  - [Static Cache](#static-cache)
- [Cache Consistency](#cache-consistency)
- [Cache Invalidation](#cache-invalidation)
- [Inline Cache](#inline-cache)
  - [Read-Through (Inline Read)](#read-through-inline-read)
  - [Write-Through (Inline Write)](#write-through-inline-write)
  - [Write-Behind / Write-Back Cache](#write-behind--write-back-cache)
- [Cache-Aside Strategy](#cache-aside-strategy)
  - [How It Works](#how-it-works)
  - [Sequence Diagram](#sequence-diagram)
- [Cache Eviction](#cache-eviction)
  - [Cache Eviction Policies](#cache-eviction-policies)
- [Cache Persistence](#cache-persistence)
- [Redis](#redis)
  - [Redis is Single-Threaded](#redis-is-single-threaded)
  - [How Redis Handles Concurrent Requests](#how-redis-handles-concurrent-requests)
  - [Redis Persistence](#redis-persistence)
  - [Append-Only Files (AOF)](#append-only-files-aof)
  - [Point-in-Time Backups (RDB)](#point-in-time-backups-rdb)
  - [Comparing RDB to AOF Persistence](#comparing-rdb-to-aof-persistence)
  - [Replication in Redis](#replication-in-redis) *(left to explore)*
- [Redis vs Memcached](#redis-vs-memcached) *(left to explore)*
- [Cache Performance](#cache-performance)

---

## Caching

For a cache to be effective, you need a really good understanding of the **statistical distribution** of data access from your application or data source. When your data access has a **normal (bell-curve) distribution**, caching is more likely to be effective compared to a **flat data access distribution**.

This is because a bell-curve distribution means a small subset of data is accessed very frequently (the **"hot" data**), while the rest is accessed rarely. Caching just that hot subset yields a high cache hit rate with a small cache. In a flat distribution, every piece of data is accessed equally often — there is no hot subset to prioritize, so you'd need to cache nearly all the data to achieve a high hit rate, which defeats the purpose of a cache.

## Static vs Dynamic Cache

There are two fundamental types of caches — **static** (sometimes called **read-only**) caches and **dynamic** (sometimes called **read/write**) caches.

### Dynamic Cache

In a **dynamic cache**, when a value is changed in the data store, that value is also changed directly in the cache — an old value is overwritten with a new value. The cache is called a dynamic cache, because the application using the cache can **write changes directly into the cache**. How this occurs can vary depending on the type of usage pattern employed, but the application using the cache can fundamentally change data in a dynamic cache.

### Static Cache

In a **static cache**, when a value is stored in the cache, it cannot be changed by the application. It is **immutable**. Changes to the data are stored directly into the underlying data store, and the out-of-date values are typically removed from the cache. The next time the value is accessed from the cache, it will be missing, hence it will be read from the underlying data store and the new value will be stored in the cache.

## Cache Consistency

A cache simply stores a copy of data held in an underlying data store. Because it is a copy of the data that is stored in the cache, when things go wrong or someone makes a mistake, it is possible that the value stored in the cache may differ from the value stored in the underlying data store. This can happen, for example, when the underlying data changes and the cache is not updated with the new value in a timely manner. When this happens, a cache is considered **inconsistent**.

**Cache consistency** is the measure of whether data stored in the cache has the same value as the source data that is stored in the underlying data store.

## Cache Invalidation

**Cache invalidation** is, quite simply, removing a value from a cache once it has been determined that the value is no longer up to date.

## Inline Cache

An **inline cache** — which can include **read-through**, **write-through**, and **read/write-through** caches — is a cache that sits in front of a data store, and the data store is accessed through the cache.

If an application wants to read a value from the data store, it attempts to read the value from the cache. If the cache has the value, it is simply returned. If the cache does not have the value, then the cache reads the value from the underlying data store. The cache then remembers this value and returns it to the calling application. The next time the value is needed, it can be read directly from the cache.

The key difference from cache-aside is that the application never talks to the data store directly — the cache handles all data store interactions transparently. On a read, the cache fetches from the data store on a miss. On a write, the cache updates itself and writes through to the data store before acknowledging the write.

### Read-Through (Inline Read)

```
 Application            Inline Cache               Data Store
     │                       │                         │
     │   GET key="3x4"       │                         │
     │──────────────────────►│                         │
     │                       │                         │
     │            ┌─────────────────────┐              │
     │            │ Has the value?      │              │
     │            └─────┬─────────┬─────┘              │
     │             YES  │         │  NO                │
     │                  │         │                    │
     │                  │         │  GET key="3x4"     │
     │                  │         │───────────────────►│
     │                  │         │                    │
     │                  │         │   value = 12       │
     │                  │         │◄───────────────────│
     │                  │         │                    │
     │                  │         │  ┌──────────────┐  │
     │                  │         │  │store in cache│  │
     │                  │         │  │"3x4" = 12    │  │
     │                  │         │  └──────────────┘  │
     │                  │         │                    │
     │   return 12      │◄────────┘                    │
     │◄─────────────────│                              │
     │                  │                              │
```

### Write-Through (Inline Write)

In a **write-through cache**, rather than having the application update the data store directly and invalidating the cache, the application updates the cache with the new value, and the cache updates the data store **synchronously**. This means that the cache maintains an up-to-date value and can still be used, yet the data store also has the newly updated value. The cache is responsible for maintaining its own **cache consistency**.

As soon as the write is complete, both the cache and the data store have the same value, and so the cache remains consistent. Anyone else accessing the value from either the cache or the data store will get the new value, consistently and correctly.

One downside of the **write-through** strategy is that the actual write is relatively **slow**, because the write call has to update both the cache and the underlying data store. Hence, **two writes** are required, and one of them is to the slow backend data store.

```
 Application            Inline Cache              Data Store
     │                       │                        │
     │   PUT "3x4" = 12      │                        │
     │──────────────────────►│                        │
     │                       │                        │
     │                       │  ┌──────────────┐      │
     │                       │  │update cache  │      │
     │                       │  │"3x4" = 12    │      │
     │                       │  └──────────────┘      │
     │                       │                        │
     │                       │   PUT "3x4" = 12       │
     │                       │───────────────────────►│
     │                       │                        │
     │                       │       write OK         │
     │                       │◄───────────────────────│
     │                       │                        │
     │   write OK            │                        │
     │◄─────────────────────-│                        │
     │                       │                        │
```

### Write-Behind / Write-Back Cache

In order to speed up the write operation, a **write-behind cache** can be used instead.

With a write-behind cache, the value is updated directly in the cache, just like the write-through approach. However, the write call then **immediately returns**, without updating the underlying data store. From the application perspective, the write was fast, because only the cache had to be updated.

At this point in time, the cache has the newer value, and the data store has an older value. To maintain cache consistency, the cache then updates the underlying data store with the new value, at a later point in time. This is typically a **background, asynchronous** activity performed by the cache.

Although this process results in a faster application write operation, there is a **tradeoff**. Until the cache updates the data store with the new value, the cache and data store hold different values. The cache has the correct value, and the underlying data store has an incorrect, or **stale**, value. This gets remedied when the write-behind operation in the cache updates the data store — but until then, the cache and data store are out of sync. The cache is considered **inconsistent**.

This would not be a problem if all access to the key was performed through this cache. However, if there is a mistake or error of some kind and the data is accessed directly from the underlying data store, or through some other means, it is possible that the old value will be returned for some period of time. Whether or not this is a problem depends on your application requirements.

```
 Application            Inline Cache             Data Store
     │                       │                        │
     │   PUT "3x4" = 12      │                        │
     │──────────────────────►│                        │
     │                       │                        │
     │                       │  ┌──────────────┐      │
     │                       │  │update cache  │      │
     │                       │  │"3x4" = 12    │      │
     │                       │  └──────────────┘      │
     │                       │                        │
     │   write OK            │                        │
     │◄──────────────────────│  (returns immediately) │
     │                       │                        │
     │                       │                        │
     │  ┌─────────────────────────────────────────┐   │
     │  │         async (later, in background)    │   │
     │  └─────────────────────────────────────────┘   │
     │                       │                        │
     │                       │   PUT "3x4" = 12       │
     │                       │───────────────────────►│
     │                       │                        │
     │                       │       write OK         │
     │                       │◄───────────────────────│
     │                       │                        │
     │              cache and data store now in sync  │
     │                       │                        │
```

## Cache-Aside Strategy

In a **cache-aside pattern**, the cache is accessed **independently** of the data store. **Cache consistency** is the responsibility of the application.

When an application needs to read a value, it first checks to see if the value is in the cache. If it is not in the cache, then the application accesses the data store directly to read the desired value. Then, the application stores the value in the cache for later use. The next time the value is needed, it is read directly from the cache.

Unlike an inline cache, in the cache-aside pattern there is **no direct connection** between the cache and the underlying data store. All data operations to either the cache or the underlying data store are handled by the application.

### How It Works

1. Consult the cache to see if there is an entry representing `"3x4"`.
2. If there is an entry, return the result from the cache. **STOP**.
3. If there is not an entry, call the service and get the result of `3 × 4`.
4. Store the result of the service call into the cache under the entry `"3x4"`.
5. Return the result.

### Sequence Diagram

**Cache Miss (first call):**

```
 Client          Multiplication Service           Cache
   │                       │                        │
   │    request "3x4"      │                        │
   │──────────────────────►│                        │
   │                       │                        │
   │                       │     GET key="3x4"      │
   │                       │───────────────────────►│
   │                       │                        │
   │                       │    MISS (not found)    │
   │                       │◄───────────────────────│
   │                       │                        │
   │                       │  ┌─────────────────┐   │
   │                       │  │compute 3x4 = 12 │   │
   │                       │  └─────────────────┘   │
   │                       │                        │
   │                       │    PUT "3x4" = 12      │
   │                       │───────────────────────►│
   │                       │                        │
   │                       │      stored OK         │
   │                       │◄───────────────────────│
   │                       │                        │
   │     return 12         │                        │
   │◄──────────────────────│                        │
   │                       │                        │
```

**Cache Hit (subsequent call):**

```
Client        Multiplication Service        Cache
  │                    │                       │
  │  request "3x4"     │                       │
  │───────────────────►│                       │
  │                    │                       │
  │                    │      GET key="3x4"    │
  │                    │──────────────────────►│
  │                    │                       │
  │                    │    HIT: value = 12    │
  │                    │◄──────────────────────│
  │                    │                       │
  │    return 12       │                       │
  │◄───────────────────│                       │
  │                    │                       │
```

On a cache miss, the Multiplication Service computes the result internally, stores it in the cache, and returns it. On a cache hit, the result is returned directly from the cache without any computation.

## Cache Eviction

A cache is typically smaller than the underlying data store, so it is necessary for the cache to contain only a subset of the data held in the underlying data store. This process of removing older or less frequently used data is called **cache eviction**, because data is evicted, or removed, from the cache.

### Cache Eviction Policies

#### 1. Least Recently Used (LRU)

In a cache with a **least-recently used (LRU)** eviction policy, when the cache is full and new data needs to be stored, the cache makes room for the new data by looking at the data already stored in the cache. It then finds the piece of data that hasn't been accessed for the **longest period of time**. It then removes that data from the cache and uses the free space to store the new data.

#### 2. Least Frequently Used (LFU)

In a **least-frequently used (LFU)** cache, when the cache is full and data needs to be evicted, the cache looks for the data that has been accessed the **fewest number of times**, and removes that data to make room for the new data.

The difference between an LRU and LFU cache is small. An LRU uses the amount of time since the data was last accessed, while the LFU uses the number of times the data was accessed. In other words, the LRU bases its decision on an **access date**, the LFU bases its decision on an **access count**.

#### 3. First-In-First-Out (FIFO)

When the cache is full, the cache looks for the data that has been in the cache the longest period of time and removes that data first. The data that was first inserted into the cache is the data that is first evicted. This eviction policy is not common in enterprise caches.

#### 4. Random Eviction

In a random eviction cache, when the cache is full, a randomly selected piece of data is evicted from the cache.

#### 5. Time-To-Live (TTL)

In a **time-to-live (TTL)** eviction, data values are given a period of time — potentially seconds, minutes, hours, days, years — that they are to be stored in the cache. After that period has elapsed, the value is removed from the cache, **whether or not the cache is full**. In Redis, this is done at the **key level**, not at the cache-eviction policy level.

## Cache Persistence

**Cache persistence** is the capability of storing a cache's contents in **persistent storage**, so that a power failure or other outage does not lose the contents of the cache.

A **volatile cache** is a cache that exists in memory and is subject to erasure when a power outage or system restart occurs. In case of failure or other issue, the contents of a volatile cache cannot be assumed to be available — the system must always be functional, even if the cache itself is removed. The performance of the system may be impacted, but the capabilities and functionality cannot be affected.

With cache persistence, contents persist even during power outages and system reboots.

Typically, a **volatile cache** is implemented in **RAM**, or some other high-performance, non-permanent memory store. A **persistent cache** typically relies on a **hard disk**, **SSD**, or other long-term persistent storage.

It is possible to implement a persistent cache in volatile memory, as long as the cache mechanism makes appropriate **redundant backups** of the cached contents into persistent memory. In this situation, if the cache fails (such as via a power outage or system reboot) such that the volatile memory is wiped clean, then the redundant cache copy in the persistent memory is used to re-create the contents of the cache in the volatile memory, before the cache comes back online. This process is called **cache rehydration**.

## Redis

**Redis** can operate as either a **volatile** or **persistent** cache. It uses **RAM** for its primary memory storage, making it act like a volatile cache, yet permanent storage can be used to provide the persistent backup and rehydration, so that Redis can be used as a persistent cache.

### Redis is Single-Threaded

Redis uses a **single thread** to handle all read and write commands. Every command — `GET`, `SET`, `DEL`, etc. — is executed sequentially by one thread, one after another.

**Why single-threaded?**

- **No race conditions** — since only one thread processes commands, there's no possibility of two operations modifying the same key at the same time.
- **No lock overhead** — no time wasted acquiring/releasing locks.
- **Atomic by default** — every Redis command is inherently atomic because nothing else can interrupt it mid-execution.

**How is it still so fast?**

- All data is in **RAM** — no disk I/O for reads/writes.
- Operations are extremely simple — each command completes in **microseconds**.
- The bottleneck is usually the **network**, not the CPU.

Starting with Redis 6, Redis uses **multiple I/O threads** for reading requests from the network and writing responses back. But the **actual command execution** is still single-threaded.

### How Redis Handles Concurrent Requests

When multiple clients send commands at the same time, the second command doesn't get lost — it sits in a **queue**. Redis uses an **event loop** (based on `epoll`/`kqueue`) to handle connections:

1. The **OS TCP buffers** hold incoming data for each client connection.
2. The Redis event loop detects which connections have data ready to be read.
3. It reads the commands and places them into an **internal execution queue**.
4. The single thread picks up commands **one by one**, executes each, and sends the response back.

```
  Client A: GET key1 ──┐
                       ▼
               ┌─────────────────┐
               │  OS TCP Buffers │  (network layer holds
               │  ┌─────────────┐│   incoming bytes until
  Client B: ──►│  │ Client A buf││   Redis reads them)
  GET key2     │  │ Client B buf││
               │  └─────────────┘│
               └────────┬────────┘
                        │
                        ▼
               ┌─────────────────┐
               │   Event Loop    │  (detects which sockets
               │   (epoll/kqueue)│   have data ready)
               └────────┬────────┘
                        │
                        ▼
               ┌─────────────────┐
               │  Command Queue  │
               │  ┌─────────────┐│
               │  │ 1. GET key1 ││  <-- picked up first
               │  │ 2. GET key2 ││  <-- waits its turn
               │  └─────────────┘│
               └────────┬────────┘
                        │
                        ▼
               ┌─────────────────┐
               │  Single Thread  │
               │  executes one   │
               │  command at a   │
               │  time           │
               └────────┬────────┘
                        │
                 1. Execute GET key1 --> respond to Client A
                 2. Execute GET key2 --> respond to Client B
```

A typical Redis `GET` command takes about **1-2 microseconds** to execute. So even though the second command is "waiting," it waits for about **1 microsecond**. At this speed, Redis can process **100,000+ commands per second** sequentially, and the waiting time is imperceptible. The latency you actually feel is almost entirely **network round-trip time** (milliseconds), not the queuing time (microseconds).

### Redis Persistence

Redis offers a range of persistent options, specifically:

1. Append-only files (AOF)
2. Point-in-time backups (RDB)
3. A combination of both

### Append-Only Files (AOF)

Redis uses a file called the append-only file (AOF), in order to create a persistent backup of the primary volatile cache in persistent storage. The AOF file stores a real-time log of updates to the cache. This file is updated continuously, so it represents an accurate, persistent view of the state of the cache when the cache is shut down, depending on configuration and failure scenarios. When the cache is restarted and cleared, the commands recorded in the AOF log file can be replayed to re-create the state of the Redis cache at the time of the shutdown. The result? The cache, while implemented primarily in volatile memory, can be used as a reliable, persistent data cache.

The option `APPENDONLY yes` enables the AOF log file. All changes to the cache will result in an entry being written to this log file, but the log file itself isn't necessarily stored in persistent storage immediately. For performance reasons, you can delay the write to persistent storage for a period of time to improve overall system performance. This is controlled via `APPENDFSYNC`. The following options are available:

- `APPENDFSYNC no`: This allows the operating system to cache the log file and wait to persist it to permanent storage when it deems necessary.
- `APPENDFSYNC everysec`: This forces a write of the AOF log file to persistent storage once every second.
- `APPENDFSYNC always`: This forces the AOF file to be written to persistent storage immediately after every log entry is created.

The result of the `APPENDONLY` command is a continuously growing log file. Redis can, in the background, rebuild the log file by removing no longer necessary entries in the log file. The following command will clean up the log file:

`BGREWRITEAOF`

The result is a log file with the shortest sequence of commands needed to rehydrate the current state of the cache in memory.

### Point-in-Time Backups (RDB)

Sometimes it is useful to create a backup copy of the current contents in the cache. This is what the RDB backup is for. This command creates a highly efficient, smallest possible, point-in-time backup file of the current contents of the cache. It can be executed at any point in time as follows:

`SAVE`

This command creates a `dump.rdb` file that contains a complete current snapshot of the Redis cache. Alternatively, you can issue the following command:

`BGSAVE`

This command returns immediately and creates a background job that creates the snapshot.

### Comparing RDB to AOF Persistence

If your goal is to create a reliable, persistent cache that can survive process crashes, system crashes, and other system failures, then the only reliable way to do that is to use AOF persistence with `APPENDFSYNC` set to `always`. No other method guarantees that the entire state of the cache will be properly stored in persistent storage at all times.

If your goal is to maintain a series of point-in-time backups for historical and system-recovery purposes (such as saving one backup per day for an entire month), then the RDB backup is the proper method to create these backups. This is because the RDB is a single file providing an accurate snapshot of the database at a given point in time. This snapshot is guaranteed to be consistent. However, RDB cannot be used to survive system failures, because any changes made to the system between RDB snapshots will be lost during a system failure.

So, depending on your requirements, both RDB and AOF can be used to solve your persistent needs. Used together, they can provide both a system-tolerant persistent cache, along with historical point-in-time snapshot backups.

### Replication in Redis

> **TODO:** Left to explore — Redis replication (master-replica architecture, async replication, sentinel, clustering, read replicas, failover).

## Redis vs Memcached

> **TODO:** Left to explore — Comparison of Redis and Memcached (data structures, persistence, replication, eviction, threading model, use cases).

## Cache Performance

Notice that talking to and manipulating the cache also takes time. So requests that must call the service also have to first check the cache and add a result to the cache. This additional effort is called the **cache overhead**.

In a cache-aside strategy, the cache overhead includes the **cache check time** (to see if the result was previously in the cache) and the **cache write time** (to store the newly calculated result in the cache). Requests that end up having to call the service anyway take more effort than simply calling the service. Put in cache terms, a **cache miss** (that is, a request that cannot be satisfied by results stored in the cache) incur additional overhead compared to just calling the service. By corollary, a **cache hit** (that is, a request that can be satisfied by results stored in the cache) are satisfied significantly faster using only the results stored in the cache.

The total time a request takes to process as a result of a **cache miss** is:

```
Request_Time = Cache_Check + Service_Call_Time + Cache_Write
```

In our multiplication service, let's make the following assumptions:

```
Cache_Check      = 1ms      # How long to get a value from the cache
Cache_Write      = 1ms      # How long to write a new value into the cache
Service_Call_Time = 25ms    # How long service takes to process multiplication request
```

Therefore, the **Request_Time** for our multiplication service, when we have a cache miss is:

```
Request_Time = 1ms + 25ms + 1ms
Request_Time = 27ms    # For a cache miss
```

Notice that this total time is greater than the time it takes for the service to process the request if there was no cache (**25ms**). The additional **2ms** is the **cache overhead**.

But requests that can be fulfilled by simply reading the cache take less effort, because they do not have to make any calls to the service. Put in cache terms, requests that **cache hit** take significantly less time by avoiding the service call time.

The total time a request takes to process as a result of a **cache hit** is:

```
Request_Time = Cache_Check
```

Therefore, the **Request_Time** for our multiplication service, when we have a cache hit is:

```
Request_Time = 1ms    # For a cache hit
```

So, some requests take significantly less time (**1ms** in our example), while other requests incur additional overhead (**2ms** in our example). Without a cache, all requests would take about the same amount of time (**25ms** in our example).

**In order for a cache to be effective, the overall time for all requests must be less than the overall time if the cache didn't exist.**

This means, essentially, that there needs to be more **cache hits** than **cache misses** overall. How many more depends on the amount of time spent processing the cache (the **cache overhead**) and the amount of time it takes to process a request to the service (**service call time**).

The greater the number of **cache hits** compared with the number of **cache misses**, the more effective the cache. Additionally, the greater the **service call time** compared with the **cache overhead**, the more effective the cache.

The **cache miss rate** is the percentage of requests that generate a cache miss. Conversely, the **cache hit rate** is the percentage of requests that generate a cache hit. Because each request must either be a cache hit or cache miss, that means:

```
Cache_Miss_Rate + Cache_Hit_Rate = 1 (100%)
```

When using our multiplication service without a cache, each request takes **25ms**. With a cache, the time is either **1ms** or **27ms**, depending on whether there was a cache hit or cache miss. In order for the cache to be effective, the 2ms overhead of accessing the cache during a cache miss must be offset by some number of cache hits. Put another way, the total request time without a cache must be greater than the total request time with a cache for the cache to be considered effective. Therefore, in order for the cache to be effective:

```
Request_Time_No_Cache >= Request_Time_With_Cache
```

```
Request_Time_With_Cache =
    ( Cache_Miss_Rate * Request_Time_Cache_Miss ) +
    ( Cache_Hit_Rate  * Request_Time_Cache_Hit )
```

And since:

```
Cache_Hit_Rate = 1 - Cache_Miss_Rate
```

You can rewrite this as:

```
Request_Time_With_Cache =
    ( Cache_Miss_Rate       * Request_Time_Cache_Miss ) +
    ( (1 - Cache_Miss_Rate) * Request_Time_Cache_Hit )
```

Therefore:

```
Request_Time_No_Cache >= Cache_Miss_Rate * Request_Time_Cache_Miss +
                         (1 - Cache_Miss_Rate) * Request_Time_Cache_Hit
```

For our multiplication example, that means:

```
25ms >= Cache_Miss_Rate * 27ms + (1 - Cache_Miss_Rate) * 1ms
25ms >= (Cache_Miss_Rate * 27ms) + 1ms - (Cache_Miss_Rate * 1ms)
25ms >= 1ms + Cache_Miss_Rate * 26ms
24ms >= Cache_Miss_Rate * 26ms
Cache_Miss_Rate <= 24/26
Cache_Miss_Rate <= 92.3%
```

Given that cache hit rate + cache miss rate = 1, we can do the same calculation using the **cache hit rate** rather than the cache miss rate:

```
25ms >= (1 - Cache_Hit_Rate) * 27ms + Cache_Hit_Rate * 1ms
25ms >= 27ms - Cache_Hit_Rate * 27ms + Cache_Hit_Rate * 1ms
25ms >= 27ms - Cache_Hit_Rate * 26ms
2ms  <= Cache_Hit_Rate * 26ms
Cache_Hit_Rate >= 2/26
Cache_Hit_Rate >= 7.7%
```

In other words, in this example, as long as a request can be satisfied by the cache (**cache hit rate**) at least **7.7%** of the time, then having the cache is more efficient than not having the cache.

Doing the math the other way, you could ask a different question. If the average request time is **25ms** without a cache, what would be the average request time if the cache hit rate was 25%? 50%? 75%? 90%?

For our multiplication service, we use this equation:

```
(1 - Cache_Hit_Rate) * 27ms + Cache_Hit_Rate * 1ms
```

**Cache Hit Rate = 25%:**

```
(1 - 0.25) * 27ms + 0.25 * 1ms
0.75 * 27 + 0.25 * 1
20.25 + 0.25
= 20.5ms
```

The average request time assuming a cache hit rate of 25% is **20.5ms**. Much faster than the 25ms for no cache!

**Cache Hit Rate = 50%:**

```
(1 - 0.5) * 27ms + 0.5 * 1ms
0.5 * 27 + 0.5 * 1
13.5 + 0.5
= 14ms
```

**Cache Hit Rate = 75%:**

```
(1 - 0.75) * 27ms + 0.75 * 1ms
0.25 * 27 + 0.75 * 1
6.75 + 0.75
= 7.5ms
```

**Cache Hit Rate = 90%:**

```
(1 - 0.9) * 27ms + 0.9 * 1ms
0.1 * 27 + 0.9 * 1
2.7 + 0.9
= 3.6ms
```

As the cache hit rate increases, the average request time improves dramatically. So if the cache hit rate increases from 25% to 90%, the average request time drops from 20.5ms to 3.6ms — **86% lower** than without a cache (3.6ms compared with 25ms)!

**In other words, the higher the cache hit rate, the more effective the cache.**

These calculations are all based on the amount of time it takes for the request to be processed by the multiplication service without the cache (25ms in our example). But this value is just an assumption. What happens if that value is larger, say **500ms**?

**Cache Hit Rate = 25%:**

```
(1 - 0.25) * 502ms + 0.25 * 1ms
0.75 * 502 + 0.25 * 1
376.5 + 0.25
= 376.75ms
```

**Cache Hit Rate = 50%:**

```
(1 - 0.5) * 502ms + 0.5 * 1ms
0.5 * 502 + 0.5 * 1
251 + 0.5
= 251.5ms
```

**Cache Hit Rate = 75%:**

```
(1 - 0.75) * 502ms + 0.75 * 1ms
0.25 * 502 + 0.75 * 1
125.5 + 0.75
= 126.25ms
```

**Cache Hit Rate = 90%:**

```
(1 - 0.9) * 502ms + 0.9 * 1ms
0.1 * 502 + 0.9 * 1
50.2 + 0.9
= 51.1ms
```

We can see that for a service that takes more resources and has a larger request time without a cache, the impact of the cache on the request time becomes much greater. In particular, at a cache hit rate of 90%, the average request time is **89.8% better** than without a cache (51.1ms compared with 500ms).

**In other words, the greater the cost of calling the un-cached service, the greater the effectiveness of the cache for a given cache hit rate.**

These calculations can and should be performed on each caching opportunity to determine whether or not — and to what extent — the application can effectively utilize a cache.
