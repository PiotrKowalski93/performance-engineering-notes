# Function Inlining

Function inlining is one of the most important compiler optimisations for performance. A normal function call incurs some overhead when called, particularly in the prologue and epilogue. The function prologue is code that prepares the stack and registers required for said function, while the epilogue performs the opposite by restoring the stack and registers to the state they were in before the function call.

Function inlining can help us minimise these overheads. When a function is inlined, we can think of the function's code being put directly in place of where the function call was originally written. If a function can be inlined, we will often observe major performance improvements.

## Objective

Compare sorting performance using C `qsort()` and C++ `std::sort()` with a custom comparator sorting structures by `key1` and `key2`.

## Implementations

**Baseline – `qsort()`**

```cpp
void solution_baseline(std::array<S, N>& arr) {
    qsort(arr.data(), arr.size(), sizeof(S), compare);
}
```
We notice we are making a call to qsort, which is a quicksort algorithm from C. The trouble here is that this function is in a compiled C library, which makes is nigh-on impossible for the compiler to "see" into it and make the optimisations required to inline it.

**Optimized – `std::sort()`**

```cpp
void solution(std::array<S, N>& arr) {
    std::sort(arr.begin(), arr.end(),
        [](const S& a, const S& b) {
            return a.key1 < b.key1 ||
                  (a.key1 == b.key1 && a.key2 < b.key2);
        });
}
```

## Benchmark results

| Implementation | Time |
|---|---:|
| `qsort()` | 657 µs |
| `std::sort()` | 428 µs |

## Verifying inlining in ASM

Look for:

```asm
call .....
```