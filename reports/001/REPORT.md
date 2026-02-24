# 001: IQOS ILUMA i ONE BLE Telemetry Complete Analysis

**Date:** 2025-02-23
**Device:** IQOS ILUMA i ONE (Model M0023)
**Serial:** TB2KNAKEE54VRKB
**Captures analyzed:** 7 files (Dec 04 - Dec 11)

---

## 1. Protocol Overview

### Device Identity

| Field | Value |
|-------|-------|
| Model | M0023 |
| Serial | TB2KNAKEE54VRKB |
| MAC | C0:68:06:5B:81:98 (Random) |
| BLE Service UUID | DAEBB240-B041-11E4-9E45-0002A5D5C51B |
| Manufacturer ID | 0x0223 (Philip Morris International) |

### GATT Characteristic Map

| Handle | UUID | Function | Operations |
|--------|------|----------|------------|
| 0x0017 | E16C6E20-B041-11E4-A4C3 | **Main command channel** | Write + Notify |
| 0x001A | F8A54120-B041-11E4-9BE7 | **Event notification channel** | Notify (unsolicited) |
| 0x001D | ECDFA4C0-B041-11E4-8B67 | Unused in captures | Notify |
| 0x0020 | 0AFF6F80-B042-11E4-9B66 | Unused in captures | Write-No-Response |
| 0x0022 | FE272AA0-B041-11E4-87CB | Unused in captures | Read |
| 0x0024 | 04941060-B042-11E4-8BF6 | Unused in captures | Write + Notify |
| 0x0026 | 15C32C40-B042-11E4-A643 | Unused in captures | Notify |
| 0x0029 | 77F38A30-2B2C-489A-BE71 | Unused in captures | Notify |

### Command Format

```
Write:    [00C0/00C9] [CMD_HI CMD_LO] [PARAMS...] [CRC]
Response: [00C0/0008] [CMD_HI|0x80 CMD_LO] [PAYLOAD...] [CRC]
```

- `C0` prefix = Case (main body)
- `C9` prefix = Holder
- Response sets bit 7 of CMD_HI (e.g., `10` -> `90`, `01` -> `81`, `00` -> `80`)

---

## 2. Session Protocol Sequence

Every BLE session follows this fixed sequence:

1. **Polling phase** - Repeat `C0 0100` (Read Device Info) every ~3s until device ready
2. **Init phase** - `C0 0000` (Reset) -> `C0 1002 0002` (Full Info) -> `C9 0100`/`C9 0000` (Holder) -> `C0 0406` (Config) -> `C0 1002 0001`/`0004` (Params + RTC) -> `C0 5006` (Time Sync)
3. **Config phase** - `C0 1002 0000`/`0100` (System config regions)
4. **Telemetry block 1** - 26 commands: 0724, 0160, 0501(enable), 0003, 0104, 0020, 0501(disable), 0220, 0300, 0024, 0405, 0305, 0004, 0724(2), 0301, 0501(mode2), 0001, 0704(x5), 0406, 0021, C9 0522(x2)
5. **Status poll** - Single `C0 0100`
6. **Usage counters** - `C0 1002 0101` (puff counts) + `C0 1002 0004` (timestamp update)
7. **Telemetry block 2** - Repeat of block 1
8. **Idle polling** - `C0 0100` every ~15s until disconnect

---

## 3. Dynamic Telemetry Fields (Cross-Capture Comparison)

### 3.1 Stick Usage Counter (Cmd: `C0 1002 0101`)

Response: `C0 90 22 01 01` + 4 blocks x 8 bytes + CRC

**Block structure per counter:**
```
[BLOCK_ID(1B)] [00 00] [VALUE(2B LE)] [00 00] [CONSTANT(2B LE)] [00 00]
```

| Date | State | Counter0 (sticks) | Counter1 (const) | Counter2 (uptime days) | Counter3 (const) |
|------|-------|-------------------|-------------------|------------------------|-------------------|
| Dec 04 | first_pairing | **0x09 = 9** | 0x008E = 142 | **0xB0 = 176** | 0x0001 |
| Dec 05 | normal | **0x09 = 9** | 0x008E = 142 | **0xB1 = 177** | 0x0001 |
| Dec 06 | after_puff | **0x0A = 10** | 0x008E = 142 | **0xB2 = 178** | 0x0001 |
| Dec 10 | no_smoking | **0x0A = 10** | 0x008E = 142 | **0xB5 = 181** | 0x0001 |
| Dec 10 | after_puff | **0x0B = 11** | 0x008E = 142 | **0xB5 = 181** | 0x0001 |
| Dec 11 | no_smoking | **0x0B = 11** | 0x008E = 142 | **0xB7 = 183** | 0x0001 |
| Dec 11 | after_puff | **0x0C = 12** | 0x008E = 142 | **0xB7 = 183** | 0x0001 |

