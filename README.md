# ScanSnap iX500 Setup Guide for Omarchy/Arch Linux

Complete guide for configuring the discontinued Fujitsu ScanSnap iX500 document scanner on Omarchy (Arch Linux-based) systems using SANE.

---

## Overview

The **Fujitsu ScanSnap iX500** is a discontinued but excellent duplex document scanner. Despite being discontinued by Fujitsu, it works perfectly on Linux via the SANE (Scanner Access Now Easy) framework using the built-in `fujitsu` backend - no proprietary drivers required.

**Supported features:**
- USB scanning (WiFi not supported on Linux)
- Duplex (double-sided) scanning
- ADF (Automatic Document Feeder) - **one page at a time**
- 50-600 DPI resolution
- Lineart, Grayscale, and Color modes
- No firmware files required (built into scanner)

**USB IDs:**
- **iX500:** `04c5:132b`
- **iX500EE (Enterprise Edition):** `04c5:13f3`

USB IDs are hardware identifiers burned into the device firmware. They never change and uniquely identify your scanner model.

**⚠️ Important Note:** The SANE backend scans **one page per command**. To scan multi-page documents, use the batch scanning script provided in this guide which loops until all pages are processed.

---

## Prerequisites

- Omarchy or Arch Linux system
- USB connection to the scanner
- Sudo privileges for package installation

---

## Step 1: Install SANE Packages

Install the core SANE scanning framework and dependencies:

```bash
# Using pacman
sudo pacman -S sane sane-backends simple-scan img2pdf imagemagick

# Or using yay
yay -S sane sane-backends simple-scan img2pdf imagemagick
```

**Packages explained:**
- `sane` - Core SANE libraries
- `sane-backends` - Scanner drivers including the `fujitsu` backend
- `simple-scan` - GNOME-based GUI scanning application
- `img2pdf` - Convert images to PDF (better quality than ImageMagick)
- `imagemagick` - Image processing (PDF fallback + deskew/straighten feature)

### Optional: OCR Support

To make scanned PDFs searchable, install `ocrmypdf` from the AUR:

```bash
# Using yay (recommended)
yay -S ocrmypdf

# During installation, you'll be prompted to select OCR language data
# For Dutch documents: select 86 (tesseract-data-nld)
# For English documents: select 30 (tesseract-data-eng)
```

**Installing additional OCR languages later:**

```bash
# Install English language support
sudo pacman -S tesseract-data-eng

# Install Dutch language support
sudo pacman -S tesseract-data-nld

# Install multiple languages
sudo pacman -S tesseract-data-eng tesseract-data-nld tesseract-data-deu
```

**Available languages:** Run `pacman -Ss tesseract-data` to see all 128+ available language packs.

### Searchable PDFs: Finding Your Documents

Once OCR is enabled, your PDFs contain searchable text. Use `pdfgrep` to search through them:

```bash
# Install pdfgrep
sudo pacman -S pdfgrep

# Search for text in all scanned PDFs
pdfgrep "Belastingdienst" ~/Documents/Scans/*.pdf

# Search with context (show 3 lines before/after)
pdfgrep -C 3 "factuur" ~/Documents/Scans/*.pdf

# Search recursively in subdirectories
pdfgrep -r "Peter Berkenbosch" ~/Documents/Scans/

# Case-insensitive search
pdfgrep -i "belasting" ~/Documents/Scans/*.pdf

# List only matching filenames
pdfgrep -l "2025" ~/Documents/Scans/*.pdf
```

---

## Step 2: Verify Scanner Detection

Check if the scanner is detected by SANE:

```bash
scanimage -L
```

**Expected output:**
```
device `v4l:/dev/video0' is a Noname Webcam virtual device
device `fujitsu:ScanSnap iX500:1566072' is a FUJITSU ScanSnap iX500 scanner
```

The `fujitsu:ScanSnap iX500:XXXXXXX` entry confirms your scanner is detected.

### Alternative detection methods:

```bash
# List all SANE devices with details
sane-find-scanner

# Check USB device
lsusb | grep -i fujitsu
# Output: Bus 001 Device 006: ID 04c5:132b Fujitsu, Ltd ScanSnap iX500
```

---

## Step 3: Configure User Permissions (Optional but Recommended)

Add your user to the `scanner` group for better USB device access:

```bash
sudo usermod -a -G scanner $USER
```

**Then log out and log back in** for the group change to take effect.

### How it works

