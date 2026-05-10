# zk-gpu-prover

> GPU-accelerated FRI commit prover in C++/CUDA. End-to-end runtime engineering — kernel offload, async execution, memory pooling, profiling. Built on top of ICICLE primitives, with a from-scratch C++ prover scaffold.

![Status](https://img.shields.io/badge/status-in_progress-orange)
![Language](https://img.shields.io/badge/C%2B%2B-17-blue)
![CUDA](https://img.shields.io/badge/CUDA-12.x-green)
![Build](https://img.shields.io/badge/build-CMake-orange)
![License](https://img.shields.io/badge/license-MIT-blue)

---

## Goal

**Achieve ~6× end-to-end FRI commit prover speedup on a single RTX 4090 within 8 weeks**, through three layers of optimization:

1. **GPU NTT** for the FRI commit phase (largest single bottleneck) — built on ICICLE primitives
2. **Memory-hierarchy optimization** — pinned host memory, CUDA streams, device memory pool, batched transfers
3. **GPU Poseidon + Merkle commitment** (secondary bottleneck)

Target hardware: NVIDIA RTX 4090 (Ada Lovelace), CUDA 12.x. Reference baseline: pure-C++ single-threaded CPU prover, implemented on the same field arithmetic for fair comparison.

## Why

ZK proving is computationally expensive — orders of magnitude more than verification. Modern STARK provers (Plonky3, SP1, Polygon Zero) hit a wall at large circuit sizes (`2^20` and beyond) on CPU. This project applies *systems-level* GPU acceleration techniques to the FRI commit phase, the dominant cost in STARK proving.

The optimization techniques here — memory-hierarchy tiling, async execution overlap, bank-conflict avoidance, batched primitives — are the same primitives that drive modern AI accelerator runtime stacks. NTT butterfly tiling maps directly to GEMM tiling; FRI fold streaming maps to attention KV streaming. The skills demonstrated transfer cleanly to NPU/GPU runtime, kernel, and compiler engineering.

**Why C++ (and not Rust):** ICICLE's core is C++/CUDA, and most production NPU/AI accelerator runtime stacks (Rebellions, FuriosaAI, Moreh, NVIDIA, etc.) are C++. Working in C++/CUDA throughout maintains language consistency with the author's prior systems work (LevelDB-ZNS, also C++17) and keeps the focus on kernel/runtime engineering rather than language onboarding.

## Architecture

```
Host (CPU, C++)
 ├── FRI Commit Prover (this project)
 │    ├── Domain construction
 │    ├── DFT / NTT                    ──┐
 │    ├── Polynomial fold                │
 │    ├── Merkle commit (Poseidon)  ──┐  │
 │    └── Query phase / openings      │  │
 │                                    │  │
 └── ICICLE C/C++ API  ───────────────┴──┴── GPU
                                              ├── NTT / iNTT kernels
                                              ├── Poseidon hash kernels
                                              ├── Merkle tree builder
                                              └── Memory pool / streams (custom)
```

Detailed diagrams: [`docs/architecture.md`](./docs/architecture.md) (TBD).

## Plan (8 Weeks)

### Phase A — FRI GPU Prover (Week 1–6)

| Week | Focus | Key Deliverable |
|---|---|---|
| 1 | C++/CUDA + ICICLE setup, NTT baseline | ICICLE NTT runs locally; CPU vs GPU NTT timing across sizes (`2^14` … `2^20`) |
| 2 | Field arithmetic + Merkle integration | BabyBear field, Poseidon-based Merkle tree on GPU validated |
| 3 | FRI commit phase scaffold | End-to-end FRI commit running; correctness validated against CPU reference |
| 4 | Memory-hierarchy optimization | Pinned host memory, CUDA streams, device memory pool; +1.5–2× |
| 5 | Batched ops + kernel fusion | Batched NTT, Poseidon-Merkle fused build |
| 6 | Comprehensive benchmark + roofline | Scaling curves, per-stage breakdown, roofline analysis |

### Phase B — FlashAttention Mini (Week 7)

A focused 7-day exercise demonstrating kernel-engineering skill transfer from ZK to ML. Same toolchain (C++/CUDA), so language switching cost is zero.

| Day | Task |
|---|---|
| 1–2 | Read FlashAttention v2 paper; study reference impl |
| 3–4 | Naive CUDA forward pass (FP16, head_dim = 64) |
| 5 | Tensor Core integration via WMMA API |
| 6 | Benchmark vs PyTorch SDPA / xFormers across sequence lengths (512 … 8192) |
| 7 | Writeup + "skills bridge" section linking FRI techniques to attention kernels |

### Phase C — Launch (Week 8)

- Polished English README (this file, finalized with results)
- Technical blog post (~2,500 words)
- Demo GIF / video
- Outreach: Korean AI accelerator startups (Rebellions, FuriosaAI, Moreh, HyperAccel) + Korean blockchain infra (KAIA, Lambda256, DSRV) + global ZK infra (Ingonyama, Succinct, Cysic)

## Tech Stack

- **Language:** C++17, CUDA 12.x
- **GPU primitives:** [ICICLE-3](https://github.com/ingonyama-zk/icicle) — Ingonyama's C/C++/CUDA library for ZK arithmetic (NTT, MSM, Poseidon)
- **Field:** BabyBear (`p = 2^31 − 2^27 + 1`) — chosen for compatibility with Plonky3 published numbers and first-class ICICLE support
- **Profiling:** Nsight Systems, Nsight Compute, `perf`, custom timing harness
- **Build:** CMake 3.20+
- **CI:** GitHub Actions (build, unit tests, fmt/clang-tidy)

## Repository Structure (planned)

```
.
├── README.md                    # This file
├── devlog.md                    # Daily progress log
├── CMakeLists.txt
├── docs/
│   ├── profile-report.md        # Bottleneck analysis (Week 2)
│   ├── architecture.md          # System diagrams
│   ├── benchmarks.md            # Detailed numbers
│   └── skills-bridge.md         # ZK ↔ ML kernel mapping
├── src/
│   ├── field/
│   │   └── babybear.cuh         # BabyBear field arithmetic
│   ├── ntt/
│   │   ├── gpu_ntt.cuh
│   │   └── gpu_ntt.cu           # GPU NTT (via ICICLE)
│   ├── merkle/
│   │   ├── poseidon.cu          # Poseidon hash kernel
│   │   └── merkle_tree.cu       # Merkle tree builder
│   ├── fri/
│   │   ├── prover.cpp           # FRI commit phase scaffold
│   │   └── commit_phase.cu
│   ├── runtime/
│   │   ├── memory_pool.{h,cpp}  # Device memory pool
│   │   └── streams.{h,cpp}      # CUDA stream management
│   └── cpu_reference/
│       └── fri_cpu.cpp          # Single-threaded CPU baseline
├── flash_attention/             # Phase B
│   ├── kernels/
│   │   └── flash_attn_v2.cu
│   └── benches/
├── benches/                     # Performance benchmarks
├── tests/                       # Correctness tests
└── plots/                       # Result figures
```

## Results

*To be filled in as project progresses. See [`devlog.md`](./devlog.md) for daily updates.*

| Stage | Wall-clock (ms) | Speedup vs CPU | Notes |
|---|---|---|---|
| CPU baseline (`2^20`) | TBD | 1.00× | Single-threaded C++, BabyBear field |
| + GPU NTT (ICICLE) | TBD | TBD | Week 3 |
| + Pinned memory + streams | TBD | TBD | Week 4 |
| + GPU Poseidon-Merkle | TBD | TBD | Week 5 |
| + Batched ops | TBD | TBD | Week 6 |

## Background & Prior Work

This project builds on the author's prior research:

- **ZK-STARK FRI Protocol: GPU/CPU Acceleration and Memory-Hierarchy Bottleneck Analysis** (2025) — preliminary investigation of where FRI prover time is spent, focused on NTT and Merkle commit bandwidth limits.
- **GPU Architecture Analysis for ZKML Acceleration** (2025) — multi-layer optimization of matrix multiplication for proof generation, including bank-conflict measurement, Tensor Core integration, and Montgomery multiplication for finite-field arithmetic.

These reports, written for SKKU Software 수준평가, established the theoretical groundwork that this implementation operationalizes.

## References

1. ICICLE. *Ingonyama.* https://github.com/ingonyama-zk/icicle
2. Ben-Sasson, E., Bentov, I., Horesh, Y., & Riabzev, M. *Scalable, transparent, and post-quantum secure computational integrity.* IACR ePrint 2018/046.
3. Plonky3. *Polygon Labs.* https://github.com/Plonky3/Plonky3 (reference for field choice & published numbers)
4. Dao, T. *FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning.* arXiv:2307.08691, 2023.
5. Jia, Z., Maggioni, M., Staiger, B., & Scarpazza, D. P. *Dissecting the NVIDIA Volta GPU Architecture via Microbenchmarking.* arXiv:1804.06826, 2018.
6. Grassi, L., et al. *Poseidon: A New Hash Function for Zero-Knowledge Proof Systems.* USENIX Security 2021.

## Skills Demonstrated

For recruiters and hiring managers:

| Skill area | Demonstrated through |
|---|---|
| GPU kernel engineering (CUDA C++) | Custom CUDA NTT integration, Poseidon kernels, WMMA / Tensor Core where applicable |
| Memory-hierarchy reasoning | Pinned host memory, on-chip tiling, batched device transfers, device memory pool |
| Runtime / async execution | CUDA streams, event-based scheduling, dependency management between compute and copy |
| Performance profiling | Nsight Systems / Compute traces, roofline analysis, per-stage breakdown |
| Cross-domain skill transfer | ZK prover acceleration techniques explicitly mapped to ML kernel design (FlashAttention) |
| End-to-end ownership | Theory → profile → implementation → benchmark → writeup |
| Systems software | CMake build, C library FFI (ICICLE), CI, reproducible benchmarks |

Direct relevance to:
- **NPU / AI accelerator runtime + kernel teams** (Rebellions, FuriosaAI, Moreh, HyperAccel, NAVER Cloud GPU infra, NVIDIA Korea)
- **ZK infrastructure** (Ingonyama, Succinct, Cysic, Polygon Zero, Risc Zero)
- **Korean blockchain infrastructure** (KAIA, Lambda256, DSRV, A41)

## Author

**Taeju Kim** (김태주) — SKKU Software, B.S. expected July 2026

Prior systems work: [LevelDB-ZNS](../leveldb-zns) — Zoned Namespace SSD-optimized LSM-Tree storage engine for blockchain workloads; achieved Device WAF of 1.001 and +45.7% throughput on overwrite workloads through hardware-aware data placement and event-driven garbage collection. Same C++17 stack.

## License

MIT
