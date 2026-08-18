# ScanSnap iX500 on Omarchy / Arch Linux

Setup guide and batch scanning script for the discontinued Fujitsu ScanSnap iX500 on Omarchy (Arch Linux-based) systems, using SANE.

The repo contains one script, [`scansnap-scan`](scansnap-scan): load the ADF, run it, get a single deskewed PDF, optionally OCR'd and optionally uploaded to Paperless-ngx.

---

## Overview

The **Fujitsu ScanSnap iX500** is discontinued but an excellent duplex document scanner. It works on Linux through SANE's built-in `fujitsu` backend, with no proprietary drivers and no firmware files.

**What works:**

- USB scanning (WiFi is not supported on Linux)
- Duplex (double-sided) ADF scanning, whole stack in one pass
- 50-600 DPI
- Lineart, Grayscale and Color modes

**USB IDs:**

- iX500: `04c5:132b`
- iX500EE (Enterprise Edition): `04c5:13f3`

USB IDs are hardware identifiers burned into the device firmware. They never change and uniquely identify the scanner model.

---

## Install

```bash
# Core: scanning + PDF assembly + image processing
yay -S sane sane-backends img2pdf imagemagick

# Optional: GUI frontend
yay -S simple-scan

# Optional: OCR (searchable PDFs). Pick your language packs.
yay -S ocrmypdf tesseract-data-eng tesseract-data-nld

# Optional: only needed for --paperless
yay -S jq 1password-cli

# Optional: full-text search across scanned PDFs
yay -S pdfgrep
```

What each package is for:

| Package | Why |
|---|---|
| `sane`, `sane-backends` | Core libraries and the `fujitsu` backend |
| `img2pdf` | Lossless image-to-PDF assembly |
| `imagemagick` | Automatic deskew and `--trim` (provides `magick`, ImageMagick 7) |
| `ocrmypdf` | `--ocr`, adds a searchable text layer |
| `jq`, `curl` | `--paperless` metadata resolution and upload |
| `simple-scan` | GUI alternative, has its own batch support |

Run `yay -Ss tesseract-data` for the full list of 128+ OCR language packs.

Then install the script:

```bash
git clone https://github.com/peterberkenbosch/omarchy-scansnap.git
install -Dm755 omarchy-scansnap/scansnap-scan ~/.local/bin/scansnap-scan
```

Make sure `~/.local/bin` is on your `PATH`.

---

## Find your scanner

```bash
scanimage -L
```

```
device `fujitsu:ScanSnap iX500:1566072' is a FUJITSU ScanSnap iX500 scanner
```

That trailing number is your unit's serial. **The script's built-in default carries the author's serial**, so export your own:

```bash
# ~/.bashrc, ~/.zshrc, or wherever you keep shell config
export SCANSNAP_DEVICE="fujitsu:ScanSnap iX500:YOURSERIAL"
```

Other detection routes:

```bash
sane-find-scanner
lsusb | grep -i fujitsu
# Bus 001 Device 006: ID 04c5:132b Fujitsu, Ltd ScanSnap iX500
```

### Permissions

Add yourself to the `scanner` group, then log out and back in:

```bash
sudo usermod -a -G scanner $USER
```

The scanner is already in the system hardware database, which is what grants SANE access:

```bash
grep -A 2 "ScanSnap iX500" /usr/lib/udev/hwdb.d/20-sane.hwdb
```

```
# Fujitsu ScanSnap iX500
usb:v04C5p132B*
 libsane_matched=yes
```

---

## Usage

```bash
# Duplex color 300dpi, all pages in the ADF into one PDF
scansnap-scan

# Single-sided grayscale at 200dpi
scansnap-scan --simplex --gray --dpi 200

# Searchable PDF
scansnap-scan --ocr

# Receipt or small document: crop the surrounding whitespace
scansnap-scan --simplex --gray --trim

