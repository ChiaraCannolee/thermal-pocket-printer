# crafts-and-co-thermal-pocket-printer

Print to the **Crafts & Co 3128** thermal pocket printer directly from your computer via Bluetooth, no app required.

The Crafts & Co 3128 (sold at Action stores in the Netherlands/Europe) is a rebranded **DP-L1S** by Xiamen Print Future Technology, using the **LuckPrinter SDK**. Its companion app ("Luck Jingle") requires excessive permissions and a persistent internet connection for a device that prints over Bluetooth. This project removes the app from the equation.

## Quick start: Open the web app

**No installation needed.** Just open this in Chrome, Edge, or Opera:

**https://ChiaraCannolee.github.io/thermal-pocket-printer/**

(Requires Web Bluetooth support – Chrome/Edge/Opera on desktop, not Firefox/Safari)

## What this does

The web app gives you everything you need to design and print labels, stickers, and receipts:

### Design tools
- **Image tab**: drop a PNG/JPG/etc. and the app auto-fits it to your label
- **Text tab**: type text, adjust size, drag it directly on the preview to reposition (or use arrow keys to nudge — Shift+arrow for bigger steps). Click "⊕ Center" to snap back to the middle.
- **Test tab**: prints an alignment pattern with a gradient bar so you can verify density and dithering settings

### Label sizes
Six presets covering the most common Action sticker rolls (29×12, 40×12, 50×30, 40×30mm rectangular labels, 30mm round stickers) plus an Auto/Receipt mode for 56mm continuous paper, with custom dimensions in mm. The white preview panel scales to match your selected label proportions.

For the 56mm continuous paper, the preview shows shaded areas on either side to indicate the actual printable zone — the printhead covers 48mm of the 56mm-wide paper, leaving 4mm margin on each side.

For round labels, content is automatically masked to a circle so anything outside doesn't print.

### Print settings
- **Density** – three levels (light, normal, dark)
- **Dither** – Floyd-Steinberg for photos and gradients
- **Invert** – swap black and white
- **Label mode** – for pre-cut labels with gaps. Off = continuous paper.

### Print preview
Before printing, you get a preview dialog with:
- **Threshold slider** – live preview of how the threshold affects your image (0-255)
- **Copies** – print up to 99 of the same design
- **Density override** – override your sidebar density just for this print
- **Feed after print** – extra paper feed in mm

The preview opens even when the printer isn't connected, so you can review your settings before connecting (handy because the printer auto-shuts-off after a while of inactivity).

### Templates
Save any design as a template with the floppy-disk icon. Templates store the image, all print settings, label size, and text position. Set one as default and it auto-loads on next visit. Stored locally in your browser (no cloud, no account).

### Other niceties
- **Undo/redo** with Ctrl+Z / Ctrl+Shift+Z (or Cmd on Mac), 30 steps deep
- **Editable printer name** – click the title to rename your printer; persists across sessions
- **ESC closes any open modal**
- **Click outside a modal to dismiss it**
- **Battery indicator** appears once connected

## Python CLI (for automation)

If you prefer command line or want to batch-process prints:

```bash
# Install dependencies
pip install bleak Pillow

# Print a test pattern
python3 print.py test

# Print an image
python3 print.py image photo.png

# Print with dithering (for photos/gradients)
python3 print.py image photo.png --dither

# Print text
python3 print.py text "Hello World"

# Print on sticker paper (label mode)
python3 print.py text "My Label" --label

# Check printer battery and status
python3 print.py info
```

## Web GUI vs CLI

The **web GUI** (`index.html`) is the easiest way to get started — no installation, just open it in Chrome.

The **Python CLI** (`print.py`) is for automation, batch jobs, and integrations. Requires `pip install bleak Pillow`.

## Usage (CLI)

```
python3 print.py <command> [options]

Commands:
  scan                  Scan for nearby BLE printers
  info                  Show printer info (model, battery, firmware)
  test                  Print a test pattern
  image <file>          Print an image (PNG, JPG, BMP, etc.)
  text <string>         Print text

Options:
  --address, -a         Printer BLE address (skip scanning)
  --density, -d 0|1|2   Print darkness (0=light, 1=normal, 2=dark)
  --dither              Floyd-Steinberg dithering (better for photos)
  --invert              Invert colours (white-on-black)
  --label               Label/sticker mode with gap detection
  --copies, -c N        Number of copies
  --width, -w N         Print width in pixels (default: 384)
  --feed, -f N          Paper feed after print in dots (default: 80)
```

## Compatible paper

The printer uses 56mm wide thermal sticker and label rolls (30mm label diameter). Compatible supplies include:

- Standard white thermal paper (included with printer)
- Crafts & Co clear glossy sticker rolls
- Crafts & Co white sticker rolls
- Crafts & Co coloured sticker rolls (pink, yellow, etc.)
- Standard 56mm thermal sticker/label rolls

The printer is "ink free": it uses heat to activate the thermal coating. Coloured papers just provide a coloured background under the black print.

## How it works

The printer communicates over BLE using a variant of the ESC/POS thermal printer protocol. The protocol was reverse-engineered from the decompiled Android APK and verified against hardware.

### Connection & Protocol

1. **Connect** to BLE service `ff00`, write to characteristic `ff02`, listen for notifications on `ff01`
2. **Advertise name**: printer broadcasts as "C&Co 3128_BLE" — note: no service UUIDs in advertisement, so the web app filters on name prefix
3. **Enable printer**: send `10 FF F1 03` (Lujiang-specific command)
4. **Wake up**: send 12 null bytes
5. **Set density** (optional): `10 FF 10 00 [0|1|2]` for light/normal/dark
6. **Send bitmap**: GS v 0 raster image (384 pixels wide, 1-bit, MSB-first)
7. **Feed paper**: `1B 4A 50` (feed 80 dots)
8. **Stop job**: `10 FF F1 45` (wait for response)

For label/sticker paper with gap detection:
- Replace step 7 with `1D 0C` (position to next label)
- Use `1F 11 51` before print and `1F 11 50` after for position adjustment

See [PROTOCOL.md](PROTOCOL.md) for complete command reference.

## Compatible printers

This tool is confirmed to work with the Crafts & Co 3128 (DP-L1S). It will likely work with other printers from the LuckPrinter family that use the same SDK and `BaseNormalDevice` class; including various DP-/DP-series, LuckP-/LuckP-series, and MiniPocketPrinter models. The print width may differ (check with `print.py info`).

**Other printer classes use different enable/stop commands:**
For the Fichero D11s and other AiYin-based label printers, see [fichero-printer](https://github.com/0xMH/fichero-printer) by 0xMH, who reverse-engineered the same SDK for a different device class.

## Background

This project started as a privacy-motivated reverse-engineering exercise. The "Luck Jingle" app requires location permissions, internet access, and various other permissions that have no business being on a Bluetooth printer. The protocol was reverse-engineered by decompiling the Android APK with JADX and reading the `PrinterImageProcessor` and `BaseNormalDevice` classes from the LuckPrinter SDK.

## Licence

MIT
