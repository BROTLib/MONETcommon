# Add a TcUnit test suite to MONETcommon

**Status: planned, not started.** Follow-up to [MONETcommon#36](https://github.com/BROTLib/MONETcommon/issues/36),
split off from #20 (M14).

## Context

MONETcommon has zero tests today: no `Tests` project, TcUnit not referenced anywhere. AstroBROT
already has a working TcUnit setup (`AstroBROTTests/`, a separate TwinCAT solution referencing
`AstroBROT` as a library) that can serve as the structural template — but AstroBROT's tests are
all pure math (`FB_CO_REFRACT`, `FB_EQ2HOR`, coordinate transforms, no hardware, no axis
references), which is a much easier testing target than MONETcommon's FBs.

**The open question that gates everything else**: can MONETcommon's FBs actually be instantiated
in a TwinCAT test project at all?

- `FB_MonetTelescopeControl` (the FB behind 3 of the 4 candidate tests below) takes
  `fbElevation`/`fbAzimuth`/`fbDerotator` as `REFERENCE TO FB_ElevationControl`/
  `FB_AzimuthControl`/`FB_DerotatorControl` (concrete HalfBROT types, not interfaces) — these
  don't declare `AT%I*`/`AT%Q*` directly, but they very likely wrap a TwinCAT `AXIS_REF` (NC
  axis reference), which needs a real or virtual NC axis configured in the target system to run
  at all. `fbCovers`/`fbBrake`/`fbHydraulics` are interfaces (`I_MirrorCovers`/`I_Brake`/
  `I_Hydraulics`), so those three are straightforwardly mockable.
- `FB_MonetCoverControl` (behind the 4th candidate test) declares `AT%Q*`/`AT%I*` hardware I/O
  *directly inside the FB itself* (open/close cover outputs, limit-switch inputs) — the same
  "only one instance per PLC" pattern flagged elsewhere in this codebase. A test project would
  need those channels to resolve to *something* (TwinCAT's virtual/simulation I/O, or an
  unlinked/default-initialized state) to even compile and run the FB.

Neither of these is necessarily a blocker — TwinCAT supports virtual NC axes and can often run
`AT%I*`/`AT%Q*` variables unlinked (defaulting to `FALSE`/`0`) in a target with no real hardware
attached — but neither has been verified here. **First step is a spike to confirm this**, before
committing to a test plan that might not actually run.

## Steps

### 1. Spike: confirm MONETcommon FBs can run in a TwinCAT test target

Create the `MONETcommonTests` project skeleton (step 2) with a single trivial test that just
instantiates `FB_MonetTelescopeControl` with a real (or virtual-axis-backed) `FB_ElevationControl`
etc., and separately one that instantiates `FB_MonetCoverControl` on its own. Confirm both compile,
download to a (real or simulation) target, and run without needing physical hardware. If
`FB_MonetCoverControl` genuinely can't run without hardware, that specific candidate test drops
out of scope until it can (see "Open questions" below) — the other three don't depend on it.

### 2. Project setup

Mirror `AstroBROTTests`' structure:
- A separate TwinCAT solution/project, `MONETcommonTests`, referencing `MONETcommon` (and
  transitively `BROTLib`/`HalfBROT`) as libraries, plus `TcUnit`.
- One `FB_X_Tests.TcPOU` per function block under test, `FUNCTION_BLOCK FB_X_Tests EXTENDS
  TcUnit.FB_TestSuite`, following the existing pattern from e.g.
  `AstroBROT/AstroBROTTests/AstroBROTTests/AstroBROTTests/POUs/FB_CO_REFRACT_Tests.TcPOU`:
  ```
  METHOD Test_SomeCase
  TEST('Some_Case_Name');
  // ... arrange, act ...
  AssertEquals_LREAL(Expected := ..., Actual := ..., Delta := ..., Message := '...');
  // or AssertTrue / AssertFalse / AssertEquals_BOOL / etc.
  TEST_FINISHED();
  ```

### 3. Initial test scope (from #36)

Four candidates, in order of how directly testable they are once the step-1 spike confirms the
telescope-control FB runs:

- **`_GotoTelescope`'s zenith guard (#13)**: call with `fRightAscension`/`fDeclination` that
  resolve to elevation > 89.5 deg, assert `bError`/`nErrorID = 16#16` and `bGoto` cleared.
- **`bSoftwareError` latch survives a scan (#7)**: force a `_GotoTelescope` timeout condition (or
  call the method directly with the right stage/timer state), assert `bError`/`nErrorID` are
  still set on the *next* scan (not wiped by the top-of-scan axis-error recompute), and cleared
  only after `bReset`.
- **`_ParkTelescope`'s emergency branch is reachable on `bError` (#8)**: set `bError := TRUE` and
  `bPark := TRUE`, assert the FB actually reaches stage 80 (brake close/axes disabled/covers
  close) within a few scans, not stuck returning early.
- **`FB_MonetCoverControl`'s movement timeout (#9)**: command a cover open with no limit-switch
  ever reporting reached, assert the output de-energizes and `abCoverError`/`afbTimeoutEvent`
  fire after the timeout elapses. Depends on the step-1 spike resolving the hardware-mapping
  question for this specific FB.

### 4. CI

Once tests exist, revisit MONETcommon#23 (S2: `release.yml` has no build/test gate) — this test
suite is the actual prerequisite for adding one.

## Open questions

- Does `FB_ElevationControl`/`FB_AzimuthControl`/`FB_DerotatorControl` need a virtual NC axis to
  run, or can they run with a null/unlinked `AXIS_REF`? Determines whether the spike needs a
  TwinCAT virtual-axis target or can use a plain simulation target.
- If `FB_MonetCoverControl` truly can't run without real I/O, is it worth factoring its pure
  logic (the timeout/state-machine decisions) out of the `AT%I*`/`AT%Q*`-mapped FB into a
  separately-testable helper, or is that too large a refactor to justify for test coverage alone?
  Not decided — flag if the spike hits this.

## Verification

The spike (step 1) is itself the verification that this plan is viable before investing further:
if both candidate FBs run in a TwinCAT test target without physical hardware, proceed with steps
2-3 as scoped; if not, come back and revise scope before writing more tests.