# Somewhere other than the default
scansnap-scan --output ~/Desktop
```

### Options

| Option | Effect |
|---|---|
| `--bw`, `--lineart` | Black & white |
| `--gray`, `--grayscale` | Grayscale |
| `--color` | Color (default) |
| `--dpi N` | Resolution, 50-600 (default 300) |
| `--simplex` | Single-sided (`ADF Front`) |
| `--duplex` | Double-sided (default) |
| `--ocr` | OCR with `ocrmypdf`, Dutch + English, PDF/A output |
| `--trim` | Crop whitespace around each page |
| `--output`, `-o DIR` | Output directory |
| `--paperless` | Upload to Paperless-ngx after scanning |
| `--title`, `--correspondent`, `--type`, `--tags`, `--created` | Paperless metadata hints |
| `--keep-local` | Keep the local PDF after a successful upload |
| `--help`, `-h` | Full help |

### Environment

| Variable | Meaning |
|---|---|
| `SCANSNAP_DEVICE` | Device string from `scanimage -L`. **Set this.** |
| `SCANSNAP_OUTPUT_DIR` | Default output directory (default `~/Documents/Scans`) |
| `PAPERLESS_URL` | Paperless-ngx instance URL, required for `--paperless` |
| `PAPERLESS_TOKEN` | Paperless API token |
| `PAPERLESS_OP_ITEM` | 1Password secret reference to read the token from |

---

## Automatic deskew

Every page is deskewed automatically. There is no flag to enable it and no flag to skip it.

The script does **not** use the fujitsu driver's `--swdeskew`. That option mislocks on security-print patterns (the tint blocks printed on bank and government letters) and has been observed rotating an upright page by roughly 30 degrees.

Instead the script runs its own projection-profile deskew:

1. Downscale the page, crop to the paper interior (70%), threshold to bilevel. Cropping first keeps scanner background and paper edges out of the measurement.
2. Score candidate angles from -3° to +3° in 0.25° steps. The score is the standard deviation of the row-brightness profile: text rows line up into sharp bands when the page is straight, so the profile's variance peaks at the correct angle.
3. Refine around the winner in 0.05° steps.
4. If the best angle sits at the search boundary, detection is treated as failed and the page is left alone.
5. Corrections under 0.05° are skipped so already-straight pages are never resampled.

The ±3° clamp is the safety property: a correction can never exceed 3°, so a misdetection costs you a barely-tilted page rather than a ruined one.

---

## Paperless-ngx upload

With `--paperless`, the finished PDF is posted to a [Paperless-ngx](https://docs.paperless-ngx.com/) instance and the local copy is deleted (pass `--keep-local` to keep it).

```bash
export PAPERLESS_URL="https://paperless.example.com"
export PAPERLESS_TOKEN="your-api-token"

scansnap-scan --ocr --paperless \
  --correspondent "Belastingdienst" \
  --type "Beschikking" \
  --tags "business,fy-2026" \
  --title "Voorlopige aanslag 2026"
