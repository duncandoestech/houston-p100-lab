# Benchmarks

All results from this one machine. Sections run from the current setup back to the earlier ones. Full tables, method
and limits for the tuning work are in [measurements/](measurements/).

- [How things were measured](#how-things-were-measured)
- [Current setup: Qwen3.8-27B on four cards](#current-setup-qwen38-27b-on-four-cards)
- [How the current settings were found (three cards)](#how-the-current-settings-were-found-three-cards)
- [Two models side by side (2+2 split)](#two-models-side-by-side-22-split)
- [Earlier setup: Qwen3.6-35B-A3B on the 29-patch build](#earlier-setup-qwen36-35b-a3b-on-the-29-patch-build)
- [Other models on the earlier setup](#other-models-on-the-earlier-setup)
- [P2P bandwidth between cards](#p2p-bandwidth-between-cards)

## How things were measured

- **Speed:** `llama-bench` (pp = prompt processing, tg = generation, in tokens/s) and real `llama-server` requests.
  MTP can only be measured through the server or `llama-speculative-simple`.
- **Agent battery:** a Hermes agent running nine error-recovery coding tasks (`err_python_env`, `err_replay_patch`,
  `err_ambiguous_edit`, `err_case_search`, `err_hidden_search`, `err_big_output`, `err_multi_dir`,
  `err_inline_script`, `err_big_file_read`) against the server, 3 repetitions each, 600 s limit per task. Pass rate
  mostly measures the model; wall time is the hardware number. `err_hidden_search` and `err_big_file_read` are the
  noisy tasks; at 3 repetitions, one or two passes either way is noise.
- **Protocol for the tuning work:** cool down to 48 °C or below before each run, a discarded warmup, arms interleaved
  (A B B A), 1 Hz `nvidia-smi` telemetry, and a watchdog that stops on a card over 80 °C or an SM clock under
  1000 MHz while busy.
- Power figures are GPU board power from `nvidia-smi` plus CPU package power (RAPL). The rest of the machine is not
  measured; draw at the wall is higher.

## Current setup: Qwen3.8-27B on four cards

Kmic-68 fork, NCCL, tensor split, `UD-Q6_K_XL`, 150 W. Command in [SOFTWARE.md](SOFTWARE.md#production-settings).

### Three cards against four (2026-10-06)

`llama-bench`, 3 repetitions, order alternated:

| Test | 3 cards | 4 cards | Change |
|---|---:|---:|---:|
| tg512 | 33.07 ± 0.17 | 39.24 ± 0.37 | +18.7% |
| pp2048, depth 0 | 505.11 ± 0.60 | 629.00 ± 1.28 | +24.5% |
| pp2048, depth 16384 | 450.96 ± 0.66 | 584.54 ± 1.46 | +29.6% |

Server with the production flags, two servers per card count (order 3, 4, 4, 3):

| | 3 cards | 4 cards | Change |
|---|---:|---:|---:|
| 64,000-token prompt (4 runs each) | 388.5 t/s | 514.5 t/s | +32.4% |
| 64,000-token prompt, total time | 165-167 s | 125-126 s | −25% |
| 512-token generation with MTP (6 runs each) | 37.4 t/s | 42.1 t/s | +12.5% |
| Peak memory per card | 10.9-11.5 GB | 8.5 GB | |
| Hottest card | 66 °C | 66 °C | |
| GPU power, mean while busy / peak | 395 W / 528 W | 475 W / 647 W | |

Output quality (KL divergence over 16,376 tokens of wikitext-2): going from three cards to four changes the output
about as much as turning NCCL on does (mean KLD 0.005, same top token 98.3%). Two four-card runs were byte-identical.

### Agent battery on four cards

| Task | Passed | Mean wall |
|---|---:|---:|
| err_python_env | 3 of 3 | 67 s |
| err_replay_patch | 3 of 3 | 65 s |
| err_ambiguous_edit | 3 of 3 | 70 s |
| err_case_search | 3 of 3 | 71 s |
| err_hidden_search | 3 of 3 | 63 s |
| err_big_output | 3 of 3 | 64 s |
| err_multi_dir | 3 of 3 | 61 s |
| err_inline_script | 3 of 3 | 88 s |
| err_big_file_read | 2 of 3 | 529 s |
| **Total** | **26 of 27 (96%)** | **120 s** |

Server over the battery: 453.9 t/s prompt, 53.9 t/s generation with MTP, 71.4% draft acceptance. The one failure
was a context overflow: the agent's conversation reached 278,695 tokens, past the 262,144 context.

| | 4 cards, 150 W | 3 cards with NCCL, 125 W | 3 cards without NCCL, 125 W |
|---|---:|---:|---:|
| Tasks passed | 26 of 27 | 24 of 27 | 23 of 27 |
| Mean wall time | 120 s | 124 s | 183 s |
| Mean wall time without `err_big_file_read` | 69 s | 84 s | 131 s |
| Server prompt speed over the battery | 453.9 t/s | 377.9 t/s | 242.2 t/s |
| Server generation with MTP | 53.9 t/s | 48.9 t/s | 48.1 t/s |

The three-card runs also used `-b 32768` and were a day earlier, so not all of the gain is the fourth card.

### Alternatives tested on four cards (2026-10-11 and 10-12)

Nothing beat the production settings.

| Question | Answer |
|---|---|
| Q8_0 instead of `UD-Q6_K_XL`? | No. Same battery score (26 of 27), 4.6% slower generation (tg512 37.43 vs 39.22), 0.8 GB more per card. Prompt speed equal. |
| q8_0 KV cache instead of q4_0? | No. The slowest option: −8.7% generation in `llama-bench`; on the server −3.5% short, −4.7% deep, −2.9% on a 64k prompt, +1.0 GB per card. |
| f16 KV cache instead of q4_0? | No. About +3% on short requests, −2.5% on a 64,000-token prompt, +2.9 GB per card. |
| Does the q4_0 cache hurt output? | Very little. Against f16 over a 65,536-token context: mean KLD 0.0021, same top token 98.0%, no rise with depth, perplexity +0.25%. |
| Other MTP settings? | No. `n-max` 2 and 5 were 7% and 10% slower on short requests; draft top-k / top-p settings were within noise. |
| What does MTP give? | +15% generation on short requests, +11% deep in context, −4.5% on a 64,000-token prompt, 474 MiB more per card. |

Server runs, one setting changed at a time (12 short generations, a 64,000-token prompt, then 3 generations on top
of it; production run first and last, which differed by 2.6-2.8%):

| Run | Short t/s | Deep t/s | 64k prompt t/s | Peak MiB per card |
|---|---:|---:|---:|---:|
| Production, run 1 | 45.78 | 41.95 | 512.1 | 8577 |
| Production, run 2 | 44.52 | 40.90 | 511.2 | 8577 |
| n-max 2 | 42.02 | 39.83 | 511.8 | 8495 |
| n-max 4 | 43.84 | 43.92 | 508.1 | 8591 |
| n-max 5 | 40.73 | 38.46 | 510.7 | 8653 |
| `LLAMA_SPEC_DRAFT_TOPP=0.95` | 43.97 | 42.58 | 511.5 | 8577 |
| `LLAMA_SPEC_DRAFT_TOPK=10` | 44.73 | 42.81 | 511.1 | 8553 |
| `LLAMA_SPEC_DRAFT_TOPK=40` | 44.50 | 41.58 | 511.1 | 8577 |
| KV q8_0 (mean of 2) | 43.56 | 39.48 | 497.2 | 9568 |
| MTP off | 39.24 | 37.31 | 536.2 | 8103 |

Details: [measurements/FOUR-CARDS.md](measurements/FOUR-CARDS.md).

## How the current settings were found (three cards)

Kmic-68 fork on three cards, Qwen3.8-27B `UD-Q6_K_XL`, tensor split, q4_0 KV, 125 W unless stated. Full data:
[measurements/](measurements/).

### The settings that mattered

"A" = CUDA graphs `=3` and `-lm none`; "B" = A plus the NCCL build with `NCCL_P2P_LEVEL=SYS`. 3 repetitions each.

| Test, t/s | Before (graphs off, mmap) | A | B |
|---|---:|---:|---:|
| tg512 | 21.40 | 29.44 ± 0.09 | 31.09 ± 0.06 |
| pp2048, depth 0 | 274.6 | 275.94 ± 0.23 | 499.75 ± 0.11 |
| pp2048, depth 16384 | 260.1 | 259.64 ± 0.38 | 444.91 ± 0.58 |
| pp2048, depth 65536 | 211.1 | 211.75 ± 0.27 | 318.66 ± 0.59 |
| 129,000-token prompt through the server, `-c 262144` | — | 211.0 | 315.9 |

- **CUDA graphs** (`GGML_CUDA_GRAPHS_PRE_VOLTA=3`): with them off, 43.6 of 45.3 ms per token was the CPU queueing GPU
  work. Decode is bound by the 2011 CPU, not the cards, until graphs cut the per-token API calls from about 19,000 to
  7,300.
- **NCCL:** +51-82% prompt speed, +5.6% decode. Without `NCCL_P2P_LEVEL=SYS`, NCCL uses a host-memory path here.
  Prompt exchanges of 26+ tokens run in BF16; output differs from the non-NCCL build by less than changing `-ub` does
  (mean KLD 0.0009 against 0.0015 over a 65,536-token context), and is repeatable run to run.
- **No NCCL tuning variable beat the defaults by 3%.** The best, 1 channel, gave +1.6% prompt speed.
- **`-ub 4096`** against 2048 gained 0.8-1.3% for 2.4 GB more per card; kept 2048. `-ub 512` cost 13%.
- **Context:** `-c 262144` fits with over 5 GB free per card on three cards (one 129,000-token prompt run).
- **Q8_0** was 7% slower in decode than `UD-Q6_K_XL` and equal in prompt speed.

### MTP settings

Mean of 3 prompts (chat, code, ~8k summary), 256 tokens, n-max 3:

| p-min | Greedy, A | Greedy, B | Sampled (3 seeds), A | Sampled (3 seeds), B |
|---|---:|---:|---:|---:|
| 0.0 | 48.36 | 47.30 | 41.3 | 43.6 |
| 0.5 | 42.47 | 39.89 | 37.9 | 38.4 |
| 0.75 | 37.47 | 39.69 | 34.7 | 36.1 |

p-min 0.0 wins under both greedy and sampled decoding. n-max 3 and 4 at p-min 0.0 are within 2.5% of each other.

### Power cap

| Cap | tg512 | pp2048 | GPU power in decode, 3 cards | Hottest card | Decode t/s per GPU watt |
|---|---:|---:|---:|---:|---:|
| 125 W | 31.25 | 506.0 | 356 W | 54 °C | 0.0877 |
| 150 W | 33.29 (+6.5%) | 530.4 (+4.8%) | 430 W | 58 °C | 0.0775 |
| 175 W | 33.66 (+7.7%) | 549.0 (+8.5%) | 484 W | 60 °C | 0.0696 |

At 125 W the driver's power-cap reason was active in up to 80% of busy samples during generation; at 175 W, almost
never in decode. Production uses 150 W.

### Dropped requests and `-b`

A client drops a 64,000-token request 78 s in. How long the slot stays busy:

| `-b` | Nothing else calling | Another client polling `/slots` 5x a second |
|---|---|---|
| 32768 | 72-74 s | 80-81 s |
| 4096 | 3.8-4.0 s | 79-80 s |
| 2048 | 3.8-4.3 s | 79-80 s |

`-b 2048` costs 0.3-0.8% of prompt speed and nothing in decode. Polling `/health` every 5 s had no effect.

### Concurrency, slots and soak

| | `-np 1` | `-np 2 --kv-unified` | `-np 2 --no-kv-unified` |
|---|---:|---:|---:|
| Second client arriving 5 s into a 64k prompt: time to first token | 162.5 s | 191.9 s | 163.8 s |
| Two short chats at once, combined | 37.2 t/s | 45.4 t/s | 47.3 t/s |
| Extra memory per card | — | +1.2 GB | +0.7 GB |

A second slot doesn't let a client in during someone else's long prompt. Production stays at one slot.

90-minute soak (one slot, mixed chats, code, 20k and 64k prompts): 349 requests, none failed, prompt speed flat,
memory constant, peak 61-64 °C.

### First look at the Kmic fork (125 W, 2026-09-27)

Before any of the tuning above (no NCCL, mmap loading, MTP 4 / 0.2, `-b 32768`, `UD-Q5_K_XL` file, three cards):
agent battery 26 of 27 (96%), 145 s mean per task, 188.0 t/s prompt and 45.1 t/s generation over the battery.
Correctness gate first: decode 21.76 t/s against 17.51 for upstream at the fork point, perplexity 2.6067 (band
2.6209 ± 0.0199), flash-attention op tests passed.

## Two models side by side (2+2 split)

Qwen3.6-35B-A3B on cards 0-1 and Qwen3.8-27B on cards 2-3, both at 262,144 context, 150 W (2026-10-21 and 10-22).
Measurement only; production stayed on four cards.

| | Qwen3.6-35B-A3B | Qwen3.8-27B |
|---|---|---|
| File | `Qwen3.6-35B-A3B-MTP-UD-Q4_K_XL` | `Qwen3.8-27B-UD-Q6_K_XL` |
| Build | 29-patch `v0.4.0` | Kmic-68 NCCL build |
| Split | layer | tensor |
| KV cache | f16 | q4_0 |
| MTP | n-max 3, p-min 0.75 | n-max 3, p-min 0.0 |

| t/s | Qwen3.8 alone | Qwen3.8, Qwen3.6 busy | Qwen3.6 alone | Qwen3.6, Qwen3.8 busy |
|---|---:|---:|---:|---:|
| Short chats | 37.67 | 37.26 | 90.85 | 90.37 |
| After a 64,000-token prompt | 35.72 | 33.81 | 60.91 | 60.61 |
| 64,000-token prompt | 342.6 | 344.6 | 404.0 | 402.2 |

- The two barely slow each other: at most 5%.
- Qwen3.8 on two cards against four: −16% short, −11% deep, −33% on the 64k prompt. A 250,000-token prompt still fits.
- Qwen3.8 on two cards scored 24 of 27 on the agent battery (26 of 27 on four).
- **Subagents:** with Qwen3.8 as the main agent, sending its subagents to Qwen3.6 finished four delegation tasks in a
  mean 427 s against 552 s with subagents on Qwen3.8, 12 of 12 passed against 11 of 12.

Choosing the Qwen3.6 file (t/s):

| | Q4_K_XL, 2 cards | Q5_K_XL, 2 cards | Q8_0, 4 cards |
|---|---:|---:|---:|
| Short chats, MTP | **90.9** | 83.0 | 85.4 |
| After a 64,000-token prompt, MTP | **61.0** | 39.6 | 55.1 |
| 64,000-token prompt, no MTP | 846 | 795 | **1208** |
| After a 250,000-token prompt, no MTP | **41.6** | 22.0 | 38.9 |

Q5_K_XL only fits on two cards with a q8_0 cache and `-ub 256`. On Qwen3.6, MTP halves prompt speed (421 against
952 t/s on a 64k prompt) but helps short chats, so it stays on. Tensor split was slower than layer split for this model
(about 50 against 70 t/s generation on four cards). Tuning found nothing better than the starting settings on either
model. Details: [measurements/QWEN36-2X2.md](measurements/QWEN36-2X2.md).

## Earlier setup: Qwen3.6-35B-A3B on the 29-patch build

Before the Kmic fork, the box ran Qwen3.6-35B-A3B (mixture of experts, ~3B active) `Q4_K_XL` on upstream `v0.4.0`
with the 29 P100 patches, **layer split**, three cards (x16/x8/x8 with a riser), 65,536 context, q8_0 KV, MTP
n-max 3 / p-min 0.75. Measured 2026-09-21 to 09-27, default 250 W unless stated, SM clocks checked at ~1,328 MHz.

### 29-patch build against stock llama.cpp

`llama-bench`, three cards, 5 repetitions:

| | pp512 | tg128 |
|---|---:|---:|
| Qwen3.6-35B-A3B Q4_K_XL, stock `v0.4.0` | 460.98 ± 11.87 | 54.69 ± 1.24 |
| Qwen3.6-35B-A3B Q4_K_XL, 29 patches | 468.18 ± 4.24 | **69.54 ± 1.31 (+27.2%)** |
| Qwen3.8-27B Q5_K_XL, stock `v0.4.0` | 141.81 ± 0.33 | 10.50 ± 0.00 |
| Qwen3.8-27B Q5_K_XL, 29 patches | 141.06 ± 1.40 | **13.22 ± 0.00 (+25.9%)** |

The patches speed up generation, not prompt processing. The patch repo's own +80% headline used a different quant,
MTP and a real sampler; this is a plain `llama-bench` comparison.

### MTP tuning

Server, one request at a time, 512 tokens, temperature 0, 4 measured requests per setting:

| Setting | Generation t/s | vs no MTP | Draft acceptance |
|---|---:|---:|---:|
| No MTP | 66.5 | — | — |
| n-max 3, p-min 0.0 | 90.8 | +36.5% | 67.5% |
| n-max 5, p-min 0.0 | 68.3 | +2.7% | 43.0% |
| **n-max 3, p-min 0.75** | **92.4** | **+39.0%** | 93.9% |
| n-max 4, p-min 0.75 | 90.3 | +35.8% | 93.1% |
| n-max 3, p-min 0.75, sampled (temp 0.7, top-p 0.8, top-k 20) | 87.0 | +30.8% | 90.6% |

On this MoE model a high p-min was the lever. On Qwen3.8 with the Kmic fork, p-min 0.0 wins instead; tune per model
and build.

### Card count

Agent battery, one request at a time, layer split. The quant was stepped down so the model fit on fewer cards:

| Cards | Quant | Generation avg | Prompt avg | Mean task wall | Passed | GPUs + CPU power, avg / peak | Energy per task |
|---:|---|---:|---:|---:|---:|---:|---:|
| 3 | Q4_K_XL | 85.2 t/s | 325.8 t/s | 70 s | 93% | 327 W / 601 W | ~23 kJ |
| 2 | Q4_K_XL | 85.0 t/s | 302.8 t/s | 83 s | 93% | 270 W / 437 W | ~22 kJ |
| 2 | Q3_K_XL | 82.0 t/s | 283.9 t/s | 72 s | 89% | 268 W / 418 W | ~19 kJ |
| 1 | Q2_K_XL | 76.0 t/s | 247.7 t/s | 98 s | 89% | 200 W / 258 W | ~20 kJ |

- **With layer split, more cards don't make one request faster.** Each card runs its layers in turn, so per-card
  utilization falls (87% on one card, 37-57% on two or three). Extra cards buy VRAM, not speed. (Tensor split with
  NCCL on the Kmic fork is different: there, three to four cards was +13% generation.)
- Quant didn't change speed (Q4_K_XL and Q3_K_XL on two cards: 85.0 vs 82.0 t/s).
- Part of the one-card gap is the Q2_K_XL quant; this series can't separate the two.

### PCIe link width

Two cards, Q3_K_XL, layer split:

| Link widths | Generation avg | Prompt avg | Mean task wall | Passed |
|---|---:|---:|---:|---:|
| x16/x16 | 82.0 t/s | 283.9 t/s | 72 s | 89% |
| x16/x8 | 83.2 t/s | 295.1 t/s | 88 s | 89% |
| x8/x8 | 80.8 t/s | 291.4 t/s | 97 s | 85% |

All within noise. Only small activations cross between cards per token in layer split, and x8 has plenty for that.
(The x8/x8 run also had worse airflow after reseating the cards: one card averaged 74.8 °C, peak 80 °C.)

### Power cap: 125 W against 250 W

Agent battery, three cards:

| | 250 W (default) | 125 W cap | Change |
|---|---:|---:|---:|
| Passed | 93% | 93% | same |
| Mean task wall | 70 s | 79 s | +13% |
| Generation avg | 85.2 t/s | 82.2 t/s | −3.5% |
| Prompt avg | 325.8 t/s | 286.0 t/s | −12% |
| GPUs + CPU, average power | 327 W | 272 W | −17% |
| GPUs + CPU, peak | 601 W | 431 W | −28% |
| Energy per task | ~22.9 kJ | ~21.5 kJ | −6% |
| GPU clock while busy | 1,327 MHz | 1,227 MHz | −7.5% |
| Hottest card, peak | 75 °C | 65 °C | −10 °C |

The cap works by lowering the clock (throttle reason "SW Power Cap" in ~29% of busy samples, no thermal throttling).
Generation is memory-bound and barely moves; prompt processing is compute-bound and takes most of the hit.

### Concurrent requests

`llama-batched-bench`, three cards, 128-token prompts and 128 generated tokens per sequence:

| Concurrent sequences | Generation t/s | Total t/s |
|---:|---:|---:|
| 1 | 69.09 | 49.95 |
| 4 | 152.56 | 224.86 |
| 8 | **174.64** | **278.29** |
| 16 | 112.97 | 193.02 |
| 32 | 125.99 | 214.67 |

Throughput peaks at 8 sequences (2.5x a single stream) and falls after. The cause wasn't determined.

## Other models on the earlier setup

Agent battery, 29-patch build, layer split.

### Qwen3.8-27B (dense) Q5_K_XL, no MTP

Plain quant with no MTP head. Task limit raised to 900 s and `max_tokens` to 12,288 because it thinks at length at
~12 t/s (a 7,500-token thinking budget was set but never reached).

| | 2 cards | 3 cards |
|---|---:|---:|
| Passed | 93% | 93% |
| Mean task wall | 291 s | 269 s |
| Prompt avg | 123.0 t/s | 157.0 t/s |
| Generation avg | 12.0 t/s | 11.9 t/s |
| GPUs + CPU, average power | 314 W | 371 W |
| Energy per task | ~91 kJ | ~100 kJ |

Same pass rate as Qwen3.6 at about 4x the task time. Highest per-card power of any run (~130 W average, ~218 W peak),
since a dense model reads every weight for every token. The same model on the tuned Kmic stack later generated
42-54 t/s with MTP.

### NVIDIA Nemotron-3.5-Lightning-30B-A3B Q4_K_XL

Hybrid Mamba + Transformer MoE (31B total, 3.5B active) with MTP. Ran on the 29-patch build unchanged, temperature
1.0, top-p 0.95.

| | 2 cards | 3 cards |
|---|---:|---:|
| Passed | 81% | 78% |
| Mean task wall | 136 s | 154 s |
| Prompt avg | 334.4 t/s | 444.4 t/s |
| Generation avg | 103.6 t/s | 103.9 t/s |
| MTP draft acceptance | 91.9% | 92.6% |
| GPUs + CPU, average / peak power | 288 W / 437 W | 352 W / 581 W |

About 22% faster generation than Qwen3.6, but tasks took 1.6-2.2x longer and passed less often; most failures were
`err_big_file_read` context overflows.

### Model comparison

| Model, Q4_K_XL | Cards | Generation | Prompt | Mean task wall | Passed |
|---|---:|---:|---:|---:|---:|
| Qwen3.6-35B-A3B | 2 | 85.0 t/s | 302.8 t/s | 83 s | 93% |
| Qwen3.6-35B-A3B | 3 | 85.2 t/s | 325.8 t/s | 70 s | 93% |
| Nemotron-3.5-Lightning-30B-A3B | 2 | 103.6 t/s | 334.4 t/s | 136 s | 81% |
| Nemotron-3.5-Lightning-30B-A3B | 3 | 103.9 t/s | 444.4 t/s | 154 s | 78% |

## P2P bandwidth between cards

NVIDIA's `p2pBandwidthLatencyTest` (built from cuda-samples `v12.9` for `sm_60`), three cards, x16/x8/x8:

| | P2P off | P2P on |
|---|---|---|
| One-way bandwidth | 6.0-6.5 GB/s | 5.2-6.6 GB/s |
| Two-way bandwidth | 6.6-6.7 GB/s | 10.3 GB/s |
| GPU-to-GPU latency | 10.5-18.2 µs | 1.2-1.3 µs |

P2P cuts latency about 10x. With all cards at x8, the out-of-tree benchmark measured 5.1-5.2 GB/s per pair in both
directions and 3.1-4.4 GB/s per card with all three sending at once
([measurements/RESULTS.md](measurements/RESULTS.md#31-p2p-characterization-valid)).

On the 29-patch build, tensor split prompt-processed 36% faster than layer split (625.8 against 459.7 t/s) but crashed
during generation on three cards (see [TROUBLESHOOTING.md](TROUBLESHOOTING.md#-sm-tensor-crashes-on-the-second-request)).
The Kmic fork's tensor split works.
