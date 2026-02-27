# Indexes

Why should you, as an application developer, care how the database handles storage and retrieval internally? You're probably not going to implement your own storage engine from scratch, but you do need to select a storage engine that is appropriate for your application, from the many that are available. In order to tune a storage engine to perform well on your kind of workload, you need to have a rough idea of what the storage engine is doing under the hood.

Appending to a file is generally very efficient. Many databases internally use a **log**, which is an **append-only data file**. In order to efficiently find the value for a particular key in the database, we need a different data structure: an **index**. The general idea behind them is to keep some additional metadata on the side, which acts as a signpost and helps you to locate the data you want. If you want to search the same data in several different ways, you may need several different indexes on different parts of the data.

An index is an additional structure that is derived from the primary data. Many databases allow you to add and remove indexes, and this doesn't affect the contents of the database; it only affects the **performance of queries**. Maintaining additional structures incurs overhead, especially on writes. For writes, it's hard to beat the performance of simply appending to a file, because that's the simplest possible write operation. Any kind of index usually slows down writes, because the index also needs to be updated every time data is written.

**This is an important trade-off in storage systems: well-chosen indexes speed up read queries, but every index slows down writes.**

## Hash Indexes

Let's start with indexes for **key-value data**. This is not the only kind of data you can index, but it's very common.

Let's say our data storage consists only of appending to a file. Then the simplest possible indexing strategy is this: keep an **in-memory hash map** where every key is mapped to a **byte offset** in the data file — the location at which the value can be found. Whenever you append a new key-value pair to the file, you also update the hash map to reflect the offset of the data you just wrote (this works both for inserting new keys and for updating existing keys). When you want to look up a value, use the hash map to find the offset in the data file, seek to that location, and read the value.

```
  In-Memory Hash Map                     Data File (append-only)
 ┌──────────┬────────┐             ┌──────────────────────────────────┐
 │   Key    │ Offset │             │ Offset  Data                     │
 ├──────────┼────────┤             ├──────────────────────────────────┤
 │ "cat"    │  72    │────────┐    │  0:  cat=whiskers                │
 │ "dog"    │  24    │──────┐ │    │ 24:  dog=rex                     │
 │ "fish"   │  48    │────┐ │ │    │ 48:  fish=nemo                   │
 └──────────┴────────┘    │ │ │    │ 72:  cat=luna      (update)      │
                          │ │ └───►│ 96:  dog=buddy     (update)      │
                          │ └─────►│                                  │
                          └───────►│                                  │
                                   └──────────────────────────────────┘

  Lookup "fish":
  1. Hash map: "fish" -> offset 48
  2. Seek to byte 48 in data file
  3. Read value: "nemo"
```

We only ever append to a file — so how do we avoid eventually running out of disk space? A good solution is to break the log into **segments** of a certain size by closing a segment file when it reaches a certain size, and making subsequent writes to a new segment file. We can then perform **compaction** on these segments.

```
  Log file grows over time (append-only):

  ┌─────────────────────────────────────────────────────────────────────────┐
  │ cat=whiskers | dog=rex | fish=nemo | cat=luna | dog=buddy | bird=polly  │
  └─────────────────────────────────────────────────────────────────────────┘
                         log file getting too large!
                                    │
                                    ▼
  Break into segments (close at max size, new writes go to new segment):

  Segment 1 (closed)          Segment 2 (closed)         Segment 3 (active)
 ┌────────────────────┐      ┌────────────────────┐      ┌────────────────────┐
 │ cat=whiskers       │      │ cat=luna           │      │ horse=spirit       │
 │ dog=rex            │      │ dog=buddy          │      │ rabbit=thumper     │
 │ fish=nemo          │      │ bird=polly         │      │  ... new writes    │
 │  (max size reached)│      │  (max size reached)│      │  append here ───►  │
 └────────────────────┘      └────────────────────┘      └────────────────────┘
         ▲                            ▲                           ▲
    read-only now               read-only now              currently writing
```

```
  Before Compaction:

  Segment 1                    Segment 2
 ┌───────────────────┐        ┌───────────────────┐
 │ cat=whiskers      │        │ cat=luna           │
 │ dog=rex           │        │ dog=buddy          │
 │ fish=nemo         │        │ bird=polly         │
 │ cat=luna   (dup)  │        │                    │
 │ dog=buddy  (dup)  │        │                    │
 └───────────────────┘        └───────────────────┘

  After Compaction (merge segments, keep latest values):

  Compacted Segment
 ┌───────────────────┐
 │ cat=luna           │
 │ dog=buddy          │
 │ fish=nemo          │
 │ bird=polly         │
 └───────────────────┘
  (duplicates removed, only latest values kept)
```

Moreover, since compaction often makes segments much smaller (assuming that a key is overwritten several times on average within one segment), we can also **merge several segments together** at the same time as performing the compaction. Segments are never modified after they have been written, so the merged segment is written to a **new file**. The merging and compaction of frozen segments can be done in a **background thread**, and while it is going on, we can still continue to serve read and write requests as normal, using the old segment files. After the merging process is complete, we switch read requests to using the new merged segment instead of the old segments — and then the old segment files can simply be deleted.

```
  Compacted Segments (after individual compaction):

  Segment 1              Segment 2              Segment 3
 ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
 │ cat=luna     │       │ bird=polly   │       │ horse=spirit  │
 │ fish=nemo    │       │ snake=kaa    │       │ rabbit=thumper│
 └──────────────┘       └──────────────┘       └──────────────┘
         │                      │                      │
         └──────────────────────┼──────────────────────┘
                                │
                      merge (background thread)
                                │
                                ▼
                  ┌──────────────────────┐
                  │   New Merged Segment  │
                  ├──────────────────────┤
                  │ bird=polly           │
                  │ cat=luna             │
                  │ fish=nemo            │
                  │ horse=spirit         │
                  │ rabbit=thumper       │
                  │ snake=kaa            │
                  └──────────────────────┘
                                │
                                ▼
                  Segments 1, 2, 3 deleted
                  Reads now use merged segment
```

Each segment now has its own **in-memory hash table**, mapping keys to file offsets. In order to find the value for a key, we first check the **most recent segment's** hash map; if the key is not present we check the second-most-recent segment, and so on. The merging process keeps the number of segments small, so lookups don't need to check many hash maps.

```
  Lookup key "fish":

  Step 1: Check most recent segment's hash map first

  Segment 3 Hash Map         Segment 2 Hash Map         Segment 1 Hash Map
 ┌──────────┬────────┐      ┌──────────┬────────┐      ┌──────────┬────────┐
 │   Key    │ Offset │      │   Key    │ Offset │      │   Key    │ Offset │
 ├──────────┼────────┤      ├──────────┼────────┤      ├──────────┼────────┤
 │ horse    │   0    │      │ bird     │   0    │      │ cat      │   0    │
 │ rabbit   │  24    │      │ snake    │  24    │      │ fish     │  24    │
 └──────────┴────────┘      └──────────┴────────┘      └──────────┴────────┘
         │                          │                          │
         ▼                          ▼                          ▼
   "fish" not found           "fish" not found           "fish" FOUND!
   try next segment ──►      try next segment ──►       offset = 24
                                                               │
                                                               ▼
                                                        Segment 1 File
                                                       ┌──────────────┐
                                                       │ 0:  cat=luna │
                                                       │ 24: fish=nemo│◄── read
                                                       └──────────────┘
                                                               │
                                                               ▼
                                                        return "nemo"
```
