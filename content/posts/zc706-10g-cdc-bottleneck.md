---
title: "The Ping That Lied: Finding the Bottleneck Behind a 10G Link"
date: 2026-06-23
tags:
  - fpga
  - ethernet
  - xilinx
  - litex
  - riscv
  - linux
description: "Ping and ARP worked from the first article, but iperf3 returned zero. Tracing it from a clean PCS to a clock-domain mismatch in LiteEth's RX FIFO, a DDR3 ceiling that blocks the obvious fix on this board, and a separate, already-known Wishbone throughput limit underneath it all."
---

## Ping isn't the test

[The previous article](zc706-10g-vexriscv-wire) closed with VexRiscv answering ARP and ICMP over the PG068 10G link. That felt like proof the link worked. It wasn't — it was proof the control plane worked. Ping sends one small frame, waits, sends another. It never asks the datapath to sustain anything.

`iperf3 -c 10.1.0.3 -R` asked. Nine seconds, 0.00 bits/sec, then the connection died mid-result-exchange.

> **Source:** [`xilinx_zc706.py`](https://github.com/luanvux/10gbe_zc706/blob/main/xilinx_zc706.py) — the SoC target. [`liteeth/mac/core.py`](https://github.com/enjoy-digital/liteeth/blob/master/liteeth/mac/core.py) — the MAC's RX/TX clock-domain crossing. `boards.py` in `linux-on-litex-vexriscv` — the board's `sys_clk_freq`.

---

## Getting iperf3 onto the board

Short version: the existing rootfs had no `iperf3`, no SSH, no root password. Rebuilding it via Buildroot against an unpinned `master` clone meant rebuilding it twice — GCC 13.x had been dropped, and the replacement pulled in a host-Python/expat pairing with an ABI mismatch. Pinning to the `2024.02.11` LTS tag fixed both in one move. LTS tags are tested package sets; unpinned HEAD is not.

One more bug on the way: the new `rootfs.cpio` was 10.7 MB, the old one 3.7 MB. `litex_json2dts_linux.py` bakes `initrd_size = os.path.getsize(initrd)` into the DTB once, at generation time — and the DTB had been built before the new file was actually in place. Kernel reserved memory for the old size, truncated the cpio unpack partway through the new one, and `libresolv.so.2` happened to land past the cut. `busybox` and `init` landed before it and worked fine, which is the worst possible failure signature: looks like a missing-library bug, is actually a stale-size bug. Regenerating the DTB after the real file was in place fixed it.

None of that touches the 10G datapath. It's plumbing. The interesting bug is what iperf3 found once it could actually run.

---

## Ruling out the PCS

First instinct: signal integrity. GTX, SFP+, Si5324 refclk jitter — any of those could plausibly drop bits under sustained load that isolated pings never stress.

Isolated pings at increasing payload size said otherwise, but not cleanly:

```
56 B payload (98 B frame):  0% loss
60 B payload (102 B frame): 0% loss
70 B payload (112 B frame): 20% loss
80 B payload (122 B frame): 90% loss
90 B payload (132 B frame): 100% loss
```

Not a hard cutoff. A smooth curve. And 112 bytes is an exact multiple of the 8-byte XGMII beat width — if this were a `last_be`/alignment bug, that size should be clean. It wasn't.

`XilinxXGMII` exposes PCS status directly as CSRs, wired straight from the IP's `core_status` output:

```python
self._core_status = CSRStatus(fields=[
    CSRField("pcs_block_lock", size=1, offset=0),
    ...
])
```

Polled `xgmii_core_status` over the JTAG bridge in a tight loop while firing the failing ping size. `pcs_block_lock` held at 1 the entire time. PCS layer clean. The bug is downstream of a working PHY.

---

## Bug: an 8 Gbps drain on a 10 Gbps line

`LiteEthMACCore`'s RX path crosses clock domains through a `stream.ClockDomainCrossing` FIFO, depth 32 by default:

```python
# liteeth/mac/core.py
# - rx_cdc_depth: Depth of RX CDC FIFO (default: 32).
rx_cdc = stream.ClockDomainCrossing(eth_phy_description(dw),
    cd_from  = "eth_rx",
    cd_to    = "sys",
    depth    = rx_cdc_depth,
    buffered = rx_cdc_buffered,
)
```

`eth_rx` is renamed to `clkmgt` on this board — PG068's recovered `coreclk`, 156.25 MHz. At 64-bit width that's exactly 10 Gbps line rate on the write side. `sys` was 125 MHz. At 64-bit width, that's 8 Gbps on the read side.

Write rate 10 Gbps, drain rate 8 Gbps. A 32-entry FIFO does not fix a permanent 20% rate deficit — it only decides how long a single frame takes to overflow it. Longer frames spend more cycles writing before they finish, giving the deficit more time to accumulate past depth 32. Short frames finish before that happens. That's the curve above, exactly: clean at 98 bytes, dead at 132.

Under iperf3's sustained back-to-back traffic, frames don't get the gap between isolated pings that lets the FIFO drain back to empty. Backlog from one frame carries into the next. The deficit never resets. Throughput collapses to zero almost immediately — which is exactly what `-R` showed.

Not a PHY bug, not a signal-integrity bug. An architectural one: the CPU's clock domain was never fast enough to drain its own MAC's PHY-side write rate.

### The fix has a ceiling of its own

Obvious move: raise `sys_clk_freq`. First attempt, 250 MHz, failed during elaboration — not a timing-closure warning, a hard `ValueError` out of `litedram`:

```python
elif memtype == "DDR3":
    f_to_cl_cwl[800e6]  = ( 6, 5)
    f_to_cl_cwl[1066e6] = ( 7, 6)
    f_to_cl_cwl[1333e6] = (10, 7)
    f_to_cl_cwl[1600e6] = (11, 8)
    f_to_cl_cwl[1866e6] = (13, 9)
    # nothing past 1866e6 — DDR3's actual JEDEC ceiling, not a margin choice
```

K7DDRPHY's `nphases=4` maps `sys_clk_freq` to an effective DDR3 rate of `8 × sys_clk_freq`. 250 MHz asks for 2000 MT/s — past what DDR3 silicon is specified to do, not just past this board's comfort margin. Solving for the actual ceiling: `sys_clk_freq ≤ 1866e6 / 8 ≈ 233 MHz`.

Tried 200 MHz next. `8 × 200e6 = 1600 MT/s` — a real DDR3 bin, margin under the JEDEC ceiling, so elaboration passed this time. Implementation did not: DDR3 timing failed to close at 200 MHz on this board. A different failure than 250 MHz's — that one was a hard `ValueError` before synthesis even started, this one is a real place-and-route timing miss. Being inside the JEDEC table is necessary, not sufficient; it says the CL/CWL parameters exist, not that this specific part closes timing at that rate.

```python
class ZC706(Board):
    soc_kwargs = {"uart_name": "crossover", "with_jtagbone": True,
                  "sys_clk_freq": int(125e6)}   # unchanged — 200e6 broke DDR3 timing closure
```

---

## Not a bug — a known ceiling: the 2-slot turnstile

Independent of whether the CDC mismatch above ever gets fixed, there's a second limit already on record from the architecture itself, and it has nothing to do with clock domains. `add_ethernet()` wires the MAC in Wishbone-slot mode: 2 RX slots, 2 TX slots, 2048 bytes each — confirmed directly from this build's `csr.csv`:

```
constant,ethmac_rx_slots,2
constant,ethmac_tx_slots,2
constant,ethmac_slot_size,2048
memory_region,ethmac,0x80000000,8192,io+linker
```

`8192 = 2×2048 (RX) + 2×2048 (TX)`. There is no DMA engine here — every byte of every frame that clears the CDC FIFO still has to leave through VexRiscv issuing Wishbone loads and stores, word by word, against those four slots.

A classic 32-bit Wishbone bus needs roughly 2 cycles per word transfer. Back of the envelope, at the current 125 MHz:

```
125 MHz / 2 cycles × 4 bytes ≈ 250 MB/s ≈ 2 Gbps   (raw bus, before driver/IRQ/protocol overhead)
```

That number scales with `sys_clk_freq`, but the mechanism doesn't change — a faster clock makes the turnstile spin faster, it's still a turnstile. Even a CDC fix that built cleanly would only have raised the FIFO's drain rate; it would not have widened the four Wishbone slots frames have to pass through next. Once enough frames are in flight, this is the wall behind the wall.

---

## Result

Neither candidate frequency survived the build. 250 MHz never reaches synthesis — it fails at elaboration, in `litedram`, before timing is even a question. 200 MHz reaches implementation and fails there: DDR3 timing does not close on this board at that rate. The CDC theory above is still believed correct — the ping-loss curve and the zero `-R` result line up with it exactly — but it remains unconfirmed on hardware, because the obvious fix for it doesn't build. `sys_clk_freq` stays at 125 MHz for now.

---

## What's next

Two open threads, not one. The CDC mismatch still needs a fix that doesn't run into the DDR3 ceiling — raising `sys_clk_freq` was the obvious move and it's currently blocked on this board. And independent of that, the Wishbone-slot ceiling above is unaffected by anything done here; closing it eventually means a DMA-capable MAC path, not a clock change. Both are still open.
