# BIOS fixes for the ASUS P9X79 PRO

Two independent mods: patching in **Above 4G Decoding** so the board can map the cards' 16 GB BARs, and unhiding the
**PCIe bifurcation** menu so one x16 slot can run two cards.

Rollback for both: re-flash a known-good `.CAP` with USB BIOS Flashback, or clear CMOS. Neither survives a stock ASUS
BIOS update; they would have to be redone against the new image.

## Above 4G Decoding

### The problem

Each P100 has a 16 GB BAR (PCI Base Address Register). Several of them are far more than the 4 GB of 32-bit address
space, so each BAR has to be mapped 64-bit, above 4 GB. That's the **Above 4G Decoding** setting, and it isn't in
this board's BIOS menu at all.

X79-era ASUS boards have an **MMIOH (Memory Mapped I/O High) limit bug**: enabling Above 4G Decoding without fixing it
gives a board that POSTs and then shows a black screen until CMOS is cleared. ASUS appears to have hidden the option
instead. So there are two problems: fix the MMIOH limit in the firmware, then make the option reachable.

### 1. Dump the live BIOS

```
flashrom -p internal -c "W25Q64BV/W25Q64CV/W25Q64FV" -r bios-dump.rom
```

- Fedora blocks this by default; boot with `iomem=relaxed` on the kernel command line.
- Read it twice and compare the two dumps, so a flaky read isn't mistaken for a good one.

### 2. Patch the MMIOH limit

Use the UEFIPatch patch **"Extend MMIOH limit to fix Above 4G Decoding (X79)"** from
[xCuri0/ReBarUEFI](https://github.com/xCuri0/ReBarUEFI).

Apply it to the **official ASUS 4801 `.CAP`**, not the raw `flashrom` dump. The raw dump lacks the ~2 KB ASUS capsule
header, and USB BIOS Flashback silently rejects a file without it.

### 3. Transplant the patched modules

1. In **UEFITool 0.28.0**, extract from the patched image as `.ffs`:
   - `IvtQpiandMrcInit` (it appears in two firmware volumes; take both)
   - `Runtime` (carries a `CpuIo2` patch the fix also needs)
2. In **MMTool 4.50.0.23**, replace those modules in a fresh, unmodified copy of the 4801 CAP.
3. Copy the board's identity module (GUID `FD44820B...`, MAC address and serial) from your own `flashrom` dump into
   the rebuilt CAP, so the board keeps its identity instead of ASUS's placeholders.

### 4. Flash

Rename to `P9X79PRO.CAP` and flash with **USB BIOS Flashback** (needs no CPU or OS). It booted normally afterwards.

### 5. Turn the hidden option on

The patched firmware fixes the bug but the option is still hidden.

1. Run **IFRExtractor-RS** on the Setup form data to find the option's NVRAM location: variable `Setup`, offset `0x1`.
2. Put `modGRUBShell.efi` on a USB stick as `EFI/Boot/bootx64.efi` (from grub-mod-setup_var).
3. Disable Secure Boot temporarily and boot the stick.
4. Run `setup_var 0x1 0x1`. Read the value before and after to confirm the write landed.

### Check it

```
lspci -vvv | grep -A20 P100
```

Every card should show `Region 1: Memory at ... (64-bit, prefetchable) [size=16G]` at its own address above 4 GB.

## PCIe bifurcation

The bifurcation settings (`IOU1/IOU2/IOU3 - PCIe Port`) exist in this BIOS, but the menu that holds them
(`Advanced > System Agent Configuration > IOH Configuration`) is hidden. Here the feature works; it just has no UI.

### Find the guard bytes

A [community guide](https://winraid.level1techs.com/t/guide-adding-bifurcation-support-to-asus-x79-uefi-bios/33018)
explains the method: find the `SuppressIf` block that hides the menu and flip it from `True` to `False`. Its byte
patterns are from a different board and BIOS (Sabertooth X79, 4701) and don't match this image, because the IDs in
those opcodes are assigned per build. Decompile the Setup module's form data with **IFRExtractor-RS** instead:

```
SuppressIf  { 0A 82 }
    True  { 46 02 }
    Ref Prompt: "PCI Subsystem Settings" ... FormId: 0x40E { 0F 0F AA 0A B0 0A 01 00 00 00 FF FF 00 0E 04 }
End

SuppressIf  { 0A 82 }
    True  { 46 02 }
    Ref Prompt: "System Agent Configuration" ... FormId: 0x41D { 0F 0F 38 08 39 08 04 00 00 00 FF FF 00 1D 04 }
End
```

There are two "System Agent Configuration" forms. `0x41C` is visible and only has link-speed and VT-d options.
`0x41D` is the full one and the only one that links to `IOH Configuration`, so unhiding `0x41D` is enough.

### The patch

Change the opcode byte `46` (`True`, always hide) to `47` (`False`, never hide) at both guards. Nothing else changes.

- Setup module: File GUID `899407D7-99FE-43D8-9A21-79EC328CAC21`, its compressed Freeform section with subtype GUID
  `97E409E6-4CC1-11D9-81F6-000000000000` (the HII form data).
- In that section's decompressed body (859,627 bytes): offsets `0xC7B86` ("PCI Subsystem Settings") and `0xC7BC9`
  ("System Agent Configuration"), `46` to `47`.

Rebuild with **UEFIReplace 0.28.0**, which handles recompression and checksums:

```
UEFIReplace WORKING_p9x79pro.cap 899407D7-99FE-43D8-9A21-79EC328CAC21 18 \
  setup_hii_patched.bin -o P9X79PRO_bifurcation.cap
```

The output was the same size as the original (8,390,656 bytes). Comparing all 197 firmware files by GUID against the
original showed only the Setup file changed, and only at the two guards. Flash with USB BIOS Flashback as
`P9X79PRO.CAP`.

### Result

New menu: `Advanced > System Agent Configuration > IOH Configuration > IOUn - PCIe Port`. `IOU1` offers `x4x4` or
`x8`; `IOU2` and `IOU3` offer `x4x4x4x4`, `x4x4x8`, `x8x4x4`, `x8x8` and `x16` (default `x16`).

On this board **IOU2 is the top slot**. With IOU2 at x8x8, its root ports (`00:02.0` and `00:02.2`) came up as two
independent Gen 3 x8 links:

```
lspci -tv
lspci -s 00:02.0 -vvv | grep -E 'LnkCap|LnkSta'
lspci -s 00:02.2 -vvv | grep -E 'LnkCap|LnkSta'
```

IOU3's root ports (`00:03.0` and `00:03.2`) were already x8x8 before the mod.

A CMOS clear resets IOU1/2/3 to their defaults (x8/x16/x16) but does not re-hide the menu, so the x8x8 setting just
needs to be set again.
