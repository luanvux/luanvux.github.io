---
title: "Giving the CPU the Wire: VexRiscv-Accessible 10G Ethernet on the ZC706"
date: 2026-06-18
tags:
  - fpga
  - ethernet
  - xilinx
  - litex
  - riscv
description: "Making the ZC706's 10G LiteEth MAC reachable from VexRiscv, why the first timing constraints were wrong, and how a pre-placement Vivado hook fixed the sys-to-refclk crossing."
---

## Why the CPU needs the wire

The hardware UDP generator from the previous build bypasses the CPU entirely: pure datapath, no protocol awareness. That is fine for bulk throughput, but it leaves no one to answer ARP requests or respond to ICMP ping.

`add_ethernet()` wires a wishbone MAC into the SoC so LiteX BIOS gets an `ethernet>` prompt with `ping`, `arp`, and `netboot`. Same API call used for 1G demos. Add `data_width=64` and the MAC runs at 10G width. Except it does not close timing. Two bugs.

> **Source:** [`xilinx_zc706.py`](https://github.com/luanvux/10gbe_zc706/blob/main/xilinx_zc706.py) — the LiteX SoC target with the `--with-10g` flag.

---

## The `add_ethernet` call

The call is small. The important detail is `with_timing_constraints=False`; that is intentional, not a workaround to hide the problem.

```python
self.add_ethernet(
    phy                     = self.xgmiiphy,
    data_width              = 64,
    with_timing_constraints = False,   # why False: explained below
    dynamic_ip              = False,
    local_ip                = "10.1.0.3",
    remote_ip               = "10.1.0.4",
    mac_address             = 0xaa1233445566,
)
```

With `data_width=64`, LiteX builds the MAC datapath at the same width as XGMII. The CPU side still reaches it through Wishbone; the Ethernet side runs in the `clkmgt` domain, which PG068 drives from its `coreclk_out` output.

---

## Bug 1: `add_false_path_constraints` breaks DDR timing

`with_timing_constraints=True` calls `add_false_path_constraints(cd_sys.clk, pll.clkin)`, which emits:

```tcl
set_clock_groups -asynchronous \
  -group {sys} \
  -group {clk200_p sys sys4x idelay eth}
```

That looks harmless because it is trying to separate the system clock from Ethernet. It is not harmless.

`sys` is alone in Group 1. `sys4x` ends up in Group 2 through `-include_generated_clocks` on `clk200_p`. Vivado now treats `sys` to `sys4x` as asynchronous, so the DDR3 IOSERDES paths between them go untimed. Silent timing hole, no DRC warning.

There is also a second problem: the constraint names `eth` as a clock. For 10G the generated clocks are `refclk_p`, `RXOUTCLK`, and `TXOUTCLK`. `eth` does not exist. The constraint partially applies and makes things worse.

The fix is to pass `with_timing_constraints=False` and handle the crossing explicitly.

---

## Bug 2: `refclk_p` does not exist when you think it does

The key insight is Vivado's two-pass XDC parse:

```text
synth_design
  ├─ top XDC parsed            ← create_clock for clk200_p lives here
  └─ IP scoped XDCs parsed     ← create_clock for refclk_p lives here
                                  (ten_gig_eth_pcs_pma_0.xdc, line 47)
opt_design → place_design
  └─ pre_placement Tcl runs    ← first moment refclk_p clock object exists
```

The port `refclk_p` is an FPGA pin and is declared in the top XDC from the start. But the clock object named `refclk_p` is only created by the IP's scoped XDC, which Vivado parses after `synth_design` completes.

If you write `set_clock_groups ... refclk_p` in the top XDC, `get_clocks refclk_p` returns an empty list. Vivado silently skips the constraint. No error. Without a clock group to declare them asynchronous, Vivado times the 125 MHz to 156.25 MHz crossing. It does not close.

This is not ZC706-specific. Any GTX, PCIe, or MIG IP creates its clocks in a scoped XDC. Constraints on those clocks from the top XDC are always silent no-ops.

The fix is a pre-placement hook:

```python
platform.toolchain.pre_placement_commands.append("\n".join([
    "set sys_clks [get_clocks -quiet crg_clkout0]",
    "set eth_clks [get_clocks -quiet {refclk_p *RXOUTCLK *TXOUTCLK}]",
    "if {[llength $sys_clks] == 0} {error {No crg_clkout0 clock found}}",
    "if {[llength [get_clocks -quiet refclk_p]] == 0} {error {No refclk_p found}}",
    "set_clock_groups -asynchronous -group $sys_clks -group $eth_clks",
]))
```

`pre_placement_commands` runs after `synth_design`, after all IP scoped XDCs are parsed, but before `place_design`. That is the first point in the flow where `refclk_p` exists as a clock object. The error guards are intentional: silent skip was the whole bug. Fail loud.

---

## CRG endpoint false paths

The PLL reset and CDC `MultiReg` synchronizer cells still need false paths, but not through `add_false_path_constraints`, which would re-introduce the `sys` to `sys4x` group problem.

The surgical alternative is false path to specific pin types, with no clock group statement:

```python
# FDCE D-pins (async-clear flops; in this design these are the reset tree)
# TODO: this matches all FDCE D-pins in the design, not just reset synchronizers.
# Fine for a small SoC; add a hierarchy filter if the design grows.
platform.add_platform_command(
    "set_false_path -quiet -to [get_pins -quiet FDCE/D]")
# PLL reset pin
platform.add_platform_command(
    "set_false_path -quiet -to [get_pins -quiet PLLE2_ADV/RST]")
# CDC MultiReg synchronizer first flop
platform.add_platform_command(
    "set_false_path -quiet "
    "-to [get_pins -quiet -filter {REF_PIN_NAME == D} "
    "-of_objects [get_cells -quiet -hierarchical -filter {mr_ff == TRUE}]]"
)
```

These constrain individual endpoints. No `-group` clause means `sys` and `sys4x` remain related, so the DDR3 IOSERDES paths stay timed.

---

## Result

Build with `--with-10g`. Timing summary from the route report:

```text
Timing summary:
  WNS: +0.178 ns    TNS: 0.000 ns    ← closed
  WHS: +0.011 ns    THS: 0.000 ns
```

Flash the bitstream, then open the BIOS console:

![`litex_term crossover` launched; BIOS boot scrolling in terminal](images/4_litex_term_crossover.png)

The JTAG server terminal shows Ethernet initializing and the IP address assigned:

![JTAG server terminal: BIOS boot with `Ethernet init... Local IP: 10.1.0.3`, SDRAM init](images/4_litex_jtag_server.png)

<!-- TODO: bios-ping screenshot -->

From the traffic station, ping the FPGA directly:

![Traffic station: `sudo ifconfig ens1f1 10.1.0.4` + `ping 10.1.0.3` -- replies 0.06 to 0.14 ms](images/4_server_icmp_ping.png)

VexRiscv is on the wire. The wishbone MAC handles ARP and ICMP while the Ethernet datapath runs independently in the `clkmgt` domain.

---

## What's next

The control plane is up. Next is adding DMA capacity so the MAC can move payload directly to DDR3 without the CPU touching it. Performance will be measured with `iperf3` on Linux. Without DMA, VexRiscv copies every byte and the result is CPU-bound, not link-bound. With DMA, `iperf3` becomes a real measure of what the 10G path can sustain.