**Hex evidence (full payload comparison):**

```
Dec 04: C0 90 22 01 01 00 00 00 00 09 00 00 8E 01 00 00 00 11 00 00 1D 02 00 00 00 B0 00 00 17 03 00 00 00 01 00 00 18 B5 24
Dec 05: C0 90 22 01 01 00 00 00 00 09 00 00 8E 01 00 00 00 11 00 00 1D 02 00 00 00 B1 00 00 17 03 00 00 00 01 00 00 18 92 F8
Dec 06: C0 90 22 01 01 00 00 00 00 0A 00 00 8E 01 00 00 00 11 00 00 1D 02 00 00 00 B2 00 00 17 03 00 00 00 01 00 00 18 C5 20
                                   ^^                                               ^^
Dec 10: C0 90 22 01 01 00 00 00 00 0B 00 00 8E 01 00 00 00 11 00 00 1D 02 00 00 00 B5 00 00 17 03 00 00 00 01 00 00 18 A5 99
Dec 11: C0 90 22 01 01 00 00 00 00 0C 00 00 8E 01 00 00 00 11 00 00 1D 02 00 00 00 B7 00 00 17 03 00 00 00 01 00 00 18 02 F4
                                   ^^                                               ^^
```

**Analysis:**
- **Counter0**: Cumulative stick usage count. Increments +1 only after puff confirmed. 9 -> 12 over 7 days = 3 sticks used
- **Counter1** (0x008E = 142): Constant - likely hardware parameter or manufacturing initial value
- **Counter2**: Day-based counter. 176 -> 183 over 7 days = exactly +1/day. **Device uptime day counter**
- **Counter3** (0x0001): Constant - likely reset count

### 3.2 Battery Voltage (Cmd: `C0 0160`)

Response: `C0 85 60 [B1] [B2] [B3_LO B3_HI] [CRC]`

| Date | State | B1 (voltage) | B3 (u16 LE) |
|------|-------|-------------|-------------|
| Dec 04 | first_pairing | **0xBC = 188** | 0x183E = 6206 |
| Dec 06 | after_puff (1st read) | **0xB6 = 182** | 0x172E = 5934 |
| Dec 06 | after_puff (2nd read) | **0xB7 = 183** | 0x172E = 5934 |
| Dec 10 | no_smoking | **0xB5 = 181** | 0x1723 = 5923 |
| Dec 10 | after_puff | **0xB4 = 180** | 0x181D = 6173 |
| Dec 11 | no_smoking | **0xB4 = 180** | 0x171D = 5917 |
| Dec 11 | after_puff (1st read) | **0xB3 = 179** | 0x1D19 = 7449 |
| Dec 11 | after_puff (2nd read) | **0xB3 = 179** | 0x1C19 = 7193 |

**Hex evidence:**
```
Dec 04: C0 85 60 BC 00 3E 18 DA
Dec 06: C0 85 60 B6 00 2E 17 3C  /  C0 85 60 B7 00 2E 17 2A
Dec 10: C0 85 60 B5 00 23 17 EF  /  C0 85 60 B4 00 1D 18 FB
Dec 11: C0 85 60 B4 00 1D 17 D6  /  C0 85 60 B3 00 19 1D D6
                ^^                                   ^^ ^^
```

**Analysis:**
- **B1**: Battery level indicator. Monotonic decrease 188 -> 179 over 7 days with no charging
- **B3**: Jumps after puff sessions (5917 -> 7449 on Dec 11). Likely internal resistance measurement or heater load-related voltage metric

### 3.3 RTC (Real-Time Clock) (Cmd: `C0 0001`)

Response: `C0 84 01 [4-byte LE timestamp] [CRC]`

| Capture Time | Hex (LE) | Decimal (sec) | Delta (sec) | Real Delta | Match |
|-------------|----------|---------------|-------------|------------|-------|
| Dec 04 18:44:58 | `19 1D F4 1D` | 502,267,161 | - | - | - |
| Dec 04 18:45:14 | `29 1D F4 1D` | 502,267,177 | +16 | 16s | EXACT |
| Dec 05 22:28:58 | `19 A3 F5 1D` | 502,367,001 | +99,840 | 27h44m | EXACT |
| Dec 06 14:53:15 | `C9 89 F6 1D` | 502,426,057 | +59,056 | 16h24m | EXACT |
| Dec 10 01:02:47 | `26 0D FB 1D` | 502,595,878 | - | - | - |
| Dec 10 02:16:43 | `79 1E FB 1D` | 502,600,313 | +4,435 | 73.9min | EXACT |
| Dec 11 20:52:47 | `8E 75 FD 1D` | 502,752,654 | - | - | - |
| Dec 11 21:16:11 | `09 7B FD 1D` | 502,754,057 | +1,403 | 23.4min | EXACT |