The system uses udev rules to recognize the scanner. Check the existing configuration:

```bash
# Verify iX500 is in the hardware database
grep -A 2 "ScanSnap iX500" /usr/lib/udev/hwdb.d/20-sane.hwdb
```

**Expected output:**
```
# Fujitsu ScanSnap iX500
usb:v04C5p132B*
 libsane_matched=yes
```

This shows the scanner is pre-configured in the system. The `libsane_matched=yes` tag allows SANE to access the device.

---

## Step 4: Test Scanning

### Command-line test

Load a document into the ADF and run:

```bash
scanimage --device "fujitsu:ScanSnap iX500:1566072" \
  --format=png \
  --output-file ~/test-scan.png \
  --progress
```

**Expected output:**
```
Progress: 100%
Scanning completed.
```

The file `~/test-scan.png` will contain your scanned document.

### Common first-run message

If you see:
```
scanimage: sane_start: Document feeder out of documents
```

This means the scanner was detected but **no paper was loaded**. Load paper into the ADF and retry.

---

## Step 5: Install GUI Frontend (Optional)

For graphical scanning, install `simple-scan` (already included in Step 1):

```bash
# Run the GUI scanner
simple-scan
```

The iX500 should appear automatically in the device list.

---

## Step 6: Advanced Configuration

### View all scanner options

```bash
scanimage --device "fujitsu:ScanSnap iX500:1566072" -A
```

**Key options for iX500:**
- `--source ADF Front` - Single-sided scanning
- `--source ADF Duplex` - Double-sided scanning (default)
- `--mode Lineart|Gray|Color` - Color mode
- `--resolution 50..600` - DPI (dots per inch)
- `--page-width` / `--page-height` - Document size
- `--brightness -127..127` - Brightness adjustment
- `--contrast -127..127` - Contrast adjustment

### Example commands

```bash
# Duplex color scan at 300 DPI to PNG
scanimage --device "fujitsu:ScanSnap iX500:1566072" \
  --source "ADF Duplex" \
  --mode Color \
  --resolution 300 \
  --format=png \
  --output-file document.png

# Single-sided B&W at 200 DPI
scanimage --device "fujitsu:ScanSnap iX500:1566072" \
  --source "ADF Front" \
  --mode Lineart \
  --resolution 200 \
  --format=png \
  --output-file bw-document.png

# Scan to PDF (requires conversion)
scanimage --device "fujitsu:ScanSnap iX500:1566072" \
  --format=pnm \
  --output-file scan.pnm && \
  img2pdf scan.pnm -o scan.pdf && \
  rm scan.pnm
```

---

## Step 7: Create Batch Scanning Script

**⚠️ Important:** The SANE fujitsu backend scans **one page at a time**. Unlike the proprietary Windows/Mac software, each scan command processes only a single page from the ADF. To scan multiple pages, we need a script that loops until all pages are processed.

Save this as `~/.local/bin/scansnap-scan`:

