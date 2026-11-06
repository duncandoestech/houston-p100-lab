# Troubleshooting

Problems hit on this build and what fixed them.

## GPU falls off the bus (Xid 79)

**Symptom:** `Xid 79` ("GPU has fallen off the bus") in the kernel log, then `Xid 154` (reset required). The card
vanishes from `nvidia-smi`. Once it caused a kernel panic in the `nvidia` module.

**Cause:** a PCIe riser. The card on it dropped about 7 minutes after every boot. These did not help: removing the
power limit, giving the riser its own PSU power, and `pcie_aspm=off` (P100s report "ASPM not supported", so it does
nothing).

**Fix:** remove the riser and put the card in a direct slot. Stable since.

If a card drops:

1. **Full power cycle**: shut down, switch the PSU off for 30 s. A reboot alone may not bring it back.
2. If it repeats in the same slot, suspect the **riser, cable or slot** before the card.
3. **Swap test**: move the card or swap it with a good one. If the fault follows the card, it's the card.

## No video and no network after moving cards

Two cards in the two x16 slots run x16/x16, which uses every PCIe lane on the P9X79 PRO and **disables the USB 3
ports**. A USB Wi-Fi dongle in a USB 3 port drops off the network, and with no integrated graphics the box looks like
it didn't POST. Use a USB 2 port.

## Clocks stuck at 405 MHz

One early benchmark ran with all the cards stuck at their 405 MHz idle clock and looked like a real result until it
was rerun. Cause unknown; it didn't recur. Check the SM clock under load before trusting a number: about
1,170-1,330 MHz busy here, depending on the power cap. Low power draw under load is the tell.

## `-sm tensor` crashes on the second request

`GGML_ASSERT(bcj.nodes[i])` in `ggml-backend-meta.cpp` on the request after the first, with 2 or 3 P100s, on the
29-patch `v0.4.0` build. Upstream: [ggml-org/llama.cpp#29466](https://github.com/ggml-org/llama.cpp/issues/29466).
On that build, use `-sm layer`.

It did not reproduce on the Kmic-68 fork: not in four scripted server tests, the agent batteries, or 56 short-then-long
request pairs in a 90-minute soak. That isn't proof it's fixed, but production has run tensor split on that fork since.

## MTP: "model doesn't contain MTP layers"

```
llama_init_from_model: context type MTP requested but model doesn't contain MTP layers
```

The regular quant logs `model has unused tensor blk.64.nextn.eh_proj.weight -- ignoring`, which makes it look like
the draft head is there. It isn't usable. Use an MTP-converted GGUF (for Qwen3.6, unsloth's
`Qwen3.6-35B-A3B-MTP-GGUF`; about 500 MB larger).

`llama-bench` doesn't support MTP at all; use `llama-server` or `llama-speculative-simple` to measure it.

## `llama-bench` out of memory with `-ts 1,1,1`

`llama-server` and `llama-cli` take `-ts 1,1,1`; `llama-bench` wants `-ts 1/1/1`. Given commas, `llama-bench` silently
puts every layer on GPU 0 and fails with `cudaMalloc failed: out of memory`. `-v` shows the `layer N assigned to device
CUDA0` lines.

## Hangs that weren't

- **No `-c` given:** `llama-cli` defaults to the model's full native context, and the KV allocation drove this 16 GB-RAM
  box into page-cache thrashing that looked like a hang. Always pass `-c`.
- **Automatic tensor split:** leaving `-ts` unset on 3 GPUs without NCCL sat in a `futex` wait at 0% GPU. Pass `-ts`
  explicitly.

## Out of memory on the host (16 GB RAM)

- `llama-server` defaults to 4 slots; at 262,144 context that was killed during model load. Use `--parallel 1`.
- The cross-request host-RAM prompt cache grew until the box ran out of memory. Use `--cache-ram 0
  --no-cache-idle-slots`.
- Load large models with `-lm none` instead of mmap.

## Illegal instruction / "CPU ISA level is lower than required"

The i7-3930K has no AVX2.

- A llama.cpp build with `-DGGML_NATIVE=OFF` crashed with an illegal instruction. Build native, or use a build that
  selects CPU code at runtime.
- Linking CUDA test programs through the conda gcc's sysroot stamped an `x86-64-v3` requirement into the binary.
  Compile with `nvcc` and the conda gcc, then link with the system `g++`.

## A dropped request keeps the GPU busy

The server only acts on a cancelled request between decode calls, and `-b` sets how much prompt goes into one call.
At `-b 32768` a dropped 64,000-token prompt kept running for 72-74 s. Use `-b 2048` (about 4 s).

If another client polls `/slots` several times a second, the server doesn't even notice the drop until prompt
processing ends (about 80 s at any `-b`). Poll `/health` instead.

## gppm can't lower P100 idle power

`gppm` saves idle power by forcing the GPU to P8. On the P100 that call reports success but the card stays at `P0`
and about 30 W, with or without a model loaded and with persistence on or off. The P100 (GP100, HBM2) has a single
memory clock (715 MHz) and its 125 W power floor is far above idle draw, so there's nothing to lower. gppm was built
for the P40 (GP102). It was removed.
