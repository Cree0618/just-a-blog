---
date: 2026-08-11
title: "Making DeepSeek V4 Flash Feel Local"
description: "What I learned serving DeepSeek V4 Flash on one Modal B300: benchmark bugs, long-context limits, real agent traffic, and the economics of self-hosting."
tags:
  - AI
  - DeepSeek
  - Modal
  - inference
  - open source
---

I started this project with a fairly simple question:

> Can one rented GPU make a 284-billion-parameter model feel like a local model for coding agents?

Not a toy demo. I wanted a real OpenAI-compatible endpoint, long contexts, tool calls, concurrency, and enough measurements to decide whether self-hosting made economic sense.

The short answer is **yes, technically**. One NVIDIA B300 can serve DeepSeek V4 Flash 0731, including very long prompts. The more interesting answer is that the economics are conditional. Context length, cache behavior, concurrency, cold starts, and output rate matter more than a single impressive tokens-per-second number.

This is a progress report from the experiments, not a leaderboard result. The data comes from warm Modal runs on one B300. Some conclusions are measured; others remain explicitly unproven.

## The deployment

The basic serving architecture is deliberately boring:

1. Download the roughly 167 GB checkpoint once on a cheap CPU-only Modal function.
2. Store it in a shared Modal Volume.
3. Start the GPU container with the weights already present and `HF_HUB_OFFLINE=1`.
4. Run vLLM behind an OpenAI-compatible API.
5. Send coding-agent traffic to the served model named `llm`.

That separation matters. Downloading model weights inside a GPU container would turn an expensive cold start into an even more expensive download job. It also makes experiments repeatable: the model volume is reused while the serving image and configuration change independently.

The hardware and software stack is:

- one NVIDIA B300 with 288 GB of HBM3e;
- DeepSeek V4 Flash 0731, with roughly 284B total parameters and around 13B active parameters per token;
- vLLM `0.25.1`;
- the native FP4 MoE expert path, with mixed FP8 components, on the Blackwell fast path;
- the checkpoint's fused DSpark speculative-decoding module;
- prefix caching, FP8 KV cache, CUDA graphs, and chunked prefill (reported in the run notes; not independently recorded in every external JSON artifact).

The speed comes from the combination, not from one magic flag. FP4 reduces the weight traffic. DSpark proposes several tokens for the target model to verify in one pass. The serving configuration tries to keep the GPU busy while requests arrive with very different prompt lengths.