```bash
#!/bin/bash
# ScanSnap iX500 Batch Scanning Script
# Scans all pages from ADF into a single PDF

DEVICE="fujitsu:ScanSnap iX500:1566072"
OUTPUT_DIR="${HOME}/Documents/Scans"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Create output directory
mkdir -p "$OUTPUT_DIR"

# Default settings
MODE="Color"
RESOLUTION="300"
SOURCE="ADF Duplex"

# Parse arguments
while [[ $# -gt 0 ]]; do
  case $1 in
    --bw|--lineart)
      MODE="Lineart"
      shift
      ;;
    --gray|--grayscale)
      MODE="Gray"
      shift
      ;;
    --color)
      MODE="Color"
      shift
      ;;
    --dpi)
      RESOLUTION="$2"
      shift 2
      ;;
    --simplex)
      SOURCE="ADF Front"
      shift
      ;;
    --duplex)
      SOURCE="ADF Duplex"
      shift
      ;;
    --output|-o)
      OUTPUT_DIR="$2"
      shift 2
      ;;
    --help|-h)
      echo "Usage: $0 [OPTIONS]"
      echo ""
      echo "ScanSnap iX500 Batch Scanner for Omarchy/Linux"
      echo ""
echo "Options:"
echo "  --bw, --lineart      Black & white scanning"
echo "  --gray, --grayscale  Grayscale scanning"
echo "  --color              Color scanning (default)"
echo "  --dpi N              Set resolution (50-600, default: 300)"
echo "  --simplex            Single-sided scanning"
echo "  --duplex             Double-sided scanning (default)"
echo "  --straighten         Auto-straighten/deskew scanned pages (requires imagemagick)"
echo "  --output, -o DIR     Output directory (default: ~/Documents/Scans)"
      echo ""
      echo "Examples:"
      echo "  $0                          # Scan all pages from ADF to PDF"
      echo "  $0 --bw --dpi 200           # Scan B&W at 200dpi"
      echo "  $0 --simplex --color        # Single-sided color scanning"
      echo ""
      echo "Note: The iX500 scans one page at a time. This script loops"
      echo "      until all pages are scanned and combines them into one PDF."
      exit 0
      ;;
    *)
      echo "Unknown option: $1"
      echo "Use --help for usage information"
      exit 1
      ;;
  esac
done

# Generate base filename
BASE_FILENAME="scan_${TIMESTAMP}"
TEMP_DIR="${OUTPUT_DIR}/.temp_${TIMESTAMP}"
mkdir -p "$TEMP_DIR"

echo "========================================"
echo "ScanSnap iX500 Batch Scanner"
echo "========================================"
echo ""
echo "Settings:"
echo "  Mode: $MODE"
echo "  Resolution: ${RESOLUTION}dpi"
echo "  Source: $SOURCE"
echo "  Output: ${OUTPUT_DIR}/${BASE_FILENAME}.pdf"
echo ""
echo "Load all pages into the ADF and press Enter..."
read -r

echo ""
echo "Scanning... Press Ctrl+C to stop."
echo ""

PAGE=1
SUCCESS=true
SCANNED_FILES=()

while [ "$SUCCESS" = true ]; do
  printf "Scanning page %d... " "$PAGE"
  
  PAGE_FILE="${TEMP_DIR}/page_$(printf "%03d" $PAGE).png"
  
  # Try to scan one page
  OUTPUT=$(scanimage --device "$DEVICE" \
    --source "$SOURCE" \
    --mode "$MODE" \
    --resolution "$RESOLUTION" \
    --format=png \
    --output-file "$PAGE_FILE" \
    --progress 2>&1)
  
  SCAN_RESULT=$?
  
  # Check if scan succeeded
  if [ $SCAN_RESULT -eq 0 ] && [ -f "$PAGE_FILE" ] && [ -s "$PAGE_FILE" ]; then
    echo "✓"
    SCANNED_FILES+=("$PAGE_FILE")
    PAGE=$((PAGE + 1))
    
    # Small delay between pages
    sleep 0.5
  else
    # Check if it's just "out of documents" (normal end)
    if echo "$OUTPUT" | grep -q "Document feeder out of documents"; then
      echo "✓ (done - no more pages)"
      SUCCESS=false
    else
      echo "✗ (error)"
      echo "Error: $OUTPUT"
      SUCCESS=false
    fi
  fi
done

echo ""

# Combine into PDF if we have scanned files
if [ ${#SCANNED_FILES[@]} -gt 0 ]; then
  FINAL_PDF="${OUTPUT_DIR}/${BASE_FILENAME}.pdf"
  
  echo "Combining ${#SCANNED_FILES[@]} pages into PDF..."
  
  # Use img2pdf if available (better quality)
  if command -v img2pdf &> /dev/null; then
    img2pdf "${SCANNED_FILES[@]}" -o "$FINAL_PDF" 2>/dev/null
  else
    # Fallback to ImageMagick
    convert "${SCANNED_FILES[@]}" "$FINAL_PDF" 2>/dev/null
  fi
  
  if [ -f "$FINAL_PDF" ]; then
    echo ""
    echo "========================================"
    echo "Scan complete!"
    echo "========================================"
    echo ""
    echo "Saved: ${FINAL_PDF}"
    echo "Pages: ${#SCANNED_FILES[@]}"
    echo ""
  else
    echo "Error: Failed to create PDF"
    echo "Individual page files are in: $TEMP_DIR"
    exit 1
  fi
  
  # Clean up temp files
  rm -rf "$TEMP_DIR"
else
  echo "No pages were scanned."
  rmdir "$TEMP_DIR" 2>/dev/null
  exit 1
fi
```

Make it executable:
```bash
chmod +x ~/.local/bin/scansnap-scan
```

### Usage examples

```bash
# Scan all pages from ADF to single PDF (default)
scansnap-scan

# Scan all pages B&W at 200dpi
scansnap-scan --bw --dpi 200

# Single-sided color scanning
scansnap-scan --simplex --color

# Custom output directory
scansnap-scan --output ~/Desktop

# Straighten tilted scans (requires imagemagick)
scansnap-scan --straighten
```