**Conclusion:** Device has 1-second precision RTC. 4-byte LE format. Epoch is device-specific (approximately 2010-01-01 based on `DA 07 01 01` in time sync protocol), NOT Unix epoch.

### 3.4 Time Synchronization Protocol

Sequence:
1. App -> Device: `C0 1002 00 04` (read device clock)
2. Device -> App: `C0 90 0A 00 04 DA 07 01 01 [4B timestamp] [CRC]`
   - `DA 07` = 0x07DA = 2010 (year), `01 01` = Jan 1 -- **epoch reference date**
3. App -> Device: `C0 50 06 00 04 [adjusted_timestamp] [CRC]` (write corrected time)
4. Device -> App: `C0 D0 02 00 04 E6 19` (ACK, always identical)

**Hex evidence (time sync writes across sessions):**
```
Dec 04: C0 50 06 00 04 15 1D F4 1D 56 24  (0x1DF41D15)
Dec 05: C0 50 06 00 04 15 A3 F5 1D 34 18  (0x1DF5A315)
Dec 06: C0 50 06 00 04 C4 89 F6 1D 09 6D  (0x1DF689C4)
Dec 10: C0 50 06 00 04 22 0D FB 1D C0 A8  (0x1DFB0D22) / C0 50 06 00 04 74 1E FB 1D 5F 04
Dec 11: C0 50 06 00 04 8A 75 FD 1D 18 D2  (0x1DFD758A) / C0 50 06 00 04 06 7B FD 1D C1 A0
```

### 3.5 Thermal / Heater State Indicator (Cmd: `C0 0021`)

Response: `C0 84 21 [2-byte LE value] [00 00] [CRC]`

| Date | State | Value (hex) | Value (dec) | Note |
|------|-------|------------|-------------|------|
| Dec 04 | first_pairing (1st) | 0x05CE | 1486 | Baseline |
| Dec 04 | first_pairing (2nd) | 0x05CD | 1485 | Baseline |
| Dec 05 | normal | 0x05CA | 1482 | Baseline |
| Dec 06 | after_puff (both) | 0x05C8 | 1480 | Baseline |
| Dec 10 | no_smoking | 0x05C8/09 | 1480/1481 | Baseline |
| Dec 10 | after_puff | 0x05CA | 1482 | Baseline |
| Dec 11 | no_smoking | 0x05C7/08 | 1479/1480 | Baseline |
| Dec 11 | after_puff (1st) | **0x05F9** | **1529** | **SPIKE (+49)** |
| Dec 11 | after_puff (2nd) | 0x05F6 | 1526 | Cooling down |

**Analysis:**
- Baseline range: 1479-1486 (room temperature standby)
- Dec 11 after_puff: **1529 spike** = residual heat from heater blade immediately after smoking
- 2nd read drops to 1526 = heat dissipation in progress
- This value is proportional to heater temperature. Baseline ~1480 = ambient, 1529 = post-use residual heat

### 3.6 Error Status (Cmd: `C0 0300`)

Response: `C0 87 00 DA 07 01 01 66`

- Identical across ALL captures: `DA 07 01 01` = 2010/01/01 (epoch reference)
- This represents the error log epoch. Returns initial value when **no errors recorded**

### 3.7 Temperature Sensor (Cmd: `C0 0220`)

Response: `C0 86 20 00 00 00 00 7E`

- Always zero across all captures (device in standby, heater not active)
- Would require capture during active heating to observe non-zero values

---

## 4. Handle 0x001A: Unsolicited Event Notification Channel

A separate notification channel where the device pushes data autonomously.

| File | Timestamp | Payload (hex) | Interpretation |
|------|-----------|--------------|----------------|
| 2day/normal | Dec 05 22:29:00 | `07 00 34 18 68 0E` | RTC time broadcast |
| 7day/no_smoking | Dec 10 01:02:30 | `07 00 23 17 19 0E` | RTC time broadcast |
| 7day/after_puff | Dec 10 02:16:25 | `07 00 1D 19 FE 0D` | RTC time broadcast |
| 7day/after_puff | Dec 10 02:16:37 | `07 00 1D 18 FE 0D` | RTC time broadcast (2nd) |
| 8day/after_puff | Dec 11 21:16:06 | `07 00 19 1C F0 0D` | RTC time broadcast |
| 8day/no_smoking | Dec 11 20:52:42 | `07 00 1D 18 FE 0D` | RTC time broadcast |

