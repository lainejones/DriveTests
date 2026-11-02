# DriveTests

Stress tests for Amiga disk drivers and file systems - four tools, three
levels, in one drawer, with a script that runs the safe ones from an icon.

| | looks at | what it does |
|---|---|---|
| **flysuite** | the **driver** | drives one unit of any trackdisk-style device driver (CMD_READ, TD64, NSD, HD_SCSICMD) and checks every answer: the HD_SCSICMD contract, reads of 1-2048 blocks through every path, the edges of the unit, several tasks at once, timed soaks, SCSI+IDE at the same time, and one test per finding of a 53C80 driver audit |
| **devsoak** | the **driver**, destructively | Simon Dick's soak tester: overwrites every sector of a range or a partition you give it, keeps per-sector state, and proves the *content* stays right for hours with several tasks and several requests in flight, plus the edge cases (bounds, alignment, zero length, the 64-bit high word, stale change counts). Where flysuite saves and restores, devsoak destroys - by hand only, on a partition you do not need |
| **AmiFSX** | the **files** | a port of fsx, the file system exerciser from NeXT/Apple still used by FreeBSD and xfstests: keeps a model of a test file and hammers the real one with random reads, writes, size changes and reopens, checking every byte |
| **AmiDefrag** | the **structures on disk** | John Hertell's block-map analyser: walks the file system's own bitmap, directories and file headers (FFS, PFS3, SFS) and maps every block; read-only in the analyse mode the kit uses |

A driver bug shows up at the level it lives at: a wrong error code or a
dropped byte in flysuite, a corrupted file in AmiFSX, a damaged structure in
AmiDefrag; a driver that silently corrupts data under load fails devsoak. All
need AmigaOS 2.04 or newer except devsoak (Kickstart 1.3); flysuite, AmiFSX
and devsoak run on a 68000.

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
    Copy DriveTests/devsoak#? C:

(`devsoak.quirks` has to sit beside `devsoak`, so both go.)

**From a PC:** the tools run on the Amiga, not on the PC. Get the archive
onto the Amiga whole and unpack it there (LhA keeps the executable
protection bits). If you unpack on the PC and copy the files over, the
bits are lost and the Shell refuses to run them: `RunTests` puts them back
by itself, by hand it is

    Protect DriveTests/flysuite +e

and the same for `AmiFSX`, `AmiDefrag`, `devsoak` and `volunit`. In an emulator
(WinUAE, Copperline) with the drawer in a host folder it runs the same way,
against the emulated drive - fine for a driver that runs in the emulator, no
use for one that only exists on real hardware.

## Running it

**From Workbench:** double-click **RunTests**. It asks for a *scratch*
drawer on the drive to test, works out the drive's device driver and unit
from it (`volunit`), then asks Quick or Full:

- **Quick** reads only and writes nothing: flysuite's conf/big/edge/dma16/
  latency/audit groups, then the AmiDefrag analysis
- **Full** also writes test files in that drawer (made, checked and deleted
  again) and runs AmiFSX there

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

Arguments: `DIR` (the scratch drawer), `Quick|Full`, `SKIPIDS=<id,id>`.
Left out, the first two are asked for with requesters. (`AMIDEFRAG` from
1.2 is still accepted; the analysis now always runs.)

or the tools one by one, read-only first:

    flysuite DEVICE=yourscsi.device UNIT=0 TESTS=conf,big,edge,audit
    flysuite DEVICE=yourscsi.device UNIT=0 TESTS=all WRITEFILE=Scratch:t
    AmiFSX Scratch:fsxtest N=100000 P=5000 C=100 XREAD
    AmiDefrag DH1: MAP WIDTH=72
    devsoak Scratch: -d -t 30m -y

`volunit Work:tests` prints the `DEVICE= UNIT=` pair for a drawer.
AmiDefrag takes the DOS device name (`DH1:`), not the volume name;
`AmiDefrag LIST` shows which is which.

devsoak is the one RunTests never runs: it takes the partition's own extent
as its range (nothing outside it is written), inhibits the file system,
and overwrites every sector in it, with several tasks at once, for as long
as you say (`-t 8h` for a real soak, `-w`/`-q` for more workers and deeper
queues; give deeper queues a longer watchdog, `-W 60` for `-w 4 -q 4` on a
slow drive, as its manual says). A `WATCHDOG` FAIL ends devsoak but leaves
its worker tasks running: reboot straight after one. A partition with a
mounted volume on it has to be confirmed by typing the volume's name back;
`-y` does not skip that. Reformat the partition afterwards. `devsoak.md` has every option; `devsoak.quirks`
(kept beside the binary) lists known driver-specific behaviour.

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
- devsoak (by hand only) had overwritten its partition before anything
  crashed: that partition is scrap either way

A crash in Quick mode is a driver finding in itself: say which test.

## Reporting

Post `flysuite.log` (every line, flushed as it goes) and the summary;
`SEED=` repeats a run. AmiFSX: the operation dump and the seed. A FAIL is
the driver's; "the TARGET sends less than its header says" is the drive's
and counts as a pass.

Issues: here for the kit, RunTests and volunit;
[DataFlyerSCSI](https://github.com/lainejones/DataFlyerSCSI) for flysuite;
[AmiFSX](https://github.com/lainejones/AmiFSX) for AmiFSX;
[devsoak](https://github.com/sidick/devsoak) for devsoak;
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
- `AmiDefrag`: ChuckyGang/AmiDefrag (proto release 0.5, 1 Nov 2026)
- `devsoak`: sidick/devsoak (release 1.1, with its `devsoak.quirks` and
  README as `devsoak.md`)
- `volunit`, `RunTests`, `RAD-FFS`, the drawer's `ReadMe`: here (`src/`)

## Credits

- **flysuite, volunit, RunTests** - Laine Jones, for the DataFlyerSCSI driver. GPL-3.0.
- **AmiFSX** - Laine Jones's port of **fsx**, the file system exerciser by
  Avadis Tevanian Jr. (NeXT) and Apple, as carried on in FreeBSD, xfstests
  and LTP. GPL-3.0 for the port.
- **AmiDefrag 0.5** - John Hertell (director) and Claude Code (code),
  [ChuckyGang/AmiDefrag](https://github.com/ChuckyGang/AmiDefrag). MIT,
  included as `LICENSE-AmiDefrag`. Thanks for a block-map analyser that
  reads FFS, PFS3 and SFS alike.
- **devsoak 1.1** - Simon Dick (sidick), [sidick/devsoak](https://github.com/sidick/devsoak).
  BSD 2-Clause, included as `LICENSE-devsoak`. Thanks for offering it to
  the kit.
- **luVWagn** - whose re-engineered `csascsi.device` for the CSA Twelve
  Gauge (another 53C80 card) was the first driver outside ours the kit was
  tidied up for.

## Licence

GPL-3.0 (`LICENSE`) for everything here except AmiDefrag, which is MIT
(`LICENSE-AmiDefrag`), and devsoak, which is BSD 2-Clause (`LICENSE-devsoak`).