```

Instead of putting the token in your shell config, store it in 1Password and let the script read it:

```bash
export PAPERLESS_OP_ITEM="op://Private/Paperless API/credential"
```

If `PAPERLESS_TOKEN` is unset, the script calls `op read "$PAPERLESS_OP_ITEM"`.

Correspondents, document types and tags are passed by name. The script resolves each name to its Paperless ID and **creates it if it doesn't exist yet** (with `matching_algorithm: 6`, "auto"). Paperless then runs its own OCR, classification and workflow rules on the uploaded file, so `--ocr` is optional here; it is useful if you also want a searchable copy outside Paperless.

The upload prints the task ID and a dashboard URL to follow processing.

---

## Getting your token

In Paperless-ngx: profile menu → **My Profile** → **API Auth Token**. Copy it into `PAPERLESS_TOKEN` or your 1Password item.

---

## Searching scanned PDFs

Once `--ocr` has run, the PDFs carry a text layer:

```bash
pdfgrep "Belastingdienst" ~/Documents/Scans/*.pdf
pdfgrep -C 3 "factuur" ~/Documents/Scans/*.pdf     # with context
pdfgrep -r -i "belasting" ~/Documents/Scans/       # recursive, case-insensitive
pdfgrep -l "2026" ~/Documents/Scans/*.pdf          # filenames only
```

---

## Manual scanning

The script is a convenience wrapper. The underlying commands, if you want them:

```bash
# Inspect every option the backend exposes
scanimage --device "$SCANSNAP_DEVICE" -A

# Single page
scanimage --device "$SCANSNAP_DEVICE" \
  --source "ADF Duplex" --mode Color --resolution 300 \
  --format=png --output-file page.png --progress

# Whole stack in one session, then assemble
scanimage --device "$SCANSNAP_DEVICE" \
  --source "ADF Duplex" --mode Color --resolution 300 \
  --format=png --batch="page_%03d.png" --batch-print --progress
img2pdf page_*.png -o document.pdf && rm page_*.png
```

Useful backend options: `--page-width` / `--page-height`, `--brightness -127..127`, `--contrast -127..127`.

---

## Desktop entry

`~/.local/share/applications/scansnap-ix500.desktop`:

```ini
[Desktop Entry]
Name=ScanSnap iX500
Comment=Scan documents with Fujitsu ScanSnap iX500
Exec=simple-scan
Icon=scanner
Type=Application
Categories=Office;Scanning;
Keywords=scan;scanner;document;scansnap;
```

```bash
update-desktop-database ~/.local/share/applications/
```

---

## Troubleshooting

### Only the front side of each duplex sheet comes out

You are running an old version of this script. Versions before August 2026 scanned with `--output-file` in a per-page loop, and `scanimage` performs exactly **one acquisition per invocation** — for a duplex sheet that is the front side only, and the back was discarded when the process exited. The fix is `--batch`, which holds a single SANE session open until the feeder empties. Pull the latest `scansnap-scan`.

### "Document feeder out of documents"

No paper in the ADF. This is also the normal, expected message at the end of a run.

### A page came out rotated by a wild amount

Not this script — it clamps corrections to ±3°. If you added `--swdeskew` to the `scanimage` call yourself, remove it. See [Automatic deskew](#automatic-deskew).

### "Invalid argument"

Wrong device string. Take the exact value from `scanimage -L` and set `SCANSNAP_DEVICE`.

### Permission denied

```bash
sudo usermod -a -G scanner $USER
# log out and back in
```

### Scanner not detected

1. Check the USB cable and that the scanner's LED is on.
2. `lsusb | grep -i fujitsu`
3. `sudo udevadm control --reload-rules && sudo udevadm trigger`
4. Unplug and replug.

### `convert: command not found`

ImageMagick 7 dropped the `convert` name. This script uses `magick`. Install `imagemagick`.

### Slow scanning on USB 3.0

Older scanners can struggle on USB 3.0 ports:

```bash
export SANE_USB_WORKAROUND=1
```

Or use a USB 2.0 port.

---

## Choosing settings

| Use case | Resolution |
|---|---|
| Email / archive | 150-200 DPI |
| OCR | 300 DPI |
| High quality | 400-600 DPI |

- **Lineart:** text documents and forms, smallest files
- **Grayscale:** documents with photos, and a good default for pages with signatures
- **Color:** full-color documents and photographs, largest files

---

## Filing scans without Paperless

If you don't run Paperless-ngx, a `pdfgrep` rule file gets you most of the way. Put the rules in `~/.config/scansnap/rules.conf`:

```ini
# PATTERN -> DESTINATION
Belastingdienst          -> ~/Documents/Filed/Tax
"ING Bank"               -> ~/Documents/Filed/Banking/ING
Factuur                  -> ~/Documents/Filed/Invoices
Invoice                  -> ~/Documents/Filed/Invoices
Contract                 -> ~/Documents/Filed/Contracts
```

And a companion script, `~/.local/bin/scansnap-organize`:

```bash
#!/bin/bash
# Move OCR'd PDFs into folders based on their content.
set -uo pipefail

RULES_FILE="${HOME}/.config/scansnap/rules.conf"
SCAN_DIR="${SCANSNAP_OUTPUT_DIR:-${HOME}/Documents/Scans}"

command -v pdfgrep >/dev/null || { echo "Install pdfgrep: yay -S pdfgrep"; exit 1; }

for pdf in "$SCAN_DIR"/*.pdf; do
  [ -f "$pdf" ] || continue
  matched=false

  while IFS= read -r line; do
    [[ "$line" =~ ^[[:space:]]*# ]] && continue
    [[ -z "${line// }" ]] && continue
    [[ "$line" != *"->"* ]] && continue

    pattern=$(echo "${line%%->*}" | xargs)
    destination=$(echo "${line##*->}" | xargs)
    destination="${destination/#\~/$HOME}"

    if pdfgrep -q -i "$pattern" "$pdf" 2>/dev/null; then
      mkdir -p "$destination"
      mv "$pdf" "$destination/"
      echo "$(basename "$pdf"): matched '$pattern' -> $destination"
      matched=true
      break
    fi
  done < "$RULES_FILE"

  [ "$matched" = false ] && echo "$(basename "$pdf"): no match, left in $SCAN_DIR"
done
```

```bash
chmod +x ~/.local/bin/scansnap-organize
scansnap-scan --ocr && scansnap-organize
```

This needs `--ocr`, since matching reads the PDF's text layer. Paperless-ngx does the same job with proper classification and is the better answer if you're willing to run a server.

---

## Reference

| Path | Purpose |
|---|---|
| `/usr/lib/udev/hwdb.d/20-sane.hwdb` | Hardware database, contains the iX500 entry |
| `/usr/lib/udev/rules.d/65-sane.rules` | udev rules for scanners |
| `/etc/sane.d/fujitsu.conf` | Fujitsu backend configuration |
| `/etc/sane.d/dll.conf` | Enabled SANE backends |
| `~/Documents/Scans/` | Default output directory |
| `~/.local/bin/scansnap-scan` | The script |

Further reading:

- [SANE project](http://www.sane-project.org/) and its [supported devices list](http://www.sane-project.org/sane-supported-devices.html)
- [Arch Wiki: SANE](https://wiki.archlinux.org/title/SANE)
- [Paperless-ngx documentation](https://docs.paperless-ngx.com/)

---

*Verified against Omarchy 4.0.0, ImageMagick 7.1.2, August 2026.*
*Scanner: Fujitsu ScanSnap iX500 / iX500EE (USB IDs `04c5:132b` / `04c5:13f3`).*