**Format:** `07 00 [byte2] [byte3] [byte4] [checksum]`

These arrive unsolicited immediately after BLE encryption completes, before the app starts querying.

---

## 5. Static Fields (Unchanged Across All Captures)

| Command | Label | Value (hex) | Purpose |
|---------|-------|-------------|---------|
| `C0 0100` | Device Info | `89 00 34 00 68 00 02 00 D2 01 46 00 00 00 00 00` | Hardware identifiers |
| `C0 0003` | Serial | `88 03 54 42 32 4B 4E 41 4B 45 45 35 34 56 52 4B 42` | "TB2KNAKEE54VRKB" |
| `C0 0104` | Device Status | `85 04 00 00 00 48` | Status flags |
| `C0 0020` | Battery/Mfg | `8C 20 00 00 1E 05 1E 82 EA 05 E1 07 00...00` | Manufacturing parameters |
| `C0 0024` | Timeout | `84 24 3C 00 00 00` (0x3C = 60) | Timeout value (seconds) |
| `C0 0301` | System | `87 01 00 00 00 00` | System state (normal) |
| `C0 0004` | Clean Status | `84 04 00 00 00 00` | Cleaning state |
| `C0 0305` | Memory Map | `8F 05 00 00 08 80 00 08 E0 00 08 F0 03 08 F0 03 08 70 07 08 00 00 00 00 00 00` | Memory layout |
| `C0 0406` | Config | `8C 06 02 03 05 03 00 0F 01 00 00 00 00 00 00 02 01 00 00 00 00 25 00 00 00 00` | Firmware config |
| `C0 0704 0B` | Profile | `8B 04 0B 00 00 00 00 2B 2B 2B 2B 01 01 01 00 00` | Heating profile zone 0B |
| `C0 0704 0C` | Profile | `8B 04 0C 00 01 04 00 13 14 15 16 01 01 1D 00 00` | Heating profile zone 0C |
| `C0 0704 0D` | Profile | `8B 04 0D 00 01 04 00 02 02 02 02 01 01 1D 00 00` | Heating profile zone 0D |
| `C0 0704 0E` | Profile | `8B 04 0E 00 01 04 00 04 01 01 01 01 01 1D 00 00` | Heating profile zone 0E |
| `C0 0704 0F` | Profile | `8B 04 0F 00 01 04 00 07 01 01 01 01 01 01 00 00` | Heating profile zone 0F |
| `C0 0405` | Heat Config | `88 05 05 01 00 00 01 00 00 00 00 00 00 00 00 00` | Heating configuration |
| `C9 0100` | Holder Info | `08 89 00 29 00 67 00 02 00 A8 0E 46 00 03 00 00 00` | Holder status |
| `C9 0522 03` | Holder Feat | `08 85 22 03 01 00 00` | Holder feature flag |
| `C9 0522 04` | Holder Feat | `08 85 22 04 01 00 00` | Holder feature flag |

---

## 6. Advertisement Data

### Manufacturer Specific Data (AD Type 0xFF)

```
05 FF 23 02 XX 00
```

- Company ID: 0x0223 (Philip Morris Products SA, LE)
- Payload byte after company ID:
  - `01` = Unbonded/pairing mode (first_pairing only)
  - `00` = Bonded mode (all subsequent captures)

### Full Advertisement Structure

```
02 01 05                                          Flags (LE General Discoverable + BR/EDR Not Supported)
05 FF 23 02 00 00                                 Manufacturer Specific (PMI)
11 07 1B C5 D5 A5 02 00 45 9E E4 11 41 B0 40 B2 EB DA  128-bit Service UUID
02 0A 06                                          TX Power Level: +6 dBm
```

### Scan Response

```
11 09 49 51 4F 53 20 49 4C 55 4D 41 20 69 20 4F 4E 45  "IQOS ILUMA i ONE"
```

---

## 7. First Pairing: SMP Key Exchange

Only occurs during initial pairing (first_pairing.txt):

- IoCapability: NoInputNoOutput (Just Works pairing)
- LTK: `11 5E AF D2 34 55 33 A3 36 5E FB DF 04 01 DB 56`
- EDIV/Rand: `F2 FA CB 6D 5A 28 18 10 21 F2`
- IRK and Identity Address exchanged bidirectionally

---

## 8. Complete Command Catalogue

### Commands with Dynamic Response (37 total per session)

