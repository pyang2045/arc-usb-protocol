# Otii Arc USB protocol (firmware 3.x)

This describes the USB protocol of the Otii Arc power analyzer, as seen on the wire with HW 1.3
and FW 3.1.3. It is an independent, unofficial description written for interoperability.
Anything not confirmed on hardware is marked *(unverified)* or *(provisional)*.

> **Disclaimer:** not affiliated with, endorsed by or supported by Qoitech AB. "Otii" and
> "Otii Arc" are trademarks of their respective owner and are used here only to identify the
> hardware. This document may contain errors; using it to drive a power supply is at your own
> risk.

## 1. Transport

The Arc is a **USB CDC-ACM** serial port, so no driver is needed:

* USB VID `0x0FCE`, PID `0xD1E6`
* macOS: `/dev/cu.usbmodem*`, Linux: `/dev/ttyACM*`
* The baud rate doesn't matter (USB CDC); use 115200 8N1, raw mode.
* Only one process can own the port.

## 2. Framing

Every packet, in both directions:

```
offset size  field
0      4     magic      a3 2c b5 7f
4      2     length     u16 LE, number of payload bytes
6      2     checksum   u16 LE = (sum of all payload bytes) & 0xffff
8      N     payload
```

All integers are little-endian. Samples are IEEE-754 float32.

### Host → device payload

```
u32 txnid     incrementing transaction id
u32 command
...           command arguments (usually 2 × u32/i32)
```

### Device → host payload

The first u32 is a packet type:

| type | meaning |
|---|---|
| `0xff99030e` | command response |
| `0xff2ff2ce` | monitor/status packet, sent unsolicited at about 8 Hz, even when idle |
| `0xff2a8369` | sample data, sent while recording |

Response: `u32 type, u32 txnid, i32 error, payload…`. Error codes seen:

| error | meaning |
|---|---|
| 0 | ok |
| -3 | unknown or unsupported property id |
| -201 | value out of range (e.g. positive `maincur`) |
| -301 | value not supported by this hardware (`pwrreg = inline` on Arc) |

> **Always write complete frames.** The device trusts the length field. If a host writes only part
> of a frame (e.g. a partial `write()` on a non-blocking port), the Arc keeps consuming every later
> byte as payload and seems to hang, although monitor packets keep flowing. To recover, send up
> to 4136 zero bytes (max frame size). That completes the stuck frame, its bad checksum gets it
> dropped, and the next hello works. A host can do this automatically when hello times out.

Example, hello:

```
TX a32cb57f 0800 1500 | 01000000 14000000
RX a32cb57f 1c00 3a03 | 0e0399ff 01000000 00000000 | 03000000 20100000 20637241 27000000
                          resp      txn 1    ok       proto=3  maxlen=4128 "Arc " feat=0x27
```

## 3. Commands

| cmd | name | args | response payload |
|---|---|---|---|
| `0x0a` | keepalive | `u32 timestamp_us, u32 70` | `u32 0` |
| `0x14` | hello | none | `u32 protocol(3), u32 max_payload(4128), u32 fourcc (BE "Arc "), u32 featurebits` |
| `0x64` | get property | `u32 prop_id, u32 0` | value (see §4) |
| `0x66` | set property | `u32 prop_id, i32 value` | echoed value |
| `0x6a` | calibrate | none | none. **The reply only arrives when calibration finishes (~7.5 s)** |
| `0x78` | channel enable | `u32 channel_id, u32 divider` (1 = native rate, 0 = disable) | none |
| `0x7c` | recording start | none | none |
| `0x7e` | recording stop | none | none |
| `0x82` | UART write | `u32 25` (TX channel) + raw bytes (max payload − 12 per frame) | none |
| `0x8c` | battery profiler start | none (configuration in props 21–24, §8) | none |
| `0x8e` | battery profiler stop | none (also accepted when idle) | none |

Examples:

```
calibrate               a32cb57f 0800 5401 | ea000000 6a000000            -> reply 7.5 s later
uart write "Hi\n"       ... | 8f000000 82000000 19000000 48690a
```

The keepalive **isn't required**: the main output stays on with no host connected, and settings
persist across port open/close. Sending one per second while streaming is harmless.

### Startup

1. Open the port, flush it, send **hello**. The device may still be streaming monitor or sample
   packets from an earlier session, so skip anything that isn't the hello response.
2. GET the properties you need (§4). `id | 0x800` returns a property's maximum value.
3. Optionally `SET 40 = 1` (UART on) and `CHANNEL 23,1` (GPI digital channel).

## 4. Properties

GET returns an `i32` for int, bool and enum properties, `N × i32` for vectors, or an ASCII string.
OR-ing `0x800` into the id returns the maximum value instead.

