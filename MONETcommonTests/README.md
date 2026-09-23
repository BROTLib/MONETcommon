# MONETcommonTests

TcUnit tests for MONETcommon. A separate PLC project, so no test code ends up in the shipped
library. References the *installed* MONETcommon (`MONETcommon, * (IAG)`), exactly like MONETN or
MONETS does. Structure and tooling copied from `BROTLib/BROTLibTests` -- see that project's
README for the underlying mechanics (XAE automation for library install/deploy, ADS polling for
results); this one only covers what's specific to MONETcommon.

## What is covered

| Suite | What it checks |
|---|---|
| `FB_MonetCoverControl_Tests` | `FB_MonetCoverControl` actually compiles and runs with no linked hardware, and that the inverted limit-switch polarity correctly reports `bError` when both switches read as active (the default state of an unlinked `AT%I*` input) |

This is a first, deliberately small slice -- see
[MONETcommon/specs/plans/2026-09-22-add-tcunit-tests.md](../specs/plans/2026-09-22-add-tcunit-tests.md)
for the open questions still blocking wider coverage (most importantly: `FB_MonetTelescopeControl`
takes `REFERENCE TO FB_ElevationControl`/`FB_AzimuthControl`/`FB_DerotatorControl`, concrete
HalfBROT types built on `AXIS_REF` -- whether those run against an unlinked/virtual axis on this
target, or need something more, is not yet confirmed).

### Why `FB_MonetCoverControl`'s new movement timeout (#9) isn't tested yet

`IabCoverOpen`/`IabCoverClosed` are `AT%I*`, mapped *inside* the FB itself -- a test in a different
FB instance has no way to force them from ST code (that needs an ADS force list, or restructuring
the limit switches into inputs the caller controls). Confirmed instead, as a first step, that the
FB runs at all without hardware, and that its *existing* both-switches-read-true-by-default
behavior is what it should be. Forcing specific switch states (to actually exercise the "never
reaches the limit switch" timeout path) is follow-up work, likely via `Run-Tests.ps1` scripting an
ADS force list after deploy, since that's outside what a self-contained ST test can do.

## Running the tests

See `BROTLibTests/README.md` for the full one-time setup and Windows 11 usermode-runtime notes
(they apply unchanged here). Short version, once TcUnit is installed and MONETcommon itself is
installed as a library:

```powershell
& "C:\Program Files\Industrial Brains B.V\TcBuild\TcBuild.exe" install MONETcommon.sln -x MONETcommon -p MONETcommon -l MONETcommon.library
cd C:\TwinCAT\3.1\Runtimes\UmRT_Default; .\Start.bat
& "C:\Program Files\Industrial Brains B.V\TcBuild\TcBuild.exe" build MONETcommonTests.sln
.\MONETcommonTests\tools\Run-Tests.ps1
```

## Things to know

Same as `BROTLibTests` -- TcUnit sized to 32 suites / 32 tests / 256 asserts, every test method
runs every PLC cycle (multi-cycle tests must guard their own state), `.tmc`/`_Boot`/`_CompileInfo`/
`_Libraries` are generated and git-ignored.
