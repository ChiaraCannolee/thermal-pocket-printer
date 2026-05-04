# C&Co 3128 / DP-L1S BLE Protocol

Reverse-engineered from decompiled Lucky Jingle APK (`com.dingdang.newprint`) and verified against hardware.

## Device Info

| Property | Value |
|---|---|
| Brand name | Crafts & Co 3128 (C&Co 3128) |
| Internal model | DP-L1S |
| SDK class | `DP_L1S` → `BaseNormalDevice` → `BaseDevice` |
| SDK | LuckPrinter SDK (`com.luckprinter.sdk_new`) |
| Manufacturer | Xiamen Print Future Technology Co., Ltd (also branded as Lujiang) |
| Printhead | 384 pixels wide (48 bytes/row) |
| DPI | 203 (8 dots/mm) |
| Paper width | ~57mm |
| Connection | BLE 4.0 |
| Battery | reports via command `10 FF 50 F1` |

## BLE Discovery

The printer advertises as **`C&Co 3128_BLE`** but does **not** include its service UUIDs in the advertisement payload.

**Implication for clients:**
- Native BLE clients (e.g. Python `bleak`) can scan all nearby devices and identify the printer by name.
- Web Bluetooth clients (Chrome) must use a `namePrefix` filter (e.g. `'C&Co'`) and declare `ff00` as an `optionalServices` entry, because filtering on `services: [0xff00]` would only match cached devices and fails on first-time pairing or new origins.

## BLE Services

| Service UUID | Write Char | Notify Char | Notes |
|---|---|---|---|
| `0000ff00-0000-1000-8000-00805f9b34fb` | `ff02` | `ff01` | Primary control + data |

## Command Format

All control commands use the format:

```
10 FF [cmd] [param...]
```

Where `0x10` = DLE, `0xFF` = prefix byte.

## Command Reference

### Device Info

| Command | Response | Description |
|---|---|---|
| `10 FF 20 F0` | ASCII model name | Get printer model (e.g. "C&Co 3128") |
| `10 FF 20 F2` | ASCII serial number | Get serial number |
| `10 FF 20 EF` | ASCII string | Get device boot info |
| `10 FF 20 F1` | ASCII version string | Get firmware version (e.g. "V1.10") |
| `10 FF 70` | Pipe-delimited string | Get all printer info (name\|mac\|...\|battery) |
| `10 FF 50 F1` | 2 bytes | Get battery level (byte[1] = percentage) |
| `10 FF 11` | 1 byte | Get current density setting |
| `10 FF 13` | 1-2 bytes | Get auto-shutdown time |
| `10 FF 20 A0` | 1 byte | Get print speed |

### Printer Status

| Command | Response | Description |
|---|---|---|
| `10 FF 40` | 1 byte (bitfield) | Get printer status |

Status bitfield:

| Bit | Meaning |
|---|---|
| 0 | Is printing |
| 1 | Cover open |
| 2 | Paper empty |
| 3 | Low battery |
| 4-6 | Overheating (bit 4 or 6 set) |
| 5 | Charging |

### Print Control

| Command | Response | Description |
|---|---|---|
| `10 FF F1 03` | - | Enable printer (Lujiang mode = 3) |
| `00 00 00 00 00 00 00 00 00 00 00 00` | - | Wake up printer (12 null bytes) |
| `1B 4A [n]` | - | Feed paper by n dots |
| `10 FF F1 45` | `AA` or `OK` | Stop print job |
| `10 FF 04` | `OK` | Recovery/reset |

### Settings

| Command | Response | Description |
|---|---|---|
| `10 FF 10 00 [n]` | `OK` | Set density (0-2) |
| `10 FF 12 [hi] [lo]` | `OK` | Set auto-shutdown time (minutes, 16-bit) |
| `10 FF C0 [n]` | `OK` | Set print speed |
| `10 FF 15 [lo] [hi]` | - | Set print width (pixels, 16-bit LE) |
| `10 FF 30 27 [n]` | `OK` | Set printer mode |
| `1F 70 01 [n]` | `OK` | Set heating level |
| `1F 80 [type] [param]` | `OK` | Set paper type |

### Label/Tag Control