### How it works

1. **Load all pages** into the ADF
2. Script scans **one page at a time** in a loop
3. Each page is saved as a temporary PNG file
4. When ADF is empty ("Document feeder out of documents"), loop stops
5. All pages are combined into a single PDF using `img2pdf`
6. Temporary files are cleaned up

**Note:** This is different from the Windows/Mac Fujitsu software which can scan continuously. The Linux SANE backend requires this loop approach for multi-page documents.

---

## Step 8: Desktop Integration

Create a desktop entry for the app launcher:

**File:** `~/.local/share/applications/scansnap-ix500.desktop`

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

Refresh desktop database:
```bash
update-desktop-database ~/.local/share/applications/
```

---

## Troubleshooting

### "Document feeder out of documents"

**Cause:** No paper loaded in ADF  
**Fix:** Load paper into the document feeder and retry

### Only one page scanned from ADF (Multi-page document)

**Cause:** The SANE fujitsu backend scans **one page per command**. Unlike proprietary Windows/Mac software, it doesn't automatically scan all pages in the ADF with a single command.

**Fix:** Use the batch scanning script provided in this guide:

```bash
scansnap-scan
```

This script loops and scans each page individually until the ADF is empty, then combines all pages into a single PDF.

**Alternative:** Use `simple-scan` GUI which has built-in batch scanning support.

### "Invalid argument" error

**Cause:** Scanner needs specific device name  
**Fix:** Use the exact device name from `scanimage -L`:
```bash
scanimage --device "fujitsu:ScanSnap iX500:1566072" ...
```

### Permission denied errors

**Cause:** User not in scanner group  
**Fix:**
```bash
sudo usermod -a -G scanner $USER
# Then log out and back in
```

### Scanner not detected

**Checklist:**
1. Verify USB cable connection
2. Check power (scanner should have LED on)
3. Verify USB device: `lsusb | grep fujitsu`
4. Reload udev rules: `sudo udevadm control --reload-rules && sudo udevadm trigger`
5. Unplug and replug the scanner

### Slow scanning on USB 3.0 ports

Older scanners may have issues with USB 3.0. Try:

```bash
# Set environment variable before scanning
export SANE_USB_WORKAROUND=1
scanimage ...
```

Or use a USB 2.0 port if available.

---

## Scanning Tips

### Resolution Guidelines

| Use Case | Resolution | Notes |
|----------|-----------|-------|
| Email/Archive | 150-200 DPI | Small file size |
| OCR | 300 DPI | Optimal for text recognition |
| High quality | 400-600 DPI | Large files, best quality |

### Color Mode Selection

- **Lineart (B&W):** Text documents, forms, smallest file size
- **Grayscale:** Documents with photos, intermediate file size
- **Color:** Full-color documents, photographs, largest file size

### Batch Scanning with ADF

**Important:** The SANE fujitsu backend processes **one page per scan command**. To scan multiple pages:

**Option 1: Use the batch script (Recommended)**
```bash
scansnap-scan
```
This automatically loops through all pages and combines them into one PDF.

**Option 2: Use simple-scan GUI**
```bash
simple-scan
```
The GUI handles batch scanning automatically when you load multiple pages.

**Option 3: Manual loop with img2pdf**
```bash
# Install img2pdf
sudo pacman -S img2pdf

# Create a script to scan all pages
for i in {1..10}; do
  scanimage --device "fujitsu:ScanSnap iX500:1566072" \
    --format=png --output-file page_$i.png 2>&1 || break
done

# Combine into PDF
img2pdf page_*.png -o document.pdf
rm page_*.png
```

---

## Files and Locations Reference

| File/Directory | Purpose |
|----------------|---------|
| `/usr/lib/udev/hwdb.d/20-sane.hwdb` | Hardware database with iX500 entry |
| `/usr/lib/udev/rules.d/65-sane.rules` | udev rules for scanners |
| `/etc/sane.d/fujitsu.conf` | Fujitsu backend configuration |
| `/etc/sane.d/dll.conf` | Enabled SANE backends |
| `~/Documents/Scans/` | Default scan output directory |
| `~/.local/bin/scansnap-scan` | Custom batch scanning script |

---

## Document Organization Automation

You can automate sorting scanned documents based on sender, receiver, or content using a simple configuration file approach.

