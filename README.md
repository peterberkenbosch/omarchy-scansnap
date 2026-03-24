# Omarchy ScanSnap

Linux scanning tools for Fujitsu ScanSnap document scanners on Omarchy (Arch Linux-based) systems.

## Supported Scanners

- **Fujitsu ScanSnap iX500** (USB ID: `04c5:132b`)
- Possibly other ScanSnap models with SANE fujitsu backend

## Features

- Batch scanning from ADF (Automatic Document Feeder)
- Duplex (double-sided) scanning support
- 50-600 DPI resolution
- Color, Grayscale, and Lineart modes
- Single command multi-page PDF output
- No proprietary drivers required

## Quick Start

```bash
# 1. Install dependencies
sudo pacman -S sane sane-backends simple-scan img2pdf

# 2. Download the script
curl -o ~/.local/bin/scansnap-scan https://raw.githubusercontent.com/peterberkenbosch/omarchy-scansnap/main/scansnap-scan
chmod +x ~/.local/bin/scansnap-scan

# 3. Scan!
scansnap-scan
```

## Documentation

See [README.md](README.md) for complete setup guide and troubleshooting.

## License

MIT License - see [LICENSE](LICENSE) file.
