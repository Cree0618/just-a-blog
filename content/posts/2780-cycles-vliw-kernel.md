---
date: 2026-08-19
title: "Scheduling a gather-bound kernel on a VLIW+SIMD machine"
description: "VLIW packing, software pipelining, and a 31-node scratch cache on Anthropic’s take-home. Measured 147,734 → 2,780 cycles. Still slower than every published Claude bar."
tags:
  - performance
  - VLIW
  - compilers
  - Anthropic
---

Very long instruction word (VLIW) machines date back to Fisher (1983): instead of hardware discovering instruction-level parallelism at runtime, the compiler packs independent operations into one bundle that issues in a single cycle. SIMD is the complementary idea that one packed op should also span several data lanes. The two together are a compiler problem more than a programming problem. The schedule *is* the program.

Anthropic’s original performance take-home is a small instance of that problem. The ISA, the simulator, the hash, and the published bars are Anthropic’s (copyright Anthropic PBC). This post is about the kernel I scheduled on top of that machine: `KernelBuilder.build_kernel` in `perf_takehome.py`. It is not a claim that I designed VLIW, and it is not a full solution dump — the starter asks people not to republish one.

The workload is a batched walk down an implicit binary tree. Each of 256 elements starts at index 0. For 16 rounds it loads a node value, mixes `val ^ node_val` through a six-stage 32-bit hash, then steps left or right according to hash parity:

```
node_val = forest[idx]
val = myhash(val ^ node_val)
idx = 2 * idx + (1 if val % 2 == 0 else 2)
idx = 0 if idx >= n_nodes else idx
```

Score is the cycle counter on a frozen copy of the simulator, standard case `forest_height=10` (2,047 nodes), `rounds=16`, `batch_size=256`.

This post focuses on the scheduling patterns that moved that number, and on the bound that did not move. Related work on out-of-order cores, auto-vectorization, and GPU kernel autotuning is in the same family and is not the subject here.

## The machine

One cycle can fill 12 scalar ALU slots, 6 vector ALU slots (`VLEN = 8`), 2 loads, 2 stores, and 1 flow.

{{< responsive-image src="images/vliw-cycle-bundle.png" alt="One VLIW cycle with ALU, VALU, load, store, and flow slots" maxWidth="720px" >}}

*One cycle of the simulated machine. Filled slots are occupied; gold marks the load bottleneck. Writes become visible at the end of the cycle.*


Three rules decide what a schedule is allowed to look like.

1. **Write visibility.** Results appear at the end of the cycle. A value cannot be consumed in the bundle that produces it. Same-cycle RAW is illegal.
2. **Scratch is the register file.** 1,536 words, no indirect indexing. A “cache lookup by idx” is a mux, not a load.
3. **`vload` / `vstore` are contiguous.** A gather of eight lanes is eight scalar `load_offset` ops.

The last rule is the bottleneck. 256 elements × 16 rounds is 4,096 node values. Two scalar loads per cycle means **2,048 cycles** if every node comes from memory and the load unit never idles. SIMD does not raise that cap. I mention the 2,048-cycle number explicitly because it is the ISA, not an estimate, and because several later ideas (deeper muxes, fancier packers) cannot beat it without deleting loads.

## Scheduling patterns

Compared with “emit one instruction, then the next,” a VLIW kernel is closer to runtime design: which engine is busy, which value is live, and which work can overlap a stall. The patterns below are the ones that actually appear in the running file.

### Pattern 1: Reduce memory traffic before packing it

The scalar starter reloads `idx` and `val`, gathers one node, hashes on the scalar ALU, and stores both back, every element, every round. `build()` emits one slot per bundle. Re-run 2026-08-19:

```
python3 -c "from origianperf_takehome import do_kernel_test; do_kernel_test(10,16,256)"
# CYCLES:  147734
```

The first structural change is not a better packer. It is to treat `idx`/`val` as 32 vectors of length 8, `vload` them once, run all 16 rounds in scratch, and `vstore` once. `learning_trace_optimization.md` records that step as **13,903 → 12,463**. I did not reconstruct those two snapshots; the pair is from the notes. The same notes give utilization for *that* trace (load ~16.9%, VALU ~17.1%). Those percentages do not describe the 2,780-cycle program.

After load-once, almost every remaining memory op is a gather. The inputs are contiguous. The tree walk is not.

### Pattern 2: Exploit a shrinking reachable set

Because every element starts at index 0, the reachable node set is tiny at the top: 1 node in round 0, 2 in round 1, 16 in round 4, then it explodes. Early rounds can be answered from scratch. Later rounds cannot.

I called this an “entropy horizon” in `optimization_architecture.md`: cache rounds 0–4 (31 nodes), saturate the load unit for rounds 5–15, estimate **~1,498 cycles**. That number is an estimate. I have not run a kernel that measured 1,498. The note also says that approaching the best published harness (1,363) would likely need the cache in round 5 or duplicate-index detection.

The running kernel is less ambitious than the note. It `vload`s 31 nodes, then special-cases **rounds 0 and 1** only. Round 0 is a broadcast of node 0. Round 1 is a two-way `vselect`. Nodes 3–30 sit in scratch and are never read.

```python
if round == 0:
    body.append(("valu", ("vbroadcast", node_vecs[vi], tree_cache[0])))
elif round == 1:
    body.append(("valu", ("==", t3, idx_vecs[vi], self.vector_const(1))))
    body.append(("valu", ("vbroadcast", t1, tree_cache[1])))
    body.append(("valu", ("vbroadcast", t2, tree_cache[2])))
    body.append(("flow", ("vselect", node_vecs[vi], t3, t1, t2)))
```

