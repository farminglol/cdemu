# The Sims: missing pregap data, not a Wine or drive-timing failure

On 2026-10-11, The Sims Complete Collection (SafeDisc 4.50.000) was tested with
Wine 11.19 Staging, an original Disc 1 in an Optiarc AD-5280S, CDEmu daemon
3.3.1 and libMirage 3.3.3.
The installed edition and disc label were `THESIMS_COMPLETE`; this result is
specific to that disc, not a claim about every Sims Deluxe release.

## Result

The game reaches the neighbourhood screen using the installed, unmodified
CDEmu daemon and a pregap-preserving NRG image, while the physical tray is open.
The game's executable and protection driver were not patched. The earlier
experimental timing, identity and SCSI-response overrides are unnecessary for
this tested configuration.

| Test | Result |
| --- | --- |
| Original CD in physical drive | Reached neighbourhood |
| Existing raw BIN/CUE with the fork's timing/response patches | Crashed during SafeDisc loading |
| Same BIN/CUE and patches, with only captured READ CD MSF pregap responses added | Reached neighbourhood; physical tray open |
| Pregap-preserving DAO NRG, stock CDEmu with default identity and no timing/response patches | Reached neighbourhood; physical tray open |

## What differed

A capture of actual Linux `SG_IO` responses from Wine found ten differing
pregap sectors. At these LBAs, the original drive returned `08` at byte offset
2047 of the 2048-byte user-data payload, while the old CUE image returned `00`:

```
-51 -39 -38 -33 -32 -26 -16 -8 -7 -6
```

These are negative LBAs, before the start of the normal data track. SafeDisc
reads this region with READ CD MSF (`B9`, user-data selection `10`). Its 72
pregap reads in the successful real-drive capture matched a separate raw
pregap capture exactly. Two full 150-sector captures also matched byte for byte.
The other shared sector-read responses matched between the real and virtual
runs. MODE SELECT status/sense differences remained, but did not prevent the
successful stock-CDEmu launch.

The original redumper log explicitly reported:

```
warning: incomplete pre-gap (session: 1, unavailable: 150/150)
```

Its CUE contained only `INDEX 01 00:00:00`, and the BIN began at LBA 0.
It therefore contained none of the physical pregap. Successful normal-file
reads, matching TOCs and similar timings did not establish image equivalence.

## Why use NRG here?

In this libMirage implementation, `mirage_parser_add_redbook_pregap()` prepends
a 150-sector NULL fragment. The CUE parser calls it even when a CUE declares
an explicit INDEX 00/01 gap. Simply prepending the recovered sectors to the BIN
and adding `INDEX 01 00:02:00` therefore shifts the normal track by another
150 sectors: the ISO volume descriptor appears at LBA 166 instead of LBA 16.
This is existing CUE-parser behaviour; this investigation does not change it
globally or claim that all CUE pregap conventions should be reinterpreted.

The NRG parser's DAOX path supports an actual file-backed pregap and explicit
offsets. CUEX sets INDEX 00 to LBA -150 and INDEX 01 to LBA 0. The resulting
disc reports 335411 normal data sectors, matching the original, and preserves
the original raw BIN bytes after the 352800-byte pregap.

Relevant source:

- `libmirage/mirage/parser.c`: `mirage_parser_add_redbook_pregap()`
- `libmirage/images/image-cue/parser.c`: final layout construction
- `libmirage/images/image-nrg/parser.c`: DAOX pregap fragments and CUEX indices

## Reproduce with an owned single-track MODE1 disc

`tools/preserve-cd-pregap` captures the original data rather than synthesizing
a protection signature. Its scope is deliberately narrow: one MODE1/2352
ISO9660 track starting at LBA 0, with a separately captured 150-sector pregap.
It is not a general multisession/multitrack image converter, and does not
preserve subchannels or perform ECC repair. Use the original disc matching
the existing BIN.

```sh
tools/preserve-cd-pregap capture --device /dev/sr0 --output disc1.pregap.bin
tools/preserve-cd-pregap pack --bin disc1.bin --pregap disc1.pregap.bin --output disc1.nrg
cdemu load 0 disc1.nrg
```

Capture requires `sg_raw` (sg3_utils) and read access to the optical device.
It reads twice and refuses inconsistent results. Requests contain at most
16 raw sectors, keeping transfers below 64 KiB. A single 352800-byte request
failed with an I/O error on the tested drive/transport. Packing requires only
Python 3 and refuses to overwrite existing files.

Disc images, captured responses, installed game files and Wine prefixes must
stay local. This repository contains only the conversion tool and findings.
