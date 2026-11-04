# Software Prefetching: Hiding Memory Latency

## Overview

This experiment explores how software prefetching can improve the performance of a memory-intensive workload by initiating memory accesses before the data is actually needed. The benchmark performs lookups in a custom hash map using a sequence of integer keys. The baseline performs each lookup sequentially. The optimized version prefetches the memory location associated with a future lookup, attempting to overlap memory latency with useful computation.

The experiment focuses on:

- Memory latency and cache misses
- Software prefetching
- Memory-level parallelism (MLP)
- Prefetch distance
- Cache pollution and other performance trade-offs

## 1. Background

### What is software prefetching?

Modern CPUs use several levels of cache to reduce the latency of accessing main memory. When a load misses in the cache hierarchy, the processor may need to retrieve the data from a lower-level cache or DRAM. Software prefetching allows a program to request that data be brought closer to the CPU before it is needed.

In GCC and Clang, the following intrinsic can be used:

```cpp
__builtin_prefetch(address, rw, locality);
```

Parameters:

| Parameter | Meaning |
|---|---|
| `address` | Address of the memory location to prefetch |
| `rw` | `0` for read, `1` for write |
| `locality` | Expected temporal locality, from `0` to `3` |

For example:

```cpp
__builtin_prefetch(ptr, 0, 3);
```

This requests a prefetch for reading, with high expected temporal locality. The intrinsic is a hint. It does not guarantee that the data will be fetched into a particular cache level or that the prefetch will complete before the subsequent access.

### Why can prefetching help?

Consider a dependent memory access:

```cpp
int value = hash_map->find(key);
```

The CPU must calculate the address before it can access the corresponding memory location. If the data is not in cache, the load may incur significant latency.
Software prefetching can initiate the memory access earlier:

```cpp
hash_map->prefetch_find(future_key);

// Perform useful work

int value = hash_map->find(key);
```

The goal is to overlap memory latency with other instructions.

This is particularly useful when:

- The access pattern is predictable
- The future memory address can be calculated in advance
- The workload has enough independent work to overlap memory latency
- Memory latency is a significant performance bottleneck

---

## 2. Baseline Implementation

```cpp
int solution_2(
    const hash_map_t* hash_map,
    const std::vector<int>& lookups
) {
    int result = 0;

    for (int val : lookups) {
        if (hash_map->find(val)) {
            result += getSumOfDigits(val);
        }
    }

    return result;
}
```

The baseline processes lookups sequentially.

Each iteration:

1. Reads the next key
2. Calculates the corresponding bucket
3. Accesses the hash map
4. Performs additional computation if the lookup succeeds

The address of the next bucket is not requested until the next iteration. If a lookup causes a cache miss, the CPU may have limited independent work available to hide the memory latency.

---

## 3. Optimized Implementation

```cpp
static constexpr auto prefetch_step = 16;

int solution(
    const hash_map_t* hash_map,
    const std::vector<int>& lookups
) {
    int result = 0;
    const auto size = lookups.size();

    if (size <= prefetch_step) {
        for (int val : lookups) {
            if (hash_map->find(val)) {
                result += getSumOfDigits(val);
            }
        }

        return result;
    }

    for (auto i = 0; i + prefetch_step < size; i++) {
        if (const int val = lookups[i]; hash_map->find(val)) {
            result += getSumOfDigits(val);
        }

        hash_map->prefetch_find(
            lookups[i + prefetch_step]
        );
    }

    for (auto i = size - prefetch_step; i < size; i++) {
        if (const int val = lookups[i]; hash_map->find(val)) {
            result += getSumOfDigits(val);
        }
    }

    return result;
}
```

### Prefetch distance

The prefetch distance is set to:

```cpp
static constexpr auto prefetch_step = 16;
```

The algorithm processes the current lookup while prefetching a lookup that is 16 elements ahead.

For example:

```text
Iteration 0:
    Process lookup[0]
    Prefetch lookup[16]

Iteration 1:
    Process lookup[1]
    Prefetch lookup[17]

Iteration 2:
    Process lookup[2]
    Prefetch lookup[18]

...

Iteration 16:
    Process lookup[16]
    Prefetch lookup[32]
```

The intention is to give the memory subsystem enough time to fetch the data before the corresponding lookup is executed. The optimal prefetch distance depends on the workload and the target CPU. A distance that works well for one workload may perform poorly for another.

### Hash map prefetching

```cpp
void prefetch_find(int val) const {
    int bucket = val % N_Buckets;

    __builtin_prefetch(
        &m_vector[bucket],
        0,
        3
    );
}
```

The future key is known, so the bucket index can be calculated in advance. The prefetch requests the memory location of that bucket. This is an example of software prefetching for an indirect memory access pattern.

---

## 4. Benchmark Results

### Baseline

Google Benchmark:

| Benchmark | Time | CPU | Iterations |
|---|---:|---:|---:|
| bench1 | 52.3 ms | 52.3 ms | 13 |