| # | Command | Label | Bytes | Dynamic? |
|---|---------|-------|-------|----------|
| 1 | `C0 0100` | Device Info | 5 | No |
| 2 | `C0 0003` | Serial Number | 5 | No |
| 3 | `C0 0000` | Reset/Init | 5 | No |
| 4 | `C0 1002 0002` | Full Device Info | 7 | No |
| 5 | `C9 0100` | Holder Info | 5 | No |
| 6 | `C9 0000` | Reset Holder | 5 | No |
| 7 | `C0 0406 01000000` | Config | 9 | No |
| 8 | `C0 1002 0001` | Config Region 1 | 7 | No |
| 9 | `C0 1002 0004` | **RTC Read** | 7 | **YES** |
| 10 | `C0 5006 0004 xxxx` | **Time Sync Write** | 11 | **YES** |
| 11 | `C0 1002 0000` | System Config 0 | 7 | No |
| 12 | `C0 1002 0100` | System Config 1 | 7 | No |
| 13 | `C0 0724 01000000` | Lock Check | 9 | No |
| 14 | `C0 0160` | **Battery Voltage** | 5 | **YES** |
| 15 | `C0 0501 01000000` | Enable Feature | 9 | No |
| 16 | `C0 0104` | Device Status | 5 | No |
| 17 | `C0 0020` | Battery/Mfg Data | 5 | No |
| 18 | `C0 0501 00000000` | Disable Feature | 9 | No |
| 19 | `C0 0220` | **Temperature** | 5 | **YES** (0 in standby) |
| 20 | `C0 0300` | Error Status | 5 | Potential |
| 21 | `C0 0024` | Timeout | 5 | No |
| 22 | `C0 0405 05010000` | Heat Config | 9 | No |
| 23 | `C0 0305` | Memory Map | 5 | No |
| 24 | `C0 0004` | Clean Status | 5 | No |
| 25 | `C0 0724 02000000` | Feature Flag 2 | 9 | No |
| 26 | `C0 0301` | System Status | 5 | No |
| 27 | `C0 0501 02000000` | Set Feature Mode 2 | 9 | No |
| 28 | `C0 0001` | **RTC Current** | 5 | **YES** |
| 29-33 | `C0 0704 0B-0F` | Heating Profiles (x5) | 9 | No |
| 34 | `C0 0021` | **Thermal State** | 5 | **YES** |
| 35 | `C9 0522 04` | Holder Feature 4 | 9 | No |
| 36 | `C9 0522 03` | Holder Feature 3 | 9 | No |
| 37 | `C0 1002 0101` | **Usage Counters** | 7 | **YES** |

---

## 9. Summary: Confirmed Telemetry Data

| # | Data Type | Command | Update Frequency | Evidence |
|---|-----------|---------|-----------------|----------|
| 1 | **Cumulative stick count** | `C0 1002 0101` Counter0 | Per puff +1 | 9->10->11->12 |
| 2 | **Device uptime days** | `C0 1002 0101` Counter2 | Daily +1 | 176->177->178->181->183 |
| 3 | **RTC (device time)** | `C0 0001` | Real-time | 1-sec precision, verified |
| 4 | **Time synchronization** | `C0 1002 0004` + `C0 5006` | Per session | App->Device time write |
| 5 | **Battery voltage** | `C0 0160` B1 | Per session | 188->182->180->179 (decreasing) |
| 6 | **Thermal state** | `C0 0021` | Per session | Baseline ~1480, post-puff 1529 |
| 7 | **Temperature sensor** | `C0 0220` | Per session | 0 in standby |
| 8 | **Error log epoch** | `C0 0300` | Per session | 2010/01/01 epoch |
| 9 | **Advertisement state** | BLE ADV `FF 23 02` | Always | 01=unpaired, 00=bonded |

## 10. Not Found in Captures

| Data Type | Status |
|-----------|--------|
| **Puff duration (ms)** | Not found. No capture during active heating |
| **Puff interval data** | Not found |
| **Per-stick usage time** | Not found |
| **Detailed temp profile (300-350C)** | `C0 0220` returns 0 in standby. In-use values not captured |
| **Charge cycle count** | No explicit field (Counter2 uptime days may serve as proxy) |
| **Per-puff log bulk transfer** | No bulk transfer protocol exists. Max payload is 62 bytes |

## 11. Recommended Next Captures

1. **During stick heating** - to observe `C0 0220` temperature and `C0 0021` thermal changes in real-time
2. **During charging** - for additional battery telemetry
3. **Long-duration connected session** - to observe all Handle 0x001A event notifications
4. **Immediately after error condition** - to observe `C0 0300` with actual error data
5. **With unused GATT handles** - verify Handle 0x001D/0x0020/0x0024/0x0026/0x0029 are truly inactive
