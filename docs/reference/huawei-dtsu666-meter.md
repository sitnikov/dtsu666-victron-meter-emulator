# Reference: DTSU666-H meter for Huawei SUN2000

What Huawei expects from the meter on RS485 (the inverter is master, the meter is slave).
**Updated from reading emulator sources with real-traffic captures + the official Huawei
manual** (2026-05-31). Sources — [`sources.md`](sources.md).
Confidence: ✅ high (confirmed by code/manual), ⚠️ needs verification on hardware.

> 🔑 **KEY CORRECTION.** The inverter polls the meter at addresses **`0x0836`
> (dec 2102)**, NOT `0x2000`. The `0x2000` range is the **native CHINT map**, from which a
> translator converts into Huawei addresses. On the Huawei↔meter bus you have:
> handshake `0x07D1`, data block `0x0836` (80 reg.), parameter block `0x08A6` (10 reg.).
> Confirmed by three sources, including a live-traffic decoder (Vuko791) and a working
> ESPHome proxy (pazzero).

## Link parameters

| Parameter | Value | Confidence |
|-----------|-------|------------|
| Address (slave/unit ID) | **11 (0x0B)**, range 11–19 | ✅ official Quick Guide + code |
| Baud rate | 9600 baud | ✅ official |
| Frame | **8N1** — the native DTSU666-H default (official Quick Guide/manual: "n.1"). Working emulators (salakrzy/ArminJo) send **8N2**; the inverter apparently tolerates both. **Start with 8N1** (device default), 8N2 is the fallback | ⚠️ verify which the inverter actually accepts |
| Read function | **0x03** (read holding registers) — required; some emulators also answer 0x04 | ✅ |
| Number format | IEEE-754 **float32**, 2 registers, **big-endian, high word first** | ✅ |

## Meter identification (handshake)

- The inverter **first** reads register **`0x07D1` (dec 2001), 1 word** — this is a liveness/
  identification probe. Request frame (captured from a real SUN2000): `0B 03 07 D1 00 01 D5 ED`. ✅
- **Reply: `0x3B11`** (dec 15121) = "meter active". On **older firmware** — `0x3F80` (high
  word of float 1.0). ✅ Confirmed: salakrzy (captured real frames) + the working pazzero
  proxy (`address: 2001 → return 15121`).
  - ⚠️ The exact "magic" code **depends on firmware** (0x3B11 vs 0x3F80) — capture it on your
    own inverter with an analyzer; start with 0x3B11.
  - ❌ The earlier "return **3**" is WRONG for this register. The "3" in emavap is, per
    pazzero, a confused register **2220 "Protocol changeover = 3"** from the parameter block.
- In the app/FusionSolar the meter type is **DTSU666-H** (3-phase; DDSU666-H is single-phase).
  Identification takes several minutes. ✅

## Data block `0x0836` (dec 2102): 80 registers = 40×float32, FC 0x03

Inverter request: `0B 03 08 36 00 50 A7 32` (0x50 = 80 reg.). The emulator returns **40
float32 values directly in engineering units** (do NOT apply the CHINT divisors — those
relate to converting from raw CHINT integers, not to what the inverter expects).
Value address = `0x0836 + 2*offset`.