The key design choice is to make the cache cheaper than a gather only while the candidate set is small. A 16-way mux at round 4 is legal and slow. `updated_perf_takehome.py` follows the 0–4 plan more closely and measures **4,268 cycles** — worse than ignoring rounds 2–4 and overlapping gathers.

### Pattern 3: Rewrite the hash into the ISA

`HASH_STAGES` is six triples. Three of them are `a + C + (a << k)`, which is `a * (1 + 2^k) + C`, which is one VALU `multiply_add`. The other three stay as two independent ops plus a combining op that waits a cycle. The index update `idx = 2 * idx + {1, 2}` is the same instruction after parity becomes an addend.

```python
if op1 == "+" and op2 == "+" and op3 == "<<":
    c_mult = self.vector_const((1 << val3) + 1)
    c_add = self.vector_const(val1)
    body.append(("valu", ("multiply_add", val_vecs[vi], val_vecs[vi], c_mult, c_add)))
```

`multiply_add` is not a hash instruction. Three stages of `myhash` happen to be one fused multiply-add each. Algebra on the ISA is real work; naming it “an intrinsic” would overclaim.

### Pattern 4: Pack two queues, pipeline across batches

A linear `schedule()` that flushes on hazards is enough to turn “a list of ops” into bundles. The gather path needs more: it has to reach forward over RAW bubbles and pull a load into the same cycle as an unrelated VALU op. `_pack_dual_lookahead` does that for a load queue and a VALU queue. The gather loop uses a mixed packer with `lookahead=32`.

Software pipelining is then a queue of vector batches of 8. While the current batch emits `load_offset`, previously enqueued batches advance through hash stages. The current batch is appended **after** its eight gather offsets, so the hash never XORs a node vector with empty lanes.

```python
# Enqueue current batch after gathers complete to preserve correctness.
active_batches.append((vi_batch, 0))
self.instrs.extend(self._pack_mixed_lookahead(mixed_ops, lookahead=32))
```

The key design choice is correctness-before-overlap. Hashing a gather that has not finished is a silent wrong answer. `do_kernel_test` compares memory against `reference_kernel2` at each pause; a packer bug is an assert, not a better score. `compute_burst = 1` is what the tree runs, not a claim that 1 is optimal. Commit `55d5723` tried a more aggressive interleave; both versions measure 2,780.

{{< responsive-image src="images/vliw-entropy-horizon.png" alt="Entropy horizon: early rounds from scratch, later rounds as gathers" maxWidth="720px" >}}

*Gold is the mux from scratch. Blue is the memory pipeline. The running 2,780-cycle kernel only muxes rounds 0–1.*


## Case study: what 2,780 is made of

After `build_kernel` for the standard test, the file produces **2,780 bundles**, which matches the cycle counter:

| Quantity | Count | Meaning |
|---|---:|---|
| `load_offset` ops | 3,584 | 256 elements × 14 uncached rounds |
| Bundles with two loads | 1,792 | every gather dual-issued |
| Bundles with one load | 126 | leftover |
| Bundles with zero loads | 862 | cannot be recovered later |
| Bundles with load and VALU | 1,093 | overlap |
| Scratch used | 1,364 / 1,536 | deep windows hit this first |

The gather traffic is packed. The hole is the other 988 cycles: setup, the round 0–1 mux and hash, pipeline fill/drain, stores, and hash that did not sit on a gather. That is the shape of 2,780. It is not the shape of 1,487.

Re-measured 2026-08-19, all three kernels matched `reference_kernel2`:

{{< responsive-image src="images/vliw-cycle-bars.png" alt="Cycle counts from 147734 down to 2780 versus the 1487 Claude bar" maxWidth="720px" >}}

*Bar widths are linear in cycles against the starter. I did not pass 1,487.*


| Kernel | File | Cycles | vs 147,734 | Status |
|---|---|---:|---:|---|
| Scalar starter | `origianperf_takehome.py` | 147,734 | 1.00× | re-run |
| Load-once | notes | 13,903 → 12,463 | — | notes, not re-run |
| ALU mux 0–4, naive later gather | `updated_perf_takehome.py` | 4,268 | 34.61× | re-run |
| Lane pipeline (current) | `perf_takehome.py` | **2,780** | **53.14×** | re-run |
| Estimated hybrid | architecture note | ~1,498 | — | **estimate** |

Official bars from `Readme.md` / `tests/submission_tests.py`. None of these are mine. I did not pass any of them.

| Cycles | Whose number |
|---:|---|
| 2,164 | Opus 4, many hours in the harness |
| 1,790 | Opus 4.5, casual session (~best human in 2 hours) |
| 1,579 | Opus 4.5, 2 hours in the harness |
| 1,548 | Sonnet 4.5, much more than 2 hours |
| 1,487 | Opus 4.5, 11.5 hours. Email if below this |
| 1,363 | Opus 4.5, improved harness |

2,780 is about 1.28× the first Claude bar and about 2× the best published harness. 53× the starter is the wrong comparison. Claude is the actual bar.

Git has `117f69c Revert depth-6 ALU lookup`. I do not have a clean cycle number for that kernel and I am not going to invent one.

## What this is not

It is not a demonstration that the kernel is close to the machine’s limit. Dual-issuing every `load_offset` is necessary and not sufficient; 862 bundles still have no load.

It is not evidence that the architecture note was the right program. The note’s 0–4 cache, implemented more faithfully, lost to a shallower cache plus a packer.

It is not a Perfetto analysis of the 2,780 kernel. I counted bundles. I did not re-open the trace.

The last edit to `perf_takehome.py` is dated 2026-01-30. If I picked this up again I would not start by chasing ~1,498. I would start by filling some of the 862 empty load bundles, or deleting the work that created them.

A document titled “the minimum most efficient program” is a plan. Cycle counts arbitrate, including against my own notes. Utilization has a timestamp. Losing to Claude is the result.