| id | name | type | device unit | notes |
|---|---|---|---|---|
| 1 | name | str | | "Arc" |
| 4 | intres | int | mΩ | battery emulation internal resistance, max 5000 Ω |
| 5 | 4wire | bool | | remote sense |
| 6 | mainsrccurrmode | enum | 0=cutoff, 1=limit | over-current behaviour |
| 8 | highrange | bool | | force high current range |
| **9** | **main** | bool | | **main output on/off** |
| 10 | pwrreg | enum | 0=voltage, 1=current, 10=inline, 100=off | regulation mode. `inline` gets -301 on Arc |
| **11** | **mainvolt** | int | µV | output voltage, max 4.55 V |
| **12** | **maxcur** | int | mA | over-current level, 0–5 A |
| 13 | maincur | int | µA | constant current in current mode, -2.5–0 A (negative = sink). Positive values get -201. **GET and the SET echo return only the low 16 bits** (see note) |
| 21 | batt_profile_steps | blob | | profiler step list, 12 bytes per step (§8). SET takes the whole blob |
| 22 | batt_profile_exit_iter | int | | profiler max iterations |
| 23 | batt_profile_exit_ocv | int | µV | profiler OCV cutoff |
| 24 | batt_profile_exit_volt | int | µV | profiler loaded-voltage cutoff |
| 30 | adcres | int | µΩ | ADC-input shunt value (0.39 Ω = 390000) |
| 40 | uart | bool | | UART enable |
| 41 | baudrate | int | baud | UART baud rate |
| 50 | gpo | mask | | write-only set/clear mask, see below |
| 51 | expvolt | int | µV | expansion-port IO voltage, 1.2–5 V |
| 52 | expoutvolt | int | µV | expansion-port 5 V output, 0 or 5000000 |
| 60 | sinkoffsetcorr | bool | | |
| 81–85, 88, 94–96 | *_CALIBRATION | int / int[4] | | factory calibration, read-only |
| 126 | bootversion | str | | |
| 128 | hwversion | str | | e.g. "1.3" |
| 129 | fwversion | str | | e.g. "3.1.3" |
| 130 | boardserial | str | | |
| 132 | uid | str | | same as serial |
| 141 | chinfo | struct[] | | channel table, see §5 |

**GPO (id 50):** each output uses 3 bits. Bit `3(n-1)` clears output *n*, and bit `3(n-1)+1` sets it.
Output 3 is the UART TX pin. The mask can't be read back (GET 50 returns -3).

| action | value |
|---|---|
| gpo1 on / off | `0x02` / `0x01` |
| gpo2 on / off | `0x10` / `0x08` |
| TX pin high / low | `0x80` / `0x40` |

**maincur note:** writing -500000 (-0.5 A) reads back as `0x00005ee0`. Whether the device applies
the full 32-bit value internally can only be checked with a load *(unverified beyond ±32 mA)*.

Examples:

```
set mainvolt 1.8 V      66: 0b000000 40771b00   (1800000 µV)
set maxcur 0.1 A        66: 0c000000 64000000   (100 mA)
set main on / off       66: 09000000 01000000 / 00000000
set pwrreg current      66: 0a000000 01000000
set maincur -0.01 A     66: 0d000000 f1d8ffff   (-9999 µA)
```

## 5. Channels

The channel table (GET 141) holds 16-byte entries:
`u16 channel_id, u16 flags (0x8000 = analog, 0x0001 = digital, 0x0002 = log), u32 0, u32 sample_rate, u32 0`.

| id | name | unit | rate | |
|---|---|---|---|---|
| 0 | mc | A | 4000 | main current |
| 1 | mv | V | 1000 | main voltage |
| 2 | ac | A | 1000 | ADC current (via adcres) |
| 3 | av | V | 1000 | ADC voltage |
| 4 / 5 | sn / sp | V | 1000 | sense − / + |
| 6 | mp | W | 4000 | main power |
| 7 | ap | W | 1000 | ADC power |
| 12/13/14 | mc1/mc2/mc3 | A | 1000/1000/4000 | per-range raw currents |
| 16 | vb | V | 1000 | USB VBUS |
| 17 | vj | V | 1000 | DC jack |
| 20 | tp | °C | 1000 | temperature |
| 22 | rx | log | | UART RX |
| 23 | gpi | digital | | GPI 1–6 |

**The sample rate is fixed on the Arc.** The `0x78` divider argument is accepted but ignored: values
1, 2, 4, 10 and 40 all gave 4000/s for `mc` and 1000/s for `mv`. Lower rates have to be produced on
the host, e.g. by averaging blocks of samples.

## 6. Monitor packet (`0xff2ff2ce`, ~8 Hz, always on)

