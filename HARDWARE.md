# Hardware

## Parts

| Part | What | Notes |
|---|---|---|
| Motherboard | ASUS P9X79 PRO (X79, LGA2011) | Needed a BIOS mod for Above 4G Decoding; see [BIOS-FIX.md](BIOS-FIX.md) |
| CPU | Intel Core i7-3930K (Sandy Bridge-E, 6c/12t, 3.2 GHz) | No AVX2. Kept instead of a Xeon E5 v2: the work is GPU-bound |
| RAM | 16 GB DDR3-1600 (4x 4 GB), no ECC | Less than the model size, which is why production loads with `-lm none` |
| GPUs | 4x Tesla P100 PCIe 16 GB (GP100) | 64 GB VRAM as four separate pools; no NVLink on the PCIe card, so all cross-card traffic is PCIe |
| PSU | Corsair RM1000x, 1000 W 80+ Gold (refurbished) | |
| Case | NZXT Source 530 full tower | Room for full-length double-wide cards |
| Fans | Arctic P12 Pro 120 mm in 3D-printed PETG shrouds | |

## PCIe layout

- **Four cards:** all at PCIe Gen 3 x8, every pair `PHB` in `nvidia-smi topo -m`, P2P working on every pair. The top
  x16 slot is split x8x8 using the bifurcation option unhidden in [BIOS-FIX.md](BIOS-FIX.md#pcie-bifurcation).
- **Three cards** came up x16/x8/x8 with a riser, and x8/x8/x8 once the riser was removed.
- **Two cards in the two x16 slots** run x16/x16, but **that uses every PCIe lane on the board and disables the USB 3
  ports**. USB 2 keeps working.
- Link width made no measurable difference to inference speed (x16/x16, x16/x8 and x8/x8 within noise; see
  [BENCHMARKS.md](BENCHMARKS.md#pcie-link-width)).
- Bus addresses shift when cards move. Identify cards by `nvidia-smi -L` (index and UUID), not PCI address.

**The USB 3 trap:** the box is headless (no integrated graphics, and P100s have no video output) and its network is a
USB Wi-Fi dongle. With the dongle in a USB 3 port, the x16/x16 layout boots with no video and no network, which looks
exactly like a failed POST. Keep the dongle on a USB 2 port, and keep a cheap PCIe display card around for debugging.

## Power

- **Each P100 takes an 8-pin EPS connector, not PCIe 8-pin.** One PCIe 8-pin lead is rated for 150 W and the card's
  board power is 250 W, so each card is fed by **two** PCIe 8-pin leads combined into one EPS 8-pin with an adapter.
  A single PCIe lead into the EPS socket is a fire risk under sustained load.
- **Power limit: 150 W per card**, set at boot (see [SOFTWARE.md](SOFTWARE.md#gpu-power-limit)). The cap is an
  average: single cards still peaked at 169-177 W for moments.
- Peak measured draw with four cards: about 650 W for the GPUs, about 710 W with the CPU (71% of the PSU).

## Cooling

The P100 is passive. It expects front-to-back server airflow that a tower case doesn't provide.

- **3D-printed shrouds in PETG**, not PLA: they sit against the heatsink exhaust and get warm.
- **Arctic P12 Pro fans**, chosen for static pressure (6.9 mmH2O) over airflow, because the fin stack is dense.
- Fan speed follows the hottest GPU through a small systemd service, not the board's fan curve, which can't see GPU
  temperatures. See [SOFTWARE.md](SOFTWARE.md#gpu-fan-control).
- Measured: at 150 W with fans at full, cards ran 34-39 °C idle and peaked at 65-68 °C under sustained prompt
  processing. In the earlier three-card layout the middle-slot card ran 15-20 °C hotter than the others and once
  throttled to 810 MHz at 80 °C on a long dense-model run, so card position matters.

## Risers

**Avoid PCIe risers for the GPUs.** A card on a riser fell off the bus (Xid 79) about 7 minutes after every boot,
even with the riser on its own power. Moved to a direct slot, it has been stable. See
[TROUBLESHOOTING.md](TROUBLESHOOTING.md#gpu-falls-off-the-bus-xid-79).
