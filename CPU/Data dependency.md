# Data Dependency

## Objective

Reduce data dependency chains and expose more instruction-level parallelism (ILP).

## Baseline

The baseline implementation uses a nested loop to search for matching values in two linked lists. Each iteration of the outer loop processes a single value from `l1`. The inner loop traverses `l2` until a match is found. The algorithm is dominated by serial linked-list traversal and dependent operations. Each pointer dereference depends on the previous one, limiting memory-level parallelism.

**Benchmark:**
- Time: 72.4 ms

## Optimization

Instead of processing one value at a time, I grouped multiple values from `l1` using `std::array`. For each group, the algorithm traverses `l2` and compares each node against multiple independent values. This exposes more independent comparisons and reduces the impact of the original serial dependency chain. The group size was tuned experimentally.

## Results

| Group size | Time |
|---:|---:|
| Baseline | 72.4 ms |
| 2 | 39.1 ms |
| 4 | 22.9 ms |
| 8 | 14.6 ms |
| **16** | **10.7 ms** |
| 32 | 36.7 ms |
| 64 | 30.4 ms |

**Best result: 16 elements per group**

- 6.8x faster than baseline
- ~85.2% reduction in execution time

## Key Takeaway

Data dependency limits how much independent work the CPU can execute concurrently. Grouping independent values allows the processor to overlap comparisons and improve instruction-level parallelism. However, increasing the group size indefinitely is not beneficial. Larger groups increase register pressure, code complexity, and potentially reduce performance.
