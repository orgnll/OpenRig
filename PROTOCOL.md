# Switch² Bluetooth LE protocol (unofficial)

Documented for interoperability so owners can control their own device. Derived from the
publicly served official web app (drdabber.app). Fields marked **(unverified)** have not yet been
confirmed on real hardware.

## GATT layout

| Purpose | UUID |
|---|---|
| Main service | `0000fee7-0000-1000-8000-00805f9b34fb` |
| Write (app → rig), write-without-response | `0000fec1-0000-1000-8000-00805f9b34fb` |
| Notify (rig → app) | `0000fec2-0000-1000-8000-00805f9b34fb` |
| Device Information (model 2a24, serial 2a25, hw 2a27, fw 2a28, maker 2a29) | `0000180a-…` |
| OTA firmware service (not used by OpenRig) | `e5c7a653-7e52-4b06-af32-960455084ae3` |

Scan filter: devices advertising service `fee7`.

## Connection sequence

1. Connect GATT, get service `fee7`, subscribe to notifications on `fec2`.
2. **Auth handshake** (below). The official app does this once per connection.
3. Send **clock sync** (`0xB1`). The rig then pushes its full state: status, all presets, custom curves.
4. Optional: request **analytics** (`0xB2`).

## Auth handshake

Smart-lock-style framing, sent to `fec1`:

```
hex = "ff02" + LEN + OP + SHA1(KEY ‖ NONCE) + NONCE + CRC16(LEN ‖ OP ‖ SHA1 ‖ NONCE)
KEY   = 61e2128b5bb08da1a4f0efeecd6ffd24ff384274   (hex bytes)
NONCE = 8 random digits, each encoded as one byte 0x00–0x09
OP    = 0000  (CMD_AUTH)
LEN   = n as 2 bytes, then the low 2 bytes of (0xFFFFFFFF XOR n), where n = len(OP+SHA1+NONCE)+2
CRC16 = custom poly 0x8005 variant (see crc16() in index.html)
```

The hex string is split into 18-byte chunks. Each chunk gets a 2-byte header: `[HEAD, SEQ]`.
HEAD for a full, non-final chunk is `0x94 | {0x00,0x20,0x40,0x60}[seq % 4]`. For the final chunk it's `chunk_len + 2`.
Replies come back with the same framing on `fec2`. A chunk whose first byte has bit 7 clear is the
last one. After stripping the 2-byte headers, payload byte 7 = `0x00` means success.

The rig also derives an SM4 session key from both nonces, but no Switch² setting command uses it.
Settings are plain packets.

## Command packets (app → rig)

Every packet is `[CMD, LEN, …payload…, CMD]` where LEN is the total packet length. Temperatures are
always sent in **both** units, big-endian 16-bit.

| CMD | Name | Layout |
|---|---|---|
| `B1` 177 | Clock sync (UTC) | `B1 09 YY YY MM DD hh mm B1` |
| `B2` 178 | Request analytics | `B2 04 FF B2` |
| `B3` 179 | Preset temperature | `B3 08 P F_hi F_lo C_hi C_lo B3` (P = 1–5) |
| `B5` 181 | Preset heating mode | `B5 05 P MODE B5` |
| `B7` 183 | Preset hold time (s) | `B7 06 P S_hi S_lo B7` (10–90) |
| `B8` 184 | Factory reset | `B8 04 FF B8` (not used by OpenRig) |
| `B9` 185 | Device settings (all at once) | see below |
| `BA`/`BB` 186/187 | Custom curve points 1–3 / 4–6 | `BA 13 P [F_hi F_lo C_hi C_lo T]×3 BA` |
| `C6` 198 | Cleaning assist | `C6 04 (AA=start / 00=stop) C6` |
| `D1`/`D2` 209/210 | Rename (chars 0–16 / 17–28) | `D1 14 c0…c16 D1`, `D2 0F c17…c28 D2` |

Heating modes: `A1` Steady, `B1` Ascent, `C1` Descent, `D1` Valley, `E1` Hill, `F1` Custom.
For Custom, first set mode `F1`, then send `BA` and `BB`. Unused points are zero.

### `B9` settings packet (20 bytes)

```
[0]  B9      [1]  14 (20)   [2]  00
[3]  active preset 1–5      [4]  00  [5] 00
[6]  light mode 0–33        [7]  00
[8]  auto shut-off minutes (0 = off, 1–60)
[9]  unit: 0F = °F, 0C = °C
[10] session: AA = heat on, 00 = off
[11] haptics: AA = on, 00 = off
[12] session extend 0–9
[13] light brightness 0–100
[14–18] 00   [19] B9
```

The packet always carries every setting, so build it from the full current state. Otherwise you'll
reset the fields you didn't mean to change.

## Notifications (rig → app)

| First byte | Meaning |
|---|---|
| `A1` 161 | Clock echo `YY YY MM DD hh mm` |
| `A2` 162 | Analytics, multi-part, selected by LEN: 20/17/19 = favorites, cycles, charge cycles; 7/11 = total time (u32); 13 = per-preset cycle counts |
| `A3` 163 | Preset temp: `[2]=P, [3–4]=°F, [5–6]=°C` |
| `A5` 165 | Preset mode: `[2]=P, [3]=MODE` |
| `A7` 167 | Preset hold: `[2]=P, [3–4]=seconds` |
| `A8` 168 | Factory reset status |
| `A9` 169 | **Live status**: `[3]` preset, `[6]` light, `[7]` session enabled, `[8]` auto-off, `[9]` auto-off remaining, `[10–11]` live temp, `[12]` unit, `[13]` session seconds left, `[14]` haptics, `[15]` extend, `[16]` battery %, `[17]` charging (AA), `[18]` brightness |
| `AA`/`AB` 170/171 | Custom curve points 1–3 / 4–6 for preset `[2]` |
| `C1`/`C2` 193/194 | Device name, two chunks |
| `C3` 195 | Serial string |
| `E1` 225 | Device / battery / PCB / chamber status bytes **(unverified meaning)** |
| `F1` 241 | Cleaning assist: `[2]` state, `[3]` seconds left |

Preset index 0 in notifications appears to be the "live/unsaved" slot **(unverified)**.

## Light modes

0 Stealth, 1 Calm, 2 Purple, 3 Blue, 4 Cyan, 5 Green, 6 Yellow, 7 Orange, 8 Red, 9 Pink,
10 Cali Sunset, 11 Purple Haze, 12 Northern Lights, 13 Vegas Nights, 14 Blue Dream, 15 Strawberry Cough,
16 Florida Groves, 17 Lime Light, 24 Blue Cheese, 25 Golden Goat, 26 Grape Ape, 27 Miami Vice,
28 Purple Urkle, 29 Candy Corn, 30 Redrum, 31 Skull & Bones, 32 Witches Brew, 33 Jack-O-Lantern.

## Limits

Temperature 250–650 °F / 121–343 °C · hold 10–90 s · auto shut-off 0 or 1–60 min · session extend 0–9.

## Why the official app feels slow

- It waits **1000 ms** between each write when saving a preset (temp → hold → mode), so one preset takes 3–4 s.
- Bulk saves wait 250 ms per write across all five presets.
- Setting changes are debounced 250 ms, plus a 100 ms minimum gap.
- On connect it reads every Device Information field before showing anything.

OpenRig sends only what changed, with a short write pacing (40/90/250 ms, user-selectable). It updates
the screen right away and confirms against the rig's echo. Device info loads in the background.
