# Software

## OS and driver

Fedora 44 Server, headless.

The NVIDIA driver comes from **RPM Fusion's legacy 580xx branch**:

```
akmod-nvidia-580xx
xorg-x11-drv-nvidia-580xx-cuda
```

Fedora's regular `akmod-nvidia` pulls driver 590+, which **dropped Pascal**. With it, the P100s aren't recognized at
all. Keep the box on the `580xx` packages; a `dnf upgrade` that lets the driver move to the newest branch would break
every GPU. Installed: `580.178.04` (reports CUDA 13.0).

`nvidia-persistenced.service` is enabled.

## CUDA toolchain

The driver runs Pascal fine, but **CUDA 13's `nvcc` can't compile for `sm_60`**:
`nvcc fatal : Unsupported gpu architecture 'compute_60'`. Builds need CUDA 12.x.

Two ways that work:

- **Docker (what production uses):** `nvidia/cuda:12.9.1-devel-ubuntu22.04` has `nvcc` 12.9 and gcc 11.4, which build
  for `sm_60` with no workarounds, and ships NCCL 2.27.3. Needs the NVIDIA Container Toolkit for `--gpus all`.
- **On the host:** install CUDA 12.9 next to CUDA 13 from NVIDIA's **Fedora 41** repo (the Fedora 44 repo only has
  13.x). Only the compiler pieces are needed: `cuda-nvcc-12-9 cuda-cudart-devel-12-9 cuda-driver-devel-12-9
  cuda-nvrtc-devel-12-9 cuda-cccl-12-9 libcublas-devel-12-9 cuda-crt-12-9`. Keep that repo disabled except for the
  install. It lands in `/usr/local/cuda-12.9`.
  - CUDA 12.9's `nvcc` rejects Fedora 44's gcc 16, and `-allow-unsupported-compiler` doesn't help (gcc 16's headers
    don't parse). Use a separate gcc 12 from conda-forge (`gxx_linux-64=12`) as the CUDA host compiler only:

```
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release \
  -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=60 -DGGML_CUDA_F16=ON \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda-12.9/bin/nvcc \
  -DCUDAToolkit_ROOT=/usr/local/cuda-12.9 \
  -DCMAKE_CUDA_HOST_COMPILER=<conda-env>/bin/x86_64-conda-linux-gnu-g++
```

## llama.cpp builds

### Kmic-68 fork (production)

[Kmic-68/llama.cpp](https://github.com/Kmic-68/llama.cpp), branch `p100-optimizations`, at `ae35056eb` (tag
`p100-optimizations-b11515-ae35056`). Its own `p100-docs/` (BUILD, QUICKSTART, FINDINGS, CHANGES) are worth reading.
Built in the CUDA 12.9.1 devel image:

```
cmake -B build-nccl -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=60 \
  -DGGML_CUDA_NCCL=ON -DGGML_NATIVE=ON -DGGML_CUDA_FA_QUANTS=all
cmake --build build-nccl --config Release
```

- **Build for `sm_60` only.** The fork's P100 matvec tuning only switches on when 60 is the only architecture. At
  `ae35056eb` a build that also lists 61 or 70 fails to compile.
- `-DGGML_NATIVE=OFF` crashed with an illegal instruction on this CPU (no AVX2); build native, or use a portable build
  that picks CPU code at runtime.

### shinbunbun 29-patch build (earlier production)

