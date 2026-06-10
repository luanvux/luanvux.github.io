---
title: "No Spare Cycles: Inside LiteEth's XGMII Reconciliation Sublayer"
date: 2026-06-10
tags:
  - fpga
  - ethernet
  - litex
  - migen
description: "A line-by-line architecture tour of liteeth/phy/xgmii.py — the open-source Reconciliation Sublayer that sits between LiteEth's MAC and a 10G PCS/PMA: inter-frame gap tracking, the deficit idle count, lane-4 shifted starts, and the RX realigner."
---

## Background

[The previous article](zc706-10g-liteeth) replaced the eval-licensed Xilinx 10G MAC on the ZC706 with LiteEth, talking to the free PG068 PCS/PMA across the XGMII boundary. I claimed there that `LiteEthPHYXGMII` "is not glue logic — it is the Reconciliation Sublayer", and promised a deep dive. This is that article.

The subject is a single file: [`liteeth/phy/xgmii.py`](https://github.com/enjoy-digital/liteeth/blob/master/liteeth/phy/xgmii.py), about 700 lines of Migen by Leon Schuermann. It converts between LiteX's generic byte stream and the 64-bit XGMII bus that every 10G PCS/PMA understands — and in doing so it implements the parts of IEEE 802.3 Clause 46 that most tutorials wave away: inter-frame gap enforcement, start-lane alignment, and the deficit idle count.

Why care about the internals? Two reasons. First, if you connect any PCS/PMA-only IP (PG068, or another vendor's equivalent), this module *is* your compliance layer — when something on the wire looks wrong, this is where you debug. Second, it is a beautifully compact case study in designing for a datapath with **zero slack**:

> 156.25 MHz × 64 bits = 10 Gb/s — exactly line rate. Every bus word matters; one wasted cycle per packet is bandwidth you never get back, and one missing cycle on RX is a lost frame.

Almost every design decision in this file falls out of that single constraint.

---

## What Clause 46 actually asks of a transmitter

XGMII itself is trivial: 64 data bits plus 8 control bits per cycle, one control bit per byte. When `ctl[i]=1`, byte `i` is a control character — `0x07` IDLE, `0xFB` START, `0xFD` END (here as seen in the 64-bit clock-domain form; on a DDR 32-bit XGMII these are two transfers). The hard part is the *protocol* the Reconciliation Sublayer must speak over it:

| Clause 46 requirement | Where it lives in `xgmii.py` |
|---|---|
| START character may only occupy **lane 0** — byte 0 or byte 4 of a 64-bit word | the unshifted/shifted start paths in the TX FSM |
| START replaces the first preamble byte; END follows the last frame byte | `unshifted_idle_transmit`, the per-byte TRANSMIT encoder |
| Inter-frame gap of ≥ 96 bit-times (12 bytes)… | `current_ifg` range tracker |
| …on **average** — the deficit idle count (46.3.1.4) may shorten individual gaps | `current_dic` / `last_packet_rem` |
| RX must tolerate starts on either lane | `LiteEthPHYXGMIIRXAligner` |

The lane-0 rule is what creates all the complexity. A frame whose length is not a multiple of 4 cannot be followed by *exactly* 12 idle bytes and still have the next frame start on a legal lane — you either pad the gap (wasting bandwidth) or borrow against the deficit idle count. Everything interesting in the TX path is machinery for making that choice every single cycle.

---

## The module at a glance

![LiteEthPHYXGMII top-level block diagram — TX, RX with aligner, CRG, and the PCS/PMA below](images/xgmii-top-block.svg)

`LiteEthPHYXGMII` bundles four pieces:

- **`LiteEthPHYXGMIITX`** — stream in, XGMII out. Owns the IFG state, the DIC, and a two-state FSM (`IDLE`/`TRANSMIT`).
- **`LiteEthPHYXGMIIRX`** — XGMII in, stream out. A realigner, a one-word lookahead, and a two-state FSM (`IDLE`/`RECEIVE`).
- **`LiteEthPHYXGMIIRXAligner`** — normalises lane-4 starts to lane 0 so the RX FSM only ever sees aligned frames.
- **`LiteEthPHYXGMIICRG`** — clocks `eth_tx`/`eth_rx` from the PCS/PMA's recovered clocks (or the sim clock in model mode).

One consequence worth flagging early: the PHY sets `self.integrated_ifg_inserter = True`, and `LiteEthMACCore` checks that flag and **skips its own gap inserter** on TX (`liteeth/mac/core.py`). Gap management has to live here, in the PHY's clock domain, because only the RS knows about lanes and the DIC. If you ever wonder why the usual `LiteEthMACGap` module is missing from a 64-bit XGMII build — this is why.

The stream interface on both sides is `eth_phy_description(64)`: `data[64]`, `last`, and `last_be[8]` — a one-hot marker for the last valid byte of the frame (`0x01` = only byte 0 valid, `0x80` = all eight valid). The TX is hard-wired to `dw=64`; an `assert` at the top makes sure.

### A note on the preamble before reading any hex

LiteEth defines `eth_preamble = 0xd555555555555555`, while IEEE 802.3 writes the preamble+SFD as `AA AA … AB`. Both are the same bits. The 64-bit register is little-endian — byte 0 (`0x55`) is `tx_data[7:0]` and goes out first — and each byte is serialised LSB-first, so `0x55` (`01010101`) appears on the wire as `0xAA` and the final `0xD5` as `0xAB`. No bit-reversal logic exists anywhere in the file; the PCS/PMA's serialisation order does the work. Keep this in mind when reading waveforms: register `55…D5` *is* the textbook preamble.

---

## TX, part 1 — the inter-frame gap is a range, not a number

The obvious implementation of "maintain a 12-byte gap" is a byte counter. The RS doesn't bother, because transmissions can only ever resume on byte 0 or byte 4 — gap sizes are only ever *decided* at 4-byte granularity. So the whole IFG state is two bits:

```python
# - 0: less than  4 bytes of IFG transmitted
# - 1: less than  8 bytes of IFG transmitted
# - 2: less than 12 bytes of IFG transmitted
# - 3: 12 or more bytes of IFG transmitted
current_ifg = Signal(max=4, reset=3)
```

Three strobe signals mutate it — `ifg_reset` (a transmission starts), `ifg_add_single` (+4 bytes, saturating at 3), `ifg_add_double` (+8 bytes) — and crucially the *next* value is computed combinationally as `next_ifg` before being registered. That lookahead is what lets the FSM decide `sink.ready` for the following cycle without ever stalling: by the time the next data word could arrive, the module already knows which start branch it would take.

A detail that surprises people: the END character **counts toward the IFG** (the START does not). So a frame ending at byte 1 of a word has already banked 7 idle-equivalent bytes by the end of that same cycle.

---

## TX, part 2 — the deficit idle count

Suppose a frame's length mod 4 is 1. After its END character the gap grows in 4-byte steps, so the achievable gaps before the next legal start are 11 bytes or 15 — never 12. Clause 46.3.1.4 resolves this with the **deficit idle count**: you may emit the 11-byte gap, provided you owe the difference back later, and your running deficit never exceeds 3 bytes.

LiteEth implements it as the code comment's "bounded counter of deleted XGMII idle characters":

```python
last_packet_rem = Signal(max=4)              # previous frame's length % 4
current_dic     = Signal(max=4, reset=3)     # deficit, bounded 0..3
```

The update rules are written into the FSM's start branches:

- **Shorten a gap** (start early): `current_dic += last_packet_rem` — allowed only while `current_dic + last_packet_rem <= 3`.
- **Stretch a gap** (start late, or full 12+ bytes seen): `current_dic = max(0, current_dic - last_packet_rem)` — extra idles pay the debt down.
- **Sit idle past a full gap** with no data pending: `current_dic = 0` — the debt is fully repaid by waiting.

The reset value of 3 is the conservative corner: at power-up the counter pretends the deficit is already maxed out, so the very first frame can never shorten its gap.

If you instantiate the PHY with `dic=False`, `last_packet_rem` becomes `Constant(0)` — note this does **not** disable gap enforcement; it just makes every "borrow" branch unreachable and lets the synthesiser sweep the arithmetic away. Frames whose length is a multiple of 4 behave identically either way.

---

## TX, part 3 — the FSM: four ways out of IDLE

![LiteEthPHYXGMIITX transmit FSM — branches A, B1, B2, C out of IDLE](images/xgmii-tx-fsm.svg)

The FSM has just two states, but `IDLE` contains a four-way priority decision, taken the moment `sink.valid` is high:

| Branch | Gap state | Condition | Start lane | DIC effect |
|---|---|---|---|---|
| **A** | `ifg == 3` (≥12 B) | always | 0 (unshifted) | pay down |
| **B1** | `ifg == 2` (8–11 B) | `rem != 0 & dic + rem ≤ 3` | 0 (unshifted) | borrow |
| **B2** | `ifg == 2` | otherwise | 4 (shifted) | pay down |
| **C** | `ifg == 1` (4–7 B) | `rem != 0 & dic + rem ≤ 3` | 4 (shifted) | borrow |
| — | else | — | keep idling | reset if `ifg ≥ 2` |

Branch A is the easy life: a full gap has elapsed, so emit `tx_ctl=0x01`, `tx_data = Cat(XGMII_START, sink.data[8:64])` — START overwrites preamble byte 0, the remaining seven preamble bytes ride along — and go to `TRANSMIT`:

![Branch A — unshifted start after a full IFG, END landing mid-word](images/xgmii-wave-unshifted.svg)

Branch B is where the DIC earns its keep. With only 8–11 bytes of gap banked, the module can either start *now* on lane 0 (B1, borrowing `rem` bytes of deficit) or wait for the lane-4 slot in the same word (B2), which guarantees 12+ bytes. B2 emits `tx_ctl=0x1F`: four IDLEs, then START, then the first three preamble bytes — the frame begins in the *upper half* of the bus word:

![Branch B2 — shifted start on lane 4 when the DIC budget is exhausted](images/xgmii-wave-shifted.svg)

Branch C is the aggressive end of the spectrum — only 4–7 bytes of gap, start shifted anyway if the deficit allows. The waveform below shows the tightest legal case: a frame ends at byte 1 (`rem=1`, banking 7 bytes of gap in that word), and the next frame starts on lane 4 of the *very next word* for a total gap of 11 bytes, with the deficit ticking from 0 to 1:

![Branch C — DIC borrow producing back-to-back frames with an 11-byte gap](images/xgmii-wave-dic.svg)

When none of the branches fire, IDLE drives `0xFF` / eight IDLE characters, advances the gap by 8 bytes, and — this is the part that keeps the pipe full — uses `next_ifg` to *pre-compute* whether the following cycle's `sink.valid` would be accepted, registering `sink.ready` accordingly. The handshake decision is always made one cycle ahead of the data.

---

## TX, part 4 — the shifted datapath: stitching half-words

![TX datapath — last_be masking, the one-cycle shift buffer, and the aligner mux feeding the FSM](images/xgmii-tx-datapath.svg)

A lane-4 start creates a permanent misalignment: the sink delivers 64 valid bits per cycle, but the wire wants them offset by four bytes. Rather than tracking an offset through the FSM, the module keeps a one-cycle delay register and a mux:

```python
If(transmit_shifted,
    adjusted_sink_valid.eq(prev_valid),
    adjusted_sink_valid_data.eq(Cat(
        prev_valid_data[(dw // 2):],   # upper half of the previous beat → low bytes
        sink.data[:(dw // 2)],         # lower half of the current beat  → high bytes
    )),
    ...
```

(Recall Migen's `Cat()` is LSB-first — the *first* argument lands in the low bits, the opposite of Verilog concatenation.)

So in shifted mode every transmitted word is half "yesterday's data", half "today's". The same stitch is applied to `last_be`, after one important hygiene step: `last_be` is masked to zero whenever `last` is not asserted, because the stream contract only defines it on the final beat — an unmasked stale value would end the frame early.

Two consequences of this delay-register trick are handled explicitly rather than accidentally:

- On the first `TRANSMIT` cycle in shifted mode, `adjusted_sink_valid` (= `prev_valid`) is still low. That's fine — the IDLE state already drove that word's pads (4 idles + START + preamble) on its way out.
- If, in shifted mode, the sink asserts `last` with the final byte in the **upper half** (`last_be & 0xF0`), the module must *stop requesting data* even though the current word isn't fully transmitted — the leftover half is already captured in the delay register and will go out next cycle. Missing this would have dropped the tail of every such frame.

### Ending a frame

Inside `TRANSMIT`, each of the eight output bytes is encoded independently against the (adjusted) `last_be` — a generate-style Python loop producing an `If/Elif/Else` per byte:

- `last_be == 0` or `last_be ≥ (1 << i)` → data byte, `ctl[i]=0`;
- `last_be == (1 << (i-1))` → this is the first byte *past* the frame → **END**, and as a side effect: `ifg_add_single` if the END sits in the lower five bytes, and `last_packet_rem ← i % 4` for the DIC;
- anything after that → IDLE.

One case has no room for the END character: a frame whose length is a multiple of 8 fills its final word completely (`last_be == 0x80`). The FSM raises `end_transmission` and spends one more `TRANSMIT` cycle emitting `{END, 7×IDLE}` — a word that conveniently banks 8 bytes of gap, so even this "extra" cycle wastes nothing:

![Frame length divisible by 8 — dedicated END word via end_transmission, still reaching a 12-byte gap with a shifted start](images/xgmii-wave-end.svg)

---

## RX — realign first, then everything is easy

The receive side could have mirrored the TX's complexity — tracking whether the current frame started on lane 0 or lane 4 through every downstream decision. Instead it spends one small FSM up front to make the problem disappear:

![RX aligner FSM (NOSHIFT/SHIFT) and the receive FSM (IDLE/RECEIVE)](images/xgmii-rx-fsm.svg)

`LiteEthPHYXGMIIRXAligner` watches for a START on byte 4. When it sees one it outputs a fully-idle word for that cycle, latches the upper half, and from then on emits `{latched half, current lower half}` — the mirror image of the TX stitch — until a lane-0 START switches it back. The transitional all-idle word is safe precisely because of an Ethernet guarantee: a receiver sees a minimum 5-byte interpacket gap, so the inserted idles can never overwrite the tail of a previous frame.

Downstream of the aligner, the RX FSM only ever deals with lane-0 frames, and three mechanisms do the rest:

**A one-word lookahead.** The aligned bus is registered once, so the FSM examines word *N* while word *N+1* is already visible. This solves an otherwise awkward problem: a frame whose END lands on byte 0 of the *next* word has no in-band end marker in its final data word — without lookahead, `last` could not be asserted on the right beat.

**A priority END scan.** A `reduce`-built `If/Elif` chain finds the lowest-indexed END character and converts its position to the one-hot `last_be` — exactly inverting the TX encoding. No END anywhere yields `last_be = 0x80` internally, i.e. "all eight bytes valid".

**Preamble matching by full-word compare.** In `IDLE` the FSM matches `ctl == 0x01` and the entire 64-bit `{START, preamble[63:8]}` pattern in one comparison, then emits a *regenerated* constant preamble to the source — byte 0 is the START on the bus but `0x55` on the stream. The flip side: any corruption in the preamble word means no match, and the frame is silently ignored. There is no error reporting at this layer; that's a deliberate simplification (a frame you couldn't lock onto is a frame you can't meaningfully flag, and the MAC's CRC check guards everything after the preamble).

And the asymmetry that explains the whole RX design: **`source.ready` is never checked**. You cannot backpressure the wire — at 10G the words arrive every 6.4 ns whether you're ready or not. The RX simply asserts `valid` and trusts the MAC-side FIFO to be drained fast enough; sizing that buffer is the system integrator's job, not the RS's.

---

## Stepping back

Two states per FSM, two bits of gap state, two bits of deficit, one delay register per direction. The file reads as dense, but the architecture is small — what makes it feel intricate is that every piece is shaped by the no-spare-cycles constraint:

- The IFG tracker is 2 bits **because** starts only happen on two lanes.
- The DIC exists **because** padding every gap to the next lane boundary would cost up to 3 bytes per frame — at minimum frame size, roughly 3–4% of the link.
- The shifted-start stitch exists **because** waiting for the next lane-0 slot would cost 4 bytes per frame instead.
- `sink.ready` is computed from a lookahead **because** a cycle of handshake latency is a cycle of lost line rate.
- The RX aligner inserts idles **because** the only free real estate on a 10G wire is the gap the standard already guarantees.

If you're auditing this module against IEEE 802.3, the mapping table at the top of this article is the checklist; if you're porting the approach to another width (32-bit XGMII, or 25G), the things that change are exactly the things derived from the 64-bit/two-lane geometry — the IFG granularity, the branch set, and the stitch — while the DIC algorithm itself carries over unchanged.

For the test side, `test/test_xgmii_phy.py` in the LiteEth tree drives this module with frames of every length mod 8 and checks gap legality on the simulated wire — a good starting point if you modify any of the branch logic.

---

## What's next

The performance question from the previous article still stands: the ZC706 link is up and the datapath is now fully understood, so the follow-up is measurement — driving UDP traffic through the LiteEth crossbar at increasing rates and seeing how close this RS + MAC combination gets to the 10 Gb/s the math says is there.
