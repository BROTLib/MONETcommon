# MONETcommon safe state per error class

Status: **final for step 7** (step 0 of [plans/2026-11-02-reference-implementation-issue-24.md](../plans/2026-11-02-reference-implementation-issue-24.md),
issue #24 aim 6; ramp rule and roof rule decided by Tim on 2026-11-02, plan D8/D9, what is still open is in plan
section 8). Living doc: update a row when its behaviour changes (step 7, one PR per row, order in "Step 7"). Step 7 PR 1
(`bCloseRoofRequest`) written on `feat/close-roof-request`, see "Roof"; the `M:` lines it cites are of that branch. Step 7 PR 3
(hydraulics error reaction) written on `feat/hydraulics-error-reaction` (from `develop` @ `7d667d8`, 0.13.0), not compiled or
run yet; the `M:` lines of its row and of the bullets marked PR 3 below are of that branch. Step 7 PR 5 (state-machine
timeouts ramp, then park) written on `feat/timeout-error-reaction` (from `develop` @ `40517f9`, PR 3 merged), not compiled or
run yet; the `M:` lines of the timeout row and of the bullets marked PR 5 below are of that branch.
Code read at `develop` @ `c34470f` (MONETcommon 0.11.2, pins BROTLib 0.16.0 / HalfBROT 0.10.2); site `MAIN`s read at
MONETS `develop` @ `3757306` and MONETN `develop` @ `9ae07bb`.

**M** = `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU`; `M:n` is the line in that file (XML included).
Line numbers are those of `c34470f` and have moved since (step 2c, 0.12.0; step 3 of the plan alone shifts the body by
+9 lines from the new declarations, and more in and after the stage-machine methods, the properties and
`_RaiseSoftwareError`). Step 3 is behaviour-neutral, no cell below changes; where a cell cites a stage machine read it by name:
the stages are `E_GotoStage`/`E_SlewStage`/`E_ParkStage`/`E_PowerOnStage` with the same numbers (stage 0/80 of the park
is `prepare`/`secure`, stage 0 of the power-on `open_covers`), the software errors 16#10/16#15/16#16/16#30/16#40 are
raised by `_RaiseSoftwareError`, and each prologue is a row of `F_StageAbort`. Step 4b (also neutral) moves the target block
of the body (refraction weather to the second hor2eq) into `_ComputeTargets`, so the body lines after it moved up by 127;
no cell cites a line of that block.
Marking as in the plan: **[verified]** = read in the code on 2026-11-02, **[unverified]** = my reading, to be checked
by a test, CI or on site. Nothing here has been run on a PLC; the rig tests named below are characterization tests on
mock axes (`MONETcommonTests`), they do not model NC timing, STO/TwinSAFE or the hydraulic brake under motion.

## Target (decision D4)

Roof closed, telescope parked if possible, covers closed as far as possible, hydraulics off. The Az/El brake is
hydraulic and locks by itself when the hydraulics go off, so switching the hydraulics off *is* the brake close; there
is no brake-before-hydraulics ordering. The roof closes at once and does not wait for `bIsParked`.

**Stop before the hydraulics drop (decided, plan D8, was open point 8.1):** ramp first, i.e. a controlled stop
(`FB_MonetStopTransaction`), then hydraulics off, so the brake locks on a standing axis, whenever the drives can still
ramp. The brake lock is the stop only when they cannot: axis error, E-stop/safety error, and a hydraulics error where
the pressure is already gone. The hydraulic brakes lock by themselves when the hydraulics go off.