[`shinbunbun/llama-cpp-p100-patches`](https://github.com/shinbunbun/llama-cpp-p100-patches): 29 patches measured on a
P100, applied to upstream tag `v0.4.0`. The repo is a Nix flake, but the patches are plain files:

```
git clone --branch v0.4.0 --depth 1 https://github.com/ggml-org/llama.cpp llama.cpp-patched
cd llama.cpp-patched
for p in ../llama-cpp-p100-patches/patches/*.patch; do
  patch -p1 -F0 < "$p" || { echo "FAILED: $p"; break; }
done
```

All 29 apply cleanly with no fuzz. Patches 19 and 20 (`fuse-pre-add-rms-norm`, `fuse-add-unary-mul`) were left off
(their upstream default): the patch set's docs say they make decode output non-deterministic on MoE models. Measured
gain over stock `v0.4.0`: about +27% decode, no change in prompt speed ([BENCHMARKS.md](BENCHMARKS.md#29-patch-build-against-stock-llamacpp)).

## GPU fan control

The shroud fans follow the hottest GPU. `gpu-fan-control.service` runs `/usr/local/bin/gpu-fan-control.sh`:

```bash
#!/bin/bash
HWMON=$(for d in /sys/class/hwmon/hwmon*; do
  [ "$(cat $d/name)" = "nct6776" ] && echo "$d"
done)

PWM="$HWMON/pwm2"
PWM_ENABLE="$HWMON/pwm2_enable"
echo 1 > "$PWM_ENABLE"

while true; do
  TEMP=$(nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits | sort -n | tail -1)
  if [ "$TEMP" -le 40 ]; then SPEED=70
  elif [ "$TEMP" -ge 75 ]; then SPEED=255
  else SPEED=$(( 70 + (TEMP - 40) * (255 - 70) / (75 - 40) )); fi
  echo "$SPEED" > "$PWM"
  sleep 5
done
```

Every 5 s it ramps the fans linearly from PWM 70 at 40 °C to full at 75 °C.

The Super I/O chip is an **nct6776** and the shroud fans are on **`pwm2`**. `pwmconfig` didn't find that reliably; it
was found by writing to each `pwmN` by hand and watching which fan responded. On different hardware, repeat that
rather than assuming `pwm2`.

For benchmark runs a second unit, `fans-full.service`, holds the fans at full (about 2,850-3,050 RPM). It isn't
enabled at boot; run only one of the two at a time.

## GPU power limit

All cards are capped at **150 W** (minimum 125 W, default 250 W). Why 150: on the production stack, 125 W was
limiting throughput (the driver's power-cap reason was active in up to 80% of busy samples during generation).

| Cap | Generation (tg512) | Prompt (pp2048) | GPU power while generating, 3 cards | Hottest card |
|---|---|---|---|---|
| 125 W | 31.25 t/s | 506.0 t/s | 356 W | 54 °C |
| 150 W | 33.29 t/s (+6.5%) | 530.4 t/s (+4.8%) | 430 W | 58 °C |
| 175 W | 33.66 t/s (+7.7%) | 549.0 t/s (+8.5%) | 484 W | 60 °C |

150 W gets most of the generation gain; 175 W adds 0.4 t/s for another 54 W.

`/etc/systemd/system/nvidia-power-limit.service`:

```ini
[Unit]
Description=Set NVIDIA GPU power limit
After=nvidia-persistenced.service
Wants=nvidia-persistenced.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/nvidia-smi -pl 150

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nvidia-persistenced.service nvidia-power-limit.service
nvidia-smi --query-gpu=index,persistence_mode,power.limit --format=csv   # expect Enabled, 150.00 W
```

With no `-i`, the limit applies to every GPU. If a different card model is added, use one
`ExecStart=/usr/bin/nvidia-smi -i N -pl W` line per GPU (check indexes with `nvidia-smi -L` first; they can shift).

## Production settings

Docker Compose, one service, Kmic-68 NCCL build, `restart: unless-stopped` (it comes back after a reboot unless it was
stopped by hand).

```
environment: GGML_CUDA_P2P=1  GGML_CUDA_GRAPHS_PRE_VOLTA=3  NCCL_P2P_LEVEL=SYS
             LLAMA_SPEC_SAMPLE_TEMP=1.0  LLAMA_SPEC_DRAFT_TOPK=20
llama-server -m Qwen3.8-27B-UD-Q6_K_XL.gguf --jinja --cache-ram 0 --no-cache-idle-slots
    --parallel 1 -c 262144 -ngl 99 -sm tensor -ts 1/1/1/1 -fa 1 -ctk q4_0 -ctv q4_0
    -b 2048 -ub 2048 -lm none -fit off
    --spec-type draft-mtp --spec-draft-n-max 3 --spec-draft-p-min 0.0
    -ngld 99 -ubd 64 -ctkd q4_0 -ctvd q4_0
    --temp 1.0 --top-k 20 --top-p 0.95 --min-p 0.0 --metrics
```

Why each setting (numbers in [BENCHMARKS.md](BENCHMARKS.md) and [measurements/](measurements/)):

| Setting | Why |
|---|---|
| NCCL build, `NCCL_P2P_LEVEL=SYS` | Prompt 276 → 500 t/s at depth 0, 211 → 316 t/s on a 129,000-token prompt; generation 29.8 → 31.2 t/s (3 cards). Without the variable NCCL picks a host-memory transport on this board. Output is repeatable but not bit-identical to the non-NCCL build. |
| `GGML_CUDA_GRAPHS_PRE_VOLTA=3` | CUDA graphs for single-token steps: decode 22.4 → 29.5 t/s on 3 cards. The fork's serving default. |
| `GGML_CUDA_P2P=1` | Direct peer copies for the tensor-split exchange. |
| `-sm tensor -ts 1/1/1/1` | Four cards against three: 64,000-token prompt 389 → 514 t/s, generation with MTP 37.4 → 42.1 t/s, 8.5 GB per card instead of about 11. |
| `-ctk q4_0 -ctv q4_0` | Smallest cache. q8_0 was 8.7% slower; f16 was 3% faster on short requests but 2.5% slower on a 64,000-token prompt and 2.9 GB more per card. q4_0 picks the same top token as f16 98% of the time, with no drift out to 65,536 tokens. |
| `-lm none` | 16 GB of RAM for a 23.5 GiB model: with mmap, generation was 11% slower and noisy, and loading took twice as long. |
| `-fit off` | Keeps the explicit `-ts` layout. |
| MTP `n-max 3`, `p-min 0.0` | Fastest under greedy and sampled decoding (the fork's reference is 4 / 0.2). MTP adds about 15% on short requests. |
| `LLAMA_SPEC_SAMPLE_TEMP=1.0 LLAMA_SPEC_DRAFT_TOPK=20` and the sampler | The fork's sampled-draft setup, paired with the model card's sampling. |
| `-b 2048` (was 32768) | The server acts on a dropped request only between decode calls. A request dropped mid-way through a 64,000-token prompt held the slot 72-74 s at `-b 32768` and about 4 s at `-b 2048`, for 0.3% of prompt speed. |
| `-c 262144` | Fits with room to spare (11.1 GB peak per card on three cards with a 129,000-token prompt). |
| `--parallel 1` | Without it the server makes 4 slots, which ran this 16 GB-RAM box out of memory during load. `-np 2` works but a second client still waits for a long prompt, and it costs 0.7-1.2 GB more per card. |
| `--cache-ram 0 --no-cache-idle-slots` | The default host-RAM prompt cache grew across large requests until the box ran out of RAM. |

Operating notes:

- **Don't poll `/slots` quickly.** With another client requesting `/slots` five times a second, the server didn't
  notice a dropped request until its prompt finished (about 80 s), at any `-b`. `/health` every 5 s has no such
  effect. Fast `/metrics` scraping might do the same; not tested.
- A container sidesteps Fedora's SELinux rule that stops systemd from running binaries under `/home` (`init_t` can't
  execute `user_home_t`). A plain systemd unit needs the binary copied under `/opt`.
