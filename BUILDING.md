# How this driver and patch were made

Everything in this repository happened on a real machine: an HP LaserJet
P2055dn connected over USB to a Mac (macOS 26, CUPS), printing single-page
PDFs that intermittently came out as ~20 mostly-blank pages of PostScript
source text. This document is the full story — provenance, the bug, the
evidence trail, the fix, and how to reproduce the build.

## Driver provenance

The P2055dn has native PostScript 3 emulation, so the correct driver is HP's
own PostScript PPD — not a raster driver:

- **PPD file:** `hp-laserjet_p2055_series-ps.ppd`
  (*HP LaserJet P2055 Postscript (recommended)*)
- **Original author:** HP Development Company, L.P. — the file ships inside
  HP's own [hplip](https://github.com/hplip/hplip) driver project (MIT
  licensed), version 3.26.4
- **Acquired from:** the hplip 3.26.4 Arch Linux package (`extra`), pulled
  from an official mirror, then extracted — no source modifications to the
  PPD's device options, only the JCL patch described below
- **Why PostScript and not PCL/Gutenprint:** a PS3 printer should be driven
  in its native language — macOS renders via its built-in Apple-signed
  filters and the printer firmware does the rest, with full device options
  (duplex, trays, resolution). Gutenprint (the usual raster fallback for
  legacy HP on macOS) is deprecated upstream.

## Packaging

The distributable is a standard macOS product package built with Apple's
own tooling — no Xcode project needed:

```sh
# payload layout — the macOS driver search path
payload/Library/Printers/PPDs/Contents/Resources/HP/hp-laserjet_p2055_series-ps.ppd

pkgbuild --root payload --scripts scripts \
    --identifier com.hp.driver.laserjet-p2055dn \
    --version 1.1.0 --ownership recommended --install-location / \
    --min-os-version 11.0 comp.pkg

productbuild --package comp.pkg \
    --identifier com.hp.driver.laserjet-p2055dn --version 1.1.0 \
    HP_LaserJet_P2055dn_Driver.pkg
```

`scripts/postinstall` registers the CUPS queue (`HP_LaserJet_P2055dn`,
USB device, A4, retry-on-error) but only when a P2055-series printer is
actually connected, never touches an existing queue, and never fails the
install. Placing the PPD in `/Library/Printers/PPDs/Contents/Resources/`
makes it appear as a selectable driver in System Settings → Printers &
Scanners.

## The bug

Symptom, reported by the user: *"single-page PDF prints as ~20 almost empty
pages, some containing PostScript notes — sometimes."* The CUPS job log
showed repeated reprints of the same file and one user-cancelled job —
classic retry-until-it-works behaviour.

## Diagnosis — evidence trail

Every step below was done **without sending a single job to the printer**
until the fix was verified; print streams were captured to files.

1. **Filter-chain capture.** macOS converts PDF → PostScript with
   `cgpdftops` and post-processes with Apple's PS wrapper. Running those
   stages manually showed the produced PostScript is structurally clean:
   one page, valid `setpagedevice`, no binary.

2. **Real-chain capture.** `cupsfilter -p <queue PPD> ...` reproduces the
   exact byte stream the scheduler would send. For the failing PDF it
   produced one clean page wrapped in a PJL envelope:

   ```
   ESC%-12345X@PJL
   @PJL JOB NAME = "…" DISPLAY = "…"
   @PJL SET USERNAME = "…"
   %%@PJL ENTER LANGUAGE = PostScript     ← corrupted!
   %!PS-Adobe-3.0
   …
   ```

3. **The decisive clue.** The corrupted line `%%@PJL ENTER LANGUAGE =
   PostScript` came from the PPD's own JCL declaration
   (`*JCLToPSInterpreter: "@PJL ENTER LANGUAGE = PostScript"`). Apple's PS
   wrapper had rewritten it with a `%%` prefix — turning a valid PJL command
   into an invalid one. The user then reported the printed garbage pages
   literally started with that exact text — byte-for-byte the point where the
   printer stopped being told what to do and began echoing the stream as
   text instead of interpreting it.

4. **Root cause.** The printer is never *reliably* switched into PostScript
   mode. The PPD asks for PJL `ENTER LANGUAGE`; macOS corrupts the command;
   the printer falls back to its default (PCL/text) and echoes the raw
   PostScript source — pages of code. When its firmware auto-detection
   happened to spot the later `%!PS-Adobe-3.0` marker in time, the job
   printed correctly: hence *intermittent*.

5. **Fix search by experiment.** Three PPD variants were captured and
   compared:

   | Variant | Stream result |
   |---|---|
   | A: stock PPD (with JCL) | `%%@PJL ENTER LANGUAGE = PostScript` before `%!PS` — broken |
   | B: `JCLToPSInterpreter` emptied | the `%!PS-Adobe-3.0` line itself got `%%`-prefixed — worse |
   | C: **all JCL/Protocols lines removed** | clean `%!PS-Adobe-3.0` at **byte 0**, zero PJL bytes |

   Variant C wins: plain PostScript with the PS marker first — deterministic
   language entry, the configuration that drove PS printers over USB for
   decades before PJL envelopes existed.

6. **Scheduler-level confirmation.** A throwaway queue pointed at a local
   socket listener (printer untouched) captured the real scheduler output
   for the failing PDF with the fixed PPD: 124,975 bytes starting
   `%!PS-Adobe-3.0`, zero `@PJL`/UEL content. The fix was then applied to
   the live queue.

## The fix

Strip the JCL framing declarations from the PPD:

```sh
grep -vE '^\*(JCLBegin|JCLToPSInterpreter|JCLEnd|Protocols)' \
    hp-laserjet_p2055_series-ps.ppd > hp-laserjet_p2055_series-ps.ppd.fixed
```

Jobs now go to the printer as plain PostScript. Device options (duplex,
trays, page size) are unaffected — they travel inside the document as
`setpagedevice` calls, not via PJL.

## Reproducing the build

```sh
# 1. fixed PPD as above (or take the one from this repo's package)
# 2. stage payload + scripts
mkdir -p payload/Library/Printers/PPDs/Contents/Resources/HP scripts
cp hp-laserjet_p2055_series-ps.ppd.fixed \
   payload/Library/Printers/PPDs/Contents/Resources/HP/hp-laserjet_p2055_series-ps.ppd
# 3. build (commands under "Packaging")
```

## Attributions

- **HP Development Company, L.P.** — the PPD
  `hp-laserjet_p2055_series-ps.ppd` is HP's official driver file, published
  under the MIT license inside the [hplip](https://github.com/hplip/hplip)
  project (v3.26.4). No HP files were modified beyond the JCL patch above.
- **Arch Linux project** — the hplip package whose mirror the PPD was
  extracted from (https://archlinux.org/packages/extra/x86_64/hplip/).
- **OpenPrinting** — foomatic printer database consulted for driver
  identification (https://openprinting.github.io/foomatic/).
- **Apple / OpenPrinting CUPS** — macOS print pipeline behaviour described
  here is empirical observation of Apple's CUPS filter chain
  (`cgpdftops`, `pstoappleps`, `pstops`); the PJL-corruption finding is
  specific to macOS's wrapper and was not reported upstream at the time of
  writing.
- **Jakub Kolcar** — packaging, diagnosis, and the JCL patch for this
  repository.

The printer itself (an HP LaserJet P2055dn, serial S288HC5) behaved
correctly throughout once it was actually sent valid PostScript — no
firmware changes were made.
