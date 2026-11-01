# DriveTests - notes for Claude (and any other AI agent) working on an Amiga disk driver

This repository is a test kit for AmigaOS (m68k) disk device drivers and
file systems. When a person asks you to test, debug or review a driver with
it, this is what you need to know.

## What is here

- `flysuite` - exercises one unit of a trackdisk-style driver (CMD_READ/WRITE,
  TD64, NSD, HD_SCSICMD). Every option and every check is documented in
  `flysuite.md` inside the release drawer and in `docs/flysuite.md` of
  github.com/lainejones/DataFlyerSCSI, where its source (`tools/flysuite.c`)
  lives.
- `AmiFSX` - fsx port, hammers one file through dos.library. Source and
  docs: github.com/lainejones/AmiFSX.
- `AmiDefrag` (John Hertell, MIT) - block-map analyser for FFS/PFS3/SFS. Use
  its analyse mode (`AmiDefrag Vol: MAP`) only; the defrag engines are beta.
- `src/RunTests` - the AmigaDOS script behind the icon: picks a scratch
  drawer, `volunit` turns it into `DEVICE= UNIT=`, then runs the three.
- `src/volunit.c` - prints the device driver and unit behind a drawer.

The tools run **on the Amiga** (real or emulated, OS 2.04+, 68000 for
flysuite/AmiFSX). You cannot run them on the PC. Ways to drive them from a
session:

- an emulator you can script (WinUAE with a host-directory drive and a
  startup script; Copperline; FS-UAE/Amiberry with a shared folder): copy
  the drawer in, run `flysuite ... LOG=<file on the shared folder>`, read the
  log from the host;
- a real machine over a remote harness (NetHarness, a serial console, an SSH
  bridge): `PUTFILE` the drawer, run, `GETFILE` the log;
- or ask the person to run it and paste `flysuite.log` back.

## How to run it, in order

1. Read-only first, never on the driver the machine boots from unless the
   person accepts that a driver bug may take the machine down:
   `flysuite DEVICE=<driver> UNIT=<n> TESTS=conf,big,edge,audit LOG=<file>`
2. Then writes, on a scratch volume the person named:
   `flysuite DEVICE=<driver> UNIT=<n> TESTS=all WRITEFILE=Scratch:t LOG=<file>`
   Raw block writes need `RAWWRITE=first,count` (saved and restored; refused
   over an RDB, a partition or a mounted volume).
3. `AmiFSX Scratch:fsxtest N=100000 P=5000 C=100 XREAD`
4. `AmiDefrag Scratch: MAP WIDTH=72` and look for "unknown" blocks.

Put the LOG on a volume of a *different* driver than the one under test (or
the emulator's host folder): flysuite closes the file after every line, so
if the driver crashes the machine the last line names the test.

`ATAPIDIR` and `RUNEND` need `DANGER` and can wedge a machine: emulator only.
`FUZZ` too.

## Reading the result

One line per check: `PASS`, `FAIL`, `SKIP`, `HANG`. Return code 0 passed, 5 a
FAIL, 10 a HANG, 20 refused (bad arguments, a range the tool will not
write). The summary at the end counts them and prints the driver counters.

- A `FAIL` is the driver's. The audit tests say what they expect:
  `ERRMAP` (error codes past the end), `DIRMIS` (data phase against the
  caller's direction - a 53C80 phase loop gets this wrong easily; it must
  not invent bytes), `CDBPASS` (byte 1 of a LUN 0 command untouched),
  `RACEOPEN` (several first opens share one unit), `ATAPIDIR`.
- "the TARGET sends less than its header says" is the drive's, counts as
  a pass.
- A `HANG` names the command that never came back; the tool aborted it and
  went on. Look for a missing timeout or a phase the driver does not expect.
- If the machine went down: the last log line is the test; reproduce with
  that group alone (`TESTS=<group>`) before changing anything.
- `SEED=<n>` from the log header repeats a run exactly.

## Fixing a driver with it

Reproduce before fixing: run the failing test alone, make the fix, run the
same test, then the whole read-only set again, then the writes. Keep the
before/after pass counts in the commit message or the issue. A fix in a phase
loop often moves a failure elsewhere - the full `TESTS=all` run afterwards is
not optional.

## Reporting back to the kit

If the tool itself is wrong (a check that cannot pass on a correct driver, a
wrong expectation for a device class), open an issue on the DataFlyerSCSI
repository with the log line and the driver, or on AmiFSX / AmiDefrag for
those. Issues about RunTests, volunit or the drawer go here.
