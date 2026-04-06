# HMF-MAC — Hybrid Miss-Frequency Adaptive Multi-Level Adaptive Cache

> **Real-Time Adaptive Cache Architecture simulated using gem5**  
> A C++ implementation of a smart, access-pattern-aware cache replacement policy designed to outperform static policies (LRU, Random) across diverse memory workloads.

---

## Overview

HMF-MAC introduces an **adaptive cache replacement policy** that monitors runtime memory access patterns and dynamically switches strategies to minimize cache misses. Built as an extension to the gem5 full-system simulator, the architecture targets the performance gap left by static replacement policies when faced with mixed or unpredictable workloads.

The key insight: no single replacement policy is optimal across all access patterns. Sequential workloads favor streaming/bypass strategies, strided access benefits from prefetch-aware policies, and random access performs best under frequency-based eviction. HMF-MAC detects which regime is active and adapts accordingly — in real time.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    gem5 Cache Hierarchy                  │
│                                                         │
│   ┌───────────┐     ┌───────────────────────────────┐   │
│   │  Cache    │────▶│     Adaptive Policy Engine     │   │
│   │ (cache.hh)│     │    (adaptive_policy.hh)        │   │
│   └───────────┘     │                               │   │
│                     │  ┌──────────┐  ┌───────────┐  │   │
│   ┌───────────┐     │  │ Access   │  │  Policy   │  │   │
│   │  Base     │────▶│  │ Pattern  │─▶│ Selector  │  │   │
│   │ (base.hh) │     │  │ Monitor  │  │  (LRU /   │  │   │
│   └───────────┘     │  └──────────┘  │  Freq /   │  │   │
│                     │               │  Random)  │  │   │
│   ┌─────────────┐   └───────────────────────────┘  │   │
│   │ Noncoherent │                                   │   │
│   │   Cache     │   ┌───────────────────────────┐   │   │
│   └─────────────┘   │   Set-Associative Base    │   │   │
│                     │   (base_set_assoc.hh)     │   │   │
│                     └───────────────────────────┘   │   │
└─────────────────────────────────────────────────────────┘
```

---

## Key Components

| File | Role |
|---|---|
| `adaptive_policy.hh` | Core adaptive replacement policy — monitors access history and selects eviction strategy dynamically |
| `base.hh` / `base.cc` | Abstract cache base class; defines the interface for all cache types |
| `base_set_assoc.hh` | Set-associative cache structure used as the hardware foundation |
| `cache.hh` / `cache.cc` | Main coherent cache implementation (L1/L2 compatible) |
| `noncoherent_cache.hh/.cc` | Non-coherent cache variant (suitable for scratchpad / private L1 use) |
| `show_stats.sh` | Shell script to extract and display gem5 stats output |

### Benchmark Workloads

| File | Access Pattern | Purpose |
|---|---|---|
| `sequential.c` | Sequential | Simulates linear array traversal — favors streaming policies |
| `random.c` | Random | Simulates pointer-chasing / graph workloads — stresses LRU |
| `strided16.c` | Stride-16 | Cache-friendly strided access |
| `strided1024.c` | Stride-1024 | Cache-hostile strided access (high conflict miss rate) |

---

## Getting Started

### Prerequisites

- [gem5](https://www.gem5.org/) (v22.0 or later recommended)
- GCC / G++ with C++17 support
- Python 3.8+ (for gem5 scripting)
- SCons build system

### Integration with gem5

1. Clone this repository:
   ```bash
   git clone https://github.com/CosmicVishnu/HMF-MAC.git
   cd HMF-MAC
   ```

2. Copy source files into your gem5 cache replacement policy directory:
   ```bash
   cp src/*.hh src/*.cc $GEM5_ROOT/src/mem/cache/replacement_policies/
   ```

3. Register the new policy in gem5's SConscript, then rebuild:
   ```bash
   cd $GEM5_ROOT
   scons build/X86/gem5.opt -j$(nproc)
   ```

4. Compile benchmark workloads:
   ```bash
   gcc -O2 -o sequential sequential.c
   gcc -O2 -o random random.c
   gcc -O2 -o strided16 strided16.c
   gcc -O2 -o strided1024 strided1024.c
   ```

5. Run a simulation (example):
   ```bash
   $GEM5_ROOT/build/X86/gem5.opt configs/se.py \
     --cpu-type=TimingSimpleCPU \
     --caches --l2cache \
     --l1d-repl=AdaptivePolicy \
     -c ./sequential
   ```

6. View results:
   ```bash
   bash show_stats.sh
   ```

---

## Results

Performance measured as **cache hit rate (%)** across access patterns compared to baseline LRU:

| Workload | LRU Hit Rate | Random Hit Rate | **HMF-MAC Hit Rate** |
|---|---|---|---|
| Sequential | ~72% | ~68% | **~85%** |
| Random | ~41% | ~39% | **~47%** |
| Stride-16 | ~78% | ~73% | **~82%** |
| Stride-1024 | ~31% | ~29% | **~38%** |

> Results are simulation-dependent and vary by cache size, associativity, and workload parameters. The adaptive policy shows strongest gains on mixed or stride-heavy workloads.

---

## Design Decisions

**Why adaptive?** Traditional replacement policies are tuned for average-case behavior. Real workloads — especially in HPC, databases, and ML inference — exhibit phase behavior where access patterns shift over time. HMF-MAC amortizes this by continuously re-evaluating policy fitness.

**Why gem5?** gem5 provides cycle-accurate microarchitectural simulation, making it ideal for evaluating cache policies without needing physical hardware. It supports full-system and syscall-emulation modes, and its modular design allows clean integration of custom replacement policies.

**Noncoherent cache variant** is provided for systems where cache coherence is managed externally (e.g., accelerator-attached scratchpads), enabling use of the adaptive policy without coherence overhead.

---

## Future Work

- Implement MESI protocol support for multi-core coherent simulation
- Add hardware performance counter integration for real-chip deployment
- Extend adaptive logic with ML-based workload classification
- Benchmark against RRIP and SHIP replacement policies
- Add Python-based stats visualization via `matplotlib`

---

## Tech Stack

![C++](https://img.shields.io/badge/C++-17-blue?logo=cplusplus)
![gem5](https://img.shields.io/badge/gem5-Simulator-orange)
![Shell](https://img.shields.io/badge/Shell-Bash-green?logo=gnu-bash)
![C](https://img.shields.io/badge/C-Benchmarks-lightgrey?logo=c)

---

## License

This project is open-source. See [LICENSE](LICENSE) for details.

---

## Acknowledgements

Built as part of computer architecture coursework exploring adaptive memory hierarchy optimization. Simulated using the gem5 architectural simulator.
