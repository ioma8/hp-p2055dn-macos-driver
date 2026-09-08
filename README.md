# HP LaserJet P2055dn — macOS Driver

Installable macOS package providing HP's official PostScript driver for the
HP LaserJet P2055dn over USB, with automatic print-queue registration.

The P2055dn speaks native **PostScript 3**, so no raster drivers are needed:
macOS renders with its built-in filters and the printer firmware does the
rest. The bundled PPD (*HP LaserJet P2055 Postscript (recommended)*) exposes
the full feature set — auto duplex, all trays, 600/1200 dpi, econo mode.

**Requirements:** macOS 11.0+ (Intel & Apple Silicon).

## What was fixed (v1.1.0)

Upstream HP PPDs declare PJL job framing (`JCLBegin` / `JCLToPSInterpreter` /
`JCLEnd`). macOS's print pipeline corrupts it, rewriting `@PJL ENTER LANGUAGE
= PostScript` into the invalid `%%@PJL ENTER LANGUAGE = PostScript`. The
printer is then not reliably switched into PostScript mode and intermittently
echoes the raw PostScript stream as text — a single-page PDF prints as many
pages of PostScript source. This package removes those JCL declarations, so
jobs ship as plain PostScript starting with `%!PS-Adobe-3.0`: deterministic,
no garbage pages.

## Install

Double-click `HP_LaserJet_P2055dn_Driver.pkg`, or run:

```sh
sudo installer -pkg HP_LaserJet_P2055dn_Driver.pkg -target /
```

The package:

1. Installs the PPD to `/Library/Printers/PPDs/Contents/Resources/HP/` — it
   then appears as *HP LaserJet P2055 Postscript (recommended)* in System
   Settings → Printers & Scanners.
2. If a P2055-series printer is connected over USB, creates and enables the
   queue `HP_LaserJet_P2055dn` (A4, retry-on-error), and makes it the system
   default only if none exists.
3. Never modifies an existing `HP_LaserJet_P2055dn` queue.

Printer not connected at install time? Re-run the installer once it is, or
add the printer manually in System Settings.

The package is unsigned; if macOS blocks the download, right-click → Open, or
run `xattr -d com.apple.quarantine HP_LaserJet_P2055dn_Driver.pkg`.

## Verify

```sh
lpinfo -m | grep -i p2055      # driver present
lpstat -p HP_LaserJet_P2055dn  # queue idle / enabled
lp -d HP_LaserJet_P2055dn file.pdf
```

## Uninstall

```sh
sudo rm /Library/Printers/PPDs/Contents/Resources/HP/hp-laserjet_p2055_series-ps.ppd
sudo lpadmin -x HP_LaserJet_P2055dn
```

## License

The PPD is HP's official driver file, taken from the MIT-licensed
[hplip](https://github.com/hplip/hplip) project (v3.26.4). See `LICENSE`.
