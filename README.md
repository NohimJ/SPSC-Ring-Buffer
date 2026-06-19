# Lock-Free SPSC Ring Buffer

This project contains a single-producer, single-consumer (SPSC) lock-free ring buffer implemented in C++ using atomics with acquire/release memory ordering, benchmarked against a mutex-protected `std::queue` baseline.

## Why

In high-frequency trading and other low latency driven systems, passing data between threads (e.g. market data ingestion → strategy logic) needs to avoid blocking. A traditional mutex forces a waiting thread to be suspended and rescheduled by the OS, which introduces unpredictable latency. This project implements a queue where neither thread ever blocks. Operations either succeed immediately or report failure immediately, with no OS involvement.

## Design

- **Fixed-capacity ring buffer.** No resizing in the data structure, capacity is set once at construction.
- **Power-of-2 capacity, enforced at construction.** This allows index wraparound to be computed with a bitwise AND (`index & (capacity - 1)`) instead of the modulus operator, since modulus requires a division instruction that is significantly slower than AND on most CPUs. Capacity is validated with `(cap & (cap - 1)) != 0` which only equals 0 for powers of 2 and is less computationally expensive than modulus.
- **One producer thread, one consumer thread.** The producer exclusively owns `tail`; the consumer exclusively owns `head`. Each thread only ever writes to its own index, which avoids write contention by design. The producer checks the consumers index to see if queue is full, and likewise the consumer checks the producers index to check if the queue is empty.
- **Acquire/release memory ordering**
  - The producer writes the element, then stores the new `tail` with `memory_order_release`, guaranteeing the element written is visible before the index update is.
  - The consumer reads `tail` with `memory_order_acquire`, pairing with the producer's release and guaranteeing it sees the actual element data, not stale memory.
  - Reads of a thread's *own* index use `memory_order_relaxed`, since there's no cross-thread visibility concern there.
- **One slot is always left empty**  This is what allows the full and empty conditions to be satisfied without an extra counter. Full is "tail one step behind head"; empty is "head equals tail."
- **Non-copyable, non-movable.** A live queue with active producer/consumer threads should never be copied or moved, so the copy/move constructors and assignment operators are explicitly deleted.
- `push`/`pop` return `bool` rather than throwing or returning the element directly, full/empty are routine, expected states under normal operation, not exceptional ones, and exceptions are too costly to use so they are not used. `pop` writes its result through a `T&` reference parameter to avoid an unnecessary copy on return.

## Benchmark

Two threads (one producer, one consumer) push/pop 10,000,000 integers as fast as possible, compared against the same workload using a `std::queue` protected by a `std::mutex`.

**Compiled with `-O3`, 5 runs each:**

| Implementation         | Time (ms)         |
|-------------------------|-------------------|
| Lock-free ring buffer   | 98–104 (avg ~100) |
| Mutex-protected queue   | 144–161 (avg ~155)|

The lock-free queue is **~35–40% faster**, and notably more consistent, the mutex-based version shows a variance in latency which is expected since its performance depends on OS thread scheduling rather than just raw memory operations. In latency-sensitive systems, that predictability matters as much as the average.

## Build & Run

```
g++ -O3 -o buffer SPSC.cpp
./buffer
```

Output is two lines: time taken by the lock-free queue, then time taken by the mutex-based baseline (in ms).

## What I'd add next

- Cache-line padding (`alignas(64)`) between `head` and `tail` to eliminate false sharing
- Move-semantics overload for `push(T&&)` to avoid copying large element types
- A `size()`-based capacity utilization view, and basic unit tests for full/empty/wraparound edge cases
