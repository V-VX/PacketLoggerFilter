# 002: Cross-Device Counter2 Analysis & Alternative Format Investigation

**Date:** 2025-02-23
**Device:** IQOS ILUMA i ONE (M0023, "black") + IQOS ILUMA i (M0022, "sentia")
**Related reports:** 001

---

## 1. Background & Objective

Report #001 identified Counter2 (Block 2, tag `0x17`) in the `C0 1002 0101` usage counter response as incrementing daily. However, the user pointed out a contradiction: black device Counter2 = 176 on Dec 04 2025 (purchase date), which counts back to ~June 2025 -- before the device was purchased.

This report:
1. Proves the exact mathematical formula for black device Counter2
2. Tests alternative encoding/format interpretations
3. Analyzes sentia device Counter2 for cross-device comparison
4. Provides a complete cross-device counter block comparison

---

## 2. Black Device Counter2: Proven Formula

### 2.1 Formula

```
Counter2 = floor(RTC_seconds / 86400) - 5640
```

Where:
- `RTC_seconds` = device real-time clock value (4-byte LE from `C0 0001` response)
- `86400` = seconds per day
- `5640` = constant offset (device-specific)

### 2.2 Verification Across All Captures

| Capture | RTC bytes (LE) | RTC decimal | floor(RTC/86400) | Counter2 | Offset |
|---------|---------------|-------------|------------------|----------|--------|
| Dec 04 18:44 | `19 1D F4 1D` | 502,537,497 | 5816 | 0xB0 = 176 | **5640** |
| Dec 05 22:28 | `19 A3 F5 1D` | 502,621,977 | 5817 | 0xB1 = 177 | **5640** |
| Dec 06 14:53 | `C9 89 F6 1D` | 502,696,393 | 5818 | 0xB2 = 178 | **5640** |
| Dec 10 01:02 | `26 0D FB 1D` | 502,992,166 | 5821 | 0xB5 = 181 | **5640** |
| Dec 10 02:16 | `79 1E FB 1D` | 503,000,441 | 5821 | 0xB5 = 181 | **5640** |
| Dec 11 20:52 | `8E 75 FD 1D` | 503,149,966 | 5823 | 0xB7 = 183 | **5640** |
| Dec 11 21:16 | `09 7B FD 1D` | 503,151,369 | 5823 | 0xB7 = 183 | **5640** |

**Offset is 5640 in ALL 7 readings.** The formula is exact -- not approximate, not statistical.

### 2.3 RTC Computation Detail

```
Bytes: 19 1D F4 1D  (LE u32)
Value: 0x19 + 0x1D*256 + 0xF4*65536 + 0x1D*16777216
     = 25 + 7,424 + 15,990,784 + 486,539,264
     = 502,537,497

floor(502,537,497 / 86,400) = floor(5816.407) = 5816
Counter2 = 176
Offset = 5816 - 176 = 5640
```

**Note:** Report #001 Section 3.3 contained incorrect decimal conversions for the RTC values (e.g., stated 502,267,161 instead of the correct 502,537,497). The hex byte data was correct; only the decimal conversions were wrong. This report corrects them.

### 2.4 Day 5640 Interpretation

RTC epoch = 2010-01-01. What calendar date is day 5640?

```
2010-01-01 to 2024-12-31:
  Regular years (2010,2011,2013-2015,2017-2019,2021-2023): 11 * 365 = 4,015
  Leap years (2012,2016,2020,2024): 4 * 366 = 1,464
  Total: 5,479 days

Day 5479 = 2025-01-01  (0-indexed from 2010-01-01)
Day 5640 = 5479 + 161 = 2025-01-01 + 161 days

161 days from Jan 1: Jan(31) + Feb(28) + Mar(31) + Apr(30) + May(31) = 151
161 - 151 = 10 more days into June

Day 5640 = 2025-06-10 (June 10, 2025)
```

**Conclusion:** The black device's Counter2 offset points to **June 10, 2025** as "day zero." Since the device was purchased December 4, 2025 (~177 days later), this date is consistent with a **manufacturing date, initial battery activation, or factory QC test date.**

This is analogous to Apple storing iPhone manufacture dates retrievable from Settings.

---

## 3. Alternative Format Analysis