| Offset | Address hex / dec | Quantity | Unit | Confidence |
|-------:|-------------------|----------|------|------------|
| 0 | 0x0836 / 2102 | Ia phase A current | A | ✅ HIGH (3 sources) |
| 1 | 0x0838 / 2104 | Ib phase B current | A | ✅ HIGH |
| 2 | 0x083A / 2106 | Ic phase C current | A | ✅ HIGH |
| 3 | 0x083C / 2108 | (average U L-N? — conflict) | V | ⚠️ LOW |
| 4 | 0x083E / 2110 | Ua phase A (L1-N) | V | ✅ HIGH |
| 5 | 0x0840 / 2112 | Ub phase B (L2-N) | V | ✅ HIGH |
| 6 | 0x0842 / 2114 | Uc phase C (L3-N) | V | ✅ HIGH |
| 7 | 0x0844 / 2116 | (average U L-L? — conflict) | V | ⚠️ LOW |
| 8 | 0x0846 / 2118 | Uab line-to-line A-B | V | ✅ HIGH |
| 9 | 0x0848 / 2120 | Ubc line-to-line B-C | V | ✅ HIGH |
| 10 | 0x084A / 2122 | Uca line-to-line C-A | V | ✅ HIGH |
| 11 | 0x084C / 2124 | **Frequency** | Hz | ✅ HIGH |
| 12 | 0x084E / 2126 | **Pt total active power** | W | ✅ HIGH (key for zero-export) |
| 13 | 0x0850 / 2128 | Pa active power A | W | ✅ HIGH |
| 14 | 0x0852 / 2130 | Pb active power B | W | ✅ HIGH |
| 15 | 0x0854 / 2132 | Pc active power C | W | ✅ HIGH |
| 16 | 0x0856 / 2134 | Qt total reactive | var | ✅ HIGH |
| 17–19 | 0x0858–0x085C / 2136–2140 | Qa/Qb/Qc reactive | var | ✅ HIGH |
| 20 | 0x085E / 2142 | St total apparent power | VA | ✅ HIGH |
| 21–23 | 0x0860–0x0864 / 2144–2148 | Sa/Sb/Sc apparent | VA | ✅ HIGH |
| 24 | 0x0866 / 2150 | PFt total cos φ | — | ✅ HIGH (scale 1, no divisor) |
| 25–27 | 0x0868–0x086C / 2152–2156 | PFa/PFb/PFc cos φ | — | ✅ HIGH |
| 28 | 0x086E / 2158 | (reactive energy imp.? — conflict) | — | ⚠️ LOW |
| 29 | 0x0870 / 2160 | (reactive energy exp.? — conflict) | — | ⚠️ LOW |
| 30–31 | 0x0872–0x0874 / 2162–2164 | (sources conflict) | — | ⚠️ LOW |
| 32 | 0x0876 / 2166 | **EP_imp active energy import** | kWh | ✅ HIGH |
| 33–35 | 0x0878–0x087C / 2168–2172 | (unknown — return 0.0) | — | ⚠️ return 0 |
| 36 | 0x087E / 2174 | **EP_exp active energy export** | kWh | ✅ HIGH |
| 37–39 | 0x0880–0x0884 / 2176–2180 | (unknown — return 0.0) | — | ⚠️ return 0 |

> The block is exactly 80 registers (2102…2181), i.e. 40×float32. The conflicting/unknown
> offsets are **not needed for zero-export** — only **Pt (offset 12 / 2126)** is critical for
> control; keep the rest plausible/zero. Must be filled with non-zero values: phase voltages
> (4/5/6), frequency (11), Pt (12).

## Parameter block `0x08A6` (dec 2214): 10 registers U16 (NOT float)

Request: `0B 03 08 A6 00 0A 27 24` (10 reg.). salakrzy returns **zeros** — and it works.
The working pazzero proxy returns meaningful values (U16, one per register):

| Address dec | Purpose | Value | Confidence |
|------------:|---------|-------|------------|
| 2214 | version/serial? | — | ⚠️ (zeros ok) |
| 2216 | energy reset (0=no) | 0 | ⚠️ |
| **2217** | **connection mode** (0=3P4W, 1=3P3W) | **0** (3P4W) | ✅ medium |
| 2218/2219 | CT ratio / PT ratio | 1 / 1 | ⚠️ |
| **2220** | protocol changeover | **3** | medium (this is the "3" in emavap!) |
| **2221** | meter address (11–19) | **11** | ✅ |
| 2222 | baudrate (1=9600) | 1 | ⚠️ |
| **2223** | **meter type** (0=1ph, 1=3ph) | **1** (3-phase) | ✅ medium |

> Start: you can **zero-fill 0x08A6** (like salakrzy — it works); if identification stalls,
> fill the parameters per pazzero (3P4W=0, addr=11, type=1). ⚠️ medium — single source.

## Power sign ✅

- **At the meter register level:** **"+" = import from grid, "−" = export.** ✅
- No emulator **inverts** the sign — they return it as-is. pazzero multiplies by −1 only
  because of its own source grid convention (SolarEdge); that is not a property of the meter.
- **The Victron sign matches** this (+import/−export) → when fed from Victron no inversion is
  needed. ⚠️ Still verify on the bench against FusionSolar — a sign error is the most common one.

## Scale ✅ (settled)

- **Return float32 directly in engineering units**: A, V, Hz, W, var, VA, cos φ, kWh. No
  divisors and no "×10".
- ❌ The earlier "power in 0.1 W (×10)" related to the integer/CT layer of specific pairs
  (ArminJo/Deye multiply integer-watts ×10 into float for compatibility with the *native*
  DTSU666), **not** to what SUN2000 expects in block 0x0836. cos φ — scale 1 (the ÷10 divisor
  in salakrzy is an artifact).
- ⚠️ Still cross-check the final Pt against the FusionSolar reading (CT ratio etc.).

## Readback from the inverter side (huawei-solar-lib, for monitoring/debug)

The inverter exposes on ITS side what it **read from the grid meter** (our emulator), in the
block **37100–37137** (read-only, FC 0x03 HoldingRegister). This is handy to see whether the
inverter "saw" the meter, **without going into the slow FusionSolar**. These are NOT the
emulator registers (`0x0836`) — this is the inverter's readout copy.

Access is via a Modbus-TCP client to the inverter (SDongle, port 502, unit_id 1); addressing
uses the **raw register number** (like 32069/40126).