```
u32 type, u32 0, then repeated TLV: u16 tag, u16 len, u32 id, value
```

`tag` 2 = present value, `tag` 1 = property-change event (`id` = property, `i32` new value; e.g.
`9 = 1/0` when the profiler switches main on/off). `len` includes the 4-byte id.

| id | len | value |
|---|---|---|
| 0–7, 16, 17, 20 | 8 | float32 present value of that channel (mc, mv, ac, av, sn, sp, mp, ap, vb, vj, tp) |
| 22 | 8 | u32 **GPI bitmask**: bit n-1 = GPI n. GPI3 is the UART RX pin |
| 65536 | 12 | 8 bytes, always zero so far *(unknown)* |
| 65537 (0x10001) | 28 / 4 | battery profiler entry (§8). Empty (len 4) = profiler finished |
| 65538 | 8 | u32 **4-wire state**: 0 = cal_invalid, 1 = disabled, 2 = inactive, 3 = active |
| 65539 (0x10003) | 24 | battery profiler exit info (§8) |

This gives instant readings of every channel without starting a recording. The monitor `mv`
reports the regulator voltage even when `main` is off.

## 7. Sample packet (`0xff2a8369`)

```
u32 type, u32 seq
repeat: u16 channel_id, u16 count, u32 sample_rate, count × float32
```

About one packet per millisecond, e.g. 4 `mc` samples at 4 kHz plus 1 `mv` sample at 1 kHz. Unlike
the monitor packet, the streamed `mv` is the real output voltage, so it reads 0 V while `main` is off.

```
CHANNEL 0,1     (mc)
CHANNEL 1,1     (mv)
REC_START (0x7c)
... sample packets ...
REC_STOP  (0x7e)
CHANNEL 0,0 ; CHANNEL 1,0
```

## 8. Battery features

### Emulator (host side)

The device has no battery model of its own. Emulation is a host loop that rewrites two properties
about every 200 ms:

```
loop:
  SET 11 (mainvolt) = OCV(used capacity)
  SET 4  (intres)   = R(used capacity)
stop: SET 4 = 0, SET 11 = previous main voltage
```

`OCV` and `R` are interpolated from a battery-profile JSON (`dischargetables[0].table[]` rows of
`capacity` [mAh used], `voltage` [V], `resistance` [Ω]). The Arc applies `intres` itself, so the
output voltage drops with load current. The host integrates the measured current to track the
state of charge.

### Profiler (on the device)

The battery goes on the main output. The device repeats a step sequence and reports one entry
per iteration until a cutoff:

```
SET 22 = max iterations
SET 23 = OCV cutoff [µV]
SET 24 = voltage cutoff [µV]
SET 21 = steps blob, per step: f32 duration [s], f32 value, u32 type (0 current [A], 1 power [W], 2 resistance [Ω])
CMD 0x8c                             start; device switches main on (monitor event 9=1)
... monitor 0x10001 entry once per iteration
... monitor 0x10003 exit info, then empty 0x10001, event 9=0
CMD 0x8e                             stop early
```

Example blob for power 50 mW/2 s, 100 Ω/3 s, 4 mA/5 s:
`00000040 cdcc4c3d 01000000 | 00004040 0000c842 02000000 | 0000a040 6f12833b 00000000`.

Exit info (0x10003): `u32 iterations, u32 reason, f32 ocv, f32 (unused), f32 voltage`. Reason 3 =
voltage cutoff was observed. The other codes are *(provisional)*: 0 script, 1 iteration, 2 ocv,
4 user, 5 disconnected, 6 aborted.

Entry (0x10001, len 28): `u32 iteration, f32 OCV [V], f32 R [Ω], f32 used capacity [mAh]`, then
8 more bytes *(provisional: not yet observed with a battery attached; with nothing connected the
device exits at once with reason 3 at ~0.02 V)*.

### Validation scripts (host side)

The Arc firmware has no script engine, so charge/discharge/sleep sequences have to run on the
host with ordinary properties:

* discharge → `pwrreg current`, `maincur = -I` (sink). Power/resistance modes are recomputed from
  the monitor voltage at ~8 Hz.
* charge → `pwrreg voltage`, `mainsrccurrmode limit`, `maxcur = I`, `mainvolt = V` (CC/CV).
* sleep → main off.

## 9. Open questions

* UART RX log data format (needs a TX→RX loopback or external sender).
* Profiler entry layout with a real battery attached (§8).
* Whether `maincur` beyond ±32 mA is applied in full (§4).
* Firmware 1.x devices use a different, text-based protocol and need a firmware update first.

## License

Copyright (c) 2026 Paul Yang. This document is licensed under the
[Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/)
(CC BY 4.0); see [LICENSE](LICENSE).
