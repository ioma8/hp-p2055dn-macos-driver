# HP LaserJet P2055dn — macOS Driver

Installable macOS package providing HP's official PostScript driver for the
**HP LaserJet P2055dn** over USB, including automatic print-queue registration.

**macOS 11.0+** (Intel & Apple Silicon). No third-party raster drivers required.

## Why this driver

The P2055dn speaks native **PostScript 3**, so the correct driver is HP's own
PostScript PPD (*HP LaserJet P2055 Postscript (recommended)*): full device
options (auto duplex, all trays, 600/1200 dpi, econo mode). macOS renders with
its built-in filters and the printer's firmware does the rest. The deprecated
Gutenprint raster driver is **not** needed.

## Install

Double-click `HP_LaserJet_P2055dn_Driver.pkg`, or from a terminal:

```sh
sudo installer -pkg HP_LaserJet_P2055dn_Driver.pkg -target /
```

During install the package:

1. Installs the PPD to `/Library/Printers/PPDs/Contents/Resources/HP/`
   (the standard macOS driver location — it appears as *HP LaserJet P2055
   Postscript (recommended)* in System Settings → Printers & Scanners).
2. If the printer is connected over USB, creates and enables the queue
   `HP_LaserJet_P2055dn` (A4, retry-on-error) and sets it as system default
   only when no default exists yet.
3. Leaves an already-existing `HP_LaserJet_P2055dn` queue untouched.

Printer not connected at install time? Re-run the installer, or add the
printer manually in System Settings once it is plugged in.

## Verify

```sh
lpinfo -m | grep -i p2055        # driver visible
lpstat -p HP_LaserJet_P2055dn     # idle / enabled
lp -d HP_LaserJet_P2055dn file.pdf
```

## Uninstall

```sh
sudo rm /Library/Printers/PPDs/Contents/Resources/HP/hp-laserjet_p2055_series-ps.ppd
sudo lpadmin -x HP_LaserJet_P2055dn
```

## Provenance & license

The PPD (`hp-laserjet_p2055_series-ps.ppd`) is HP's official driver file,
extracted from the MIT-licensed [hplip](https://github.com/hplip/hplip) 3.26.4
project. See `LICENSE`.
