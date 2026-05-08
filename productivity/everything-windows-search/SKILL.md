---
name: everything-windows-search
description: "Integrate Everything search engine (Windows) from WSL2. Uses es.exe CLI to perform instant filesystem searches across all Windows drives."
version: 1.0.0
author: agent
prerequisites:
  commands: [iconv]
---

# Everything Search from WSL2

Everything is a Windows file search engine that indexes all files instantly. This skill integrates it into WSL2 via es.exe (the CLI interface).

## Setup

### 1. Install es.exe

Download es.exe from voidtools and place it in an accessible Windows location:

```bash
curl -sL "https://www.voidtools.com/ES-1.1.0.26.zip" -o /tmp/es.zip
sudo apt install -y unzip
unzip -o /tmp/es.zip -d /tmp/es_tmp/
```

Copy to a user-writable Windows path (NOT `C:\Program Files\` — permission denied from WSL):

```bash
cp /tmp/es_tmp/es.exe "/mnt/c/Users/$USER/"
```

Verify Everything service is running (check via Windows cmd or look for the tray icon). es.exe communicates with the Everything service via named pipes.

### 2. Create WSL Wrapper

`~/.local/bin/es`:

```bash
#!/bin/bash
# Everything Search from WSL2
# Usage: es <search_term> [options]
#   -n N    : limit results (default: 30)
#   -r      : regex search
#   -w      : whole word match
#   -c      : case sensitive
WINE_EXE="/mnt/c/Users/silen/es.exe"
ARGS=()

while [[ $# -gt 0 ]]; do
  case "$1" in
    -n) ARGS+=("-n" "$2"); shift 2 ;;
    -r) ARGS+=("-regex"); shift ;;
    -w) ARGS+=("-whole-word"); shift ;;
    -c) ARGS+=("-case"); shift ;;
    *) ARGS+=("$1"); shift ;;
  esac
done

# Run es.exe and convert output from GBK to UTF-8
"$WINE_EXE" "${ARGS[@]}" 2>&1 | iconv -f GBK -t UTF-8 2>/dev/null || "$WINE_EXE" "${ARGS[@]}"
```

```bash
chmod +x ~/.local/bin/es
export PATH="$HOME/.local/bin:$PATH"
```

## Usage

```bash
# Basic search
es -n 10 keyword

# File type search
es -n 5 *.pdf
es -n 5 *.jpg

# Regex search
es -r -n 10 "neuro.*hud"

# Whole word match
es -w Tesla

# No limit (will show ALL results — use wisely!)
es important_document
```

## How It Works

es.exe communicates with the Everything service (running as a Windows service or system tray app) via a named pipe. The Everything service maintains a real-time index of NTFS volumes, so searches return results instantly — typically within milliseconds.

From WSL2, es.exe runs via the Windows binary compatibility layer (WSL can run Windows PE executables directly through `/mnt/c/` paths).

## Pitfalls

### GBK Encoding (Chinese Characters)
Windows console uses GBK encoding. Chinese filenames display as garbage without conversion. Always pipe through `iconv -f GBK -t UTF-8`.

### Path Format
Paths are returned in Windows format (`C:\Users\...`). To use them in WSL commands, convert with `wslpath`:
```bash
wslpath "$(es -n 1 somefile | head -1)"
```

### Permission Denied
Cannot write to `C:\Program Files\` from WSL without elevated privileges. Put es.exe in user-writable paths like `C:\Users\<username>\`.

### Everything Service Must Be Running
If es.exe returns no output (exit code 0), the Everything service probably isn't running. Check the system tray for the Everything icon or start it manually.

### Escaping Special Characters
Use quotes for search terms with spaces or special characters:
```bash
es "NeuroHUD project files"
```
