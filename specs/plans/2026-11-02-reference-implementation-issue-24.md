# MONETcommon as a reference implementation (#24, code review finding D)

Date: 2026-11-02. Status: **proposed, plan only, no code changed. Decisions D1-D7 taken (Tim, 2026-11-02, section 7);
the open items left are in section 8.** Steps 0-1 written 2026-11-02 on `feat/safe-state-table-and-tests`: the
table is [../design/monetcommon-safe-states.md](../design/monetcommon-safe-states.md) (cells corrected against 0.11.2), the suite
`MonetSafeState_Tests`; not compiled or run yet. Step 2a released in BROTLib 0.17.0. Step 2c written 2026-11-02 on
`feat/motion-config-input` (status at the step); not compiled or run yet. Step 3 written 2026-11-02 on
`refactor/stage-machines` (status at the step); not compiled or run yet.
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
  they are not. The draft table (to copy into the design doc and check against the code in step 1):

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

### Step 7 - Safe-state fixes (aim 6, one PR per row of the step-0 table)
- Each row where today differs from the target becomes a PR: flip the `_Current` test, change M (expected first
  candidates: software errors and hydraulics/cover errors park; hydraulics off at the end of every emergency park).
  Safety-relevant: on-site check per PR, E-stop at hand. Release: minor per behaviour change, MONETS first.

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
- TcUnit suite limit (40, all 40 used after step 3; step 4's telemetry case must go into an existing suite or step 6
  must free slots first) and per-suite limits (32 tests, 256 asserts).

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
  commands before step 5 (step 4c).
- **D4 Safe state.** Roof closed, park if possible, covers closed as far as possible, hydraulics off. The brakes are
  hydraulic and lock when the hydraulics go off, so no brake-before-hydraulics ordering is needed. The roof does not
  wait for `bIsParked`.
- **D5 Split depth.** Option A: `_ComputeTargets()` method in M; `FB_MonetTargets` only if A proves insufficient.
- **D6 Axes coupling.** Keep `REFERENCE TO` the concrete axes, rationale in the README, `I_Axis` not widened.
- **D7 Internal timings.** Tracking delay, Wake&Shake window, interrupted auto-clear (and the other internal timers)
  are not site policy and stay constants outside the config.

## 8. Still open

1. **Ramp or lock per error class** (step 0 table, column "Stop before hydraulics off"): confirm the proposed rule
   (controlled stop first whenever the drives can still ramp; the hydraulic brake lock is the stop only for axis
   error, E-stop/safety error, and a hydraulics error with the pressure already gone). Settle in the step-0 PR.
2. **Roof for classes other than the watchdog:** which `MAIN`/MONETRoof signal closes it (e.g. on `bError` or
   READY_STATE -1), so the table can name it. Settle in step 0.
3. **pybrotlib findings** (step 4c) decide edge-triggered inputs vs methods only for step 5.

## 9. Not in this plan

- `FB_CommandArbiter` and the template body (SM plan 5.1-5.4); the rest of the IAG50cm work there.
- HalfBROT blocks writing their own inputs (separate HalfBROT issues to open); TwinSAFE / H5 (#6, #61).
- Pendant behaviour (#177, #10) and the mode state machine (#168), beyond the dead inputs in step 6.
