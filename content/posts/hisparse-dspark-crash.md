---
date: 2026-08-14
title: "Composing speculative decode with hierarchical sparse attention"
description: "SGLang HiSparse plus DSpark on a B300: 1,055 seconds to uvicorn, a null spec_info, $2.35 billed. Gate 3 then failed on 147k+1024 > 147456. Not a speedup claim."
tags:
  - inference
  - SGLang
  - DeepSeek
  - postmortem
---

Speculative decoding (Leviathan et al. 2023) is a feedback loop of a specific kind: a cheap draft proposes several tokens, a target model verifies them in one forward pass, and accepted tokens become free relative to autoregressive decode. Hierarchical sparse attention is a different loop: keep an index of the prefix hot on the GPU, evict ordinary attention KV off device, swap selected pages back when a query needs them.

Composing the two is not “turn both flags on.” Each assumes a batch object, a page geometry, and a cache that the other may have dismantled. This post is a record of that composition on DeepSeek V4 Flash 0731, one NVIDIA B300, SGLang. It is not a tokens-per-second result. The server never became healthy.

I already had a vLLM path for this model — FP4 experts, DSpark, prefix cache, agent traffic — written up separately as [Making DeepSeek V4 Flash Feel Local](/posts/deepseek-v4-flash-modal/). Those tables belong to a different engine. What they left was a narrower question: first-token time grew much faster than decode as prompts got long. Can the prefix *index* stay resident while ordinary KV leaves HBM?

This post focuses on that attempt. Matched A/B throughput, adaptive DSpark, and the economics of cache hits versus the official API are out of scope here; they are in the earlier write-up.

I did not write SGLang. LMSYS did. I wrote the campaign around it: an isolated Modal app so a failed boot could not take down the vLLM serving I already trusted, CPU dataset checks, a spend guard, a smoke client, later a DeepSeek-V4 coordinator on a cloned tree, and the failure JSON.

## Two flags, one incompatibility

The V1 command was intentionally un-clever. One B300, tensor parallel 1, official Flash 0731 (`7872f01b…`), SGLang v0.5.17 at `29481685`, published CUDA 13.0 image, FP8 KV, context 524288, FlashInfer MXFP4 MoE, DSpark at the checkpoint default of five proposed tokens:

```text
--enable-hisparse
--disable-radix-cache
--speculative-algorithm DSPARK
```

I wanted radix. A 350K agentic trace is mostly mature prefix reuse. SGLang v0.5.17 will not start HiSparse unless radix is off. `hisparse_hook.py` still says so: “Hierarchical sparse attention currently requires `--disable-radix-cache`.” So the V1 arm could not meet the prefix-reuse requirement the experiment was for. I recorded that in the failure JSON rather than pretending I had a HiSparse-off control or a cached-prefix result.

The dataset existed before the GPU did. Ninety-six independent Mooncake sessions, 360 turns each, seed 42, growing from a 15,240-token start to a 350,546-token finish. Validated on CPU. None of that traffic reached a live endpoint.

The paid entrypoint refused to run unless `DSV4_RUN_PAID=YES`. It took a Modal billing baseline, started a three-hour / ~$21 guard, and only then set `min_containers=1`.

## Pattern 1: Spend the cheap checks first

A stray `modal run` on the CPU downloader cannot launch a B300. Image contents, dataset shapes, and flag legality are CPU problems. The GPU is for the residual.

That residual, on 10 August 2026, was:

| Quantity | Value | Status |
|---|---|---|
| Container start | 22:15:42 | measured |
| Uvicorn on `:30000` | 22:33:17 | **1055 s** |
| Scheduler e2e | 964.0 s | measured |
| Weight load | 527.80 s (518.13 target + 9.68 draft) | measured |
| HBM before weights | 266.86 GB available | measured |
| HBM after pools/graphs | 23.84 GB available (~243 GB allocated) | measured |
| Health | `503; never became ready` | measured |
| `boot_to_healthy_seconds` | `null` | measured |
| Billed | **$2.35497550** | `startup_failure.json` |

HiSparse allocated: `host_to_device_ratio=2`, two 33.61 GB host paged-pools (`dsv4_hisparse_c4` for target and draft), 21 layers, 42,749 pages. DSpark initialized: `gamma=5`, `verify_num_draft_tokens=6`. Acceptance was never reached. There was nothing to accept. The tree cache came up as `SWAChunkCache` with `hierarchical=False`, which is what you get when radix is disabled.

This was not a missing-weight or wrong-GPU failure. The machine was doing real work.

## Pattern 2: Warmup is the first real decode

SGLang’s built-in 256-token startup warmup prefills a new sequence, then transitions the batch to decode. The scheduler called `batch.prepare_for_decode()`, which called `spec_prepare_for_decode()`, which did this:

```text
batch.spec_info.prepare_for_decode(batch)
```

`batch.spec_info` was `None`. `AttributeError` at `sglang/srt/speculative/spec_utils.py:1038` on the v0.5.17 image. Timestamp 22:33:34, seventeen seconds after uvicorn. Modal reported `SGLang exited during startup with -9`.