| Command | Response | Description |
|---|---|---|
| `1B BB CC` | - | Mark print first (start label sequence) |
| `1B BB BB` | - | Mark print last (end label sequence) |
| `1B BB AA` | - | Mark print not-last (continue) |
| `1D 0C` | - | Position to next label |
| `1F 11 [n]` | - | Auto-adjust label position |
| `1F 11 51` | - | Adjust position start (0x51 = 81) |
| `1F 11 50` | - | Adjust position end (0x50 = 80) |

### Platform

| Command | Response | Description |
|---|---|---|
| `FC FF 00 02 45 02 00 46` | - | Set platform identifier |

## Print Sequence

### Normal Print (receipt/continuous paper)

```
1. 10 FF F1 03          Enable printer
2. 00 00 00 00 ...      Wake up (12 null bytes)
3. 1D 76 30 00 ...      GS v 0 raster image (header + bitmap data)
4. 1B 4A 50             Feed paper 80 dots
5. 10 FF F1 45          Stop print job → wait for AA or OK
```

### Label/Sticker Print

```
1. 10 FF F1 03          Enable printer
2. 00 00 00 00 ...      Wake up (12 null bytes)
3. 1F 11 51             Adjust position (start, first label only)
4. 1D 76 30 00 ...      GS v 0 raster image
5. 1D 0C                Position to next label
6. 1F 11 50             Adjust position (end, last label only)
7. 10 FF F1 45          Stop print job
```

### Tattoo Print

```
1. 10 FF F1 03          Enable printer
2. 00 00 00 00 ...      Wake up (12 null bytes)
3. 1F 80 01 40          Set paper type (tattoo)
4. 1D 76 30 00 ...      GS v 0 raster image
5. 1B 4A [endLineDot]   Feed paper
6. 10 FF F1 45          Stop print job
```

## Bitmap Format (GS v 0, uncompressed)

The printer accepts standard ESC/POS raster image commands:

### Header (8 bytes)

```
1D 76 30 [m] [wL] [wH] [hL] [hH]
```

| Byte | Value | Description |
|---|---|---|
| `1D 76 30` | fixed | GS v 0 command |
| `m` | 0-3 | Mode: 0=normal, 1=double width, 2=double height, 3=both |
| `wL` | low byte | Width in bytes (pixels / 8, rounded up) |
| `wH` | high byte | Width in bytes (high byte, usually 0) |
| `hL` | low byte | Height in pixels (low byte) |
| `hH` | high byte | Height in pixels (high byte) |

For the C&Co 3128: width = 384px = 48 bytes, so `wL=0x30, wH=0x00`.

### Pixel Data

- 1 bit per pixel, 8 pixels per byte
- MSB first (bit 7 = leftmost pixel)
- Dark pixel = 1 (prints black), light pixel = 0
- Total bytes = width_bytes × height_pixels
- Threshold: RGB average < 128 = dark (configurable in the web app via the threshold slider in the print preview)

### Compressed Format (not required)

The SDK also supports a compressed format with header `1F 10`, using
proprietary `Compress.codeESC()` or `Compress.codeLihu()` from native
code (`libcompress.so`). The C&Co 3128 accepts the uncompressed GS v 0
format, so compression is unnecessary.

## Related Projects

This printer is part of the LuckPrinter family. The SDK supports 159+
printer models. Related reverse-engineering work:

- [fichero-printer](https://github.com/0xMH/fichero-printer) by 0xMH:
  Fichero D11s (AiYin variant, 96px label printer, same SDK)

## Device Class Hierarchy

From the decompiled SDK:

```
BaseDevice
└── BaseNormalDevice (print flow, bitmap encoding, command protocol)
    ├── BaseNormalDevice.base
    │   ├── BaseA4Device (A4 paper printers)
    │   ├── BaseLuckPA4Device
    │   └── BaseLujiangNormalDevice
    ├── DP_L1S (this printer)
    ├── DP_L1 / DP_D1 / DP_S1 / etc.
    ├── LuckP_L1 / LuckP_D1 / etc.
    └── MiniPocketPrinter / PPD1 / PPS1 / etc.
```

The `DP_L1S` class sets:
- `printWidth = 384`
- `compress = true` (but uncompressed also works)
- `compressWay = 0` (would use `Compress.codeESC()`)
- `endLineDot = 80`
- `enablePrinterMode = 3`
