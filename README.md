# DriveTests

Stress tests for Amiga disk drivers and file systems - three tools, one per
level, in one drawer, with a script that runs them from an icon.

| | looks at | what it does |
|---|---|---|
| **flysuite** | the **driver** | drives one unit of any trackdisk-style device driver (CMD_READ, TD64, NSD, HD_SCSICMD) and checks every answer: the HD_SCSICMD contract, reads of 1-2048 blocks through every path, the edges of the unit, several tasks at once, timed soaks, SCSI+IDE at the same time, and one test per finding of a 53C80 driver audit |
| **AmiFSX** | the **files** | a port of fsx, the file system exerciser from NeXT/Apple still used by FreeBSD and xfstests: keeps a model of a test file and hammers the real one with random reads, writes, size changes and reopens, checking every byte |
| **AmiDefrag** | the **structures on disk** | John Hertell's block-map analyser: walks the file system's own bitmap, directories and file headers (FFS, PFS3, SFS) and maps every block; read-only in the analyse mode the kit uses |

A driver bug shows up at the level it lives at: a wrong error code or a
dropped byte in flysuite, a corrupted file in AmiFSX, a damaged structure in
AmiDefrag. All three need AmigaOS 2.04 or newer; flysuite and AmiFSX run on a
68000.

flysuite was written for the [DataFlyerSCSI](https://github.com/lainejones/DataFlyerSCSI)
driver (an NCR 53C80 card) and found six real bugs in it; it runs against any
driver - `DEVICE=yours.device UNIT=n` is all it needs.

## Install

There is no installer: the drawer is the installation. Unpack
`DriveTests.lha` **on the Amiga**:

    LhA x DriveTests.lha Work:

and a `DriveTests` drawer with its icon appears. Nothing is copied anywhere
else - no libraries, no assigns, no ENV: variables.

**Shell only (no Workbench, or you want the tools on the path):**

    Copy DriveTests/flysuite C:
    Copy DriveTests/AmiFSX C:
    Copy DriveTests/AmiDefrag C:
    Copy DriveTests/volunit C:

**From a PC:** the tools run on the Amiga, not on the PC. Get the archive
onto the Amiga whole and unpack it there (LhA keeps the executable
protection bits). If you unpack on the PC and copy the files over, the
bits are lost and the Shell refuses to run them: `RunTests` puts them back
by itself, by hand it is

    Protect DriveTests/flysuite +e

and the same for `AmiFSX`, `AmiDefrag` and `volunit`. In an emulator
(WinUAE, Copperline) with the drawer in a host folder it runs the same way,
against the emulated drive - fine for a driver that runs in the emulator, no
use for one that only exists on real hardware.

## Running it

**From Workbench:** double-click **RunTests**. It asks for a *scratch*
drawer on the drive to test, works out the drive's device driver and unit
from it (`volunit`), then asks Quick or Full:

- **Quick** reads only and writes nothing: flysuite's conf/big/edge/audit
  groups
- **Full** also writes test files in that drawer (made, checked and deleted
  again) and runs AmiFSX there

The AmiDefrag analysis is **off by default** (`AMIDEFRAG` on the Shell line
turns it on): in our tests on OS 3.2.3 - a real A1200 and WinUAE, FFS
volumes - AmiDefrag 0.3 printed its block map in full and then rebooted the
machine when it exited. It is still in the drawer for the map, which is
good; run it by hand on a machine with nothing unsaved. Reported upstream.

The output shows in its window, every log is saved in the drawer
(`flysuite.log`, `amifsx.log`, `amidefrag.log`) and a requester at the end
says how it went.

**From a Shell:** the same, with the answers on the line (the script runs
from its own drawer):

    CD Work:DriveTests
    Execute RunTests Work:tests Quick

A SCSI bus with a network adapter on it (a DaynaPORT emulation on a
BlueSCSI/ZuluSCSI) needs that id kept out of the probes, or the network
driver and the test talk to it at once:

    Execute RunTests Work:tests Quick SKIPIDS=5

Arguments: `DIR` (the scratch drawer), `Quick|Full`, `SKIPIDS=<id,id>`,
`AMIDEFRAG` (also run the AmiDefrag analysis). Left out, the first two are
asked for with requesters.

or the tools one by one, read-only first:

    flysuite DEVICE=yourscsi.device UNIT=0 TESTS=conf,big,edge,audit
    flysuite DEVICE=yourscsi.device UNIT=0 TESTS=all WRITEFILE=Scratch:t
    AmiFSX Scratch:fsxtest N=100000 P=5000 C=100 XREAD
    AmiDefrag Scratch: MAP WIDTH=72

`volunit Work:tests` prints the `DEVICE= UNIT=` pair for a drawer.

Nothing writes unless you say so. flysuite's raw block writes need
`RAWWRITE=first,count` (max 1 MB): the blocks are saved first and restored
at the end and on every error path, and a range over an RDB, a partition or
a mounted volume is refused. File-level writes go through `WRITEFILE=`.
Two tests that can wedge a machine (ATAPIDIR, RUNEND) need `DANGER` and are
meant for an emulator. Every wait is bounded: a request that never comes
back is reported as HANG with the command that caused it, and the run ends
with a summary instead of a frozen machine. Return codes: 0 passed, 5 a
FAIL, 10 a HANG, 20 refused.

The `ReadMe` in the drawer lists the test groups and what each audit test
looks for in a 53C80 driver; `flysuite.md`, `AmiFSX.md` and `AmiDefrag.md`
document every option.

## If it crashes

flysuite itself does not freeze a machine, but a driver can take it down (a
Guru, or a reset with nothing on screen). Then:

- reboot; a volume that was being written validates itself - normal
- read `flysuite.log` in the scratch drawer: it is written line by line, so
  the last line names the test that was running. That line is the report.
- Full mode leaves test files behind (`flysuite.*`, `fsxtest*` in the
  scratch drawer): delete them
- a `RAWWRITE` range (by hand only; RunTests uses none) keeps its saved copy
  in RAM, so after a crash the range holds test data - unused space outside
  every partition, by the tool's own refusal, not your files
- AmiDefrag's analysis and Quick mode write nothing

A crash in Quick mode is a driver finding in itself: say which test.

## Reporting

Post `flysuite.log` (every line, flushed as it goes) and the summary;
`SEED=` repeats a run. AmiFSX: the operation dump and the seed. A FAIL is
the driver's; "the TARGET sends less than its header says" is the drive's
and counts as a pass.

Issues: here for the kit, RunTests and volunit;
[DataFlyerSCSI](https://github.com/lainejones/DataFlyerSCSI) for flysuite;
[AmiFSX](https://github.com/lainejones/AmiFSX) for AmiFSX;
[AmiDefrag](https://github.com/ChuckyGang/AmiDefrag) for AmiDefrag.

## Using it with Claude (or another AI agent)

The tools run on the Amiga, but the reading and the fixing can be done with
an AI. Two ways:

- **claude.ai / Claude Desktop:** run flysuite with `LOG=`, then give Claude
  the log (paste it, or attach the file) with the driver's source and ask
  what failed and why. The log is one line per check with the command and
  the expected/got values, which is what it needs; `flysuite.md` in the
  drawer explains every check if it asks.
- **Claude Code (or any coding agent) in the driver's repository:** clone
  this repository next to it, or copy `CLAUDE.md` from here into the
  driver's. It tells the agent what the tools are, in which order to run
  them, where to put the log so it survives a crash, how to read a FAIL /
  HANG / crash, and to reproduce before fixing. The agent still needs a way
  to run things on an Amiga: a scriptable emulator (WinUAE with a
  host-directory drive, Copperline), a remote harness on a real machine, or
  you running it and pasting the log back.

This is how flysuite was used on the DataFlyerSCSI driver: audit, test,
reproduce each finding, fix, re-run the same test, then everything again.

## Source

- `flysuite`: `tools/flysuite.c` in the DataFlyerSCSI repository
- `AmiFSX`: the AmiFSX repository
- `AmiDefrag`: ChuckyGang/AmiDefrag (release 0.3)
- `volunit`, `RunTests`, `RAD-FFS`, the drawer's `ReadMe`: here (`src/`)

## Credits

- **flysuite, volunit, RunTests** - Laine Jones, for the DataFlyerSCSI driver. GPL-3.0.
- **AmiFSX** - Laine Jones's port of **fsx**, the file system exerciser by
  Avadis Tevanian Jr. (NeXT) and Apple, as carried on in FreeBSD, xfstests
  and LTP. GPL-3.0 for the port.
- **AmiDefrag 0.3** - John Hertell (director) and Claude Code (code),
  [ChuckyGang/AmiDefrag](https://github.com/ChuckyGang/AmiDefrag). MIT,
  included as `LICENSE-AmiDefrag`. Thanks for a block-map analyser that
  reads FFS, PFS3 and SFS alike.
- **luVWagn** - whose re-engineered `csascsi.device` for the CSA Twelve
  Gauge (another 53C80 card) was the first driver outside ours the kit was
  tidied up for.

## Licence

GPL-3.0 (`LICENSE`) for everything here except AmiDefrag, which is MIT
(`LICENSE-AmiDefrag`).
