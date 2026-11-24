# Houston: a Tesla P100 local LLM server

`houston` is an old X79 desktop turned into a local LLM inference box. It was built with four NVIDIA Tesla P100
16 GB cards (64 GB of VRAM); since November 2026 it runs two, plus a Tesla M10 for small helper models. This repo is
the build notes and the measured results: what it took to get the cards working, what settings it runs, and how
fast it is.

## Now: two cards (November 2026)

The two P100s are shared between an image server and a language model, one at a time. Same agent battery, same
two cards, newest build of each engine, measured 2026-11-22 and 23:

| Model | Engine | Agent battery (9 tool-use tasks x 3) | Mean per task | Prompt / generation |
|---|---|---|---|---|
| **Inkling 2 Mini** (NVFP4-Q8_0, 512k context) | Megla's `ink2`, build `d05521d` | **27 of 27** | **57 s** | **474 / 81 t/s** |
| Qwen3.8-27B (`UD-Q6_K_XL`, 262k context, MTP) | Kmic-68's fork, build `bbad6fcfb` | 26 of 27 | 129 s | 342 / 52 t/s |

Inkling 2 Mini was faster on every task, with 39% faster prompt reading and 57% faster generation than Qwen3.8-27B.
Qwen3.8-27B on two cards was close to its four-card result (129 s against 120 s per task): prompt speed dropped about a
quarter, generation hardly at all. Settings and per-task tables:
[BENCHMARKS.md](BENCHMARKS.md#current-setup-two-cards-november-2026).

## The box

| | |
|---|---|
| GPUs | 2x Tesla P100 PCIe 16 GB (Pascal, HBM2), no NVLink; four until November 2026. Plus a Tesla M10 |
| Board / CPU | ASUS P9X79 PRO, Core i7-3930K (6 cores, 2011) |
| RAM | 16 GB DDR3-1600 |
| PSU | Corsair RM1000x |
| Cooling | 3D-printed PETG shrouds with Arctic P12 Pro fans, driven by GPU temperature |
| OS | Fedora 44 Server, NVIDIA 580xx driver, CUDA 12.9 toolchain |

Details: [HARDWARE.md](HARDWARE.md). The board needed a BIOS mod before it would map the cards at all:
[BIOS-FIX.md](BIOS-FIX.md).

## What it ran on four cards (October 2026)

Qwen3.8-27B (unsloth `UD-Q6_K_XL`, 23.5 GiB) on [Kmic-68's P100 fork of llama.cpp](https://github.com/Kmic-68/llama.cpp),
built with NCCL, tensor split across all four cards, 262,144-token context, MTP speculative decoding, 150 W per card,
in Docker. Full command and the reason for each setting: [SOFTWARE.md](SOFTWARE.md#production-settings).

| Measurement | Result |
|---|---|
| 64,000-token prompt through the server | 514 tokens/s |
| Generation with MTP, 512 tokens | 42 tokens/s (45-56 on short chats) |
| `llama-bench` pp2048 / tg512 (no MTP) | 629 / 39.2 tokens/s |
| Agent battery (9 tool-use tasks x 3) | 26 of 27 passed, 120 s mean per task |
| Memory per card | 8.5 GB of 16 GB |
| Peak power | about 650 W for the four cards, 710 W with the CPU |

All measured 2026-10-06 to 10-12. More in [BENCHMARKS.md](BENCHMARKS.md).

## What mattered

- **Above 4G Decoding had to be patched into the BIOS.** Without it, the board can't map four 16 GB cards.
- **Driver and compiler are both pinned.** Fedora's current NVIDIA driver dropped Pascal (use the 580xx branch), and
  CUDA 13's compiler can't build for `sm_60` (use CUDA 12.9).
- **Tensor split with NCCL is what makes more cards faster.** With plain layer split, one, two and three cards all
  generated at about the same speed; more cards only bought VRAM. On the Kmic fork with NCCL and
  `NCCL_P2P_LEVEL=SYS`, going from three cards to four gave +32% prompt speed and +13% generation.
- **NCCL with `NCCL_P2P_LEVEL=SYS`** raised prompt processing 51-82% on three cards. Without the variable NCCL picks
  a slower host-memory path on this board.
- **CUDA graphs for single-token steps** (`GGML_CUDA_GRAPHS_PRE_VOLTA=3`) raised decode from 22.4 to 29.5 tokens/s
  on three cards. Without them the 2011 CPU, not the GPUs, was the limit.
- **150 W per card** is the power sweet spot: +6.5% generation over the 125 W minimum. 175 W adds almost nothing.
- **`-lm none`** (no mmap): the box has 16 GB of RAM for a 23.5 GiB model, and mmap loading cost 11% of generation.
- **MTP speculative decoding** adds about 15% to generation on this model and needs an MTP-converted GGUF.
- **PCIe link width didn't matter**: x16/x16, x16/x8 and x8/x8 measured the same.
- **Avoid PCIe risers.** A card on a riser kept falling off the bus.

## Files

| File | What's in it |
|---|---|
| [HARDWARE.md](HARDWARE.md) | Parts, PCIe lanes, power wiring, cooling |
| [BIOS-FIX.md](BIOS-FIX.md) | Patching in Above 4G Decoding and unhiding PCIe bifurcation on the P9X79 PRO |
| [SOFTWARE.md](SOFTWARE.md) | Driver, CUDA toolchain, builds, fan control, power limit, production settings |
| [BENCHMARKS.md](BENCHMARKS.md) | All results, current and earlier setups |
| [TROUBLESHOOTING.md](TROUBLESHOOTING.md) | Problems hit and how they were fixed |
| [measurements/](measurements/) | Full data behind the tuning: every table, method, limits, raw summaries and scripts |

All numbers come from this one machine. The measurements were planned, run and written up with an AI coding
assistant (Claude Code) working on the machine under my direction; I chose what to measure and reviewed the
results. Check anything you rely on.