| addr | quantity | Quantity | Type | Scale | Confidence |
|----:|:--------:|----------|------|-------|------------|
| **37100** | 1 | **Meter status** (0=offline, 1=online) | U16 | — | ✅ (address/type) |
| 37101 / 37103 / 37105 | 2 | Voltage L1 / L2 / L3 (L-N) | I32 | ÷10 → V | ✅ addr; scale ⚠️ |
| 37107 / 37109 / 37111 | 2 | Current L1 / L2 / L3 | I32 | ÷100 → A | ✅ addr; scale ⚠️ |
| **37113** | 2 | **Grid active power (Pt)** | I32 | W | ✅ addr/type; sign ⚠️ |
| 37115 | 2 | Reactive power | I32 | var | ✅ addr |
| 37117 | 1 | Power factor | I16 | ÷1000 | ⚠️ |
| 37118 | 1 | Grid frequency | I16 | ÷100 → Hz | ✅ |
| 37119 / 37121 | 2 | Energy export / import | I32 | ÷100 → kWh | ⚠️ |
| **37125** | 1 | Meter type (0=1ph, 1=3ph) | U16 | — | ✅ |
| 37132 / 37134 / 37136 | 2 | Active power L1 / L2 / L3 | I32 | W | ✅ addr |

> **For debugging you need two:** **37100** (does the inverter see the meter) and **37113**
> (what the inverter "thinks" the grid power is). The rest is reference, in case of
> currents/phases. Reading all 38 registers makes no sense — extra transactions on a busy dongle.
>
> **How to read just the two:** the simplest is **two small reads** — `37100` (q=1, status,
> U16) and `37113` (q=2, Pt, I32 signed). Assemble Pt from the two words: `(hi << 16) | lo`
> (a JS function yields a signed 32-bit automatically). Alternative — a single read `37100`
> **q=15** (reaching the low word of Pt at 37114) and parse words 0 and 13–14.
>
> ⚠️ **Verify by readback (not confirmed on our inverter):**
> - Addresses/types — from `wlcrs/huawei-solar-lib` (the same source as for 47415/47416,
>   marked ✅), but the 37100 block specifically has not been verified on our 15KTL-M2.
> - **Sign of 37113:** compare live with the value the emulator sends (+import/−export,
>   already verified against Victron/FusionSolar). Matches → same convention; inverted →
>   handle it downstream.
> - Scales (÷10/÷100) — Huawei standard; confirm by plausibility (V≈230, f≈50.00).

## Gotchas (from practice)

- **Address = 11**, not 1. ✅
- **handshake 0x07D1 → 0x3B11** (old firmware 0x3F80). ✅
- **Polled at 0x0836, not 0x2000.** ✅
- **Non-zero:** phase voltage (4/5/6), frequency (11), Pt (12) — must be valid; otherwise
  "meter offline/abnormal". ✅
- **Timing:** the master polls roughly once per second and expects fast RTU replies; keep the
  reply path non-blocking (ESPHome RTU-slave timing risk). ✅
- **Firmware:** the export-limitation menu and the exact handshake code depend on the version. ⚠️
- **Before trusting zero-export** cross-check the emulated Pt against FusionSolar. ✅

## Donor implementations (reviewed line by line)

- **`salakrzy/DTSU666_CHINT_to_HUAWEI_translator`** — ESP32 (ESP-IDF) + eModbus
  (`ModbusServerRTU`), slave 0x0B, 9600 8N2, FC03, handshake 0x07D1→0x3B11, block 0x0836
  (80 reg.) + 0x08A6 (10 reg., zeros). **The main format donor** (captured real frames).
- **`Vuko791/sun2000-dtsu666h-sniffer`** — Node-RED decoder of **live** inverter↔meter
  traffic; per-register layout 2102…2174. **Ground truth.**
- **`pazzero/esphome-dtsu666h-modbus-proxy`** — a working **ESPHome** proxy; confirms the
  handshake (2001→15121), the data block and parameters 2214–2223. **Direct ESPHome reference.**
- `MichielfromNL/DTSU666Emulator` — ESP8266/ESP32, C++ class; loop()-Modbus (modbus-esp8266),
  not eModbus; solves blocking via push-MQTT (important for timing).
- `ArminJo/Arduino-DTSU666H_PowerMeter` — measures CT current, answers 0x2014/0x151E/0x0006;
  the "0.1 W units" is about the float interpretation of the native DTSU666, not about the
  SUN2000 block 0x0836.
- `emavap/dtsu666-emulator` — HA Modbus-TCP; puts "3" into 0x7d1 and a 0x2000-style map —
  **a different convention** (TCP); for RTU-SUN2000 the salakrzy combo (0x3B11/0x0836) is correct.

URLs — in [`sources.md`](sources.md).