DSpark is in SGLang’s `dflash` family. The prepare-for-decode dispatcher for that family assumes `spec_info` exists. The V1 HiSparse path can produce a warmup batch that has already left prefill without attaching that object.

The recorded decision:

> No paid retry: v0.5.17 and current upstream both unconditionally dereference nullable `spec_info` in this DSpark path; upstream documents HiSparse/spec incompatibility and no obvious supported DSpark fix was found.

Patching the dereference locally would have been a one-line change and a dishonest experiment. I would have been measuring “a warmup guard,” not HiSparse. I would also have been measuring a stack that cannot turn radix on. Paying another 17 minutes of B300 time to confirm the same `AttributeError` would have been vanity.

Modal started a second container at 22:33:49 because the guarded deploy had `min_containers=1`. That is keep-alive, not a retry I designed. I stopped the app. The smoke client never wrote a result file. The AIPerf campaign never selected a concurrency. The report template still says `NOT RUN`.

I do not have a tokens-per-second number for this stack. I will not invent one.

{{< responsive-image src="images/hisparse-spec-info.png" alt="DSpark assumes spec_info exists; HiSparse V1 warmup leaves it None" maxWidth="720px" >}}

*The entire measured V1 result: a successful load, then AttributeError at spec_utils.py:1038.*


## Pattern 3: Page geometry is an admission rule

V1 is `--enable-hisparse` plus `--disable-radix-cache` on 0.5.17. Dead end for this model if you care about prefix reuse and DSpark.

V2 is a different design. Upstream HiSparse V2 (PR #32314) keeps radix. Evicted attention KV goes to HiCache with write-back. The indexer stays resident. Selected C4 records swap into a small device buffer. DeepSeek V4 is not the geometry that PR was written around.

DSA V2 treats one sparse position as one radix token. DSV4 does not:

- Radix and HiCache own 256-token full pages.
- C4 attention and the C4 indexer store 64 records per full page.
- C128 attention stores 2 records per full page.
- The sliding-window attention tail is 128 tokens, smaller than one full page.

I wrote a coordinator for that geometry in `sglang-hisparse-v2-auditfix1`, on upstream base `2ffbf20ae`. Five commits, mine: coordinator and Triton kernels; last 256-token full page stays request-owned so the 128-token SWA window stays live; SWA prefix lock released after unlock; prove the tail before radix unlock; stash inserts keep the same tail rule and charge full-token units.

The DSpark part is a correctness trade, not a throughput one. Target verify sees `gamma+1` query rows (width 6). Each row gets a disjoint C4 hot-slot bank so one query cannot evict a selected record another query in the same verify block still needs. Physical `temp_slots` is 512 + 512×6 = 3584. That wastes HBM on purpose. I have not measured a unioned-bank optimization, because I have not measured the path.

CPU checks ran first: source invariants, page-id churn under a test-only flag, Triton 3.6 compile-only lowering to non-empty SM103 cubins. Compile-only does not execute the kernels. Those checks passed. Then Allocation 1.

V2 fixes ordinary KV capacity at **147,456** tokens. That is the device pool the *control* server can actually hold. Control is not HiSparse. The old Gate 3 was a single long request: **147k + 1024 > 147456**. Control returned **0 tokens**. The candidate never got a fair token-ID compare, because there was no golden.

That is a harness bug, not a kernel bug. Official V2 geometry is the other way around: *each* request, including decode, must fit the device pool. Overflow is created by concurrent unique working set, not by one illegal request.

I reshaped Gates 3 and 4 in `87448cc` and stopped.

| Gate | Shape | Intent |
|---|---|---|
| Old Gate 3 | 147,000 + 1,024 > 147,456 | illegal control golden; returned 0 tokens |
| New Gate 3 | 145,920 + 1,024 = 146,944 ≤ 147,456 | long-context token match, **not** eviction |
| Gate 4 | two requests, shared 65,536 prefix, distinct 73,728 suffixes; unique set 212,992 > 147,456 | the case V2 is for |

The offline tests now refuse the old 147,000-token Gate 3. I have not re-run the paid A/B. CPU preparation is still green. GPU correctness is not. I will not say V2 works.

## Implications

It is not a serving recommendation. Production in this repo is still vLLM.

It is not a HiSparse benchmark. There is no decode rate, no TTFT, no acceptance percentage, no 350K AIPerf table.

It is not a claim that I fixed SGLang. I did not merge the coordinator. I did not patch `spec_utils.py` on the paid image. I did not prove swap-in on a B300.

The useful habit is the same one the vLLM campaign taught, applied to an engine that refused to start. Write the command down before the GPU exists. Spend the cheap checks first. When the paid job dies, write the crash as JSON with the billed dollars on the same page as the traceback. Do not turn a 503 into a blog win.

Long-context attention is still the wall. HiSparse is still a plausible way through it, if radix, HiCache, DSpark verify banks, and page ownership can be made to agree. Until that A/B runs, the honest sentence is the one on the invoice.

The serving stack would not boot. I stopped.
