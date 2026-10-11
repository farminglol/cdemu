# Drive behaviour emulation (fork patches)

Experimental additions to `cdemu-daemon` that make a virtual drive behave more
like a specific physical drive: read timing and verbatim command responses.
They were written to study how disc-check copy protections (SafeDisc 4) tell a
virtual drive from a real one. Everything is off unless the environment
variables below are set when the daemon starts.

The Sims Complete Collection investigation is now resolved: the original image
omitted data in the first track's pregap. A pregap-preserving NRG works with stock
CDEmu, without these overrides. See [the findings and recovery tool](README.sims-pregap.md).

## Read timing

| Variable | Meaning |
| --- | --- |
| `CDEMU_SLOW_SECTORS` | File with one `lba delay_us` pair per line. Reading a listed sector takes that long; the sector right after one costs an extra 45 ms (a real drive re-seeks after a weak sector). |
| `CDEMU_SEEK_US` | Base cost of a seek, in µs. |
| `CDEMU_SEEK_NS_PER_SECTOR` | Extra seek cost per sector of distance, in ns. |
| `CDEMU_SEEK_MIN` | Jumps of at most this many sectors are not seeks (default 16). |
| `CDEMU_SECTOR_US` | Transfer time per sector read, in µs. |

The delays apply to READ (10/12), READ CD and READ CD MSF, on top of the
existing DPM/transfer-rate emulation.

## Verbatim responses

`CDEMU_OVERRIDE_DIR` names a directory of responses captured from a real drive:

- `inquiry.bin` — standard INQUIRY data.
- `modesense10-2a.bin` — MODE SENSE (10) response for page 2Ah (current values).
- `cdb-<hex>.bin` — data returned, with GOOD status, for a command whose CDB
  bytes are exactly `<hex>` (6, 10 or 12 bytes by opcode group).
- `cdb-<hex>.sense` — `key asc ascq` in hex; the command fails with that sense.

Per-CDB files are checked before the built-in command implementation, so they
can replace any command's answer, including TOC and disc-structure reads.