The following alternative interpretations of value 176 (0xB0) were tested and ruled out:

| Method | Interpretation of 0xB0 = 176 | Result | Verdict |
|--------|------------------------------|--------|---------|
| BCD encoding | 0xB0 as BCD | B is invalid BCD digit | **Rejected** |
| Day-of-year | Day 176 of 2025 | = June 25, but counter changes daily (not constant) | **Rejected** |
| Week counter | 176 weeks from epoch | ~3.38 years from 2010 = mid 2013 (meaningless) | **Rejected** |
| 12-hour units | 176 half-days | ~88 days = ~Sep 7, 2025 (doesn't fit purchase or mfg) | **Rejected** |
| Hour counter | 176 hours active | 7.3 days of operation (plausible but refuted by formula proof) | **Rejected** |
| Unix epoch days | Day 176 from 1970 | June 25, 1970 (meaningless) | **Rejected** |
| Signed i16 | 0x00B0 as signed | Same value (176, positive) | No change |
| IEEE754 half-float | 0x00B0 | Denormalized ≈ 1.075e-5 (meaningless) | **Rejected** |
| Modulo 256 | 5816 mod 256 | = 0xB8 = 184 (not 176) | **Rejected** |
| Different epoch (2000) | 5640 days from 2000-01-01 | = ~Jun 2015 (meaningless) | **Rejected** |

**The proven formula `floor(RTC/86400) - 5640` is the only interpretation that produces a constant offset across all 7 captures.** No alternative encoding yields a more meaningful result.

---

## 4. Sentia Device Counter Analysis

### 4.1 Device Architecture Differences

| Feature | Black (M0023, ILUMA i ONE) | Sentia (M0022, ILUMA i) |
|---------|---------------------------|------------------------|
| Form factor | All-in-one (standalone) | Two-piece (case + holder) |
| BLE targets | C0 only | C0 (case) + C9 (holder) |
| Counter blocks (C0) | 4 blocks | **5 blocks** |
| Counter blocks (C9) | N/A | 4 blocks |
| Battery cmd `C0 0160` | Present | **Absent** |
| Thermal cmd `0021` | On C0 | On **C9 only** |
| Battery cmd `0020` response | `8C 20` (12-byte payload) | `84 20` (4-byte payload) |

### 4.2 Sentia C0 Counter Block Structure (5 blocks, response `902A`)

Full payload (Nov 30):
```
C0 90 2A 01 01
  00 00 00 00  82 05 00 1E   Block 0: Counter0=0x0582=1410, tag=0x1E
  01 00 00 00  9A 00 00 1D   Block 1: Counter1=0x009A=154,  tag=0x1D
  02 00 00 00  6F 02 00 17   Block 2: Counter2=0x026F=623,  tag=0x17
  03 00 00 00  6C 00 00 18   Block 3: Counter3=0x006C=108,  tag=0x18
  04 00 00 00  05 00 00 14   Block 4: Counter4=0x0005=5,    tag=0x14
D5 5D
```

### 4.3 Sentia C9 Counter Block Structure (4 blocks, response `9022`)

Full payload (Nov 30):
```
08 90 22 01 01
  00 00 00 00  4F 06 00 8E   Block 0: Counter0=0x064F=1615, tag=0x8E
  01 00 00 00  8D 05 00 20   Block 1: Counter1=0x058D=1421, tag=0x20
  02 00 00 00  A3 03 00 17   Block 2: Counter2=0x03A3=931,  tag=0x17
  03 00 00 00  19 00 00 18   Block 3: Counter3=0x0019=25,   tag=0x18
47 B8
```

### 4.4 Sentia Counter Evolution Table

#### C0 (Case) Counters

| Date | State | Ctr0 (0x1E) | Ctr1 (0x1D) | Ctr2 (0x17) | Ctr3 (0x18) | Ctr4 (0x14) |
|------|-------|-------------|-------------|-------------|-------------|-------------|
| Nov 27 | first_pairing | 1385 | ?(trunc) | ?(trunc) | ?(trunc) | ?(trunc) |
| Nov 30 | paired_2nd | **1410** | **154** | **623** | 108 | 5 |
| Dec 05 16:32 | before_puff | **1424** | **156** | **632** | 108 | 5 |
| Dec 05 20:45 | after_puff | **1425** | 156 | 632 | 108 | 5 |

#### C9 (Holder) Counters

| Date | State | Ctr0 (0x8E) | Ctr1 (0x20) | Ctr2 (0x17) | Ctr3 (0x18) |
|------|-------|-------------|-------------|-------------|-------------|
| Nov 27 | first_pairing | 1584 | ?(trunc) | ?(trunc) | ?(trunc) |
| Nov 30 | paired_2nd | **1615** | **1421** | **931** | 25 |
| Dec 05 16:32 | before_puff | **1629** | **1436** | **941** | 25 |
| Dec 05 20:45 | after_puff | **1630** | **1437** | **942** | 25 |

*(Nov 27 first_pairing payloads were truncated in the original capture; only Block 0 is recoverable)*

#### Deltas (Nov 30 -> Dec 05 before puff, then -> Dec 05 after puff)

| Counter | C0 delta (5d) | C0 delta (puff) | C9 delta (5d) | C9 delta (puff) |
|---------|--------------|-----------------|--------------|-----------------|
| Ctr0 (puffs) | +14 | +1 | +14 | +1 |
| Ctr1 | +2 | 0 | +15 | +1 |
| Ctr2 (tag 0x17) | **+9** | **0** | **+10** | **+1** |
| Ctr3 | 0 | 0 | 0 | 0 |
| Ctr4 | 0 | 0 | N/A | N/A |

### 4.5 Sentia Counter2 Does NOT Follow the Black Device Formula

Applying `floor(RTC/86400) - Counter2` to sentia:

**C0 Counter2:**

| Date | RTC bytes (LE) | RTC decimal | floor(RTC/86400) | Ctr2 | Offset |
|------|---------------|-------------|------------------|------|--------|
| Nov 30 | `1C 48 EE 1D` | 502,155,292 | 5812 | 623 | **5189** |
| Dec 05 16:32 | `88 4F F5 1D` | 502,615,944 | 5818 | 632 | **5186** |
| Dec 05 20:45 | `E8 8A F5 1D` | 502,631,144 | 5818 | 632 | **5186** |

Offset varies: 5189 vs 5186 (delta = 3). **Formula does not hold.**

**C9 Counter2:**

| Date | RTC bytes (LE) | RTC decimal | floor(RTC/86400) | Ctr2 | Offset |
|------|---------------|-------------|------------------|------|--------|
| Nov 30 | `1D 48 EE 1D` | 502,155,293 | 5812 | 931 | **4881** |
| Dec 05 16:32 | `8A 4F F5 1D` | 502,615,946 | 5818 | 941 | **4877** |
| Dec 05 20:45 | `E7 8A F5 1D` | 502,631,143 | 5818 | 942 | **4876** |

Offset varies: 4881, 4877, 4876. **Formula does not hold.** C9 Counter2 also incremented after a puff (+1 from 941 to 942), unlike the black device Counter2 which never increments on puff.

### 4.6 Sentia Counter2 Increment Rate Analysis

**C0 Counter2:** +9 in 5.33 RTC days (Nov 30 08:34 -> Dec 05 16:32) = **~1.69 per day**
- Does not match any clean time quantum (12h, 16h, 8h all fail)
- No increment after single puff -> not strictly puff-based
- 14 puffs occurred in same period -> +9 counter is not 1:1 with puffs

**C9 Counter2:** +10 in 5.33 RTC days = **~1.88 per day**, plus +1 after puff
- C9 Counter2 incremented both over time AND after puff event
- 14 puffs produced +10 time-based increments + 1 puff increment = +11 total

**Hypothesis:** Sentia Counter2 is a **composite counter** combining:
1. A time-based component (like black device, but with smaller quantum)
2. An event-based component (stick/session usage on C9 holder)

This cannot be decomposed without additional capture points at known intervals.

### 4.7 Sentia Counter2 Offset Date Estimates

Despite the inconsistency, the approximate offset dates suggest manufacturing timelines:

| Component | Avg Offset | Day from epoch | Approx Date |
|-----------|-----------|---------------|-------------|
| C0 (case) | ~5187 | 5187 | ~2025-03-14 (March 2025) |
| C9 (holder) | ~4878 | 4878 | ~2023-05-14 (May 2023) |

- Sentia C0 manufactured ~March 2025, device purchased ~April 2025. **Plausible.**
- Sentia C9 manufactured ~May 2023, about 2 years before purchase. This is unusual but possible if the holder component was manufactured earlier or the counter includes a different base offset.

---

## 5. Cross-Device Tag Byte Analysis

The tag byte (byte 7 of each 8-byte block) appears to encode the counter type:

| Tag | Black C0 | Sentia C0 | Sentia C9 | Interpretation |
|-----|----------|-----------|-----------|---------------|
| `0x8E` | Block 0 (puffs: 9-12) | -- | Block 0 (puffs: 1584-1630) | **Puff counter (standalone/holder)** |
| `0x1E` | -- | Block 0 (puffs: 1385-1425) | -- | **Puff counter (case component)** |
| `0x1D` | Block 1 (const: 17) | Block 1 (154-156) | -- | **Low-frequency counter** |
| `0x20` | -- | -- | Block 1 (1421-1437) | **Puff mirror counter (holder)** |
| `0x17` | Block 2 (176-183) | Block 2 (623-632) | Block 2 (931-942) | **Day/time-based counter** |
| `0x18` | Block 3 (const: 1) | Block 3 (const: 108) | Block 3 (const: 25) | **Static parameter** |
| `0x14` | -- | Block 4 (const: 5) | -- | **Static parameter (case only)** |

Key observations:
- **Tag `0x17` is universal** across all devices and components -> same counter type
- **Tag `0x8E` vs `0x1E`**: Both count puffs, but `0x8E` = standalone/holder, `0x1E` = case. Bit difference: `0x8E ^ 0x1E = 0x90` (bit 7 and bit 4 differ)
- **Tag `0x18` is universal** but values differ (1, 25, 108) and all are constant -> hardware/calibration parameter
- Sentia C9 Block 1 (tag `0x20`) tracks puffs parallel to Block 0, suggesting redundant or categorized puff counting

---

## 6. Cross-Device Counter0 (Puff Count) Comparison

| Device | Component | Nov 27 | Nov 30 | Dec 04 | Dec 05 | Dec 06 | Dec 10a | Dec 10b | Dec 11a | Dec 11b |
|--------|-----------|--------|--------|--------|--------|--------|---------|---------|---------|---------|
| Sentia C0 | Case | 1385 | 1410 | -- | 1424/1425 | -- | -- | -- | -- | -- |
| Sentia C9 | Holder | 1584 | 1615 | -- | 1629/1630 | -- | -- | -- | -- | -- |
| Black C0 | Standalone | -- | -- | 9 | 9 | 10 | 10/11 | -- | 11/12 | -- |

- Sentia counters are much higher (1300-1600+) vs black (9-12), consistent with sentia being purchased ~8 months earlier (~April 2025) and used heavily
- Both devices increment Counter0 by exactly +1 per puff event (verified across before/after captures)
- Sentia C0 and C9 puff counts differ (1410 vs 1615 on Nov 30), suggesting independent tracking or different component histories

---

## 7. Sentia-Specific Dynamic Fields

### 7.1 Thermal State (C9 `0021`)

| Date | State | Value (hex LE) | Value (dec) |
|------|-------|---------------|-------------|
| Nov 27 | first_pairing (1st) | 0x0E6C | 3692 |
| Nov 27 | first_pairing (2nd) | 0x0E6E | 3694 |
| Nov 30 | paired (both reads) | 0x0EA3 | 3747 |
| Dec 05 16:32 | before_puff (1st) | 0x0EAF | 3759 |
| Dec 05 16:32 | before_puff (2nd) | 0x0EAE | 3758 |
| Dec 05 20:45 | after_puff (both) | 0x0EAD | 3757 |

- Range: 3692-3759 (vs black device: 1479-1529)
- **No post-puff spike** in sentia (3759 before -> 3757 after), unlike black device which showed +49 spike on Dec 11
- This makes sense: sentia `0021` queries the **holder** (C9), which was reinserted into the case and cooled before the BLE capture
- Gradual upward trend over 8 days (3692 -> 3759 = +67) may reflect seasonal temperature change or sensor drift

### 7.2 Battery Level (C0 `0020`)

Sentia: `C0 84 20 0B 02 00 00 70` (4 bytes payload)
Black:  `C0 8C 20 00 00 1E 05 1E 82 EA 05 E1 07 ...` (24 bytes payload)

- Response length codes differ: sentia `84` (4 bytes) vs black `8C` (12 bytes usable)
- Sentia returns `0B 02 00 00` where `0x020B` = 523 (likely raw ADC reading)
- Black returns a much richer payload with manufacturing-era timestamp data
- Sentia battery reading was identical across all captures (device always fully charged from case)

### 7.3 Battery Voltage (`C0 0160`)

**Completely absent from sentia protocol.** This command is specific to the ILUMA i ONE (M0023) all-in-one form factor. The two-piece ILUMA i (M0022) likely handles battery monitoring through the case's charging circuitry instead.

---

## 8. Block Format Specification (Refined)

```
Offset  Size  Field
0       1     Block ID (0x00, 0x01, 0x02, 0x03, 0x04)
1       3     Padding (always 0x00 0x00 0x00)
4       1     Value low byte
5       1     Value high byte
6       1     Padding (always 0x00)
7       1     Tag byte (counter type identifier)
```

Total: 8 bytes per block.

Value is u16 LE at offset 4-5. Tag byte at offset 7 identifies the counter type across devices.

Response header format: `[prefix] 90 [length] 01 01 [blocks...] [CRC16]`
Where length = total payload bytes (excluding prefix and CRC):
- Black C0: `22` = 34 bytes (4 blocks * 8 + 2 for `01 01`)
- Sentia C0: `2A` = 42 bytes (5 blocks * 8 + 2 for `01 01`)
- Sentia C9: `22` = 34 bytes (4 blocks * 8 + 2 for `01 01`)

---

## 9. Summary of Findings

### Confirmed

1. **Black Counter2 = floor(RTC/86400) - 5640** with perfect precision across 7 readings
2. **Day 5640 = June 10, 2025** -- plausible manufacturing/activation date (177 days before Dec 4 purchase)
3. **All alternative encodings tested and rejected** -- the formula is the only consistent interpretation
4. **Sentia Counter2 uses a DIFFERENT mechanism** -- offset varies by 3-5 depending on capture, indicating non-pure-time counting
5. **Battery voltage command `0160` is M0023-specific** -- absent in M0022 (sentia)
6. **Tag byte `0x17`** identifies the "day counter" type across all devices/components
7. **Block format** is identical (8-byte) across both devices; only block count differs

### Hypotheses Requiring More Data

1. **Sentia C0 Counter2** may be a composite of time-based + event-based increments (daily increments + extra increments from usage sessions)
2. **Sentia C9 Counter2** appears event-linked (increments after puff) but also has a time component
3. **Counter2 absolute offset** may encode device manufacture or activation date for standalone devices (black), but meaning is less clear for multi-component devices (sentia)

### Corrections to Report #001

- Section 3.3 RTC decimal values were incorrectly computed. Correct values:
  - Dec 04: 502,537,497 (not 502,267,161)
  - Dec 05: 502,621,977 (not 502,367,001)
  - Dec 06: 502,696,393 (not 502,426,057)
  - Dec 10 01:02: 502,992,166 (not 502,595,878)
  - Dec 11 20:52: 503,149,966 (not 502,752,654)
- Counter2 label "Device uptime days" is misleading. More accurate: **"Days since activation (day 0 = manufacturing date)"**

---

## 10. Recommended Next Steps

1. **Capture sentia at known intervals** (e.g., connect every 12 hours without puffing) to isolate the time-based component of Counter2
2. **Capture sentia after exactly 1 stick** to verify C9 Counter2 +1 per stick hypothesis
3. **Compare Counter2 tag `0x17` across more IQOS models** to determine if it's a universal "device age" metric
4. **Investigate sentia Counter1 (C9 tag `0x20`)** which mirrors puff count -- determine if it counts a different puff category or is simply redundant
5. **Capture during active heating** to observe non-zero `C0/C9 0220` temperature values and correlate with Counter2 changes in real-time