**Roof (decided, plan D9, was open point 8.2):** the roof closes only for the serious classes: E-stop/safety error,
power failure, hydraulics error, axis error (Az/El/Der). It does **not** close for the recoverable ones: timeouts
16#10/16#30/16#40, config errors 16#31/16#32, goto target 16#15/16#16, invalid clock, cover error, and focus error
(kept as no close; whether a focus error is serious enough is still open, plan 8.4). The MQTT watchdog is unchanged
and always closes (`FB_MonetMqttWatchdog`). The cabinet over-temperature (#32) is not in either list and stays to
decide (plan 8.6).

- Today this library only closes the roof in `FB_MonetMqttWatchdog`. Neither site `MAIN` closes the roof on `bError`,
  READY_STATE -1.0 or any other telescope signal [verified: MONETS `MAIN.TcPOU:210` and MONETN `MAIN.TcPOU:192` are the
  only roof closes, both through the watchdog; the weather close in MONETS `MAIN.TcPOU:232-235` is commented out].
  Whether MONETRoof closes on its own (weather, power) is not part of this table [unverified].
- **Split:** the library only provides the signal; the closing logic lives in the site `MAIN` (MONETS, MONETN). The
  signal is implemented (step 7 PR 1, below); the `MAIN` part is not (PR 2).
- **Signal (implemented, step 7 PR 1 on `feat/close-roof-request`, not compiled or run yet):** `VAR_OUTPUT
  bCloseRoofRequest : BOOL` of M (M:69-75 on that branch). Derived in the body right after `bHwError`/`bError` (M:331-345)
  from the serious classes only, not from `bError` (which also carries focus and the software errors): the cause
  `bCloseRoofCause := fbElevation.Error OR fbAzimuth.bError OR fbDerotator.bError` (axis, the terms of `bHwError` without
  `fbFocus.Error`) `OR fbHydraulics.Error` (also read by the brake rule) `OR bEstopTriggered OR bSafetyError OR NOT
  bMainReady` (power failure is the phase monitor, not M's `bPowerFailure`, which is a power-on failure: brake clearing
  missing or an error during the power-on). Set on the rising edge of the cause (`bCloseRoofCauseLast`, M:204-208), so a
  cause that is gone again (a short E-stop, a phase dip) still requests the close; cleared by `bReset` once no cause is
  active. READY_STATE -1.0 is not used: it also covers the recoverable software errors. Details as built:
  - The clear sits in the same block as the set, not in `_HandleReset` (the cause is computed after `_HandleReset`; one
    place is easier to read). Clear before set, so a new cause in the reset scan wins.
  - `bReset` is held while `bError` (end of the body), so for an axis error one reset clears the axis in the reset scan
    and the request in the next scan. A hydraulics error is not reset by M (HalfBROT `Reset()`, `MAIN` button): a reset
    while it is active keeps the request, a second reset after the hydraulics reset clears it.
  - Nothing in the body returns early, so it is evaluated in every automatic scan, in the error branch and in E-stop.
    `SuspendAutomatic` does not run it: in manual the output keeps the value of the last automatic scan (the `MAIN` gate
    below ignores it there); a cause that appears in manual and is still active on the return sets it in the first
    automatic scan, one that comes and goes during manual is not seen.
  - **PLC start (plan 8.7, chosen for PR 1):** no arming. `bCloseRoofCauseLast` starts FALSE, so a cause already present
    at the start (E-stop pressed, no phase) sets the request in the first automatic scan, and it needs a `reset=1` once
    the cause is gone. Conservative (closing is the safe direction) and the simplest; the on-site check of PR 2 decides
    whether that is a nuisance. To arm only after the start, initialise `bCloseRoofCauseLast` with TRUE (then a cause
    present at the start does not set it until it goes and comes back; with several causes an arming would rather need
    per-cause edges or an arming after the first scan without a cause).
  - Not logged (no event): a signal only, watched by ADS on site.
- **`MAIN` (per site, proposed):** a new `bRoofCloseOnErrorEnable : BOOL := TRUE` (switch for test runs) and, after the
  `MqttWatchdog` call, `IF bRoofCloseOnErrorEnable AND eTelescopeMode = E_TelescopeMode.automatic AND
  TelescopeControl.bCloseRoofRequest THEN RoofControl.Close(); END_IF` (every scan while requested, like the watchdog).
  The mode term is the cabinet selector: `eTelescopeMode` is manual/off when `CabinetControl.KeyLocal` (MONETS
  `MAIN.TcPOU:76-80`, MONETN `MAIN.TcPOU:68-72`). In manual mode `TelescopeControl` is not called at all, only
  `SuspendAutomatic()` (MONETS `MAIN.TcPOU:128/147`, MONETN `MAIN.TcPOU:114/132`), so the output is not updated there
  anyway; the gate keeps a request latched in automatic from acting after the switch to manual.
- **E-stop:** the E-stop circuit also cuts the roof drives, so a close during an active E-stop cannot act. The request
  stays latched and the close takes effect after the release and restart of the roof drives (as long as the request is
  still latched and the selector is on automatic) [unverified on site: whether `FB_RoofControl.Close()` called during
  the E-stop is remembered or must be called again after the release; calling it every scan covers both].

## What the telescope block does on any error [verified]

These lines are shared by several rows below.

- `bHwError` is elevation, azimuth, focus or derotator error (M:283-286), and since step 7 PR 3 also the hydraulics error
  (`bHydraulicsError`, M:337-343 on `feat/hydraulics-error-reaction`); `bError := bHwError OR bSoftwareError` (M:287). Covers,
  E-stop, safety error and `bMainReady` are **not** part of `bError`.
- On `bError` every move flag is dropped and all three axes are disabled in the same scan (M:312-324). No ramp: the
  disable is the stop. Only exception (PR 3): during the ramp of a hydraulics error (`bHydraulicsRamp`, only while the
  hydraulics error is the only error) the disable waits until that stop has settled (M:417-421 on the PR 3 branch). Second
  exception (PR 5): during the ramp and park of a state-machine timeout (`bTimeoutPark`, only without a hardware error,
  E-stop or safety error) the axes stay enabled until the park disables them (M:455-458 on `feat/timeout-error-reaction`). An error raised inside a stage method (timeouts) takes effect from the next scan.
- Elevation error: the brake is closed once on the rising edge (#155, M:328-336).
- Brake rule (#121/#151/#155): whenever the elevation is disabled the brake is closed, except in the Wake&Shake
  window, where E-stop, hydraulics error, safety error or covers not open still close it (M:741-769,
  `F_BrakeMustClose`).
- Emergency park (#86): only on the rising edge of `bHwError`, not parked, not in E-stop, not the Wake&Shake fault of
  the first power-on (M:545-554, `F_ErrorParkRequest`, `F_WakeShakeErrorExempt`). With `bError` the park bypasses the
  ready gate (M:606-612) and parks **in place**: brake closed, axes disabled, covers closed (M:1072-1080), then
  hydraulics off once the brake reports closed (M:1139-1142), `bIsParked` once the covers report closed
  (M:1143-1150). `bParkedInPlace` is set (#158). Since PR 5 the state-machine timeouts 16#10/16#30/16#40 also park, on the
  rising edge of the timeout error (not parked, no E-stop/safety/hardware error), but that park drives to the park position
  when `F_ParkAxesCanMove` (M:1086-1087 on `feat/timeout-error-reaction`); the other software errors still start no park.
- Software errors are latched in `bSoftwareError`/`nSoftwareErrorID` until `bReset` (M:1860-1864). `bReset` is held while
  an error is active that the block resets itself (axes, focus, software errors); since PR 3 written out as that list
  (M:796 on the PR 3 branch), so a hydraulics error, which only HalfBROT resets, does not hold it (same as before PR 3).
- READY_STATE: -1.0 on `bError OR bSoftwareError OR bPowerFailure OR bEstopTriggered`; online needs
  `bReady AND bMainReady AND NOT bEstopTriggered`; E-stop also aborts the startup (M:801-806, `FB_StartupControl`).
- `bReady` needs homed axes, covers opened, the three axes enabled, the brake open and no error (M:787-792).
- Command timeouts (`tonCommandTimeout`: power-on 12 h, park/gohome/goto/slew 2 min, track 12 h; since plan step 2c the
  `stMotion` fields `tPowerOnTimeout`, `tCommandTimeout`, `tTrackTimeout`, with these defaults) only log
  (`TCSCTOEvent`, M:2016-2019); only the stop timeout acts (stop transaction, M:1970).

## Table

"Today" is what the code does, with the line; "Test" pins it on the mock rig. Where today differs from the target the
test carries the suffix `_Current` (it flips in step 7). "Step 7" says whether the row differs from the target (a
step 7 candidate, bold) and which PR of the order below fixes it. The roof signal `bCloseRoofRequest` exists since step 7
PR 1 (M:331-345 on `feat/close-roof-request`) with one case per cause in `MonetSafeState_Tests` (`Roof_Request_On_*`);
no `MAIN` acts on it yet (PR 2), so the roof cell of those rows still differs.

| Error class | Trigger, latched where | Today [verified, file:line] | Target (D4, ramp rule D8, roof D9) | Step 7 | Reset path | Test |
|---|---|---|---|---|---|---|
| Axis error Az/El/Der | `fbAzimuth.Error`/`fbElevation.bError`/`fbDerotator.bError` (M:256-270, 283-286), latched in the axis | All axes disabled (M:312-324), El: brake closed on the edge (M:328-336), Az/Der: brake closed by the #121 rule (M:751-766). Emergency park in place: covers closed, hydraulics off, `bIsParked`, `bParkedInPlace`, READY_STATE -1.0 (M:545-554, 1072-1080, 1139-1150). Not in E-stop, not on a parked telescope, not for the Az/El fault of the first power-on (#113). `bCloseRoofRequest` set (M:331-345), no `MAIN` acts on it yet | brake lock is the stop (drives not controllable, D8); park in place; covers close; hydraulics off; roof close (D9, `bCloseRoofRequest`) | **differs: roof only** (signal done in PR 1, `MAIN` close PR 2); stop, park, covers and hydraulics already as the target | `reset=1` (axis `Reset()` M:258-269; the same reset clears the roof request in the next scan), then `power=1` | `MonetErrorPark_Tests.Elevation_Error_Starts_The_Emergency_Park`, `.Error_During_Estop_Does_Not_Park`, `.First_PowerOn_AzEl_Error_Is_Exempt_From_The_Park`, `MonetBrakeRelease_Tests`, `MonetSafeState_Tests.Roof_Request_On_Axis_Errors` |
| Focus error | `fbFocus.Error` (M:271-275, 285) | Same as an axis error: the three axes are disabled (M:312-324) and the emergency park parks **in place** (M:1072), it does not drive to the park position. Plan draft said "drive to park": wrong for today | ramp, then drive to park if `F_ParkAxesCanMove`, else in place; covers close; hydraulics off; **no roof close** (D9, recoverable; open: plan 8.4) | **differs: no ramp, parks in place** (PR 8) | `reset=1`, then `power=1` | `MonetErrorPark_Tests.Focus_Error_Starts_The_Emergency_Park`, `MonetSafeState_Tests.No_Roof_Request_On_Focus_Error` |
| Software error: timeout goto 16#10, home 16#30, slew 16#40 | `tonGotoTimeout`/`tonHomeTimeout`/`tonSlewTimeout`, `stMotion.tStateMachineTimeout` 600 s (M:500-502); raised in M:895, 1003, 1530 (`_RaiseSoftwareError`); latched `bSoftwareError` (M lines of `feat/timeout-error-reaction`) | **Implemented (step 7 PR 5, not compiled or run yet; M lines of `feat/timeout-error-reaction`).** `bError`, READY_STATE -1.0 as before. **Ramp first** (M:379-407): on the rising edge of the timeout error (`bTimeoutError`, the latched ID is 16#10/16#30/16#40), in the scan after the raise, unless parked or a hardware error (`bHwError`, incl. hydraulics), E-stop or safety error is active (`bTimeoutCanMove`), `bTimeoutPark` is set, `bStop` if an enabled axis moves (also homing or tracking; stop transaction #96, the focus stops too) and `Park()`, queued behind the stop (`bParkAfterStop`) or at once at standstill; logged once (`TimeoutParkEvent`). While `bTimeoutPark` the error branch keeps the axes enabled (M:455-458) and the park's prepare stage **drives to the park position** if `F_ParkAxesCanMove` (calibrated, enabled, brake open), else parks in place (M:1086-1087); disable_axes/secure then disable the axes, the #121 rule closes the brake, the covers close and the hydraulics go off (M:1154-1156), `bIsParked`, `bParkedInPlace` FALSE. `bTimeoutPark` ends when parked, when the error is reset, when a hardware error, E-stop or safety error appears, when no stop or park is pending (`stop=1` cancels the park; the axes are then disabled) or on the stop timeout (the lock takes over, the queued park runs in place); a park still driving with the error then restarts in its prepare stage, i.e. in place. `bCloseRoofRequest` not set. `SuspendAutomatic` stops the drives during it (M:1356-1358), `bPark` kept (#118) | ramp, then drive to park; covers close; hydraulics off; **no roof close** (D9, recoverable) | matches since PR 5 (no PR left) | `reset=1` (M:1810), then `power=1`; a reset during the park drive clears the error and the park goes on as a normal park (`bPark` kept, #85) | `MonetTimeouts_Tests.Slew_Times_Out_After_600_Seconds`, `.Slew_Timeout_Ramps_Parks_And_Recovers_With_Reset` (16#40, reset path), `MonetSafeState_Tests.Goto_Times_Out_Ramps_Then_Drives_To_Park` (was `Goto_Times_Out_Without_Park_Current`; also: no roof request), `.Home_Times_Out_Ramps_Then_Drives_To_Park` (was `Home_Times_Out_Without_Park_Current`) |
| Software error: goto/track target 16#15 (below `stMotion.fGotoMinElevation`, default 2 deg) / 16#16 (above `fGotoMaxElevation`, default 89.5 deg) | `F_GotoTargetCheck` in `_GotoTelescope` (M:890-899), latched | Same as the timeouts: axes disabled, no park. Plan draft did not list these | as the timeouts (the telescope has not moved yet: stop, no park needed); **no roof close** (D9) | matches if raised at standstill; differs (no ramp) only if raised under motion on a track target [unverified], then with PR 5 | `reset=1` | `F_GotoTargetCheck_Tests` (pure), `.Goto_Window_From_stMotion_Refuses_The_Pole` (rig) |
| Software error: config 16#31 (home/park below horizon) | `F_HomeBelowHorizon`, set **every scan** (M:248-253) | Axes disabled while the configuration is wrong; a reset does not clear it | stays a startup refusal (nothing moves); **no roof close** (D9) | matches, no PR | fix `telescopeConfig` | `F_HomeBelowHorizon_Tests` (pure) |
| Software error: config 16#32 (invalid `stMotion`, plan step 2c) | `BROTLib.F_MotionConfigValid` FALSE on the values in use (`stMotion` with the deprecated aliases applied, `_MotionConfig`), set **every scan** right after 16#31 (16#32 wins if both) | As 16#31: axes disabled while the configuration is wrong, a reset does not clear it; the defaults of `ST_MotionConfig` stay in use meanwhile, so no timer, window or velocity runs with an invalid value | stays a startup refusal (nothing moves); **no roof close** (D9) | matches, no PR | fix `stMotion` (or the alias), then `reset=1` | `MonetSafeState_Tests.Invalid_Motion_Config_Is_Software_Error_32` (also: no roof request) |
| Hydraulics error | `fbHydraulics.Error` (HalfBROT `FB_HydraulicsControl`, latched there until its `Reset()`), read every scan into `bHydraulicsError` and `bHwError` (M:337-344, PR 3 branch) | **Implemented (step 7 PR 3, not compiled or run yet; M lines of `feat/hydraulics-error-reaction`).** A telescope error like an axis error: `bError`, READY_STATE -1.0, moves rejected, a power-on fails (`bPowerFailure`). **Ramp first** (M:346-367): on the rising edge, if the drives can still ramp (`bHydraulicsCanRamp`: `fbBrake.BrakeOpen`, `fbHydraulics.OilPressureOK`, no E-stop, no safety error, no other axis/focus/software error) and an enabled axis moves (also homing or tracking), `bStop` is set in that scan (stop transaction #96, the focus stops too; a tracking axis first drops tracking, BROTLib#101) and the error branch keeps the axes enabled (M:417-421) until the stop has settled, the drives cannot ramp any more, the hydraulics error is gone or the stop timeout (15 s) runs out; logged once (`HydraulicsRampEvent`). The emergency park of the same scan (M:516-528) is queued behind the stop (`bParkAfterStop`) and then parks **in place**: brake closed, axes disabled, covers closed, hydraulics off, `bIsParked`, `bParkedInPlace` (M:1033-1124). **Lock is the stop** when the brake is no longer open or the pressure is gone (or E-stop/safety/other error): disabled in the scan of the error, park in place at once, as for an axis error; also at standstill (nothing to ramp). Exemptions as #86: no park on a parked telescope, none during an E-stop (no ramp either); never the #113 Wake&Shake exemption (hydraulics in `bOtherHwError`, M:523). `bCloseRoofRequest` set (M:375-383), no `MAIN` acts on it yet. `nErrorID` stays 0 for a hydraulics error alone (no ID) | ramp in the same scan while the pressure holds; the lock is the stop if the pressure is already gone (D8); park in place; covers close; hydraulics off; roof close (D9, `bCloseRoofRequest`) | **differs: roof only** (signal done in PR 1, `MAIN` close PR 2); stop, park, covers and hydraulics as the target since PR 3 | HalfBROT `Reset()` (MAIN reset button, which also calls the telescope `Reset()` in automatic): `bError` goes by itself once the hydraulics error is gone; `reset=1` (after or with it) clears the roof request; `power=1`. A `reset=1` while the hydraulics error is still there neither clears it nor is held (M:796), so the park is not aborted | `MonetSafeState_Tests.Hydraulics_Error_While_Tracking_Ramps_Then_Parks_In_Place` (was `..._Keeps_Tracking_Current`), `.Hydraulics_Pressure_Loss_While_Tracking_Parks_In_Place_Without_Ramp`, `.Roof_Request_On_Hydraulics_Error`, `MonetErrorPark_Tests.Hydraulics_Error_On_A_Parked_Telescope_Does_Not_Park`, `.Hydraulics_Error_During_Estop_Does_Not_Park`, `.Hydraulics_Error_Park_Recovers_With_Hydraulics_Reset_And_Reset`, `MonetBrakeRelease_Tests.Window_Brake_Closes_On_Hydraulics_Error` |
| Cover error | `fbCovers.Error` (HalfBROT `FB_CoverControl`, latched there) | **M never reads `fbCovers.Error`** [grep]. Plan draft "`bReady` drops" is wrong: `bReady` reads `Opened` (M:788), and HalfBROT's `Opened` is "all three open switches", independent of the error, so online nothing changes. HalfBROT stops cover motion on the error, so: a power-on waits in stage 0 for `Opened` (M:1214-1217), with `bIsParked` already FALSE (M:1212), until the 12 h command timeout, which only logs; a park stops in stage 80 waiting for `Closed` (M:1143) with axes disabled, brake closed and hydraulics off, `bPark` held, READY_STATE 0.3, never `bIsParked` | ramp; drive to park; covers as far as possible; hydraulics off; park ends without `Closed`; **no roof close** (D9, recoverable) | **differs: park and power-on hang** (PR 4) | HalfBROT `Reset()` (MAIN reset button) | `MonetSafeState_Tests.Cover_Error_At_PowerOn_Waits_In_Stage_0_Current`, `.Cover_Error_Holds_The_Park_Before_Parked_Current` |
| E-stop | `bEstopTriggered` (MAIN: `SafetyHandling.estop`) | Axes disabled every scan (M:823-827), brake closed by the #121 rule, no emergency park (M:546), moves rejected (`F_CommandAcceptance` `estop`), READY_STATE -1.0 and the startup is aborted (M:802, 806), Wake&Shake window closed (M:744). Covers not closed, hydraulics stay on, the move flags are not cleared by the E-stop [verified]; how a slew ends on real STO'd axes is [unverified]. `bCloseRoofRequest` set (M:331-345) and held after the release until a `reset=1` without a cause, no `MAIN` acts on it yet | brake lock is the stop (STO, D8); park in place; covers as possible after release; hydraulics off; roof: close requested at once (D9, `bCloseRoofRequest`), but the E-stop circuit also cuts the roof drives, so it cannot act during the E-stop; the held request closes the roof after release and restart | **differs: no park, hydraulics stay on, roof** (PR 6; roof: signal done in PR 1, `MAIN` close PR 2) | release, safety reset, `power=1`; `reset=1` for the roof request | `MonetErrorPark_Tests.Error_During_Estop_Does_Not_Park`, `MonetBrakeRelease_Tests.Window_Brake_Closes_On_Estop`, `MonetSafeState_Tests.Estop_During_Slew_Disables_The_Axes_Without_Park_Current`, `.Roof_Request_On_Estop` |
| Safety error (no E-stop) | `bSafetyError` (MAIN: `SafetyHandling.error`, #155) | Used only by the brake rule in the Wake&Shake window (M:752, 760); no disable, no park. On site the STO would raise axis errors, which then park [unverified]. `bCloseRoofRequest` set (M:331-345), no `MAIN` acts on it yet | as E-stop: lock is the stop (D8); park in place; hydraulics off; roof close requested (D9, `bCloseRoofRequest`) [unverified whether a safety error also cuts the roof drives; if so, held as for the E-stop] | **differs: no stop, no park, roof** (PR 6; roof: signal done in PR 1, `MAIN` close PR 2) | safety reset; `reset=1` for the roof request | `MonetBrakeRelease_Tests.Window_Brake_Closes_On_Safety_Error`, `MonetSafeState_Tests.Roof_Request_On_Safety_Error` |
| Power failure / `bMainReady` FALSE | `bMainReady` (MONETS `MAIN.TcPOU:143` `ready` = `PowerMonitoring.phaseOK`, MONETN `MAIN.TcPOU:128` `mainReady`) | Blocks a power-on in stage 0 (M:1215) and its completion (M:1282), **and** gates online: READY_STATE -2.0 while FALSE, back to 1.0 when TRUE again without a new Init (M:805, transient interlock of `FB_StartupReadiness`). A running goto/track goes on; no stop, no park. Plan draft "only blocks power-on" was incomplete. What the drives do on a real phase loss is [unverified]. `bCloseRoofRequest` set (M:331-345), also by a phase loss at the PLC start, no `MAIN` acts on it yet | ramp if the drives are powered, else lock (D8); park if possible; covers as possible; hydraulics off; roof close (D9, `bCloseRoofRequest` from `NOT bMainReady`) [unverified: whether the roof drives have power on a phase loss] | **differs: goto/track go on, roof** (PR 7; roof: signal done in PR 1, `MAIN` close PR 2) | automatic when the supply is back; `reset=1` for the roof request | `MonetSafeState_Tests.Power_Failure_During_Goto_Keeps_Moving_Current`, `.Roof_Request_On_Power_Failure` |
| Invalid clock | `fbTime.bValid` FALSE (M:229-238) | A goto/slew/track flag set while the clock is invalid takes the stop path (`bStop`, M:236-238) and the move is rejected `clock_invalid` (`_CommandState` M:2487); no park. **Only reachable before the first sync**: BROTLib `FB_AstroClockSync` latches `bValid` TRUE after the first successful read, so "invalid clock during track" cannot happen today. Plan draft "stop, no park" holds for that window only | ramp (stop path); drive to park; covers close; hydraulics off; **no roof close** (D9, recoverable) | differs (no park), only before the first sync, when the telescope is parked after a restart anyway (PR 9, optional) | automatic (first sync) | `MonetSafeState_Tests.Clock_Invalid_Slew_Takes_The_Stop_Path` |
| MQTT loss (watchdog) | `FB_MonetMqttWatchdog`, 30 s without `bConnected` after the first connect | `iTelescope.Park()` and `iRoof.Close()` in the same scan, repeated every scan while triggered (`FB_MonetMqttWatchdog.TcPOU:29-35`). Matches D4. Park then routes like any park | ramp via the park routing; drive to park; covers close; roof close at once (D4; unchanged by D9, always closes) | matches, no PR | automatic (connection back) | `MonetSafeState_Tests.Watchdog_Parks_And_Closes_The_Roof_In_One_Scan` |
| Cabinet over-temperature (#32) | `FB_MonetCabinetControl` `tempCritical` (> 60 C until < 50 C) | `iTelescope.Park()` every scan while critical (`FB_MonetCabinetControl.TcPOU:45-46`); **the roof is not closed** (comment at :44, "this is not weather"). Differs from D4 | ramp via park; drive to park; covers close; hydraulics off; roof: to decide (not in the D9 list; plan 8.6) | roof open (plan 8.6); no PR until decided | automatic (< 50 C) | none (needs an `AT%I*` probe like `FB_MonetPendantProbe`) |
| Manual mode under motion (#118) | MAIN calls `SuspendAutomatic()` (M:1306) | One stop of the moving axes (`F_SuspendStop`, M:1338-1395), queued moves dropped, `bPark` kept, `bInitRequired` (#72), READY_STATE never 1.0 (M:1415) | ramp (#118); no automatic park; operator | matches, no PR | `power=1` back in automatic | `MonetSuspendUnderMotion_Tests`, `MonetTimeouts_Tests.Suspend_Stuck_Stop_Hands_Over_And_Keeps_The_Abort` |
| PLC restart | - | `bIsParked` TRUE at start (BROTLib `FB_BaseTelescopeControl` default), `bParkedInPlace` FALSE, no automatic motion; the Az/El error of the first power-on is exempt from the park and opens the Wake&Shake window (M:174, 547-550, 741-749) | none needed (hydraulics off at start); no automatic motion | matches, no PR | - | `MonetErrorPark_Tests.First_PowerOn_AzEl_Error_Is_Exempt_From_The_Park`, `F_WakeShakeWindow_Tests`, `MonetBrakeRelease_Tests` |

## Step 7: proposed order

Lowest risk and highest value first. Each item is its own PR (MONETcommon unless named otherwise) with its own
on-site check on MONET/S first, then MONET/N, E-stop at hand; each flips its `_Current` test(s) or adds the missing
case. A row is done when its "Today" cell is rewritten to match the target.

1. **`bCloseRoofRequest` output in M** (rows axis, hydraulics, E-stop, safety, power). **Written** on
   `feat/close-roof-request` (2026-11-02, not compiled or run yet; details in "Roof" above). Adds a signal only, no motion
   changes, nobody acts on it yet. Tests: one case per cause (set, latched after the cause is gone, cleared by
   `bReset` only without an active cause; focus and software errors do not set it). On site: watch the output with an
   ADS watch on an E-stop and a hydraulics stop. Release: minor (new output).
2. **Roof close in `MAIN` behind `bRoofCloseOnErrorEnable`** (MONETS PR, then MONETN PR, re-pin of 1). Highest value:
   the roof closes on every serious error. Low risk: closing is the safe direction, the switch turns it off for test
   runs, manual mode does not close. On site: switch on, E-stop in automatic: roof does not move during the E-stop and
   closes after release and restart; hydraulics stop: roof closes; same with the switch off and with the selector on
   manual: no close.
3. **Hydraulics error** (row hydraulics): stop transaction while the pressure holds, else accept the lock; park in
   place; covers close; hydraulics off. Until 0.13.0 the axes kept being commanded against a locked brake.
   **Written** on `feat/hydraulics-error-reaction` (2026-11-02, not compiled or run yet; details in the row and in plan step 7):
   `Hydraulics_Error_While_Tracking_Keeps_Tracking_Current` flipped to `Hydraulics_Error_While_Tracking_Ramps_Then_Parks_In_Place`.
   On site: hydraulics stop (pump off) while tracking and while slewing, and an oil-low error while parked; watch `bHydraulicsRamp`,
   `bStop`, the axis enables and the brake; then the MAIN reset button and `power=1`.
4. **Cover error** (row cover): the park ends without `Closed` (hydraulics off, `bIsParked` semantics to decide in the
   PR) and the power-on leaves stage 0 on a cover error instead of waiting for the 12 h timeout. Removes two hangs,
   little motion. Both `Cover_Error_*_Current` flip. On site: cover error injected at power-on and at park.
5. **Software errors 16#10/16#30/16#40** (row timeouts, and 16#15/16#16 if raised under motion): ramp instead of the
   disable in the error branch (M:312-324 at `c34470f`), then drive to park, covers close, hydraulics off.
   `Goto_Times_Out_Without_Park_Current`, `Home_Times_Out_Without_Park_Current` flip. Touches the error branch every
   error passes, hence after 3-4. **Written** on `feat/timeout-error-reaction` (2026-11-02, not compiled or run yet; details
   in the row and in plan step 7) for the timeouts 16#10/16#30/16#40 only: the flipped tests are
   `Goto_Times_Out_Ramps_Then_Drives_To_Park`, `Home_Times_Out_Ramps_Then_Drives_To_Park`, plus
   `MonetTimeouts_Tests.Slew_Timeout_Ramps_Parks_And_Recovers_With_Reset`. 16#15/16#16 under motion are not included (their row
   is unchanged). On site: a goto, a home and a slew with `stMotion.tStateMachineTimeout` shortened (e.g. 20 s) while the axes
   still move; watch `bTimeoutPark`, `bStop`, the axis enables, the park drive, brake, covers and hydraulics; then `reset=1`
   and `power=1`; once more with `stop=1` during the park drive (axes disabled after the stop).
6. **E-stop and safety error** (rows E-stop, safety): park in place after the lock, hydraulics off, covers after the
   release. `Estop_During_Slew_Disables_The_Axes_Without_Park_Current` flips. Needs the release/restart sequence on
   site (STO, TwinSAFE are not on the rig).
7. **Power failure** (row power): ramp if the drives are powered, else lock; park if possible; hydraulics off.
   `Power_Failure_During_Goto_Keeps_Moving_Current` flips. Last of the serious rows because what the drives (and the
   roof) do on a real phase loss is unknown [unverified]; needs a phase-loss test on site.
8. **Focus error** (row focus): ramp and drive to park if `F_ParkAxesCanMove` instead of parking in place. Changes the
   #86 park path that works today, and parking in place is already safe, so low value for the risk.
9. **Invalid clock** (optional): park before the first sync. Lowest value (the telescope is parked after a restart).

Not in the order: the cabinet over-temperature roof (plan 8.6) and the focus error roof (plan 8.4), until decided.

## Findings that change the plan

- The emergency park already switches the hydraulics off at its end (M:1139-1142), for every park that reaches stage
  80, so the step-7 candidate "hydraulics off at the end of every emergency park" is already true on this path. What
  is missing is the hydraulics-off for the classes that do not park (software errors, hydraulics, E-stop, power).
- Software errors disable the axes under motion without a ramp (M:312-324): today the lock is the stop for them too,
  although the drives could ramp. Fixed for the timeouts 16#10/16#30/16#40 by step 7 PR 5 (ramp, then park).
- A cover error can hold a park forever (stage 80, M:1143) and a power-on until its 12 h timeout (stage 0), and the
  power-on clears `bIsParked` before anything has opened (M:1212). No timeout acts on either (command timeouts only log).
- "Invalid clock during track" is unreachable (the clock validity latches); the row only matters before the first sync.
- The cabinet over-temperature park does not close the roof; D4 says the roof closes.
- Neither `MAIN` closes the roof on a telescope error; decided (D9): M provides `bCloseRoofRequest` for the serious
  classes (written, step 7 PR 1), each `MAIN` closes on it behind an enable switch and only in automatic (PR 2).