Performance counters:

```text
34,664,970,612  cycles
11,601,460,994  instructions

24,882,149      l2_rqsts.demand_data_rd_hit
134,075,131     l2_rqsts.demand_data_rd_miss

682,119,510     mem_load_retired.l1_hit
71,367,273      mem_load_retired.l1_miss

692,985         mem_load_retired.l2_hit
70,536,771      mem_load_retired.l2_miss

5,988,049       mem_load_retired.l3_hit
64,626,943      mem_load_retired.l3_miss
```

### Optimized — Software Prefetching

Google Benchmark:

| Benchmark | Time | CPU | Iterations |
|---|---:|---:|---:|
| bench1 | 26.5 ms | 26.5 ms | 28 |

Performance counters:

```text
9,515,787,601   cycles
19,261,417,443  instructions

42,970,061      l2_rqsts.demand_data_rd_hit
190,538,501     l2_rqsts.demand_data_rd_miss

1,143,773,326   mem_load_retired.l1_hit
91,881,709      mem_load_retired.l1_miss

1,036,623       mem_load_retired.l2_hit
91,055,503      mem_load_retired.l2_miss

7,770,414       mem_load_retired.l3_hit
83,382,844      mem_load_retired.l3_miss
```

### Results comparison

| Metric | Baseline | Prefetching | Change |
|---|---:|---:|---:|
| Runtime | 52.3 ms | 26.5 ms | -49.3% |
| Speedup | 1.00x | 1.97x | +97.4% |

The optimized implementation reduced the measured runtime by approximately 49.3%, achieving nearly 2x speedup. The results suggest that software prefetching was beneficial for this particular workload. However, the performance counters should not be interpreted as a direct comparison of cache-miss rates without checking the measurement methodology. The raw counter values are not all lower in the optimized version. In particular, the reported L2 and L3 miss counts are higher.

---

## 5. Why Did Prefetching Improve Performance?

### Overlapping memory latency

The optimized version initiates future memory accesses while processing the current lookup. This can allow the CPU to perform useful work while waiting for data to arrive.
Instead of waiting for every lookup to complete before starting the next one, the processor can have multiple memory requests in flight. This increases memory-level parallelism (MLP).

### Increasing memory-level parallelism

Memory-level parallelism refers to the number of independent memory operations that can be outstanding simultaneously. Without prefetching, a workload with dependent or serialized memory accesses may not fully utilize the available memory subsystem. Software prefetching can increase the number of outstanding memory requests. This can improve throughput even if the latency of an individual memory access does not change.

### Prefetch distance

The prefetch distance determines how far ahead the memory request is initiated.

If the distance is too small:

- The data may not arrive in time
- The processor may still stall on the actual load

If the distance is too large:

- The prefetched data may arrive too early
- It may be evicted before it is used
- Additional cache and memory bandwidth may be consumed

---

## 6. When Can Prefetching Hurt Performance?

Software prefetching is not universally beneficial. It can introduce additional overhead and compete with useful memory accesses.

### 1. Cache pollution

Prefetched data occupies cache space. If the prefetched data is not used soon enough, it may evict useful data from the cache. This can increase the number of cache misses for other operations.

### 2. Excessive memory bandwidth consumption

Prefetching can generate additional memory requests. If the workload is already bandwidth-bound, prefetching may compete with demand loads for memory bandwidth. The result may be slower execution rather than improved performance.

### 3. Incorrect prefetch distance

A prefetch that arrives too late provides little benefit. A prefetch that arrives too early may be evicted before use. The distance must be tuned for the workload and hardware.

### 4. Unpredictable access patterns

Prefetching is most useful when future addresses are known or can be predicted. If the access pattern is highly unpredictable, software prefetching may generate requests for data that is never used.

### 5. Hardware prefetchers

Modern CPUs already include hardware prefetchers. For sequential or regular memory access patterns, hardware prefetching may already be effective. Software prefetching can duplicate work or interfere with hardware prefetching.

### 6. Additional instruction overhead

Software prefetching requires additional instructions to calculate future addresses and issue prefetch requests. For workloads that are already compute-bound, this overhead may outweigh any benefit.

### 7. Low cache-miss cost

If the data is already in L1 or L2 cache, prefetching may provide little or no benefit. The additional instructions may actually make the code slower.

### 8. Multithreaded workloads

In a multithreaded application, aggressive prefetching can increase pressure on shared cache and memory resources. One thread may prefetch data that evicts useful data belonging to another thread. This can make performance less predictable.

---

## 7. Key Takeaways

- Software prefetching attempts to hide memory latency by initiating memory accesses before the data is needed
- Prefetching can increase memory-level parallelism
- The effectiveness of prefetching depends on the access pattern and the CPU's memory subsystem
- Prefetch distance is an important tuning parameter
- Prefetching can hurt performance through cache pollution, additional memory traffic, and instruction overhead
- Hardware prefetchers may already handle regular access patterns effectively