# MONETcommon safe state per error class

Status: **proposed** (step 0 of [plans/2026-11-02-reference-implementation-issue-24.md](../plans/2026-11-02-reference-implementation-issue-24.md),
issue #24 aim 6). Living doc: update a row when its behaviour changes (step 7, one PR per row).
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

Proposed rule for the stop before the hydraulics drop (open point 8.1 of the plan, settle in the step-0 PR): a
controlled stop first (ramp, `FB_MonetStopTransaction`) whenever the drives can still ramp; the brake lock is the stop
only when they cannot (axis error, E-stop/safety error, hydraulics error with the pressure already gone).

Who closes the roof (open point 8.2): this library only closes it in `FB_MonetMqttWatchdog`. Neither site `MAIN`
closes the roof on `bError`, READY_STATE -1.0 or any other telescope signal today [verified: MONETS `MAIN.TcPOU:210`
and MONETN `MAIN.TcPOU:192` are the only roof closes, both through the watchdog; the weather close in MONETS
`MAIN.TcPOU:232-235` is commented out]. Whether MONETRoof closes on its own (weather, power) is not part of this
table [unverified].

## What the telescope block does on any error [verified]

These lines are shared by several rows below.

- `bHwError` is elevation, azimuth, focus or derotator error only (M:283-286); `bError := bHwError OR bSoftwareError`
  (M:287). Hydraulics, covers, E-stop, safety error and `bMainReady` are **not** part of `bError`.
- On `bError` every move flag is dropped and all three axes are disabled in the same scan (M:312-324). No ramp: the
  disable is the stop. An error raised inside a stage method (timeouts) takes effect from the next scan.
- Elevation error: the brake is closed once on the rising edge (#155, M:328-336).
- Brake rule (#121/#151/#155): whenever the elevation is disabled the brake is closed, except in the Wake&Shake
  window, where E-stop, hydraulics error, safety error or covers not open still close it (M:741-769,
  `F_BrakeMustClose`).
- Emergency park (#86): only on the rising edge of `bHwError`, not parked, not in E-stop, not the Wake&Shake fault of
  the first power-on (M:545-554, `F_ErrorParkRequest`, `F_WakeShakeErrorExempt`). With `bError` the park bypasses the
  ready gate (M:606-612) and parks **in place**: brake closed, axes disabled, covers closed (M:1072-1080), then
  hydraulics off once the brake reports closed (M:1139-1142), `bIsParked` once the covers report closed
  (M:1143-1150). `bParkedInPlace` is set (#158).
- Software errors are latched in `bSoftwareError`/`nSoftwareErrorID` until `bReset` (M:1860-1864).
- READY_STATE: -1.0 on `bError OR bSoftwareError OR bPowerFailure OR bEstopTriggered`; online needs
  `bReady AND bMainReady AND NOT bEstopTriggered`; E-stop also aborts the startup (M:801-806, `FB_StartupControl`).
- `bReady` needs homed axes, covers opened, the three axes enabled, the brake open and no error (M:787-792).
- Command timeouts (`tonCommandTimeout`: power-on 12 h, park/gohome/goto/slew 2 min, track 12 h; since plan step 2c the
  `stMotion` fields `tPowerOnTimeout`, `tCommandTimeout`, `tTrackTimeout`, with these defaults) only log
  (`TCSCTOEvent`, M:2016-2019); only the stop timeout acts (stop transaction, M:1970).

## Table

"Today" is what the code does, with the line; "Test" pins it on the mock rig. Where today differs from the target the
test carries the suffix `_Current` (it flips in step 7).

| Error class | Trigger, latched where | Today [verified, file:line] | Target (D4; ramp rule proposed) | Reset path | Test |
|---|---|---|---|---|---|
| Axis error Az/El/Der | `fbAzimuth.Error`/`fbElevation.bError`/`fbDerotator.bError` (M:256-270, 283-286), latched in the axis | All axes disabled (M:312-324), El: brake closed on the edge (M:328-336), Az/Der: brake closed by the #121 rule (M:751-766). Emergency park in place: covers closed, hydraulics off, `bIsParked`, `bParkedInPlace`, READY_STATE -1.0 (M:545-554, 1072-1080, 1139-1150). Not in E-stop, not on a parked telescope, not for the Az/El fault of the first power-on (#113) | brake lock is the stop (drives not controllable); park in place; covers close; hydraulics off; roof close | `reset=1` (axis `Reset()` M:258-269), then `power=1` | `MonetErrorPark_Tests.Elevation_Error_Starts_The_Emergency_Park`, `.Error_During_Estop_Does_Not_Park`, `.First_PowerOn_AzEl_Error_Is_Exempt_From_The_Park`, `MonetBrakeRelease_Tests` |
| Focus error | `fbFocus.Error` (M:271-275, 285) | Same as an axis error: the three axes are disabled (M:312-324) and the emergency park parks **in place** (M:1072), it does not drive to the park position. Plan draft said "drive to park": wrong for today | ramp, then drive to park if `F_ParkAxesCanMove`, else in place; covers close; hydraulics off; roof close | `reset=1`, then `power=1` | `MonetErrorPark_Tests.Focus_Error_Starts_The_Emergency_Park` |
| Software error: timeout goto 16#10, home 16#30, slew 16#40 | `tonGotoTimeout`/`tonHomeTimeout`/`tonSlewTimeout` 600 s (M:486-488); raised in M:875-883, 986-993, 1591-1599; latched `bSoftwareError` | Axes disabled from the next scan, also under motion, no ramp (M:312-324); brake closed by the #121 rule (M:751-766). **No park** (`bHwError` only, M:545), covers stay open, hydraulics stay on. READY_STATE -1.0 | ramp, then drive to park; covers close; hydraulics off; roof close | `reset=1` (M:1863), then `power=1` | `MonetTimeouts_Tests.Slew_Times_Out_After_600_Seconds`, `MonetSafeState_Tests.Goto_Times_Out_Without_Park_Current`, `.Home_Times_Out_Without_Park_Current` |
| Software error: goto/track target 16#15 (below `stMotion.fGotoMinElevation`, default 2 deg) / 16#16 (above `fGotoMaxElevation`, default 89.5 deg) | `F_GotoTargetCheck` in `_GotoTelescope` (M:890-899), latched | Same as the timeouts: axes disabled, no park. Plan draft did not list these | as the timeouts (the telescope has not moved yet: stop, no park needed) | `reset=1` | `F_GotoTargetCheck_Tests` (pure), `.Goto_Window_From_stMotion_Refuses_The_Pole` (rig) |
| Software error: config 16#31 (home/park below horizon) | `F_HomeBelowHorizon`, set **every scan** (M:248-253) | Axes disabled while the configuration is wrong; a reset does not clear it | stays a startup refusal (nothing moves) | fix `telescopeConfig` | `F_HomeBelowHorizon_Tests` (pure) |
| Software error: config 16#32 (invalid `stMotion`, plan step 2c) | `BROTLib.F_MotionConfigValid` FALSE on the values in use (`stMotion` with the deprecated aliases applied, `_MotionConfig`), set **every scan** right after 16#31 (16#32 wins if both) | As 16#31: axes disabled while the configuration is wrong, a reset does not clear it; the defaults of `ST_MotionConfig` stay in use meanwhile, so no timer, window or velocity runs with an invalid value | stays a startup refusal (nothing moves) | fix `stMotion` (or the alias), then `reset=1` | `MonetSafeState_Tests.Invalid_Motion_Config_Is_Software_Error_32` |
| Hydraulics error | `fbHydraulics.Error` (HalfBROT `FB_HydraulicsControl`), read only in M:751-753 and M:758 | Not a telescope error: no `bError`, no stop, no park, the axes stay enabled and commanded (a running track stays dispatched, M:676-679). The brake rule acts only with the elevation disabled. If the pressure goes and the brake locks, `bReady` drops (M:791) and READY_STATE is -2.0 while the axes still track. Plan draft "only the brake rule (#155)" was right but incomplete | ramp in the same scan; the lock wins if the pressure is already gone; park in place; covers close; hydraulics off; roof close | HalfBROT `Reset()` (MAIN reset button) | `MonetBrakeRelease_Tests.Window_Brake_Closes_On_Hydraulics_Error`, `MonetSafeState_Tests.Hydraulics_Error_While_Tracking_Keeps_Tracking_Current` |
| Cover error | `fbCovers.Error` (HalfBROT `FB_CoverControl`, latched there) | **M never reads `fbCovers.Error`** [grep]. Plan draft "`bReady` drops" is wrong: `bReady` reads `Opened` (M:788), and HalfBROT's `Opened` is "all three open switches", independent of the error, so online nothing changes. HalfBROT stops cover motion on the error, so: a power-on waits in stage 0 for `Opened` (M:1214-1217), with `bIsParked` already FALSE (M:1212), until the 12 h command timeout, which only logs; a park stops in stage 80 waiting for `Closed` (M:1143) with axes disabled, brake closed and hydraulics off, `bPark` held, READY_STATE 0.3, never `bIsParked` | ramp; drive to park; covers as far as possible; hydraulics off; roof close; park ends without `Closed` | HalfBROT `Reset()` (MAIN reset button) | `MonetSafeState_Tests.Cover_Error_At_PowerOn_Waits_In_Stage_0_Current`, `.Cover_Error_Holds_The_Park_Before_Parked_Current` |
| E-stop | `bEstopTriggered` (MAIN: `SafetyHandling.estop`) | Axes disabled every scan (M:823-827), brake closed by the #121 rule, no emergency park (M:546), moves rejected (`F_CommandAcceptance` `estop`), READY_STATE -1.0 and the startup is aborted (M:802, 806), Wake&Shake window closed (M:744). Covers not closed, hydraulics stay on, the move flags are not cleared by the E-stop [verified]; how a slew ends on real STO'd axes is [unverified] | brake lock is the stop (STO); park in place; covers as possible after release; hydraulics off; roof close | release, safety reset, `power=1` | `MonetErrorPark_Tests.Error_During_Estop_Does_Not_Park`, `MonetBrakeRelease_Tests.Window_Brake_Closes_On_Estop`, `MonetSafeState_Tests.Estop_During_Slew_Disables_The_Axes_Without_Park_Current` |
| Safety error (no E-stop) | `bSafetyError` (MAIN: `SafetyHandling.error`, #155) | Used only by the brake rule in the Wake&Shake window (M:752, 760); no disable, no park. On site the STO would raise axis errors, which then park [unverified] | as E-stop | safety reset | `MonetBrakeRelease_Tests.Window_Brake_Closes_On_Safety_Error` |
| Power failure / `bMainReady` FALSE | `bMainReady` (MONETS `MAIN.TcPOU:143` `ready` = `PowerMonitoring.phaseOK`, MONETN `MAIN.TcPOU:128` `mainReady`) | Blocks a power-on in stage 0 (M:1215) and its completion (M:1282), **and** gates online: READY_STATE -2.0 while FALSE, back to 1.0 when TRUE again without a new Init (M:805, transient interlock of `FB_StartupReadiness`). A running goto/track goes on; no stop, no park. Plan draft "only blocks power-on" was incomplete. What the drives do on a real phase loss is [unverified] | ramp if the drives are powered, else lock; park if possible; covers as possible; hydraulics off; roof close | automatic when the supply is back | `MonetSafeState_Tests.Power_Failure_During_Goto_Keeps_Moving_Current` |
| Invalid clock | `fbTime.bValid` FALSE (M:229-238) | A goto/slew/track flag set while the clock is invalid takes the stop path (`bStop`, M:236-238) and the move is rejected `clock_invalid` (`_CommandState` M:2487); no park. **Only reachable before the first sync**: BROTLib `FB_AstroClockSync` latches `bValid` TRUE after the first successful read, so "invalid clock during track" cannot happen today. Plan draft "stop, no park" holds for that window only | ramp (stop path); drive to park; covers close; hydraulics off; roof close | automatic (first sync) | `MonetSafeState_Tests.Clock_Invalid_Slew_Takes_The_Stop_Path` |
| MQTT loss (watchdog) | `FB_MonetMqttWatchdog`, 30 s without `bConnected` after the first connect | `iTelescope.Park()` and `iRoof.Close()` in the same scan, repeated every scan while triggered (`FB_MonetMqttWatchdog.TcPOU:29-35`). Matches D4. Park then routes like any park | ramp via the park routing; drive to park; covers close; roof close at once (D4) | automatic (connection back) | `MonetSafeState_Tests.Watchdog_Parks_And_Closes_The_Roof_In_One_Scan` |
| Cabinet over-temperature (#32) | `FB_MonetCabinetControl` `tempCritical` (> 60 C until < 50 C) | `iTelescope.Park()` every scan while critical (`FB_MonetCabinetControl.TcPOU:45-46`); **the roof is not closed** (comment at :44, "this is not weather"). Differs from D4 | ramp via park; drive to park; covers close; hydraulics off; roof close (D4; or keep "not weather", to decide) | automatic (< 50 C) | none (needs an `AT%I*` probe like `FB_MonetPendantProbe`) |
| Manual mode under motion (#118) | MAIN calls `SuspendAutomatic()` (M:1306) | One stop of the moving axes (`F_SuspendStop`, M:1338-1395), queued moves dropped, `bPark` kept, `bInitRequired` (#72), READY_STATE never 1.0 (M:1415) | ramp (#118); no automatic park; operator | `power=1` back in automatic | `MonetSuspendUnderMotion_Tests`, `MonetTimeouts_Tests.Suspend_Stuck_Stop_Hands_Over_And_Keeps_The_Abort` |
| PLC restart | - | `bIsParked` TRUE at start (BROTLib `FB_BaseTelescopeControl` default), `bParkedInPlace` FALSE, no automatic motion; the Az/El error of the first power-on is exempt from the park and opens the Wake&Shake window (M:174, 547-550, 741-749) | none needed (hydraulics off at start); no automatic motion | - | `MonetErrorPark_Tests.First_PowerOn_AzEl_Error_Is_Exempt_From_The_Park`, `F_WakeShakeWindow_Tests`, `MonetBrakeRelease_Tests` |

## Findings that change the plan

- The emergency park already switches the hydraulics off at its end (M:1139-1142), for every park that reaches stage
  80, so the step-7 candidate "hydraulics off at the end of every emergency park" is already true on this path. What
  is missing is the hydraulics-off for the classes that do not park (software errors, hydraulics, E-stop, power).
- Software errors disable the axes under motion without a ramp (M:312-324): today the lock is the stop for them too,
  although the drives could ramp.
- A cover error can hold a park forever (stage 80, M:1143) and a power-on until its 12 h timeout (stage 0), and the
  power-on clears `bIsParked` before anything has opened (M:1212). No timeout acts on either (command timeouts only log).
- "Invalid clock during track" is unreachable (the clock validity latches); the row only matters before the first sync.
- The cabinet over-temperature park does not close the roof; D4 says the roof closes.
- Neither `MAIN` closes the roof on a telescope error; open point 8.2 needs a decision and a `MAIN` line per site.