### Example: scansnap-organize

Create `~/.config/scansnap/rules.conf`:

```ini
# Document organization rules
# Format: MATCH_PATTERN -> DESTINATION_FOLDER

# Dutch Tax Authority
Belastingdienst -> ~/Dropbox/PBCBV/Belastingdienst

# Example company patterns
"PHBX Holding" -> ~/Dropbox/PBCBV/PHBX
"Peter Berkenbosch Consultancy" -> ~/Dropbox/PBCBV/Correspondence
"ING Bank" -> ~/Dropbox/PBCBV/Banking/ING

# Match by document type
"Factuur" -> ~/Dropbox/PBCBV/Invoices
"Invoice" -> ~/Dropbox/PBCBV/Invoices
"Contract" -> ~/Dropbox/PBCBV/Contracts

# Date-based patterns (YYYY format)
"202[0-9]" -> ~/Dropbox/PBCBV/ByYear
```

Then create the automation script `~/.local/bin/scansnap-organize`:

```bash
#!/bin/bash
# Auto-organize scanned PDFs based on content

RULES_FILE="${HOME}/.config/scansnap/rules.conf"
SCAN_DIR="${HOME}/Documents/Scans"

# Check if PDF has OCR
if ! command -v pdfgrep &> /dev/null; then
    echo "Install pdfgrep: sudo pacman -S pdfgrep"
    exit 1
fi

# Process each PDF
for pdf in "$SCAN_DIR"/*.pdf; do
    [ -f "$pdf" ] || continue
    
    matched=false
    
    # Read rules and match
    while IFS='->' read -r pattern destination; do
        # Skip comments and empty lines
        [[ "$pattern" =~ ^#.*$ ]] && continue
        [[ -z "$pattern" ]] && continue
        
        # Trim whitespace
        pattern=$(echo "$pattern" | xargs)
        destination=$(echo "$destination" | xargs)
        destination="${destination/#\~/$HOME}"
        
        # Search in PDF
        if pdfgrep -q "$pattern" "$pdf" 2>/dev/null; then
            echo "Match: '$pattern' found in $(basename "$pdf")"
            
            # Create destination folder
            mkdir -p "$destination"
            
            # Move file
            mv "$pdf" "$destination/"
            echo "  → Moved to: $destination"
            matched=true
            break
        fi
    done < "$RULES_FILE"
    
    if [ "$matched" = false ]; then
        echo "No match for: $(basename "$pdf")"
        echo "  → Kept in: $SCAN_DIR"
    fi
done
```

Make it executable:
```bash
chmod +x ~/.local/bin/scansnap-organize
```

### Usage workflow

```bash
# 1. Scan with OCR
scansnap-scan --signatures --ocr

# 2. Auto-organize the scanned PDFs
scansnap-organize

# Or combine both:
scansnap-scan --signatures --ocr && scansnap-organize
```

### Advanced: Hook into scansnap-scan

Add this to the end of `scansnap-scan` (after line ~230):

```bash
# Auto-organize if --auto-organize flag is set
if [ "$AUTO_ORGANIZE" = true ]; then
    if [ -f "${HOME}/.local/bin/scansnap-organize" ]; then
        echo "Auto-organizing..."
        scansnap-organize
    fi
fi
```

Then scan and auto-organize in one command:
```bash
scansnap-scan --signatures --ocr --auto-organize
```

---

## Additional Resources

- **SANE Project:** http://www.sane-project.org/
- **SANE Supported Devices:** http://www.sane-project.org/sane-supported-devices.html
- **Arch Wiki - SANE:** https://wiki.archlinux.org/title/SANE
- **img2pdf documentation:** `man img2pdf`

---

## Summary

The Fujitsu ScanSnap iX500 works out-of-the-box on Omarchy/Arch Linux with SANE:

1. ✅ Install `sane sane-backends simple-scan`
2. ✅ Scanner detected automatically (USB IDs: `04c5:132b` or `04c5:13f3`)
3. ✅ No firmware files needed
4. ✅ Full duplex ADF support
5. ✅ 50-600 DPI resolution range
6. ✅ Color, Grayscale, and Lineart modes

**Quick start command:**
```bash
scansnap-scan --color --dpi 300
```

---

*Last updated: 2026-03-24*
*System: Omarchy (Arch Linux-based)*
*Scanner: Fujitsu ScanSnap iX500 / iX500EE (USB IDs: 04c5:132b / 04c5:13f3)*
