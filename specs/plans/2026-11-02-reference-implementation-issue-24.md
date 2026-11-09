# MONETcommon as a reference implementation (#24, code review finding D)

Date: 2026-11-02. Status (checked against git 2026-11-06): **code merged to `develop` and released; issue #24 closed 2026-11-06 on request, the on-site checks stay in IAG test plan T17/T18.**
Steps 0-6 and step 7 PR 1-8 (PR 4/6/7/8 squash-merged as #190/#194/#192/#193), the E-stop covers follow-up (#205) and the
cabinet over-temperature roof request (#206) are in MONETcommon 0.15.0/0.15.1 (`develop` is at 0.17.0); MONETS and MONETN
`MAIN` have `bRoofCloseOnErrorEnable` (PR 2); IAG50cm dropped its local `ST_TelescopeConfig` (step 2b, BROTLib 0.17.0 pin). CI builds
and runs the rig suites. Not run on a telescope: IAG test plan T17 steps 6-12 (timeouts partial, #213 open; cover, hydraulics,
roof close, E-stop/safety, power failure, focus error open); no IAG50cm smoke test for step 2b found. The "not compiled, not
run" lines in the step statuses below are the state at writing (2026-11-02/03) and are not updated.
Original status line: **proposed, plan only, no code changed. Decisions D1-D9 taken (Tim, 2026-11-02, section 7;
D8/D9 settle the old open points 8.1/8.2, the safe-state table is final for step 7); the open items left are in
section 8.** Steps 0-1 written 2026-11-02 on `feat/safe-state-table-and-tests`: the
table is [../design/monetcommon-safe-states.md](../design/monetcommon-safe-states.md) (cells corrected against 0.11.2), the suite
`MonetSafeState_Tests`; not compiled or run yet. Step 2a released in BROTLib 0.17.0. Step 2c written 2026-11-02 on
`feat/motion-config-input` (status at the step); not compiled or run yet. Step 3 written 2026-11-02 on
`refactor/stage-machines` (status at the step); not compiled or run yet. Step 4a written 2026-11-02 on
`refactor/telemetry-split` (status at the step); not compiled or run yet. Step 4b written 2026-11-02 on
`refactor/compute-targets` (status at the step); not compiled or run yet. Step 7 PR 1 (`bCloseRoofRequest`) written
2026-11-02 on `feat/close-roof-request` (status at step 7); not compiled or run yet. Step 7 PR 3 (hydraulics error reaction)
written 2026-11-02 on `feat/hydraulics-error-reaction` (status at step 7); not compiled or run yet. Step 7 PR 5 (state-machine
timeouts ramp, then park) written 2026-11-02 on `feat/timeout-error-reaction` (status at step 7); not compiled or run yet. Step 7 PR 7 (power failure
reaction) written 2026-11-02 on `feat/power-failure-reaction` (status at step 7); not compiled or run yet. Step 7 PR 8 (focus error
reaction) written 2026-11-02 on `feat/focus-error-reaction` (status at step 7); not compiled or run yet. Step 7 PR 6 (E-stop / safety
error reaction) written 2026-11-02 on `feat/estop-safety-reaction` (status at step 7); not compiled or run yet. Step 5
(read-only command inputs) written 2026-11-03 on `feat/read-only-command-inputs` (status at the step); not compiled or run yet.
Repo: BROTLib/MONETcommon (steps 2a/2b also BROTLib and IAG50cm). Base: read at `develop` @ `adc1013` (0.11.1, pins
BROTLib 0.15.1, HalfBROT 0.10.1); line numbers are from that commit. `origin/develop` has since moved to `c34470f`
(0.11.2, pins BROTLib 0.16.0 / HalfBROT 0.10.2; pendant release-on-exit #3, null-reference fix #156); none of that
changes the findings below, but recheck line numbers when a step starts.
Short name: **M** = `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU` (2806 lines incl. XML at `adc1013`).
Sources: issue #24, review [2026-09-20-code-review.md](2026-09-20-code-review.md) (D, H1, M2, M3; AstroBROT M1),
`../IAG/specs/design/monetcommon-monetn-monets-unification-plan.md`, and BROTLib plan
`2026-10-21-telescope-state-machine-unification.md` (BROTLib#25, "SM plan" below), which already owns most of aim 2.

Marking: **[verified]** = read or grepped on 2026-11-02; **[unverified]** = my reading, to be checked by a test, CI
or on site. Nothing here has been run on a PLC.

## 1. Where we are, per aim

| Aim | State today | Left for this plan |
|---|---|---|
| 1 Site differences are inputs | Both sites build against the same pinned library; the MONETN telescope copy is gone (SM plan, [verified]: no `TCS_command` in MONETN/MONETS). Wrap from the axis limits (#2, `F_AxisTurn`). Pointing model injected from `MAIN`. Only per-site telescope input today: `fElevationMoveVelocity` (MONETS 3.0, MONETN default 5.0) [verified, both `MAIN.TcPOU`]. All `FB_Monet*` blocks except `FB_MonetStopTransaction`/`FB_MonetTempSensor` (used internally / by the site auxiliary blocks) are used by both sites [verified, grep] | the remaining constants of aim 4 |
| 2 One stage-machine pattern, split | `_ReadAxes`/`ST_AxisStatus`, `_LogEvents`, `_HandleReset` extracted (SM step 3); pure units copied to BROTLib `Telescope/Command/` (SM step 4.1) but **MONETcommon still compiles its own copies** [verified: both trees]. Stages are bare `USINT` (`nStage` in `VAR_INST`, values 0/10/25/50/75/100). Four different guard prologues [verified M:871, 982, 1042, 1171, 1584]. Software errors raised by four copies of `bError := TRUE; nErrorID := ...; bSoftwareError := TRUE; nSoftwareErrorID := ...`. `FB_CommandArbiter` (SM step 5.1) not started | enums, one error path, guard as a policy table, telemetry split, `_ComputeTargets`; arbiter stays in the SM plan |
| 3 Inputs read-only | **Verified, still open.** M writes the inherited BROTLib inputs `bPower` (10 assignments), `bGoHome` (13), `bPark` (5), `bTrack` (16), `bGoto` (15), `bSlew` (15), `bStop` (4), `bReset` (1). HalfBROT writes its own inputs: `FB_CoverControl` `bOpen`/`bClose`/`bReset`, `FB_HydraulicsControl` `bEnable`/`bReset`/`bOpenBrake`/`bCloseBrake`, `FB_FocusControl` `bEnable`/`bReset`/`bHomeAxis`, `FB_AxisControl` `bReset`. MONETS `MAIN` writes `HydraulicsControl.bOpenBrake := brakeClearing` every scan. The MONETcommon-own blocks write none of their `VAR_INPUT`s (only `comm` in `FB_Init`). The PLC-side MQTT `SET` handler (BROTLib `FB_Comm_MQTT_Influx`, `origin/develop`) calls methods (`Init`, `Park`, `Home`, `Track`, `Slew`, `Stop`, `Reset`), not flags [verified] | M's command flags; HalfBROT inputs go to HalfBROT issues |
| 4 Site constants | Wrap literals and `440.0` **gone** (#2; `440` only occurs inside a GUID) [verified]. `fHorizon + 0.2` is now `VAR CONSTANT fHorizonMargin := 0.2` (#34, M:210), still not configurable. Still literal [verified]: 12x azimuth/derotator `Velocity := 10.0`; `T#600S` x3 (goto/home/slew), `T#2M` x4 (park/gohome/goto/slew command), `T#12H` x3 (idle park, power-on, track), `T#15S` x2 (stop, suspend stop); goto window `2.0`/`89.5` (M:890); track cutoff `5.0`/`89.5` (M:2105). Internal timings (`T#5S` interrupted auto-clear, `T#5500MS` tracking delay, `T#250MS`, `T#10M` dut1, `T#15M` Wake&Shake, MQTT cadence) stay constants (D7) | move to `ST_MotionConfig` (D1) |
| 5 Coupling | **Forks gone** [verified git]: `FB_MonetHydraulicsControl` (#15, `ceab23e`), `FB_MonetFocusControl` (#16, `752b65c`), `FB_MonetCoverControl` (`d7801b1`, 0.10.0, now `HalfBROT.FB_CoverControl`). Still uneven [verified]: M takes `I_Focus`/`I_MirrorCovers`/`I_Brake`/`I_Hydraulics` but `REFERENCE TO FB_ElevationControl/FB_AzimuthControl/FB_DerotatorControl`; M reads members `I_Axis` does not have (`fMinPosition`, `fActualPosition`, `bEnable`, `bError`, `Tracking`, `fbAxis` status). Pendant takes `REFERENCE TO` all seven concrete blocks; `fbBrake`/`fbHydraulics` are declared and wired in both `MAIN`s but never used (still so on `origin/develop`) | dead pendant inputs, README rationale (D6) |
| 6 Safe state per error class | Emergency park on a new axis/focus error (#86, tested `MonetErrorPark_Tests`), brake rules (#121/#151/#155, tested), M1 latched software errors, covers stop on error (now HalfBROT). Not written down as one table; untested: hydraulics error, cover error, E-stop under motion, power failure, software errors other than 16#40, invalid clock, MQTT loss, cabinet over-temperature (#32). `FB_MonetMqttWatchdog` calls `Park()` and `iRoof.Close()` in the same scan (lines 28-35) [verified]; that is now the decided behaviour (D4) | table + tests + fixes |

**`ST_TelescopeConfig` and IAG50cm [verified 2026-11-02].** The struct already exists in BROTLib and is passed as
`telescopeConfig` by both MONET `MAIN`s. Since BROTLib **0.16.0** (PR #145, IAG50cm#17) it is a superset: the
equatorial fields `houranglePark`/`declinationPark`/`hourangleHome`/`declinationHome`/`focusHome` are in it, marked
"equatorial mounts (IAG50cm)"; 0.15.1 does not have them. IAG50cm `origin/develop` pins BROTLib 0.16.0 but still
compiles its own `IAG50cm/DUTs/ST_TelescopeConfig.TcDUT` (Id `{eb9f05db-...}`, BROTLib's is `{13de919a-...}`). Its
uses: `FB_TelescopeControl` input `telescopeConfig : ST_TelescopeConfig` (I:52), reads of `longitude`/`latitude`/
`altitude`, `houranglePark`/`declinationPark` (park, three places); `MAIN` `VAR CONSTANT` initialiser by field name
(`longitude`, ..., `houranglePark`, `declinationPark`, `focusPark`, `hourangleHome`, `declinationHome`, `focusHome`,
info fields); test project: `IAG50cmTests/tools/Sync-Sources.ps1` line 34 copies the DUT,
`IAG50cmTests.plcproj` compiles `POUs\ST_TelescopeConfig.TcDUT`, `FB_IagRig` and `FB_IagRig_Tests` initialise it.
Every field IAG50cm uses exists with the same name and type in BROTLib's struct, so dropping the local copy is a
type swap, no field mapping.

## 2. Approach

- Same rule as the SM plan: **characterize first, then change**. Every safety-relevant refactor lands only after
  rig tests pin today's behaviour; a refactor PR edits no test. A behaviour change is its own PR that flips a test.
- Aim 2's arbitration part stays in the SM plan (BROTLib `FB_CommandArbiter`, steps 5.1-5.4). This plan does the
  MONETcommon-local work that makes 5.2 a move of finished pieces, and does not duplicate it.
- Defaults reproduce today's literals exactly, so steps 2-4 are non-breaking for MONETS/MONETN `MAIN` (new inputs
  with defaults, old inputs kept as aliases, D2). The breaking items (read-only command inputs, dropped pendant
  inputs) are batched late.

## 3. Steps (one PR each, in this order)

Every step: branch from `develop`, PR, CI `tcbuild.yml` green. Test runs on this machine (the CI runner) with
`tctool`/XAE/`Run-Tests.ps1` only when Tim asks; otherwise the PR says exactly what to run. On-site checks are
Tim's. Suite budget: `MAXNUMBEROFTESTSUITES` is 40 in `MONETcommonTests.plcproj` and about 37 are used, so new
cases go into existing suites where they fit; step 6 frees slots by moving duplicated suites to BROTLib.

### Step 0 - Safe-state table (docs only)
- New `specs/design/monetcommon-safe-states.md` (the folder does not exist yet: add it with an `index.md` and a line
  in `specs/index.md`; README link). One row per error class: trigger, latched where, what the block does now
  (file:line), target, reset path, test name.
- **Target safe state (D4):** roof closed, telescope parked if possible, covers closed as far as possible,
  hydraulics off. The brakes are hydraulic and **lock by themselves when the hydraulics go off**, so switching the
  hydraulics off *is* the brake close; no separate ordering between brake and hydraulics. The roof closes at once
  and does not wait for `bIsParked`.
- The open point per class is whether the axes get a **controlled stop (ramp, `FB_MonetStopTransaction`) before the
  hydraulics drop**, or whether the brake lock is accepted as the stop (axes may still be moving when the pressure
  goes). Rule proposed below: ramp first whenever the drives are still controllable; the lock is the stop only when
  they are not (decided as D8). The draft table (to copy into the design doc and check against the code in step 1;
  the design doc is the current version, its roof column follows D9, which this draft predates):

| Error class | Drives controllable? | Stop before hydraulics off | Park | Covers | Roof | Today [read from code, pin in step 1] |
|---|---|---|---|---|---|---|
| Axis error Az/El/Der | no (error branch disables all axes) | brake lock is the stop | in place (#86/#87) | close | close | #86 park in place, brake closed on the El error edge (#155) |
| Focus error | yes | ramp | drive to park if `F_ParkAxesCanMove`, else in place | close | close | #86 emergency park |
| Software error 16#10/16#30/16#40 (timeouts), 16#31/16#32 (config) | yes, but the error branch disables the axes today | ramp, then park | drive to park if possible | close | close | axes disabled, brake closed by #121, no park |
| Hydraulics error | no brake hold once pressure is gone | ramp in the same scan; the lock wins if pressure is already gone | in place | close | close | only the brake rule (#155); no telescope reaction |
| Cover error | yes | ramp | drive to park | as far as possible | close | `bReady` drops; nothing else |
| E-stop / safety error | no (STO, TwinSAFE) | brake lock is the stop | in place | as possible after release | close | axes disabled, no park (#86 excludes E-stop) |
| Power failure / `bMainReady` FALSE | depends on what is lost | ramp if drives are powered, else lock | if possible | as possible | close | only blocks power-on |
| Invalid clock (`fbTime.bValid` FALSE) | yes | ramp (existing stop path) | drive to park | close | close | stop, no park |
| MQTT loss (watchdog) | yes | ramp via the park routing | drive to park | close | close at once (D4) | `Park()` + `iRoof.Close()` same scan: matches D4 |
| Cabinet over-temperature (#32) | yes | ramp via park | drive to park | close | close | `Park()` from `FB_MonetCabinetControl` |
| Manual mode under motion | yes | ramp (#118 suspend stop) | no automatic park (operator) | operator | operator | #118 |
| PLC restart | pose unknown | none needed (hydraulics off at start) | no automatic motion | as found | site | Wake&Shake window (#151) |

- Who closes the roof for classes other than the watchdog is the site `MAIN` / MONETRoof, not this library; the
  table names the signal M provides (e.g. `bError`, READY_STATE) and the `MAIN` line that should act on it.
  Decided (D9): a new M output (proposed `bCloseRoofRequest`) for the serious classes only, acted on in each `MAIN`
  behind an enable switch and only in automatic; the design doc has the derivation and the `MAIN` line.
- Verified by review only. Release: none.

### Step 1 - Characterization suites for the untested error classes (aim 6, tests only)
- Files: `MONETcommonTests/.../POUs/` new `MonetSafeState_Tests.TcPOU`, `FB_CoversMock`/`FB_BrakeHydraulicsMock`
  gain an injectable error (and the hydraulics mock: brake locks when the hydraulics are off),
  `MAIN.TcPOU`, `MONETcommonTests.plcproj`, `MONETcommonTests/README.md` table. Optional
  `FB_MonetMqttWatchdog_Tests` with a mock `I_Telescope`/`I_Roof` (counts against the 40-suite budget).
- Cases (pin today, `_Current` suffix where it differs from the step-0 target): hydraulics error while tracking
  (are the axes still commanded when the brake locks?); cover error during power-on and during park; E-stop during
  slew (axes disabled, no park, READY_STATE); power failure during goto; goto 600 s timeout 16#10 and home 16#30
  (time seam `nTimeDivisor`); clock invalid during track (stop path); watchdog: park and roof close in one scan.
- Verified: suites green against unchanged MONETcommon; each red test is a rig bug or a finding in step 0's table.
  Release: none (test project).

### Step 2a - BROTLib: `ST_MotionConfig` (D1)
- New `BROTLib/BROTLib/Telescope/ST_MotionConfig.TcDUT`, **alt-az only** (comment says so; equatorial mounts do not
  use it). Fields with defaults equal to today's M literals: azimuth/derotator/elevation move velocity (10.0 / 10.0 /
  5.0); state-machine timeout 600 s; command timeout 2 min; idle-park / power-on / track timeout 12 h; stop and
  suspend-stop timeout 15 s; goto elevation window 2.0/89.5; track elevation window 5.0/89.5; horizon margin 0.2;
  azimuth/derotator track margin 1.0 and track time cap 14400 s.
- New pure `F_MotionConfigValid` (velocities > 0, windows ordered, timeouts > 0, margins >= 0) next to
  `F_HomeBelowHorizon` in `Telescope/Command/`, with `F_MotionConfigValid_Tests` in BROTLibTests, including one test
  that every default equals the old literal.
- Release: BROTLib minor (additive, e.g. 0.17.0). Consumers unaffected until they re-pin.

### Step 2b - IAG50cm: use BROTLib's `ST_TelescopeConfig`, re-pin (D1)
- Delete `IAG50cm/DUTs/ST_TelescopeConfig.TcDUT` and its `<Compile>` entry in `IAG50cm.plcproj`; pin the BROTLib
  release of 2a (needs >= 0.16.0 for the equatorial fields). `FB_TelescopeControl`/`MAIN` keep the unqualified
  name, which then resolves to `BROTLib.ST_TelescopeConfig` **[unverified: the BROTLib placeholder is not
  "qualified only", check in the build; otherwise write `BROTLib.ST_TelescopeConfig`]**.
- Tests: remove `ST_TelescopeConfig.TcDUT` from `Sync-Sources.ps1` (line 34) and `IAG50cmTests.plcproj`, re-pin the
  test project's BROTLib; `FB_IagRig`/`FB_IagRig_Tests` initialisers unchanged. IAG suites green.
- Retained/persistent data: the struct is a `VAR CONSTANT` in `MAIN`, so an online change has no retain mapping to
  carry over [verified for `MAIN`]; a full download is still advisable because the type identity changes.
- Release: IAG50cm patch. On site: IAG50cm smoke test (park, home, goto), since park reads `houranglePark`.
- This also resolves the struct clash the SM plan lists as a blocker of its step 5.3.

### Step 2c - MONETcommon: site constants from `ST_MotionConfig` (aim 4, behaviour-neutral)
- Pin the BROTLib release of 2a. M: new `VAR_INPUT stMotion : BROTLib.ST_MotionConfig;`, all listed literals
  replaced by fields, `VAR CONSTANT fHorizonMargin` removed. Internal timings stay constants (D7).
- **Deprecated aliases for one release (D2):** `fElevationMoveVelocity`, `fAzimuthTrackMargin`,
  `fDerotatorTrackMargin`, `fAzimuthTrackTimeCap`, `fDerotatorTrackTimeCap` stay as inputs; an alias set to a
  value other than its old default wins over `stMotion`, and its use is logged once. Removed in the next minor.
  MONETS `MAIN` (passes `fElevationMoveVelocity := 3.0`) builds unchanged.
- `F_MotionConfigValid` FALSE = software error 16#32 at startup, like 16#31 (`F_HomeBelowHorizon`). README error
  table, `MONETcommon.plcproj` pins.
- Tests: all existing suites green, unedited; two rig cases with a non-default config (goto window 85 refuses an
  86 deg target; command timeout 1 min fires at 1 min via the seam); one alias case (MONETS 3.0 still used).
- Consumers: none required; MONETS may move its 3.0 into `stMotion` in its next re-pin. Release: MONETcommon minor
  (additive). On site: smoke test (power-on, goto, slew, stop, park) on MONET/S, then MONET/N.
- **Status (2026-11-02): written on `feat/motion-config-input`, not compiled, not run (CI on push).** BROTLib pinned to
  0.17.0 in both projects; the test project also moves from BROTLib 0.15.1 / HalfBROT 0.10.1 to 0.17.0 / 0.10.2 (it
  lagged; with 0.15.1 the tree would resolve to a BROTLib without `ST_MotionConfig`). M: private `_MotionConfig()` at the
  start of the body builds `stMotionUsed` (stMotion, aliases applied, logged once through `MotionAliasEvent` per
  change of the set in use); every listed literal reads it. Invalid = 16#32 every scan after 16#31 **and the defaults
  stay in use** (added: otherwise a 0 s timeout or a 0 velocity would act before the error branch). `SuspendAutomatic`
  uses the `tSuspendStopTimeout` of the last automatic scan (defaults after a cold start in manual). The plan's "README
  error table" does not exist: README got a `stMotion` paragraph, the safe-state table a 16#32 row.
  Tests, all in existing suites (39 of 40 suites, step 3 still has a slot):
  `MonetTimeouts_Tests.Command_Timeout_From_stMotion_Fires_At_1_Minute` (ET 60 ms with seam 1000),
  `F_GotoTargetCheck_Tests.Goto_Window_From_stMotion_Refuses_The_Pole` (deviation: window max 30 refuses dec -89 at 31-33 deg with 16#16, instead of 85/86 deg, because an 86 deg
  RA/Dec target depends on the clock), `MonetSafeState_Tests.Invalid_Motion_Config_Is_Software_Error_32`,
  `FB_MonetRig_Tests.Monets_Elevation_Velocity_Alias_Is_Used` (MONETS 3.0). MONETS/MONETN `MAIN` read: only MONETS sets
  an alias (`fElevationMoveVelocity := 3.0`), nobody reads `fHorizonMargin`; no `MAIN` change. Version not bumped
  (separate release commit).

### Step 3 - Stage machines: enumerated stage, one error path, guard as a table (aim 2, neutral)
- 3a: `E_GotoStage`, `E_SlewStage`, `E_ParkStage`, `E_PowerOnStage` (values equal to today's numbers, so logs and
  ADS watch values stay readable); `nStage` becomes `eStage`; read-only properties `GotoStage` etc. for the rig.
  `_HomeTelescope` has no stage today: unchanged.
- 3b: one private `_RaiseSoftwareError(nId : UINT, sText : STRING)` replacing the four copies; latched until
  `Reset()` as today (M1).
- 3c: the guard prologues become one pure `F_StageAbort(eMachine, bReset, bError, bStop, bInterrupted, bCommand)`
  whose per-machine differences are an explicit table (goto/slew abort on error and on the flag drop; home does not
  reset a stage; park ignores `bError` for the emergency park; power-on aborts on stop). This is the SM plan's
  dropped 3.5 done as a policy table rather than one guard, so step 5.1 can take it as arbiter input.
  `F_StageAbort_Tests` with one row per machine and flag.
- Tests: all rig suites unedited and green; new pure suite. Consumers: none (private). Release: MONETcommon patch.
  On-site smoke test as step 2c.
- **Status (2026-11-02): written on `refactor/stage-machines` (from `develop` @ `359ce5b`, 0.12.0), one commit per
  sub-step, not compiled, not run (CI on push). Version not bumped.**
  - 3a: the four enums (values = old numbers; `E_GotoStage` idle/untrack/start_move/wait_moves/wait_standstill/done,
    `E_SlewStage` idle/untrack/start_move/wait_standstill/done, `E_ParkStage` prepare/untrack/start_move/wait_moves/
    disable_axes/secure/parked, `E_PowerOnStage` open_covers/home_axes/start_move_home/wait_home/complete), no explicit base
    type (INT, like every enum in the BROT repos; the base type syntax has no precedent here). Deviation: the stage is not
    in the method's `VAR_INST` any more but in the FB's `VAR` (`eGotoStage`, ...), because a property cannot read a
    method's `VAR_INST`; both live in the instance, so nothing changes at run time. `nLastStage` stays a `USINT` in
    `VAR_INST`; the progress log compares and prints `TO_USINT(eStage)`, so `GoTo progress: 25` is unchanged. The power-on's
    `nStage >= 10` is `ePowerOnStage <> open_covers` (same set). No rig suite read `nStage` (it was not reachable from
    outside), so no suite was touched.
  - 3b: `_RaiseSoftwareError(nId, sText)`, the same four assignments in the same order. `sText` goes to a new diagnostic
    `sSoftwareErrorText : STRING(80)` (ADS only, cleared in `_HandleReset` with the latch); the log line is unchanged.
    16#31/16#32 in the body set only the latch (no `bError`/`nErrorID` before the recompute) and were left as they are.
  - 3c: `F_StageAbort` + `E_StageMachine` (goto/slew/gohome/park/poweron, the `E_TCSCommand` names), the table is a
    `CASE` with five policy booleans per row. Exact rows: goto/slew `bReset OR bError OR bInterrupted OR NOT bGoto/bSlew`;
    home `bReset OR bError OR bInterrupted`; park `bReset OR bStop OR bInterrupted`; power-on `NOT bPower OR bReset OR
    bError OR bStop OR bInterrupted`, after the failure checks that may clear `bPower` (those checks, which use their own
    `bReset OR bStop OR bInterrupted`, stay in `_PowerOn`). Correction to the step text and to SM plan 3.5: the power-on
    also aborts on `bError` and on `NOT bPower`, not only on stop. `F_StageAbort_Tests` takes the 40th suite slot (40 of
    40 now): five per-machine tests (one assertion per input) and an exhaustive 32-combination check per machine against
    the old IF conditions.
  - Read while doing it, **not changed** (behaviour kept, for Tim / step 5.1): (1) `_HomeTelescope`'s timeout branch sets
    `_HomeTelescope := TRUE` without `RETURN`; the done check below overwrites it, and if all three `MoveDone` were TRUE in
    that scan it would also log "Telescope is homed." and copy the current position after raising 16#30 [unverified
    whether reachable]. (2) Home does not abort on `NOT bGoHome` and park not on `NOT bPark`; both are dispatched only with
    their flag set, so today this only matters if a later caller (the arbiter) calls them otherwise. (3) The park's
    `rtrigError` is clocked only when the prologue passes, so an error edge during a call that aborts (reset/stop/
    interrupt) or while no park runs is seen as an edge at the next park call. (4) The `#77` comment says "a
    reset/interrupt/stop call", but `bStop` is not in the goto/slew/home rows: a stop reaches them only through the
    arbitration (flags dropped, `bInterrupted`). (5) Not a prologue: stage 0 of the slew waits for `bStopped AND NOT
    bTracking`, stage 0 of the goto does not.
  - Unverified until CI: `TO_USINT` of a strict enum (HalfBROT uses `TO_INT(eManualHome)`), enum return type of the
    properties, several statements per line in the `CASE` of `F_StageAbort`. Safe-state doc: line numbers noted as moved,
    no cell changed. To run (Tim): `tcbuild.yml` on push, or the library install + `tctool MONETcommonTests.sln build run`
    of `MONETcommonTests/README.md`; on site the step-2c smoke test (power-on, goto, slew, stop, park) on MONET/S, then N.

### Step 4 - Split telemetry and targets (aim 2, neutral; D5)
- 4a telemetry: `_SendTelemetry` (67 publishes) into `FB_MonetTelescopeTelemetry` fed with a snapshot struct
  `ST_MonetTelemetry` filled by M once per publish. Test: a recording `I_Comm` mock (extend `FB_NullComm`) asserts
  topic list and values for a fixed snapshot, in an existing suite if the budget is tight; no topic renamed
  (pyobs/Influx depend on them).
- 4b targets (D5, option A): the eq2hor/pointing/turn/derotator block (M:322-466) into a private
  `_ComputeTargets()` method in M (the SM plan D3 hook name). Covered by the existing rig suites; no new FB. A
  separate `FB_MonetTargets` only if A proves insufficient (e.g. a target bug that the rig cannot reach), as its own
  later PR.
- Release: patch each; consumers unaffected. The command part (arbitration) is SM plan 5.1/5.2, scheduled there
  after this step.
- **Status 4a (2026-11-02): written on `refactor/telemetry-split` (from `develop` @ `644dce5`, 0.12.0, step 3 merged), not
  compiled, not run (CI on push). Version not bumped. 4b: see its status below.**
  - Catalogue of `_SendTelemetry` at `644dce5` (all `'telescope'`/`'dome'`, after `SUPER^._PublishTelemetry()`): 44 publishes per
    call (54 publish statements with the branches; the "67" above counted something else, probably SUPER's too), in this
    order: `TELESCOPE.READY_STATE` (LREAL `fReadyState`), `TELESCOPE.MOTION_STATE` (LREAL of the INT `nMotionState`),
    `TELESCOPE.INTERRUPTED_AUTO_CLEAR_COUNT` (INT of the UDINT counter); `WEATHER.REFRACTION.PRESSURE`/`TEMPERATURE`
    (`Publish` of `LREAL_TO_STRING`), `PRESSURE_LIVE` (`BOOL_TO_STRING(fRefractionPressure <> 0.0)`), `TEMPERATURE_LIVE`
    (`BOOL_TO_STRING(bRefractionTemperatureSet)`); `OBJECT.EQUATORIAL.RA_ICRS`/`DEC_ICRS`, `OBJECT.HORIZONTAL.ALT`/`AZ` (calc,
    unconditional), `POINTING.SLEWTIME`, `POINTING.OFFSETS.ALT`/`AZ`, `POSITION.LOCAL.SIDEREAL_TIME`/`JD`/`LATITUDE`/`LONGITUDE`/
    `HEIGHT`; per axis ALT then AZ: `POWER_STATE` (LREAL 1.0/0.0 on `Enable`), `REFERENCED` (LREAL 1.0/0.0 on `Calibrated`),
    `ERROR_STATE` (INT `ErrorID` if `Error`, else 0), `MOTION_STATE` (INT 8 busy+tracking, 1 busy, 0), then ALT `REALPOS`
    (current - offset), `CURRPOS`, `TARGETPOS` (`Position`), `CURRSPEED` (`Velocity`), `TARGETDISTANCE` (ABS), `OFFSET`; AZ the
    same but `CURRPOS` before `REALPOS` (current - `F_AzimuthOffsetOnAxis(offset, fElevationCurrent)`, M7);
    `POSITION.HORIZONTAL.ALT`/`AZ` (sky), `POSITION.EQUATORIAL.RA_ICRS`/`HA_ICRS`/`DEC_ICRS` (current). Rate in the body:
    `tonMQTTTimer` 500 ms while `bSlewing OR bTrack OR bTracking`, else 1000 ms, restarted after each publish (so PT plus one
    cycle); manual mode (`SuspendAutomatic`) sends only READY_STATE on the same timer.
  - Done: `ST_MonetTelemetry` (43 fields, each with the type of its source, so the implicit conversions are the old ones) and
    `FB_MonetTelescopeTelemetry` (inputs `comm`, `stTelemetry`; no state; the old statements with the sources replaced by
    snapshot fields, comments kept). `_SendTelemetry` keeps `SUPER^._PublishTelemetry()` first, fills the instance variable
    `stTelemetry` and calls `fbTelemetry` once; rate and call site unchanged. Checked by a script: the new block's statements
    with the snapshot fields replaced by the fill's sources are textually the old ones (only `fSlewtime`/`fSlewTime` case).
    `comm` is a per-call input, not `FB_Init`, because the telescope's `fbComm` is only set in its own `FB_Init`.
  - Test, in `FB_MonetRig_Tests` (40 of 40 suites): `FB_RecordingComm` (test project, extends `FB_NullComm`, records method/
    topic/value/address); `Telemetry_Publishes_The_Snapshot` (44 records, order, method, topic, value, all to telescope/dome)
    and `Telemetry_Axis_State_Branches` (the other side of every condition). The test drives the block directly, so it does not
    check `_SendTelemetry`'s fill; that is covered by the script comparison only.
  - Read while doing it, **not changed** (behaviour kept): (1) `_SendTelemetry` has no `fbComm = 0` guard (BROTLib's
    `_PublishTelemetry` returns on it, #7), so an unassigned comm would stop the task in the first MONET publish; the new block
    keeps that. (2) Fields published twice per record: `POSITION.LOCAL.*` (5, identical, SUPER and here),
    `POSITION.INSTRUMENTAL.AZ/ALT.OFFSET` (FB_AltAzTelescopeControl and here, identical), `OBJECT.HORIZONTAL.ALT/AZ` (differ
    during a slew, the comment says the later one is what is on the wire). (3) `TELESCOPE.READY_STATE` is also sent by the
    `fbStartup.PublishDue` path before the timer, so it can go out twice in one scan. (4) Type mix on the wire:
    `TELESCOPE.MOTION_STATE` is LREAL, the per-axis `MOTION_STATE`/`ERROR_STATE` are INT, `POWER_STATE`/`REFERENCED` LREAL, the
    weather values strings via `Publish`. (5) No derotator telemetry here. (6) `_SendTelemetry` is a public method (no
    `PRIVATE`), kept so; nothing outside calls it (MONETS/MONETN/HalfBROT grepped).
  - Unverified until CI: the overrides of `FB_RecordingComm` (signatures copied from `FB_NullComm`), passing the FB instance
    `rec` to the `I_Comm` input, the `LREAL_TO_STRING` texts `'1000.5'`/`'-12.25'` asserted literally. To run (Tim):
    `tcbuild.yml` on push, or the library install + `tctool MONETcommonTests.sln build run` of `MONETcommonTests/README.md`;
    on site: telemetry in Influx/pyobs unchanged (same fields and rate) on MONET/S, then N.
- **Status 4b (2026-11-02): written on `refactor/compute-targets` (from `develop` @ `87197a0`, 0.12.0, steps 3 and 4a merged),
  not compiled, not run (CI on push). Version not bumped.**
  - The block at `87197a0` (M:367-497, after the #155 brake-on-error block, before the horizon guard): refraction weather
    (`F_RefractionWeather` or the zero estimate, the two rejected-value `R_TRIG`/`ADSLOGSTR`), `eq2hor` (JD `fJd + fTimeOffset`),
    `fbPointing` and the offsets, `F_TargetBelowHorizonWarning` (#94), `F_DerotatorPosition2` + `fDerotatorOffset`, the three
    velocities, `F_AxisTurn` of azimuth and derotator (#2) and `bAzimuthNoTurn`/`bDerotatorNoTurn`, the current axis positions,
    `fbPointingInverse`, both `hor2eq`. The plan's "M:322-466" was the same block at `adc1013` (approximate there too). The
    weather is included because it feeds eq2hor and both hor2eq (its own comment: one weather for commanded and reported); the
    horizon guard is not (SM plan D3 has it as its own hook `_HorizonGuard`).
  - Done: `METHOD PRIVATE _ComputeTargets` (no return value, no inputs, no variables of its own), folder `internal methods`, the
    131 lines moved verbatim (checked by `diff` against the old body lines); the body calls `_ComputeTargets();` at the same
    place. Every variable read or written stays in the FB's `VAR` (body, stage machines, properties and telemetry read them);
    `eq2hor`/`hor2eq` are the instances of `FB_AltAzTelescopeControl`. No `LineIds` entry added (the siblings `_ReadAxes`,
    `_StopTx*` have none either; TwinCAT regenerates them).
  - Test: none added. The rig suites run the body every scan and so the method (goto/track/slew/stop suites); the pure parts
    have their own suites (`F_AxisTurn_Tests`, `F_TargetBelowHorizonWarning_Tests`, `F_AzimuthOffsetOnAxis_Tests`,
    `F_GotoTargetCheck_Tests`). A case could not check anything specific to the move (a private method is not callable from a
    test, and the order is the old one textually). Gap that exists before and after: no rig test asserts the computed target or
    reported position values, nor the body-level no-turn refusal (`bAzimuthNoTurn` path) or the live-weather path.
  - Read while doing it, **not changed** (behaviour kept): (1) `eq2hor` uses `fJd + fTimeOffset`, both `hor2eq` use `fJd`, so with a
    non-zero `fTimeOffset` the reported RA/Dec are not the inverse of the commanded ones; BROTLib's `FB_RaDecCoordinates` adds
    the offset to all its conversions [unverified whether `fTimeOffset` is ever non-zero on MONET]. (2) The second `hor2eq`
    (sky position) has no `ha =>`, so there is no sky hour angle (none is published). (3) `fbPointing`/`fbPointingInverse` are
    `REFERENCE TO` and called without `__ISVALIDREF` (like the axes before #156); `SuspendAutomatic` does not call the block, so
    in manual mode `fElevationCurrent`/`fAzimuthCurrent` (the `Altitude`/`Azimuth` properties) and the RA/Dec stay at the last
    automatic scan. (4) The block runs also while `fbTime.bValid` is FALSE (with the last valid `fJd`); only the motion is
    refused then (`bStop`). (5) The velocities are computed from the azimuth before `F_AxisTurn` moves it by a turn
    (turn-invariant, so harmless).
  - Unverified until CI: that the method compiles as the body did (same scope, no locals). To run (Tim): `tcbuild.yml` on push,
    or the library install + `tctool MONETcommonTests.sln build run` of `MONETcommonTests/README.md`; on site the step-2c smoke
    test (goto, track: commanded and reported RA/Dec as before) on MONET/S, then N.

### Step 4c - Client check before read-only inputs (D3, investigation, no code)
- Consumers of the command interface: **pybrotlib** (Python library) and its users **pyobs-brot** and **brotgui**.
  None is checked out on this machine [verified: not under `Documents/`].
- Read pybrotlib's command path and record in this plan: does it send commands as MQTT `SET` (PLC side calls the
  methods, [verified]) or also by ADS writes of `bGoto`/`bTrack`/`bPark`/...; does any path hold a flag TRUE and rely
  on re-triggering (a second `goto` while `bGoto` is still TRUE), or re-send on a timer; does it read the flags back
  as status instead of READY_STATE / `TELESCOPE.*` telemetry. Same for brotgui's direct ADS use, if any, and for the
  TwinCAT Visu and both `MAIN`s (direct writes like `HydraulicsControl.bOpenBrake`).
- Outcome: either step 5 as written (edge-triggered inputs plus methods) or "methods only" if no client writes flags.
  Release: none.

### Step 5 - Read-only command inputs (aim 3, **breaking**, after SM 5.1 and 4c)
- M stops writing `bPower`, `bGoHome`, `bPark`, `bTrack`, `bGoto`, `bSlew`, `bStop`, `bReset`. Internal request
  latches (`stReq` in M, or in the arbiter if 5.1 has landed) are set by the methods (`Init`, `Park`, `Stop`, ...)
  and by the **rising edge** of the input; status goes to outputs (`bGotoActive`, ...). The direct-write gate
  (#83 `b*Checked`) becomes the edge detector.
- The inputs are declared in BROTLib `FB_BaseTelescopeControl`; their doc comments change to "edge-triggered" in a
  BROTLib PR, ideally with SM 5.4. HalfBROT's own-input writes: open HalfBROT issues (covers, hydraulics, focus,
  axis `bReset`) and MONETS `MAIN`'s `bOpenBrake` level write, not fixed here.
- Breaks: whatever 4c finds writing the flags or reading them back (pybrotlib / pyobs-brot / brotgui, Visu).
  `FB_MonetMqttWatchdog`/`FB_MonetCabinetControl` use `iTelescope.Park()`, unaffected.
- Tests: rig suites that set flags directly are edited once (one commit, reviewed separately), new cases: a held
  flag starts the command once; the flag falling does not abort; method and edge in the same scan count once.
- Release: MONETcommon minor (breaking pre-1.0), BROTLib comment update, MONETS/MONETN re-pin, client release if
  4c requires one. Full on-site check (the SM plan's step 6 list, MONETcommon#63/#64) on MONET/S, then MONET/N, with
  pyobs-brot and brotgui driving it.
- **Status (2026-11-03): written on `feat/read-only-command-inputs` (from `develop` @ `265bc94`, 0.14.1, pins BROTLib 0.17.0 /
  HalfBROT 0.11.0), not compiled, not run (CI on push). Version not bumped.** Three commits: code, the mechanical edit of the rig
  suites (reviewable alone), the new cases; docs separately. Decided by 4c / D3: edge-triggered inputs plus methods.
  - Latches: the read-only outputs `bPowerActive`, `bGoHomeActive`, `bParkActive`, `bTrackActive`, `bGotoActive`, `bSlewActive`,
    `bStopActive`, `bResetActive` (no `stReq` struct: one variable per request is both the latch and the status output, so no copy
    at the end of the scan can lag behind a method call; no new DUT). Every place that read or cleared an input now reads or
    clears its latch, statement for statement (a scripted rename of the code outside comments and strings, checked by diff): stage
    machines and `F_StageAbort` calls, stop branch and stop transaction release (`bParkAfterStop` -> `bParkActive`), `_HandleReset`
    and the reset hold, the #118 `bPark` kept in manual (`SuspendAutomatic`), the horizon guard, the idle park, the error reactions
    of step 7 (hydraulics/power ramps and the timeout/focus/safety parks set `bStopActive` or call `Park()`), `_CommandState`,
    the events, `F_TargetBelowHorizonWarning`/`F_AxisTurn` (`bHold`), `F_WakeShakeWindow`, READY_STATE `Parking`. The deprecated
    `stMotion` aliases are untouched.
  - Edges: `b*Checked` (the #83 gate) is the edge memory: the input of the last scan (snapshot at the end of the body), set TRUE by
    the method of that command (before its accept decision, so a rejected call is not decided twice either). The rising edge of
    `bStop`/`bPark`/`bReset`/`bPower` calls `Stop()`/`Park()`/`Reset()`/`Init()` at the start of the body (where a direct write
    acted before; order stop, park, reset, power, so `Init()` refuses a power-on together with a stop or park, #87); the move
    inputs are decided at the old gate with `_AcceptMove(cmd, no_command)` like the methods and set the latch when accepted.
    `Track()`/`TrackRaDec()` mark `bGoto` and `bTrack` as decided. "Same scan" = from one body call to the next; a method called
    after the body and an edge in the next scan are one command, a method after the body and an edge already seen by that body are
    two.
  - Behaviour changes (all for direct writers of the inputs; methods/MQTT unchanged): (1) the input is never cleared, a new
    command needs FALSE then TRUE; (2) the input falling does nothing (`bTrack` FALSE used to end a track by the arbitration, a
    dropped `bGoto` aborted the goto stage machine); (3) a written `bPark` during a stop is queued like `Park()` (was dropped by
    the stop branch), a written `bPower` passes `Init()`'s refusals (was only `BeginScan`), a written move while the clock is
    invalid is rejected without a stop (was: stop path; `Clock_Invalid_Slew_Takes_The_Stop_Path` -> `Clock_Invalid_Slew_Input_Is_
    Rejected`, same Id); (4) an edge while the same command runs re-commands (replace) like the method; (5) BROTLib's
    `_PublishTelemetry` still selects `OBJECT.EQUATORIAL.RA/DEC` (and `OBJECT.HORIZONTAL.*`, which `fbTelemetry` overwrites
    anyway) by the input `bSlew`; `_SendTelemetry` republishes the two RA/Dec fields from `bSlewActive` where they differ, so the
    wire keeps the 0.14 values (during a slew `fRightAscensionCalc`, which MONET never computes: 0.0, unchanged oddity).
  - Tests (41 of 41 suites, no new suite): the rig suites read `rig.fbTel.b*Active` instead of the inputs (one commit, ~210 reads,
    no assertion changed except the clock case above); `FB_MonetRig_Tests` (10 tests) gets `Held_Slew_Input_Starts_The_Slew_Once`,
    `Slew_Input_Falling_Does_Not_Abort_The_Slew`, `Slew_Method_And_Input_Edge_In_One_Scan_Count_Once`,
    `Repeated_Track_Retargets_And_Second_Stop_Works`; rig methods `SetSlewInput`, `SetSlewTargetInput`.
  - Not done here: the BROTLib doc comments of the eight inputs in `FB_BaseTelescopeControl` ("edge-triggered", separate BROTLib
    PR), BROTLib `_PublishTelemetry` reading a status instead of `bSlew`; the MONETS/MONETN `Telescope.TcVIS` `TOGGLE` switches
    (TAP or method helpers, 4c item 1) and BROTweb (4c item 2) must change in the same release; HalfBROT `FB_PendantControl`
    level writes (SM 5.4). Comments in M written before step 5 still name the requests by the inputs (noted at the declaration).
  - Unverified until CI: the build (outputs written in methods, `Init()`/`Stop()`/`Reset()` called from the body, the read of
    `nCommandRejectCount`/`fActualPosition` in the new tests), the test timing (second stop at scan 2 of the first). To run (Tim):
    `tcbuild.yml` on push, or the library install + `tctool MONETcommonTests.sln build run`; on site: the full check above with the
    Visu changed, plus an ADS write of each input held TRUE (one command) and released (nothing happens).

### Step 6 - Coupling (aim 5, small breaking cleanup; D6)
- Pendant: drop the unused `fbBrake`/`fbHydraulics` inputs (one-line removal in both `MAIN`s); covers via
  `I_MirrorCovers`, focus via `I_Focus` where the pendant only calls interface members. Coordinate with #177/#168
  (pendant in automatic, mode state machine), which rewrite the same block.
- Axes stay `REFERENCE TO` the concrete HalfBROT blocks (D6); `I_Axis` is not widened. README gets the rationale:
  M needs `fMinPosition`/`fMaxPosition`, `fActualPosition` and the `fbAxis` NC status that `I_Axis` does not carry;
  BROTLib cannot reference HalfBROT; the mock rig subclasses the concrete blocks, so tests do not need an interface.
- Remove MONETcommon's duplicate pure units now in BROTLib (`F_CommandAcceptance`, `F_ErrorParkRequest`, ...) and
  call the BROTLib ones, deleting the duplicated test suites (frees suite slots). Diff the copies first; neutral only
  if identical.
- Tests: rig suites green. Release: minor (pendant signature), MONETS/MONETN `MAIN` edit + re-pin.
- **Status (2026-11-03): written on `refactor/coupling-cleanup` (from `develop` @ `c086ac6`), not compiled, not run (CI on push).
  Version not bumped.**
  - Pendant (breaking): `fbBrake`/`fbHydraulics` removed (declared, never read). `fbFocus` is `BROTLib.I_Focus` (the pendant only
    uses `I_BaseAxis` members: `Enable`, `Ready`, `InPosLimit`, `InNegLimit`, `MovePos`, `MoveNeg`, `Reset()`, `Error`; HalfBROT's
    `FB_FocusControl` implements it, so `fbFocus := FocusControl` in `MAIN` is unchanged; the guards are `fbFocus <> 0`). The
    covers stay `REFERENCE TO HalfBROT.FB_CoverControl`: the pendant calls `Stop()` and reads `bError`, `I_MirrorCovers` has
    neither (it has `Error`). MONETS and MONETN `MAIN` (develop and main): delete the two lines `fbBrake := HydraulicsControl,`
    and `fbHydraulics := HydraulicsControl,` of the `PendantControl(...)` call (MONETS lines 228-229, MONETN 238-239), with the
    re-pin. #177 (gate the pendant on the mode) and #168 (mode state machine) touch the same block and call: they add
    `eTelescopeMode` and gating in the CASE branches, this changes only the declarations and the focus guards, so whoever comes
    second rebases a few lines; the `MAIN` edit of #177 (`eTelescopeMode := eTelescopeMode`) goes into the same call.
  - Axes (D6): unchanged; the README has the rationale (limits, `fActualPosition`, `fbAxis` NC status; BROTLib cannot reference
    HalfBROT; the rig subclasses the concrete blocks).
  - Duplicates, diffed against BROTLib 0.17.0 `Telescope/Command/` (CR/BOM stripped): identical except the object Id:
    `F_ErrorParkRequest`, `F_GotoTargetCheck`, `F_HomeBelowHorizon`, `F_TargetBelowHorizonWarning`, `F_TCSCommandText`,
    `E_CommandDecision`. Same logic, other signature: `F_ResetMoveAbort`, `F_SuspendStop` (BROTLib: one `bAnyAxisMoving` instead
    of `bAzMoving`/`bElMoving`/`bDrMoving`, OR-ed inside). Different (kept): `F_CommandAcceptance` (MONETcommon has the #72 row
    `bInitRequired` -> `init_required`), `F_CommandRejectText` (its text), `E_CommandRejectReason` (`init_required`),
    `ST_CommandState` (`bInitRequired`); `E_CommandDecision` stays with them. M calls `BROTLib.F_*` for the seven, the OR of the
    three axes is at the two call sites; the seven local units are removed.
  - Tests: the six suites of the removed functions (`F_ErrorParkRequest_Tests`, `F_GotoTargetCheck_Tests`,
    `F_HomeBelowHorizon_Tests`, `F_TargetBelowHorizonWarning_Tests`, `F_ResetMoveAbort_Tests`, `F_SuspendStop_Tests`) are the same
    cases as in BROTLibTests and are deleted: 35 of 41 suites. The rig case `Goto_Window_From_stMotion_Refuses_The_Pole` (tests M,
    not the function) moved to `FB_MonetRig_Tests` (10 tests, 55 assert lines). `F_CommandAcceptance_Tests` stays.
  - Unverified until CI: the build (`BROTLib.I_Focus` as `VAR_INPUT`, an `FB_FocusMock` `VAR_IN_OUT` passed to it in the pendant
    tests, `BROTLib.F_*` qualified calls), the rig suites green. On site: the pendant check (#69 list), focus on selector 5.

### Step 7 - Safe-state fixes (aim 6, one PR per row of the step-0 table)
- Each row where today differs from the target becomes a PR: flip the `_Current` test, change M (expected first
  candidates: software errors and hydraulics/cover errors park; hydraulics off at the end of every emergency park).
  Safety-relevant: on-site check per PR, E-stop at hand. Release: minor per behaviour change, MONETS first.
- Order (2026-11-02, after D8/D9; details per PR in the design doc, "Step 7: proposed order"): 1 `bCloseRoofRequest`
  output in M (signal only); 2 roof close in MONETS then MONETN `MAIN` behind the enable switch; 3 hydraulics error
  reaction; 4 cover error hangs (park, power-on); 5 software-error timeouts ramp and park; 6 E-stop/safety error park
  and hydraulics off; 7 power failure; 8 focus error drive to park; 9 invalid clock (optional). The emergency-park
  hydraulics-off candidate is already true (design doc, findings).
- **Status PR 1 (2026-11-02): written on `feat/close-roof-request` (from `develop` @ `1c157aa`), not compiled, not run (CI on
  push). Version not bumped.**
  - M: `VAR_OUTPUT bCloseRoofRequest : BOOL`, derived right after `bError` from `bCloseRoofCause := fbElevation.Error OR
    fbAzimuth.bError OR fbDerotator.bError OR fbHydraulics.Error OR bEstopTriggered OR bSafetyError OR NOT bMainReady`; set on
    its rising edge (`bCloseRoofCauseLast`), cleared by `bReset` while no cause is active (clear before set). Nothing reads it;
    no existing output changed. Evaluated in every automatic scan (no early return in the body, so also in the error branch and
    in E-stop); in manual (`SuspendAutomatic`) it keeps its last value, the `MAIN` of PR 2 gates on automatic anyway.
  - Settled here (8.5): name `bCloseRoofRequest`; latch on the rising edge, cleared by `bReset` once no cause is active. The
    clear is next to the set, not in `_HandleReset` (deviation from the design doc's wording; the cause is computed after it).
    Because `bReset` is held while `bError`, one reset clears an axis error and, in the next scan, the request; a hydraulics
    error needs its HalfBROT reset first, then a second `reset=1`. The `MAIN` switch name (`bRoofCloseOnErrorEnable`) is PR 2.
  - Chosen for 8.7 (PLC start), easy to change: **no arming**. `bCloseRoofCauseLast` starts FALSE, so an E-stop or a missing
    phase at the start sets the request (conservative: closing is the safe direction; needs `reset=1` after the cause is gone).
    Arming instead = initialise `bCloseRoofCauseLast` with TRUE (one line). Check on site before PR 2 whether the restart
    case is a nuisance.
  - Tests (`MonetSafeState_Tests`, 40 of 40 suites, 16 tests): `Roof_Request_On_Axis_Errors` (El, Az, Der in turn; one reset
    clears error and request), `Roof_Request_On_Hydraulics_Error`, `Roof_Request_On_Estop`, `Roof_Request_On_Safety_Error`,
    `Roof_Request_On_Power_Failure` (each: set, kept by a reset with the cause active, latched after the cause is gone, cleared
    by a reset without a cause), `No_Roof_Request_On_Focus_Error`; one extra assert each in
    `Goto_Times_Out_Without_Park_Current` (16#10) and `Invalid_Motion_Config_Is_Software_Error_32` (16#32): no request. No clock
    and no power-on needed (the request is computed from the first scan; the rig is parked at the start, so an axis error does
    not start the emergency park). New rig method `FB_MonetRig.InjectDerotatorError`.
  - Unverified until CI: the build (new output and variables, the rig method), the test timing (3 scans per step), and that the
    HalfBROT `Reset()` of the axis mocks clears the injected error within the reset scan as assumed. Not covered by a test: the
    manual-mode behaviour (output held) and the PLC-start case. To run (Tim): `tcbuild.yml` on push, or the library install +
    `tctool MONETcommonTests.sln build run` of `MONETcommonTests/README.md`; on site (design doc, order item 1): ADS watch of
    `bCloseRoofRequest` on an E-stop and a hydraulics stop, and after a PLC restart with the E-stop pressed.
- **Status PR 3 (2026-11-02): written on `feat/hydraulics-error-reaction` (from `develop` @ `7d667d8`, 0.13.0), not compiled, not run
  (CI on push). Version not bumped.** Row "Hydraulics error" of the design doc (now implemented, M lines of that branch).
  - Entry: `fbHydraulics.Error` goes into `bHwError` (`bHydraulicsError`), so it reuses everything an axis error has: `bError`,
    READY_STATE -1.0, the error branch, the #86 emergency park on the rising edge of `bHwError` (`F_ErrorParkRequest` unchanged)
    and its park in place (`_ParkTelescope` stage `prepare` with `bError`: brake closed, axes disabled, covers closed; `secure`:
    hydraulics off; `bIsParked`, `bParkedInPlace`). No parallel park path.
  - Ramp first (D8): new `bHydraulicsCanRamp` = brake open AND `OilPressureOK` AND no E-stop AND no safety error AND no other
    (axis/focus/software) error. On the rising edge of the hydraulics error, with `bHydraulicsCanRamp` and an enabled axis moving
    (InMotion/InRamp/Homing or tracking), M sets `bStop` (the existing stop transaction, #96) and `bHydraulicsRamp`; the error
    branch then skips only its three `Enable := FALSE` while `bHydraulicsRamp`. The emergency park of the same scan sees `bStop`
    and is queued (`Park()` -> `bParkAfterStop`), the stop release starts it, and it parks in place (bError). The ramp ends when
    `bStop` has been released (settled), the drives cannot ramp any more (brake closed / pressure gone / E-stop / safety / other
    error), the hydraulics error is gone, or on the stop timeout (`tStopTimeout`, 15 s; the lock takes over). Without
    `bHydraulicsCanRamp`, or at standstill, no ramp: disabled in the scan of the error, park in place at once (= axis error).
  - Reset: the hydraulics error is latched in HalfBROT and reset there (MAIN reset button: `HydraulicsControl.Reset()` and, in
    automatic, `TelescopeControl.Reset()`). M does not reset it; `bError` drops by itself once it is gone. The `bReset` hold at
    the end of the body is now written as the errors M resets (axes, focus, software errors), i.e. the old `bError`; a held reset
    on the hydraulics error would abort the park every scan (`F_StageAbort` park row) and could keep the covers open while an
    unresettable fault (oil low) persists. So a `reset=1` during a hydraulics error stays a one-scan pulse, as before PR 3, and
    `Roof_Request_On_Hydraulics_Error` (PR 1) holds unedited.
  - Exemptions, decided here: parked telescope -> no park (as #86; the park would do nothing). E-stop -> no park and no ramp (as
    #86: drives cut, people on site, covers must not move on release; the E-stop row is PR 6). First power-on (#113) -> **not
    exempt**: the hydraulics error is added to `bOtherHwError` of `F_WakeShakeErrorExempt`, so it parks even together with an
    Az/El fault (conservative: covers close). The Wake&Shake window (`F_WakeShakeWindow`) is unchanged (a hydraulics error in it
    closes the brake, #155; with the elevation error already latched there is no new edge, so no park, as before).
  - `SuspendAutomatic`: `F_SuspendStop` gets `bError AND NOT bHydraulicsRamp`, so a switch to manual during the ramp goes on with
    that stop instead of leaving a tracking/coasting axis unstopped. The edge memory `bHydraulicsErrorLast` is not set in manual:
    a hydraulics error that appears in manual and is still there on the return ramps then if an axis moves (no park:
    `bHwErrorLast` is set on the first manual scan, #86).
  - New log `HydraulicsRampEvent` (WARN, once per ramp). `nErrorID` stays 0 for a hydraulics error alone (no ID invented).
  - Tests (40 of 40 suites; `MonetSafeState_Tests` 17 tests, `MonetErrorPark_Tests` 9): flipped
    `Hydraulics_Error_While_Tracking_Keeps_Tracking_Current` -> `Hydraulics_Error_While_Tracking_Ramps_Then_Parks_In_Place` (same
    method Id); new `Hydraulics_Pressure_Loss_While_Tracking_Parks_In_Place_Without_Ramp`, `MonetErrorPark_Tests.Hydraulics_Error_On_
    A_Parked_Telescope_Does_Not_Park`, `.Hydraulics_Error_During_Estop_Does_Not_Park`, `.Hydraulics_Error_Park_Recovers_With_
    Hydraulics_Reset_And_Reset` (standstill: no ramp; reset not held; hydraulics reset clears `bError`; roof request until the next
    reset; `power=1` online again). Other classes: their suites are unedited (axis/focus park, E-stop, timeouts, roof request,
    brake release, suspend).
  - Unverified until CI: the build; the test timing (ramp settles in about 10 scans on the mocks); that the second `Init()` after the
    in-place park goes online on the rig (no earlier test re-inits after a park). Not modelled: how fast the real pump stop drops
    `BrakeOpen` (HalfBROT drops the brake output as soon as `bPumpRunning` falls, so on site the ramp may end within a few scans and
    the lock takes over), `OilPressureOK` debounce, NC behaviour of a tracking axis (`InMotion` while tracking). To run (Tim):
    `tcbuild.yml` on push, or the library install + `tctool MONETcommonTests.sln build run`; on site (design doc, order item 3).
- **Status PR 5 (2026-11-02): written on `feat/timeout-error-reaction` (from `develop` @ `40517f9`, PR 3 merged), not compiled, not
  run (CI on push). Version not bumped.** Row "Software error: timeout" of the design doc (now implemented, M lines of that branch).
  - Entry: `bTimeoutError` = `bSoftwareError` with the latched ID 16#10/16#30/16#40. The timeouts are raised inside the stage
    methods (dispatch), so the reaction runs on the rising edge in the next scan, next to the PR 3 ramp block and before the error
    branch; the axes are still enabled and may move then. Same mechanism as PR 3: `bStop` (stop transaction #96) if an enabled axis
    moves (also homing or tracking) and `Park()` (queued behind the stop as `bParkAfterStop`, or at once at standstill). No new
    park path: `F_ErrorParkRequest` (hardware errors) is not used, its comment still says software errors do not park (now true for
    16#15/16#16/16#31/16#32 only; not edited, the function is unchanged).
  - Drive to park (D4 "park if possible"): new `bTimeoutPark` is TRUE from the edge until the park ends; while it is set the error
    branch skips its three `Enable := FALSE` (next to `bHydraulicsRamp`) and the park's prepare stage treats the software error like
    no error (`(bError AND NOT bTimeoutPark) OR NOT bAxesCanMove`), so it drives to the park position when `F_ParkAxesCanMove`, else
    parks in place. The park's own end (disable at standstill, #121 brake close, covers close, hydraulics off) is unchanged. The
    dispatch already lets a park run under `bError` (M2 bypass of the ready gate).
  - Exemptions, decided here: parked (nothing to do, as #86); E-stop, safety error, any hardware error incl. hydraulics
    (`bTimeoutCanMove` FALSE: those keep their own reaction, the error branch disables at once; the E-stop/safety rows are PR 6). The
    reaction ends (`bTimeoutPark` FALSE) when parked, when the error is reset, when one of those appears, when no stop or park is
    pending any more (an operator `stop=1` cancels the park: the axes ramp and are then disabled) or on the stop timeout (the lock
    takes over, as PR 3). If it ends while the park still drives (stage untrack/start_move/wait_moves) with the error still there,
    the park stage is set back to prepare so it finishes in place (`bParkedInPlace` right, #158); a park in disable_axes/secure goes
    on. `bMainReady` (power failure) is not an exemption here (row PR 7).
  - Not included: 16#15/16#16 (goto target window, can be raised during a goto) and 16#31/16#32 (config, nothing moves); their rows
    are unchanged. Adding 16#15/16#16 would be one more ID in `bTimeoutError` if wanted.
  - Reset: unchanged, `reset=1` clears the latch (`_HandleReset`), `bReset` is held while the software error is there (as before).
    A reset during the park drive clears the error, `bTimeoutPark` ends, and the park goes on as a normal park (`bPark` kept, #85).
    No roof close request (D9; `bCloseRoofCause` does not read software errors, unchanged).
  - `SuspendAutomatic`: `F_SuspendStop` gets `bError AND NOT bHydraulicsRamp AND NOT bTimeoutPark`, so a switch to manual during the
    ramp or the park drive stops the axes once instead of leaving a commanded move running; `bPark` is kept (#118) and resumes in
    automatic (with `bTimeoutPark` still set it drives, if the axes can still move). `bTimeoutErrorLast` is not set in manual (no
    timeout can be raised there).
  - New log `TimeoutParkEvent` (WARN, once per reaction). `nErrorID` unchanged (16#10/16#30/16#40).
  - Tests (40 of 40 suites; `MonetSafeState_Tests` 17 tests, `MonetTimeouts_Tests` 5): flipped
    `Goto_Times_Out_Without_Park_Current` -> `Goto_Times_Out_Ramps_Then_Drives_To_Park`, `Home_Times_Out_Without_Park_Current` ->
    `Home_Times_Out_Ramps_Then_Drives_To_Park` (same method Ids): bStop in the scan after the timeout with the axes enabled, MC_Stop
    before any disable, then the park position (el 60) not in place, axes disabled, brake and covers closed, hydraulics off, error
    latched, no roof request. New `MonetTimeouts_Tests.Slew_Timeout_Ramps_Parks_And_Recovers_With_Reset` (16#40, same checks, then
    `reset=1` clears the error without a roof request and `power=1` goes online). Other classes: their suites are unedited
    (`Slew_Times_Out_After_600_Seconds` holds unedited; the only other test text change is the name in a comment of
    `No_Roof_Request_On_Focus_Error`).
  - Unverified until CI: the build (enum comparisons on `eParkStage` in the body, the new variables); the test timing (the park move
    is set back to 60 mock scans once `bPark` is seen, after the stop has aborted the slow move); that `Init()` after the driven park
    goes online on the rig. Not modelled: NC ramp time of a real stop, whether a timed-out move is stuck for a reason that also
    stops the park drive (then the park waits in wait_moves; the 2 min park command timeout only logs), drive behaviour of the park
    after a long slew. Not covered by a test: the stop timeout end, a hardware error during the park drive (restart in place),
    `stop=1` during the park drive, manual mode during the reaction. To run (Tim): `tcbuild.yml` on push, or the library install +
    `tctool MONETcommonTests.sln build run`; on site (design doc, order item 5).
- **Status PR 4 (2026-11-02): written on `feat/cover-error-reaction` (from `develop` @ `40517f9`, PR 3 merged), not compiled, not run
  (CI on push). Version not bumped.** Row "Cover error" of the design doc (now implemented, M lines of that branch).
  - Not a telescope error: `fbCovers.Error` stays out of `bError`/`bHwError` (the drives are controllable, the target is "ramp; drive to
    park", which the error branch and the emergency park in place would not do) and out of `bCloseRoofRequest` (D9).
  - Online (tracking, slewing, at rest): the rising edge on a telescope that is not parked and not in E-stop calls `Park()`
    (`F_ErrorParkRequest` reused with its own edge memory `bCoverErrorLast`; set TRUE on the first manual scan like `bHwErrorLast`,
    #86). The park routing gives the ramp: a goto/slew/home is stopped first (#84), a track is dropped (stage `untrack`), then the park
    drives to the park position if `F_ParkAxesCanMove`, else in place; during a stop it is queued (`bParkAfterStop`). Decided here as
    the row's target ("drive to park"), not only the two hangs of the order item. WARN `CoverErrorParkEvent`.
  - Park: stage `secure` ends without `Closed` when `fbCovers.Error`, the brake is closed and the stage was not sent back to
    `disable_axes` in that scan (axes disabled, hydraulics switched off above); a flag carries it to stage `parked`, which sets
    `bIsParked`, clears `bPark` and raises software error 16#21 via `_RaiseSoftwareError` (raised there, not in `secure`: `bError` in
    the next scan would reset the park stage to `prepare` through `rtrigError` and re-run it in place). Applies to every park, also the
    emergency park of an axis/focus/hydraulics error whose covers are in error.
  - `bIsParked` decided: TRUE = secured as far as possible (axes disabled, brake closed, hydraulics off), the open covers are reported by
    16#21. Not FALSE, because then every repeated `Park()` (watchdog and cabinet every scan, idle timeout) would re-run the park in place
    while the cover error lasts, and `F_ErrorParkRequest` would park again on the next axis error.
  - Error ID, no ID was reserved in the plan or the design doc. Candidates (16#xx scheme: 16#1x goto, 16#2x park, 16#3x home/config,
    16#4x slew; IAG50cm uses 16#20 for its park timeout): 16#20 (park; kept free for a park timeout as in IAG50cm), **16#21** (park:
    ended without closed covers; chosen, next free in the park range), 16#50 (a new "covers" range). Logged as `Internal command error,
    ID: 33` (the existing `SoftwareErrorEvent` prints decimal), text in `sSoftwareErrorText`.
  - Power-on: `_PowerOn` fails on `fbCovers.Error` after the `bError` check (existing power-on failure path: `bPowerFailure`, READY_STATE
    -1.0, log `power failure: cover error (reset the covers, then power=1)`), before the cover command of stage 0, so the telescope stays
    parked; no software error ID (as the other power-on failures). A cover error that appears during a power-on starts the park, which
    aborts the power-on (#87).
  - Reset: the cover error is latched in HalfBROT; the `MAIN` reset button calls `CoverControl.Reset()` and, in automatic,
    `TelescopeControl.Reset()` (clears 16#21). Then `power=1` opens the covers and goes online, `park=1` closes them. `reset=1` alone
    clears 16#21; `power=1` keeps failing while the cover error is there. `bReset` is held on 16#21 (a software error) for one scan
    as for every software error.
  - Behaviour changes (all only while `fbCovers.Error` is TRUE; with healthy covers nothing changes): (1) a new cover error on an
    unparked telescope outside E-stop parks it; (2) a park with covers in error ends (`bIsParked`, `bPark` FALSE, READY_STATE -1.0
    via 16#21) instead of hanging at 0.3; (3) a power-on with covers in error fails at once (-1.0, stays parked) instead of waiting
    12 h at 0.7 with `bIsParked` FALSE; (4) an emergency park whose covers are in error also ends with 16#21 in addition to its
    hardware error; (5) during the first power-on after a PLC restart (#113 Wake&Shake exemption, covers open) a cover error now parks
    in place (the window closes on the park). Unchanged: `bError` of the cover error alone, `bCloseRoofRequest`, the brake rule,
    the E-stop, the other error classes.
  - Tests (40 of 40 suites; `MonetSafeState_Tests` 18 tests, `MonetErrorPark_Tests` 11): flipped `Cover_Error_At_PowerOn_Waits_In_
    Stage_0_Current` -> `Cover_Error_At_PowerOn_Fails_And_Stays_Parked` and `Cover_Error_Holds_The_Park_Before_Parked_Current` ->
    `Cover_Error_During_Park_Ends_The_Park_Without_Closed` (same method Ids); new `Cover_Error_While_Tracking_Drives_To_Park`,
    `MonetErrorPark_Tests.Cover_Error_Park_Recovers_With_Cover_Reset_And_Reset`, `.Cover_Error_During_Estop_Does_Not_Park`. Every test
    asserts no roof request where it applies. Other suites unedited.
  - Unverified until CI: the build (reading the inherited `bPowerFailure` from a test, like `fbStartup`); the test timing; that the
    `Init()` after the park goes online on the rig (as PR 3). Not modelled: a real cover error with `Opened` FALSE while tracking (then
    `bReady` is FALSE and the park waits for `bStopped`, as any external park would), HalfBROT's timing of `Reset()` (the error clears
    in the next call of the cover block). Not covered by a test: a cover error in the middle of stage 0 (the mock covers open in one
    scan), the manual-mode edge memory. To run (Tim): `tcbuild.yml` on push, or the library install + `tctool MONETcommonTests.sln
    build run`; on site (design doc, order item 4).
- **Status PR 7 (2026-11-02): written on `feat/power-failure-reaction` (from `develop` @ `40517f9`, 0.13.0 with PR 3 merged; rebased
  onto `develop` @ `692dbbf`, PR 5 and PR 4 merged), not compiled, not run (CI on push). Version not bumped.** Row "Power failure" of the design doc (now implemented, M lines of that branch).
  - What "power failure" is: `bMainReady` = `PowerMonitoring.phaseOK` (MONETS; MONETN without the L3 voltage warning), TRUE only while
    no EL3483 guard bit is set, **warnings included** (frequency, quality, unbalance, voltage per phase), not debounced (the 100 ms / 2 s
    timers of `FB_MonetPowerMonitoring` only gate its WARN log). So it covers a real phase loss (drives, pump, covers, maybe the roof
    lose their supply) and a power-quality warning (everything still runs). M cannot tell them apart and does not know whether the
    drives still have their supply. Reaction chosen to be right for both: ramp while the drives demonstrably can (brake open with
    pressure, no axis error), else the lock; then park **in place** (not a drive to the park position on a supply in question).
  - Entry: `bMainPowerLost := NOT bMainReady` goes into `bHwError` (as the hydraulics error in PR 3): `bError`, READY_STATE -1.0
    (was -2.0), moves rejected, a power-on fails in its first scan (before the covers; was: covers opened, wait in `open_covers`), a
    running power-on aborts, the #86 emergency park in place on the edge (`F_ErrorParkRequest` unchanged; exemptions parked and
    E-stop; never the #113 Wake&Shake exemption: `bOtherHwError` gets `bMainPowerLost`). Not latched: `bError` drops by itself when
    the supply is back. `nErrorID` stays 0. `bReset` not held on it (the hold list of PR 3 is unchanged, only its comment).
  - Ramp (D8): a second copy of the PR 3 ramp block with its own flags (`bMainPowerRamp`, edge memory `bMainPowerLostLast`,
    `MainPowerRampEvent`), using the same `bHydraulicsCanRamp` as the "drives can still ramp" test; it also ends when the supply is
    back. The error branch skips the disable while `bHydraulicsRamp`, `bTimeoutPark` (PR 5) or `bMainPowerRamp`; `F_SuspendStop` gets
    `bError AND NOT bHydraulicsRamp AND NOT bTimeoutPark AND NOT bMainPowerRamp`. A power failure is a hardware error, so it ends
    a running timeout park (`bTimeoutCanMove` FALSE, the park finishes in place) and blocks a new one. A real phase loss that stops the pump drops `BrakeOpen`/`OilPressureOK` (and the drives may
    raise undervoltage errors): that ends the ramp and the lock is the stop, as D8 asks; on a warning-only blip the ramp runs fully.
  - Flicker: a one-scan blip of any guard bit now parks the telescope (in place at standstill; under motion ramp, and if the supply
    is back before the stop has settled the queued park drives to the park position). The end state is the ordinary parked state:
    `reset=1` (roof request), `power=1`. Not hard to recover from, but it ends an observation; no new timer added. If warnings blip on
    site, the fix belongs into `FB_MonetPowerMonitoring` / `MAIN` (e.g. `phaseOK` from the guard errors only, or a short on-delay),
    not into M. Alternatives not taken: (a) react only after a delay in M (new timer, delays the stop on a real loss); (b) keep it out
    of `bError` and only stop + park (keeps READY_STATE -2.0 and the transient interlock, but needs a second in-place park path and a
    second disable path); (c) drive to the park position after the ramp (needs the supply to be trusted; not conservative); (d) the
    lock always, no ramp (simpler, but against D8 when the drives are fine, e.g. a warning).
  - Behaviour changes (all on `NOT bMainReady` only): READY_STATE -1.0 instead of -2.0, also on a parked telescope (0.0 before) and at
    the PLC start without phase; online is lost (completed startup dropped), a new `power=1` is needed after the supply is back (was:
    back to 1.0 by itself); moves rejected; under motion: stop (ramp) or disable, then park in place, covers closed, hydraulics off;
    at standstill online: park in place; power-on refused in its first scan with `bPowerFailure` (READY_STATE -1.0 until the next
    `power=1`), a running power-on aborted and parked in place; new WARN log on the ramp; the emergency park log says "axis error"
    (text unchanged). Unchanged: `bCloseRoofRequest` (PR 1), the hydraulics/axis/focus/software/E-stop reactions.
  - Tests (40 of 40 suites; `MonetSafeState_Tests` 18 tests / 161 asserts, `MonetErrorPark_Tests` 11 / 86 before the rebase;
    with PR 5 and PR 4: 19 / 189 and 13 / 103): flipped
    `Power_Failure_During_Goto_Keeps_Moving_Current` -> `Power_Failure_During_Goto_Ramps_Then_Parks_In_Place` (same method Id; also:
    supply back -> `bError` gone, parked, not online without Init); new `Power_Failure_With_Pressure_Loss_Parks_In_Place_Without_Ramp`
    (phase loss with the pump: lock), `MonetErrorPark_Tests.Power_Failure_At_Standstill_Parks_In_Place_And_Recovers` (no ramp; reset not
    held; supply back; reset clears the roof request; `power=1` online), `.Power_Failure_On_A_Parked_Telescope_Does_Not_Park` (no park;
    `power=1` during the failure opens nothing). Other classes: their suites are unedited (`Roof_Request_On_Power_Failure` holds: the
    reset during the failure is a pulse).
  - Unverified until CI: the build; the test timing (as PR 3); the re-Init after the in-place park on the rig. Not modelled: whether
    the drives, the covers and the roof have their supply on a real phase loss, drive undervoltage errors, how fast `BrakeOpen` drops
    when the pump stops. To run (Tim): `tcbuild.yml` on push, or the library install + `tctool MONETcommonTests.sln build run`; on site
    (design doc, order item 7).
- **Status PR 8 (2026-11-02): written on `feat/focus-error-reaction` (from `develop` @ `40517f9`, PR 3 merged; rebased onto `develop` @ `008120e`,
  PR 5, PR 4 and PR 7 merged), not compiled, not run (CI on push). Version not bumped.** Row "Focus error" of the design doc (now
  implemented, M lines of that branch). Written while PRs 4-7 ran in parallel; rebased onto them (same error branch, same suites, same table).
  - Entry unchanged: the focus error is in `bHwError` and starts the #86 emergency park on the edge of `bHwError` (`F_ErrorParkRequest`
    and `F_WakeShakeErrorExempt` unchanged). New `bFocusErrorOnly` = focus error AND no axis/hydraulics/software error AND no power failure (`bMainPowerLost`, since the
    rebase onto PR 7) AND no E-stop AND no safety error. On that edge, with `bFocusErrorOnly`, not parked, `fbHydraulics.OilPressureOK` and `F_ParkAxesCanMove` (calibrated, enabled,
    brake open), M sets `bFocusErrorPark` (before the error branch, so the axes are not disabled in that scan); the error branch skips its
    three `Enable := FALSE` while it is set, and `_ParkTelescope` stage `prepare` parks in place only for `bError AND NOT bFocusErrorPark`
    (or when the axes cannot move), so the park drives to the park position and disables the axes itself (stage `disable_axes`).
  - Ramp first (D8), the PR 3 pattern: if an enabled axis moves (InMotion/InRamp/Homing or tracking), `bStop` in the same scan (stop
    transaction #96); the emergency park is queued (`bParkAfterStop`) and starts when the stop has settled. No separate ramp flag:
    `bFocusErrorPark` covers ramp and drive.
  - End of `bFocusErrorPark`: another error, E-stop or safety error (if the park is in `untrack`/`start_move`/`wait_moves` it is set back
    to `prepare` and parks in place: there is no new `bError` edge to reset it, and disabled axes never report `Busy` in `start_move`);
    `bIsParked`; no park requested any more (`bPark`, `bParkAfterStop`, `bStop` all FALSE, e.g. after a `Stop()`: the error branch then
    disables the stopped axes, no new park because there is no new edge); the stop timeout during the ramp (the lock takes over, then in
    place); the focus error gone (the park goes on as a normal park); the first manual scan (`SuspendAutomatic`).
  - Focus step of the park, chosen (conservative): a faulty focus is **not** sent to `focusPark` (`IF NOT fbFocus.Error`); neutral on
    HalfBROT, whose `FB_FocusControl` overwrites the target with the actual position while in error. No park stage waits for the focus
    (verified: `prepare` only writes the target; `start_move`/`wait_moves`/`disable_axes` read the three axes, `secure` the brake and covers;
    `bBusy`, which includes `fbFocus.Busy`, gates only the non-error park dispatch), so a stuck focus cannot hold the park. No focus
    timeout was added. Alternatives: keep writing the target (no effect on HalfBROT while in error); wait for the focus with a timeout (a new
    stage, more code, no gain for the safe state).
  - Reset (chosen): `bReset` is not held while `bFocusErrorPark` (end of the body), so a `reset=1` during that park is one scan: it calls
    `fbFocus.Reset()` once and restarts the park from `prepare` once (`_HandleReset` and the `F_StageAbort` park row); the park then drives on.
    Held as before it would abort the park in every scan while the focus stays faulty, with the axes enabled and the brake open (a new hang).
    Alternatives: keep the hold and end the drive on a reset (the lock would stop moving axes); ignore the reset in the park during the
    drive (two more guards). Outside this park the hold is unchanged (also after the park with a stuck focus, as today).
  - Manual mode (chosen): `F_SuspendStop` gets `bError AND NOT bHydraulicsRamp AND NOT bTimeoutPark AND NOT bMainPowerRamp AND NOT
    bFocusErrorPark` (after the rebase; the moving axes get the suspend stop), then `bFocusErrorPark := FALSE`: after the return the kept `bPark` parks in place (no automatic drive after a manual interval,
    in the spirit of #72; the operator may have moved or disabled the axes). Alternative: keep it and drive after the return like a normal
    park.
  - Exemptions checked: first power-on (#113) never exempts a focus error (`bOtherHwError`), unchanged; parked telescope: no park, no flag;
    E-stop: no park, no flag (an E-stop during the drive ends it, park in place: the covers get the close command, as with the in-place
    park before PR 8, where it was issued in the first park scan); edge detection: `bHwError AND NOT bHwErrorLast`, the same edge as
    `F_ErrorParkRequest` (a focus error on top of another hardware error has no edge and changes nothing). Roof: no change (the focus is
    not a cause of `bCloseRoofRequest`; D9, plan 8.4 still open).
  - New log `FocusErrorParkEvent` (WARN, once per start). `nErrorID` unchanged (the focus `ErrorID`).
  - Tests (40 of 40 suites; `MonetErrorPark_Tests` 14 tests before the rebase, 18 tests / 162 asserts with PR 5, PR 4 and PR 7): flipped `Focus_Error_Starts_The_Emergency_Park` ->
    `Focus_Error_At_Standstill_Parks_At_The_Park_Position` (same method Id); new
    `Focus_Error_While_Tracking_Ramps_Then_Parks_At_The_Park_Position`, `Focus_Error_With_An_Axis_Error_Parks_In_Place`,
    `Axis_Error_During_The_Focus_Error_Park_Parks_In_Place`, `Stuck_Focus_Does_Not_Hold_The_Focus_Error_Park` (new
    `FB_FocusMock.InjectStuckError`: a reset does not clear it), `Focus_Error_Park_Recovers_With_Reset_And_Power_On`. No roof request is
    asserted in the drive and recovery cases; `MonetSafeState_Tests.No_Roof_Request_On_Focus_Error` unedited (parked rig).
  - Rebase onto PR 5, PR 4 and PR 7: the error branch skips the disable while `bHydraulicsRamp`, `bTimeoutPark`, `bMainPowerRamp`
    or `bFocusErrorPark`; the park's `prepare` parks in place for `bError AND NOT bTimeoutPark AND NOT bFocusErrorPark` (the two
    flags exclude each other: a timeout park needs no hardware error, a focus-error park no software error). A power failure ends
    `bFocusErrorPark` like any other error (it is not the only error any more) and the park restarts in place; the hydraulics and
    power-failure ramps never start with a focus error (`bHydraulicsCanRamp` needs no focus error). A cover error is independent:
    a focus-error park with covers in error ends in `secure` with 16#21 (PR 4), and the faulty focus is still not sent to
    `focusPark`. All events are kept, each with its one-scan flag reset.
  - Read while doing it, **not changed**: (1) a focus error during a normal park that is already at standstill in `disable_axes` restarts
    it (edge of `bError` in `_ParkTelescope`) and drives again from the park position; before PR 8 it parked in place there. (2) The park's
    `start_move` waits for all three `Busy` at once; a real NC axis already at its park position may report `Busy` only briefly
    [unverified], which would hold `start_move` for any park, not only this one. (3) A held `reset=1` with a focus that stays faulty still
    aborts every later park (outside `bFocusErrorPark`), as before.
  - Unverified until CI: the build (new variables, `F_ParkAxesCanMove` in the body, enum compares on `eParkStage`); the test timing (mock
    moves 60 scans); `fActualPosition` and `Position` read from the mocks; that the second `Init()` after the park goes online on the rig. Not
    modelled: a real focus fault (whether HalfBROT's `Reset()` clears a real stuck focus in one call), NC behaviour of a tracking axis. To run
    (Tim): `tcbuild.yml` on push, or the library install + `tctool MONETcommonTests.sln build run`; on site (design doc, order item 8).
- **Status PR 6 (2026-11-02): written on `feat/estop-safety-reaction` (from `develop` @ `40517f9`, 0.13.0, PR 3 merged; rebased
  onto `develop` @ `a0e1589`, PR 5, PR 4, PR 7 and PR 8 merged), not compiled, not run (CI on push). Version not bumped.** Rows "E-stop" and "Safety error" of the design doc (now implemented, M lines of that
  branch).
  - One condition for both: `bSafetyStop := bEstopTriggered OR bSafetyError`, in a block of its own right after the #86 emergency park
    block (before the arbitration, so the stop branch cannot clear the park in the scan of the release). D8: the lock is the stop, no
    ramp. The axes are disabled every scan while `bSafetyStop` (the old E-stop disable at the end of the body, now also for a safety
    error alone).
  - **When (decided here, the conservative reading of the doc):** on the rising edge, at once: hydraulics off (`fbHydraulics.Enable :=
    FALSE`; the brake locks; harmless because HalfBROT's `OpenBrake()` restarts the pump at the next power-on), every move ends (move
    flags dropped, goto/slew/home stage machines reset with the #26 one-shot interrupt as the park branch does it, `bMoveAbortPending`
    and `bSupersedeRequest` cleared; no stop transaction: a disabled axis is never sent an MC_Stop, #150) and, if the telescope is not
    parked, the park in place is **held** (`bSafetyParkPending`). It runs once (`Park()`, WARN `SafetyParkEvent`) when the E-stop **and**
    the safety error are both gone (`SafetyHandling.estop` is "E-stop block not in RUN", so on site this is after the safety reset
    [unverified]). Reason: the covers may hang on the E-stop circuit, and the doc's target is "covers as possible after release"; M
    does not command them while the circuit is open, so no command latched in HalfBROT starts them when the power comes back.
  - The park after the release takes the emergency-park bypass of the ready gate (`bSafetyPark`, park branch `IF bError OR
    bSafetyPark`) and parks in place because the axes are disabled (`F_ParkAxesCanMove` FALSE); `_ParkTelescope` is not changed.
    `bSafetyPark` ends with its park (`bPark`/`bParkAfterStop` gone). Without the bypass the park would wait for `NOT bBusy`, which an
    STO'd axis may not give [unverified] (the mock axis keeps `Busy` when disabled).
  - Exemptions: parked at the edge -> no pending (as #86; also skipped if it got parked meanwhile). A safety error alone while the
    Wake&Shake window is open -> no park (the window survives a safety error by #155 design, the operator goes on; an E-stop ends the
    window and does park). #86: `F_ErrorParkRequest` is unchanged, so an axis/hydraulics error during the E-stop still starts no park
    of its own; the E-stop's park covers it after the release. What changes against #86 is that the E-stop itself now leads to a park.
    The PR 3 test `Hydraulics_Error_During_Estop_Does_Not_Park` is kept unedited in its asserts (no park, no cover command, no ramp,
    axes disabled during the E-stop: all still true); only its comment ("covers must not move on the release") is corrected.
  - Not changed (kept, listed for review): a park requested by someone else during the E-stop (watchdog, cabinet, operator) and a park
    already running at the edge go on as today and may command the covers during the E-stop; a safety error alone is still not
    `bError`, does not reject moves and is not in the READY_STATE -1.0 condition; the edge memory `bSafetyStopLast` is not set in
    `SuspendAutomatic`, so a safety stop still there on the return from manual acts then (conservative; #86 does the opposite).
  - Alternatives (not taken): (a) park in place at once during the E-stop (covers commanded while unpowered; they might start on the
    release or raise a cover error); (b) wait for an explicit `reset=1` or `park=1` after the release before the park (no automatic
    cover motion after the release, but covers stay open until someone acts); (c) release on `NOT bEstopTriggered` only (a safety
    error that stays after the E-stop would then not hold the park); (d) hold every park during the E-stop, also the watchdog's and an
    operator's (consistent covers, but changes the watchdog/cabinet rows); (e) hydraulics off in every scan of the safety stop instead
    of on the edge; (f) treat a safety error alone fully as an E-stop (reject moves, READY_STATE -1.0); (g) also park in the Wake&Shake
    window on a safety error (flips `Window_Brake_Closes_On_Safety_Error`).
  - Tests (40 of 40 suites; `MonetSafeState_Tests` 20 tests after the rebase): flipped `Estop_During_Slew_Disables_The_Axes_Without_Park_Current` ->
    `Estop_During_Slew_Locks_Then_Parks_In_Place_After_The_Release` (same method Id: axes disabled and hydraulics off in the E-stop scan,
    no bStop/MC_Stop, slew ended, no park and no cover command during the E-stop and after a reset in it, park in place after the
    release, roof request until a reset, `power=1` online); new `Safety_Error_While_Tracking_Locks_Then_Parks_In_Place_When_It_Clears`;
    `Roof_Request_On_Estop` one more assert (parked: no park after the release). Unedited and expected green: `MonetErrorPark_Tests` (E-stop
    cases only commented), `MonetBrakeRelease_Tests.Window_Brake_Closes_On_Estop`/`_On_Safety_Error` (the latter pins the window exemption),
    the other classes' suites.
  - Rebase onto PR 5/4/7/8 (2026-11-03): the safety-stop block stays right after the #86 block, now after the PR 4 cover-error
    park request. The ramp/park flags of the others already end on an E-stop or a safety error before it runs (`bTimeoutPark`
    via `bTimeoutCanMove`, `bFocusErrorPark` via `bFocusErrorOnly`, `bHydraulicsRamp`/`bMainPowerRamp` via `bHydraulicsCanRamp`),
    so the edge block does not fight them; a park they started goes on (as "a park already running at the edge", above). The
    prepare condition of `_ParkTelescope` (`(bError AND NOT bTimeoutPark AND NOT bFocusErrorPark) OR NOT bAxesCanMove`) is
    kept as it is: the safety park parks in place through `NOT bAxesCanMove` (axes disabled), as before the rebase.
    `MonetErrorPark_Tests.Cover_Error_During_Estop_Does_Not_Park` (PR 4) asserted "no park after the release"; with PR 6 the
    held park runs then and ends with 16#21 (PR 4 cover path), so its last step is changed to assert that.
  - Unverified until CI: the build; the test timing; that `Init()` after the in-place park goes online (as in PR 3, after an E-stop abort
    of the startup); the mock axis keeps its move when disabled, so the E-stop test ends it with `SetMoveScans` before the release. Not
    modelled: STO, TwinSAFE, the safety reset sequence, whether the covers and the roof hang on the E-stop circuit, whether a safety
    error cuts the drives. Risk: a `reset=1` held by an axis error that does not clear (STO may raise axis errors) aborts the park in
    every scan (`F_StageAbort` park row), as for any emergency park today. To run (Tim): `tcbuild.yml` on push, or the library install
    and `tctool MONETcommonTests.sln build run`; on site (design doc, order item 6), E-stop at hand.
- **Status PR 6 follow-up (2026-11-03): "E-stop stops the covers", written on `fix/estop-stops-covers` (from `develop` @ `52725c8`),
  not compiled, not run (CI on push). Version not bumped.** Owner: the covers are **not** on the E-stop circuit (they stay powered),
  and "E-stop means stop". So the reason given above ("the covers may hang on the E-stop circuit") is replaced: on the rising edge of
  `bSafetyStop` and in every scan while it holds (end of the body, after the stages) M calls `fbCovers.Stop()`; no stage commands
  `Open()`/`Close()` while it holds (`_PowerOn` `open_covers` waits without the command, keeps `bIsParked` and does not advance;
  `_ParkTelescope` `prepare` in place and `secure` skip `Close()` and wait, the hydraulics still go off). After the release the held park
  (`bSafetyPark`) closes the covers as before; a power-on that was opening them is aborted by that park (#87) and does not open them
  again; a park that was closing them goes on closing. This also covers the case listed above as "not changed": a park requested by
  someone else or already running at the edge no longer commands the covers during the E-stop. `Stop()` is not in `I_MirrorCovers`, so
  `fbCovers` of M is now `REFERENCE TO HalfBROT.FB_CoverControl` (as the pendant, step 6); `MAIN` passes `CoverControl` unchanged.
  Alternatives: add `Stop()` to BROTLib `I_MirrorCovers` (cleaner, but a BROTLib + HalfBROT release first); a second optional input for
  the stop (needs a `MAIN` change); stop on the E-stop only, not on a safety error alone (the owner's rule is for the E-stop; PR 6 treats
  both alike, so both stop). Not covered: manual mode (M not called, the pendant keeps its hold-to-run). Tests: `FB_CoversMock` now
  extends `HalfBROT.FB_CoverControl` (no `SUPER^()`), counts `Stop()` (`nStopCalls`) and has an optional travel model
  (`FB_MonetRig.SetCoverTravelScans`, the rig calls its body after the telescope); new `MonetErrorPark_Tests.Estop_During_Cover_Opening_Stops_The_Covers`,
  `.Safety_Error_During_Cover_Opening_Stops_The_Covers`, `.Estop_During_Cover_Closing_Stops_The_Covers`; one assert more in
  `MonetSafeState_Tests.Estop_During_Slew_...` and `.Safety_Error_While_Tracking_...` (Stop() in the scan of the edge).

- **Status cabinet over-temperature roof (2026-11-03): written on `feat/cabinet-overtemp-roof-request` (from `develop` @ `0188ae0`),
  not compiled, not run (CI on push). Version not bumped.** Owner decision (settles 8.6, added to D9): a critical cabinet temperature
  parks everything, the roof included. `FB_MonetCabinetControl` gets `VAR_OUTPUT bCloseRoofRequest : BOOL`, `:= bCabinetOverTemp` in every call (set
  above 60 C, held until below 50 C, no latch beyond that, no reset); the park every scan is unchanged. D9 list of the classes that
  close the roof: E-stop/safety error, power failure, hydraulics error, axis errors (`FB_MonetTelescopeControl.bCloseRoofRequest`) and
  cabinet over-temperature (`FB_MonetCabinetControl.bCloseRoofRequest`); the watchdog closes on its own. No site change until each
  `MAIN` (MONETS, MONETN) ORs the new output into its roof close condition and re-pins. Test:
  `FB_MonetTempSensor_Tests.Cabinet_Over_Temp_Requests_Roof_Close` through the new seam `FB_MonetCabinetProbe` (writes BT1). On site:
  `bRoofCloseOnErrorEnable` off first, raise BT1 (or force `inCabinetTemp`) above 60 C: park, `CabinetControl.bCloseRoofRequest` TRUE,
  roof does not move; then the switch on: roof closes; below 50 C the request drops.

Order rationale: 0-1 make the safety net; 2a-2b put the shared types in place before MONETcommon consumes them;
2c-4 are neutral and unblock tests and the SM plan's 5.2; 4c settles the client question; 5-6 break consumers and
are batched with re-pins; 7 is policy and goes row by row. Step 3 can come before 2a-2c; 7 can start any time after
1 for a row the table has settled.

## 4. Consumers and versions

- PLC consumers: MONETS and MONETN `MAIN` (exact pin of MONETcommon), IAG50cm (BROTLib pin, step 2b), HalfBROT
  (`FB_PendantControl` references `FB_AltAzTelescopeControl`, affected only by SM 5.4).
- Command clients (D3): **pybrotlib**, **pyobs-brot**, **brotgui**. Affected by step 5 only, depending on 4c.
- Steps 2c-4: no `MAIN` change (new inputs have defaults, aliases kept). Step 5: semantics change for anyone writing
  command flags. Step 6: pendant call loses two parameters in both `MAIN`s.
- Version bumps via `release.yml` (`<ProjectVersion>` + `Global_Version.TcGVL`). Pre-1.0: minor = breaking or
  behaviour change, patch = neutral refactor. Release chain for 2a: BROTLib minor, then IAG50cm re-pin (2b),
  MONETcommon re-pin (2c), HalfBROT re-pin at its next release, MONETS/MONETN re-pin with the MONETcommon release.

## 5. Risks

- Neutral refactors of a 2806-line motion block change scan order by accident (steps 3/4). Mitigation: rig suites
  unedited, small PRs, smoke test on site per release.
- Rig coverage is characterization, not proof: mocks do not model NC timing, the hydraulic brake lock under motion
  or TwinSAFE. Steps 5 and 7 need hardware checks.
- Wrong default in `ST_MotionConfig` = wrong speed or timeout on a real mount. Mitigation: the defaults test in 2a,
  the alias test in 2c (MONETS 3.0).
- Step 2b changes a type identity on IAG50cm: build error or a mismatched initialiser shows at compile time; a
  wrong park pose would not, hence the IAG smoke test.
- Brake lock under motion (step 0 rows "lock is the stop"): stresses brake and gearing; the ramp-first rule keeps
  that to the classes where the drives cannot ramp anyway.
- Step 5 can break pybrotlib clients silently (a held `bGoto` no longer re-triggers). Gated on 4c.
- Overlap with open work in the same file: #168 (mode state machine), #177 (pendant), SM plan 5.x, #26, #17.
  Whoever merges second rebases; do not run two of these in parallel on M.
- TcUnit suite limit (40, all 40 used after step 3; step 4a's telemetry cases went into `FB_MonetRig_Tests`; step 6
  frees slots) and per-suite limits (32 tests, 256 asserts).

## 6. Verification summary

| Step | Unit / rig tests | On site (Tim) |
|---|---|---|
| 0 | - | - |
| 1 | new suites green on current MONETcommon | - |
| 2a | `F_MotionConfigValid_Tests` in BROTLibTests | - |
| 2b | IAG50cm suites green | IAG50cm smoke test |
| 2c | all green unedited + config and alias cases | smoke test S, N |
| 3 | all green unedited + `F_StageAbort_Tests` | smoke test S, N |
| 4 | all green unedited + telemetry case | telemetry check in Influx/pyobs |
| 4c | - (code reading of pybrotlib, pyobs-brot, brotgui) | - |
| 5 | edited flag tests + edge cases | full check S, N with pyobs-brot and brotgui |
| 6 | all green | pendant check (#69 list) |
| 7 | flipped `_Current` test per row | per row, safety |

## 7. Decisions (Tim, 2026-11-02)

- **D1 Config home.** Unify with IAG50cm: BROTLib's `ST_TelescopeConfig` (a superset since 0.16.0) is the one
  struct; IAG50cm drops its local copy (step 2b). Motion limits (elevation windows, velocities, timeouts) go into a
  separate alt-az-only `ST_MotionConfig` in BROTLib (step 2a), consumed by MONETcommon (step 2c).
- **D2 Deprecated inputs.** `fElevationMoveVelocity` and the track margin/cap inputs stay as deprecated aliases for
  one release.
- **D3 Clients.** The command clients are pyobs-brot and brotgui, both via pybrotlib. Inspect how pybrotlib sends
  commands before step 5 (step 4c). Checked in step 4c ([2026-11-02-step-4c-client-check.md](2026-11-02-step-4c-client-check.md)):
  MQTT verbs only, mapped to the methods by the PLC; the MONET Visu writes the flags. Outcome: edge-triggered inputs plus
  methods (step 5, done as written).
- **D4 Safe state.** Roof closed, park if possible, covers closed as far as possible, hydraulics off. The brakes are
  hydraulic and lock when the hydraulics go off, so no brake-before-hydraulics ordering is needed. The roof does not
  wait for `bIsParked`.
- **D5 Split depth.** Option A: `_ComputeTargets()` method in M; `FB_MonetTargets` only if A proves insufficient.
- **D6 Axes coupling.** Keep `REFERENCE TO` the concrete axes, rationale in the README, `I_Axis` not widened.
- **D7 Internal timings.** Tracking delay, Wake&Shake window, interrupted auto-clear (and the other internal timers)
  are not site policy and stay constants outside the config.
- **D8 Stop before hydraulics off** (was 8.1). Ramp first: a controlled stop (`FB_MonetStopTransaction`), then
  hydraulics off, so the brake locks on a standing axis, whenever the drives can still ramp. The brake lock is the stop
  only for axis errors, E-stop/safety errors, and a hydraulics error where the pressure is already gone. The hydraulic
  brakes lock by themselves when the hydraulics go off.
- **D9 Roof** (was 8.2). Close the roof only for serious errors: E-stop/safety error, power failure, hydraulics error,
  axis errors. Not for the recoverable ones: timeouts 16#10/16#30/16#40, config errors 16#31/16#32, goto target
  16#15/16#16, invalid clock, cover error, focus error (kept as no close, see 8.4). The closing logic lives in the site
  `MAIN` (MONETS, MONETN) behind an enable switch that can be turned off for test runs, and does not close while the
  cabinet selector is on manual. The library only provides the signal (proposed: M output `bCloseRoofRequest`,
  derivation in the design doc). The MQTT watchdog is unchanged (always closes). The E-stop circuit also cuts the roof
  drives: a close during an active E-stop cannot act, the request is held and takes effect after release and restart.
  Step 7 PR 1 (the signal) written on `feat/close-roof-request`; PR 2 (`MAIN`) not started.
  Added 2026-11-03 (owner, settles 8.6): the cabinet over-temperature (#32) closes the roof too ("a critical cabinet
  temperature parks everything, the roof included"). The D9 list of classes that close the roof is now: E-stop/safety error,
  power failure, hydraulics error, axis errors, cabinet over-temperature (signal `FB_MonetCabinetControl.bCloseRoofRequest`,
  ORed in `MAIN`); plus the MQTT watchdog on its own.

## 8. Still open

1. Decided as D8 (ramp or lock per error class).
2. Decided as D9 (roof for classes other than the watchdog); what is left of it is in 8.4-8.7.
3. **pybrotlib findings** (step 4c) decide edge-triggered inputs vs methods only for step 5. Settled: edge-triggered inputs
   plus methods (D3, step 4c); written in step 5.
4. **Focus error and the roof:** kept as no close (D9); is a focus error serious enough to close the roof?
5. **Exact signal:** name (`bCloseRoofRequest` proposed), latch and reset (proposed: latched on the rising edge,
   cleared by `bReset` once no cause is active) and the name of the `MAIN` enable switch (`bRoofCloseOnErrorEnable`
   proposed; a variable, a Visu switch or a cabinet input). Settled in step 7 PR 1: name and latch/reset as proposed (status
   at step 7); the `MAIN` switch is left to PR 2.
6. **Cabinet over-temperature (#32) and the roof:** decided 2026-11-03 (owner): close (added to D9). Written on
   `feat/cabinet-overtemp-roof-request` (status at step 7).
7. **Request at PLC start:** does `NOT bMainReady` or an active E-stop at boot set the request on every restart, so
   that it needs an arming (like the watchdog's first connect) or a debounce? Check on site before step 7 PR 2. PR 1 chose
   no arming (a cause at the start sets it; one-line change to arm, status at step 7).
   Also [unverified]: whether the roof drives have power on a phase loss, and whether a safety error (no E-stop)
   cuts them like the E-stop.

## 9. Not in this plan

- `FB_CommandArbiter` and the template body (SM plan 5.1-5.4); the rest of the IAG50cm work there.
- HalfBROT blocks writing their own inputs (separate HalfBROT issues to open); TwinSAFE / H5 (#6, #61).
- Pendant behaviour (#177, #10) and the mode state machine (#168), beyond the dead inputs in step 6.
