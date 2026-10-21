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
| `F_CommandAcceptance_Tests` | `F_CommandAcceptance` / `F_CommandRejectText` (#83, #87): one test per row of the command-acceptance table (reject reasons, park wins over power-on, park deferred during a stop, latest move replaces) |
| `F_AxisTurn_Tests` | the turn of an azimuth/derotator target inside the axis limits (#2): MONETS/MONETN limits, inclusive endpoints, no valid turn, longest track time with the 4 h cap, nearest turn without velocity, the margin, and that a tracking axis keeps its turn and stops at the margin instead of jumping 360 deg |
| `F_TargetBelowHorizonWarning_Tests` | the 'Target below safe horizon' condition (#94): only a requested goto/track below `fHorizon` warns, an idle or parked telescope with a stale RA/Dec target does not, the comparison is strict |
| `F_ErrorParkRequest_Tests` | the request for the emergency park on a new axis error (#86): only a rising edge of a hardware error on a telescope that is not parked and not in E-stop, no repeat while the error persists, no loop after the park |
| `F_BrakeMustClose_Tests` | the brake close while the elevation is disabled (#121): TRUE when disabled, FALSE when enabled |
| `F_ParkAxesCanMove_Tests` | the park decision (#87, #121): drive only with all three axes calibrated and enabled and the brake open, otherwise park in place |
| `F_ResetMoveAbort_Tests` | the abort stop after a reset (#56, #66): kept while an enabled axis moves or homes, cleared at rest, on an error and while a park runs |
| `F_SuspendStop_Tests` | the stop at the switch to manual (#118): moving or homing axis, running park and running stop are stopped, not at rest or on an error |
| `F_WakeShakeErrorExempt_Tests` | the Wake&Shake exemption from the emergency park (#113): only the first power-on after a PLC restart, only while powering, only for azimuth/elevation errors |
| `F_OilLevelValid_Tests` | HalfBROT `F_OilLevelValid` (MONETS#32): the raw oil level range, with the values measured on MONET/S (wire break 0, sensor error 330, empty 883..1330, float switch 9416, over-range 16383/32767) and the no-guard defaults |
| `F_OilSuctionLatch_Tests` | HalfBROT `F_OilSuctionLatch` (MONETS#32): equal to the old suction SR while the level is valid; invalid: no start by level, a running suction kept while the main pump is enabled, float switch still starts |
| `FB_MonetStopTransaction_Tests` | `FB_MonetStopTransaction` (#96): one MC_Stop per axis and stop, a latched StopAxis level written low first, no second stop while an axis is Stopping after StopDone (the #96 guard), no settling on a bare standstill sample, three scans at rest after the last StopDone, disabled axes, repeated stop (D2) and the timeout cases (D1) |
| `StopLoop_Sim_Tests` | scan-level simulation of the stop (#96): three mock axes with the real `BROTLib.FB_AxisCommand` and an MC_Stop model; the 0.9.0 rule restarts MC_Stop with skewed Done times, the transaction sends one stop per axis and settles |
| `F_GotoTargetCheck_Tests` | the elevation window of a goto/track target (#109): below 2 deg 16#15, above 89.5 deg 16#16, both limits inside, and that with `fHorizon` 1 deg every target that warns is also refused (a warning alone only exists when `fHorizon` is above the floor) |
| `FB_MonetRig_Tests` | the mock rig for the telescope state machine tests (MONETcommon#36, BROTLib plan 2026-10-21-telescope-state-machine-unification): `FB_ElevationMock`/`FB_AzimuthMock`/`FB_DerotatorMock`/`FB_FocusMock` subclass the HalfBROT blocks with the body replaced by a motion model (`FB_AxisMockCore`); a mock axis homes and reports `MoveDone`, and the real `FB_MonetTelescopeControl` runs on the mocks (`FB_CoversMock`, `FB_BrakeHydraulicsMock`) without an error. The nested `fbAxis` outputs are written through pointers, they are not writable from outside |
| `MonetStopDuringSlew_Tests` | `FB_MonetTelescopeControl` | step 1 of the unification plan, characterization of a stop during a slew on the mock rig (`FB_MonetRig`: mock axes with a stop model and a homing that ends at the calibration cam, the elevation mock opens the brake, scanned in the MONETS MAIN order; each test owns a rig and runs over many PLC cycles, the first ones wait for the astronomical clock): one MC_Stop for each of the three axes a slew moves (#96), the moves are dropped in the first scan and every axis has its stop request by the second (`_StopTelescope`), the stop drops `bSlew`, keeps `bMoveAbortPending` until the stop has settled (#84) and leaves the telescope online. They pin what the code does today, they are not a specification |
| `MonetParkDuringPowerOn_Tests` | `FB_MonetTelescopeControl` | step 1 of the unification plan, on the mock rig: a park that arrives while the power-on has only commanded the covers aborts it and parks in place (axes disabled, covers closed again, READY_STATE 0.0, #87/#113); a park during the homing stops each homing axis once and the aborted homing does not calibrate; a stop during the power-on aborts the startup (READY_STATE -2.0), parks nothing and leaves the covers open (#80); an `Init()` during a park is refused with a reason (#87) |
| `MonetResetUnderMotion_Tests` | `FB_MonetTelescopeControl` | step 1 of the unification plan, on the mock rig: a reset during a slew drops the slew, keeps the abort stop until the axes have stopped (one stop each) and resets no moving axis (#56, #84, #85); the reset aborts the startup, so an online telescope drops to READY_STATE -2.0; a reset during a park keeps the park and sends no abort stop; a reset at rest drops an accepted slew that has not started |

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

Same as `BROTLibTests` (CI runs these tests through `tcbuild.yml`) -- TcUnit sized to 32 suites / 32 tests / 256 asserts, every test method
runs every PLC cycle (multi-cycle tests must guard their own state), `.tmc`/`_Boot`/`_CompileInfo`/
`_Libraries` are generated and git-ignored.

## Failed checks in CI

TcUnit writes the text of a failed assertion only to the ADS log, which CI cannot see. The scenario tests therefore also call `F_Note` next to every assertion, which appends `test: message;` to `GVL_Diag.sFailures`; `tools/Run-Tests.ps1` prints it as `Failed checks: ...` when a test case failed. A failure without a note comes from a test that does not use `F_Note` (the older suites).
