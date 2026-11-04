# Data Packing: Reducing Memory Footprint and Improving Cache Efficiency

## Overview

This experiment explores how the memory layout of a C++ structure affects its size and the performance of data-intensive operations.
The main objective is to reduce the memory footprint of a structure by minimizing padding, using more compact data types, and leveraging bit-fields where appropriate.
The experiment is based on the Data Packing lab from PerfNinja.

## 1. Baseline

```cpp
struct S {
    int i;
    long long l;
    short s;
    double d;
    bool b;

    bool operator<(const S& s) const {
        return this->i < s.i;
    }
};
```

### Memory layout

Compilers insert padding to satisfy the alignment requirements of individual members.
On a typical x86-64 ABI, the structure may have the following layout:

| Member | Size | Offset |
|---|---:|---:|
| `int i` | 4 B | 0 |
| Padding | 4 B | 4 |
| `long long l` | 8 B | 8 |
| `short s` | 2 B | 16 |
| Padding | 6 B | 18 |
| `double d` | 8 B | 24 |
| `bool b` | 1 B | 32 |
| Tail padding | 7 B | 33 |

Total size: **40 bytes**
The exact layout is implementation- and ABI-dependent.

### Why does padding exist?

Each type has an alignment requirement. The compiler may insert padding between members to ensure that each member is properly aligned.
The structure itself must also satisfy its alignment requirement, which can result in tail padding. Padding does not store application data, but it contributes to the memory footprint.

## 2. Optimized Structure

```cpp
struct S {
    unsigned l : 14;
    unsigned i : 7;
    unsigned s : 7;
    bool b : 1;
    float d;

    bool operator<(const S& s) const {
        return this->i < s.i;
    }
};
```

The optimized version uses bit-fields and a smaller floating-point type.

### Bit-fields

Bit-fields allow a member to occupy a specified number of bits instead of the full size of its underlying type.

For example:

```cpp
unsigned i : 7;
```

A 7-bit unsigned field can represent values in the range:

```text
0 - 127
```

This is sufficient if the application guarantees that the value never exceeds this range.

The optimized structure uses:

| Member | Bit width | Value range |
|---|---:|---:|
| `l` | 14 bits | 0 - 16,383 |
| `i` | 7 bits | 0 - 127 |
| `s` | 7 bits | 0 - 127 |
| `b` | 1 bit | 0 - 1 |

Total: 29 bits of bit-field data.

On the tested implementation, these fields occupy 4 bytes, followed by a 4-byte `float`.
Expected total size: **8 bytes**
The actual size should be verified using `sizeof` and `alignof`.

## 3. Benchmark Results

| Version | Runtime | Speedup |
|---|---:|---:|
| Baseline | 48.2 ms | 1.00x |
| Optimized | 8.06 ms | 5.98x |

The optimized implementation reduced the measured runtime by approximately 83.3%.

## 4. Why Can Data Packing Improve Performance?

### Reduced memory footprint

The optimized structure is significantly smaller. For a large array of structures, this reduces the amount of memory required to store the data.

### Improved cache efficiency

A smaller structure allows more elements to fit into a cache line.

Assuming a 64-byte cache line:

- 40-byte structures: approximately 1.6 structures per cache line
- 8-byte structures: 8 structures per cache line

This reduce the number of cache lines accessed during sequential traversal.

### Reduced memory bandwidth pressure

Smaller structures require less data to be transferred from memory when the workload accesses the entire structure.
This can improve performance in memory-bound workloads.

## 5. Trade-offs and Limitations

### Bit-fields are implementation-dependent

The allocation order of bit-fields, their alignment, and whether they cross allocation-unit boundaries are implementation-defined.
Bit-fields should not be used as a portable binary serialization format.

### Range restrictions

A bit-field must be wide enough to represent all valid values.
For example, a 7-bit unsigned field cannot represent values greater than 127.

### Access cost

Accessing or modifying bit-fields may require additional instructions, such as masking and shifting.
A smaller memory footprint does not necessarily mean faster access in every workload.

### Floating-point precision

Replacing `double` with `float` reduces storage from 8 bytes to 4 bytes, but also reduces precision.
This change is only valid if the application can tolerate the loss of precision.

### Concurrency considerations

Bit-fields are not independently addressable and cannot be used directly as atomic objects.
Concurrent modifications to different bit-fields within the same storage unit can introduce data races.

## 6. Key Takeaways

- Structure padding can significantly increase memory consumption
- Field ordering can reduce unnecessary padding
- Bit-fields are useful when the valid range of a value is known
- Smaller data structures can improve cache efficiency and reduce memory bandwidth requirements
- Data packing should be evaluated using benchmarks rather than assumptions
- Memory layout optimizations may introduce portability, precision, and concurrency trade-offs

## References

- PerfNinja: Data Packing
- [C++ Data Layout and Object Representation](https://en.cppreference.com/w/cpp/language/objects.html)