The deployment recipe follows [vLLM's DeepSeek V4 recipe](https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Flash) and the [official Modal GPU documentation](https://modal.com/docs/guide/gpu). The implementation and raw reports are in the [project repository](https://github.com/Cree0618/deepseek-v4-flash-modal).

## The first successful boot was not a benchmark

The first important result was not a speed number. It was proving that the intended path was actually running.

The 164K-token smoke test passed the useful checks:

- the container saw a B300;
- vLLM used the FP4 path rather than a BF16 or Marlin fallback;
- a prompt of 161,870 tokens was accepted;
- token-level prompt-cache hit rate was 99.8% in the synthetic shared-prefix setup;
- the tiny sample's mean DSpark accepted length was 7.0 (this metric includes the target token).

But the request stopped after only 21 output tokens. Its apparent decode rate was therefore meaningless. It proved that the model loaded and the prompt fit. It did not prove sustained throughput.

That distinction became a recurring theme: every number needs a description of exactly what it measured.

## Before optimizing the model, I had to fix the benchmark

An early throughput sweep produced a suspiciously attractive result. The problem was in my client, not in vLLM.

The benchmark used `asyncio.to_thread`, which silently relied on Python's default thread pool. On the development machine that pool had about twelve workers. Asking for concurrency 16, 64, or 256 did not actually create that many in-flight requests. The server was being tested at roughly twelve-way concurrency while the report claimed to test much higher levels.

I changed the benchmark to use a dedicated thread pool sized to the requested concurrency. I also changed the warm-up behavior: each sweep level now warms the server at that same concurrency, so CUDA graphs are captured before the timed batch instead of appearing as enormous latency outliers in the measurement.

This was probably the most valuable optimization in the whole project. It made the results less flattering, but more useful.

## The context curve

After fixing the client, I ran a fixed-output benchmark with 589 generated tokens per request. The table below shows the best clean point at each context size. The metric is **aggregate completion tokens divided by batch wall time**. It includes prompt work, cache work, TTFT, and scheduling; it is not pure decode speed.

| Context | Best concurrency | Aggregate throughput | Median TTFT | Planning users* |
| ---: | ---: | ---: | ---: | ---: |
| 15K | 16 | 1,737 tok/s | 1.10 s | 64 |
| 32K | 16 | 1,609 tok/s | 1.24 s | 60 |
| 64K | 16 | 1,395 tok/s | 2.19 s | 52 |
| 164K | 24 | 851 tok/s | 11.48 s | 31 |
| 300K | 24 | 509 tok/s | 22.0 s | 18.8 |
| 512K | 24 | 309 tok/s | 37.3 s | 11.4 |

\* These are derived capacity figures for a particular trace-shaped workload, assuming 82.6 steps per session and 589 output tokens per step. They are planning numbers, not a 52-user production replay.

The important pattern is not simply that throughput falls as prompts get longer. It is *why* it falls. In the measured 15K-to-164K decomposition, TTFT grows 10.4× while the decode rate falls about 30%. At 300K and 512K, long-context request work still dominates, but the saved artifacts do not isolate the individual cache, prefill, and decode phases. A short-context decode benchmark can therefore make the machine look much stronger than it feels to an agent waiting for its next tool call.

Concurrency also changes with context. Sixteen concurrent requests was the clean peak around 15K–64K. At 164K and beyond, 24-way batches produced the best aggregate throughput in this synthetic test, but not necessarily the best latency. More concurrency is not automatically better, though: at some point it becomes queueing rather than useful parallelism.

A separate 768K smoke test accepted a 762,639-token prompt. That is a useful prompt-fit result. It is not a claim that one B300 can sustain a productive 768K workload; the request generated only 16 output tokens.

## Synthetic traffic and real agents tell different stories

The fixed-output test intentionally repeats a shared scaffold. That gives it roughly 99% token-level prompt-cache hits, which is useful for isolating a serving path but unlike a group of agents whose contexts grow independently.

The next step was real tool-shaped traffic: agents reading files, writing files, searching code, and running commands in synthetic repositories. At 64K context, the best saved burst reached 2.71 successful requests per second at 16 agents. Increasing the burst to 20 agents collapsed to 0.54 requests per second, with most work queued behind only a couple of active prefills.

The full-session test was more revealing:

- 16 agents;
- 12 turns each;
- 176 attempted requests;
- 170 successful requests and 6 errors;
- 3.4% request error rate;
- median prompt size around 71K tokens;
- 62.4% draft-token acceptance, with a mean DSpark accepted length of 5.37;
- 46.5% token-level cache hit rate.

The middle of the session reached 27–28 seconds of median TTFT when all 16 agents were active. Later turns became faster as agents finished and concurrency fell.

That is the difference between a benchmark and an application. The synthetic benchmark reported roughly 99% token-level prompt-cache hits. The growing multi-agent session reported 46.5%. The server still worked, but cache pressure, queueing, and different conversation lengths changed the operating point completely.

The session was still a successful proof of concept. It was not a polished production service: it had a small error rate, long mid-session waits, and incomplete per-agent capacity curves. Those limitations are part of the result.

## The workload ladder: OpenCode, mini-SWE, and AgentX

The experiments became progressively less synthetic. Each tool exposed a different bottleneck, so I stopped treating them as interchangeable benchmarks.

### OpenCode: real CLI behavior and thinking mode

I then ran the endpoint through OpenCode-style coding sessions with real tools and `thinking` enabled at the high setting. The experiment used one, four, and eight concurrent agents, a 393K context limit, and a warm-up before each lane.

The eight-agent lane took about 205 seconds. The agents did actual work rather than emitting fixed-length text: they inspected a generated repository, edited files, ran tests, and reviewed the result. A separate per-agent audit artifact from that run examined 48 modules containing 10,560 handlers, ran `pytest` successfully, and found no functional anomalies. It also exposed a more realistic engineering problem: the workspace had only one test, covering roughly 0.009% of the generated handlers.

At eight agents, first reasoning/tool events arrived roughly 30–37 seconds into the requests. The measured lane was about 95% cached by token count. Using the Baseten rate card, its full-wall accounting was approximately $0.404 of GPU cost versus $0.228 of API-equivalent cost, or 1.77× more expensive. At an assumed 60% inference utilization it approached parity, but that assumption is exactly the sort of thing that needs to be measured rather than quietly inserted into a spreadsheet.

This was not an apples-to-apples comparison with the short non-thinking benchmark. It was valuable for a different reason: it measured what a coding CLI experiences when reasoning, tool latency, growing conversations, and inference are all mixed together.

### mini-SWE-agent: closed-loop SWE-bench traffic

The next step was a small real-agent pilot using mini-SWE-agent against SWE-bench Verified repositories. Eight workers drove real sandboxed tasks through the deployed endpoint, with a second group of standby tasks and a closed-loop replay path. The run was deliberately a pilot; it did not claim a SWE-bench score.

Across the long-context run, the server recorded roughly 9.9M prompt tokens, 9.5M cached prompt tokens, and 170K generated tokens. There were no preemptions. DSpark's effective accepted length was 5.38 tokens, substantially healthier than the fixed-output synthetic runs, and the replay artifacts make it possible to compare future serving configurations without paying to rerun the sandboxes.

The lesson was that a real coding agent is a stochastic process. It spends time waiting for tools, sometimes produces a long response, and grows its prompt in uneven increments. A clean throughput sweep can select a candidate configuration, but only a closed-loop agent run tells us whether that configuration survives contact with a repository.

### AgentX: the harder economic test

AgentX was the most sobering benchmark. It used an AIPerf AgentX scenario backed by long coding traces rather than a handful of fixed prompts. The valid run lasted 900 seconds at the recommended concurrency of 16.

| Metric | AgentX result |
| --- | ---: |
| Output throughput | 243.70 tok/s |
| Request throughput | 0.235 req/s |
| Prompt tokens | 21.57M |
| Cached prompt tokens | 19.32M (89.56%) |
| Speculative acceptance | 65.61% draft-token acceptance; mean DSpark accepted length 4.59 |
| TTFT P50 / P90 | 1.23 s / 3.32 s |
| Failed turns | 0 |

The short 120-second search briefly reached 334 tok/s, but the longer run is the number I trust for planning. Increasing the AgentX session-tree concurrency made the system worse: c24 reduced output throughput by 15.9%, while c32 reduced it by 44.1% and pushed TTFT P90 to 18.83 seconds. There was a persistent waiting queue at c32.

The contexts were much heavier than the earlier mini-SWE mix: median 92.5K tokens, P90 168.8K, and a maximum of 373.6K. This also explains why the AgentX result is not comparable to the 1,700 tok/s short-context figure. AgentX concurrency counts live session trees, not simply simultaneous HTTP requests; most of the time the agents are in tool gaps, while the occasional long prefill still determines the GPU's useful rate.

The economic result was a clear stop signal. At the Baseten prices used for this run, the processed work was worth about $3.56 per hour at the API, while the B300 cost $7.10 per hour. Self-hosting was therefore 1.99× the API cost at the useful c16 operating point. The cold start took 10.73 minutes and cost another roughly $1.27. For this workload, one B300 was not viable without a different batching profile, more utilization, cheaper hardware, or a better reason to self-host than token price alone.

That result is not a failure of the deployment. It is a much better answer to the original question. The same GPU is attractive for some fresh-heavy or shorter-context traffic and unattractive for this long, cache-heavy AgentX mix.

## What changed during the optimization work

The project began as a search for maximum decode speed. It gradually became a project about managing context and admission to the GPU.

The latest configuration work includes:

- increasing `max_num_batched_tokens` from 8,192 to 16,384 and then to 32,768 for the 128K–384K target workload;
- persisting FlashInfer autotuning results and TileLang/FlashInfer JIT artifacts across cold starts;
- raising GPU memory utilization from 0.92 to 0.95, with an explicit OOM check still required;
- keeping prefix caching and chunked prefill enabled for long agent turns;
- making greedy sampling the default for this agent-oriented deployment, intended to improve determinism and draft/target agreement; that effect has not been isolated in a matched A/B test;
- recording actual in-flight request counts, prompt-token counts, cache counters, failures, and Prometheus counter resets.

These changes are hypotheses until the post-change A/B runs are complete. A configuration change is not a measurement.

There is also an adaptive DSpark source-build arm based on an open vLLM implementation. It is kept separate from the stable vLLM 0.25.1 arm because source builds are slow and speculative decoding behavior can change under load. The matched adaptive-versus-fixed experiment has not yet produced a result I am willing to publish.

## The economics forced a correction

At first, it was tempting to say that one B300 was simply cheaper than an API. That statement was too broad.

There are also two different API price cards involved, and mixing them produces very convincing but invalid spreadsheets. The table below records the historical rates used in the August 6 analysis; the official tier was flagged for a significant price increase on August 6.

| Product | Input | Cached input | Output |
| --- | ---: | ---: | ---: |
| Baseten DeepSeek-V4-Flash-0731 | $0.13/M | $0.028/M | $0.26/M |
| Official DeepSeek API, historical 2026-08-06 rate card | $0.14/M | $0.0028/M | $0.28/M |

These are different products. Check the [official DeepSeek pricing page](https://api-docs.deepseek.com/quick_start/pricing/) before making a current comparison.

Using the historical 2026-08-06 official-API rate card, the synthetic shared-prefix runs gave these self-hosted/API ratios:

| Context — historical 2026-08-06 official-price comparison | Self-hosted / API |
| ---: | ---: |
| 15K | 0.53× |
| 32K | 0.55× |
| 64K | 0.59× |
| 164K | 0.82× |
| 300K | 1.11× |
| 512K | 1.42× |

Below 1× is cheaper. These values exclude cold starts, idle windows, failed work, and the operational cost of running the service. They describe one synthetic traffic shape, not a universal break-even point.

The full 16-agent session had a different accounting result: about $0.421 of measured GPU cost versus $0.9166 at the official API rate card, or 0.46×. That is encouraging, but it is still one run and does not settle the cost of cold starts or always-on availability.

The later economics audit exposed two mistakes in the earlier reasoning:

1. I mixed the Baseten and official DeepSeek price tiers.
2. I treated a very high synthetic cached-token rate as if it were a fresh-prefill ceiling, and effectively treated cached tokens as free GPU work.

They are not free. In a cache-heavy agent workload, the GPU still spends time moving and attending to cached state. The current conclusion is more nuanced: fresh-heavy traffic can be a good fit for self-hosting; cache-heavy traffic can be close to parity or worse, especially against an API with very cheap cache reads. Actual billed wall time matters more than a throughput-only estimate.

## What we know, and what we do not

### Established

- One B300 can load and serve DeepSeek V4 Flash 0731 with the FP4 fast path.
- A 164K prompt works reliably in the tested setup.
- A 762K prompt fits in a smoke test.
- Warm aggregate throughput ranges from about 1,700 tok/s at short context to about 300 tok/s at 512K in the measured fixed-output runs.
- Real tool traffic achieves healthy, non-zero DSpark acceptance, including 62.4% over the full session.
- Context management and concurrency limits matter as much as kernel selection.

### Still unproven

- The causal speedup from DSpark. There is no matched no-spec benchmark on the same deployment, so I will not claim a DSpark multiplier.
- Sustained 768K throughput.
- A full 52-user replay.
- Universal API savings.
- The production value of adaptive verification.
- How the latest 32K batch-token configuration changes the long-context curve.

## What comes next

The next experiments are less glamorous than the first deployment, but more decisive:

1. Run a matched DSpark/no-spec A/B at the same context, concurrency, checkpoint, and sampling settings.
2. Compare adaptive and fixed-k DSpark on the same source-built vLLM image.
3. Repeat the 128K–384K sweep after the batch-budget change.
4. Replay a realistic multi-turn workload while recording cache counters and actual Modal billing.
5. Treat context trimming, concurrency caps, and cold-start discipline as first-class product features rather than benchmark details.

The biggest lesson is that deploying a large model is only the beginning. The first benchmark asks, “How many tokens per second can this GPU produce?” The useful benchmark asks, “How long does an agent wait, how many agents can share the GPU, what happens when their contexts grow, and what does the bill look like at the end?”

One B300 is surprisingly capable. It is not a magical replacement for an API, but with the right context budget and traffic shape it can make a very large open model feel local—and give you much better visibility into where the performance and money actually go.

## Further reading

- [Project repository and deployment guide](https://github.com/Cree0618/deepseek-v4-flash-modal)
- The detailed benchmark, AgentX, OpenCode, mini-SWE, and economics reports are kept with the experiment artifacts in the project workspace.
- [NVIDIA B300 component data](https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/components.html)
- [Modal pricing](https://modal.com/pricing)
- [vLLM's DeepSeek V4 recipe](https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Flash)
