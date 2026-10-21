# The switch to manual does not stop moving axes (#118)

Date: 2026-10-21. Status: **plan agreed (section 11), implemented in a PR (0.9.3 after the release). Nothing compiled locally, run on a PLC or run on a telescope.**
Repo: BROTLib/MONETcommon. Base: `develop` = `origin/develop` = `0103577` (0.9.2: #56 reset under motion, #66 part 1, command
timeout per command). Line numbers below are from `0103577`.

Main file: `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU` (FBMTC below). Also read: `FB_MonetStopTransaction`,
`F_ResetMoveAbort`, `F_CommandAcceptance`, `ST_CommandState`, `FB_MonetPendantControl`, `FB_MonetCabinetControl`, BROTLib
`FB_Axis` / `FB_BaseAxis` / `FB_StartupControl` (`develop`), HalfBROT `FB_AxisControl`, `FB_ElevationControl`, `FB_AzimuthControl`,
`FB_DerotatorControl`, `FB_HydraulicsControl` (0.5.6), MONETS `MAIN` (`develop` `bce44f3`) and MONETN `MAIN` (`develop` `dd64349`).
Issues read: #118, #3 (pendant interlock), #72 (manual-to-automatic policy), BROTLib#92. Plans read: #56
(`2026-10-21-preempted-move-keeps-moving-56.md`, the model for this one), 0.8.5 (`2026-10-20-slew-reset-fixes.md` 2.4), #83/#87
(`2026-10-20-command-acceptance-83-87.md` 3.8), the code review (H2).

Everything in sections 1-2 is **read from the code only**. Section 2.5 lists what only a telescope can answer.

## 0. Summary

- `SuspendAutomatic` (FBMTC 1374-1395) is what MAIN calls instead of the body while the cabinet key is on local
  (`manual`, or `off` without hydraulics). On its first scan it aborts a running power-on. It does not stop anything else, and
  from the second scan on it only publishes READY_STATE.
- So a goto, slew, gohome, park, track or power-on move that is running at the switch is no longer driven, but the axes keep the
  last levels the automatic code wrote: a running `MC_MoveAbsolute`/`MC_Home` runs to its end, and a tracking axis keeps its
  external setpoint generator on the last target.
- Nobody owns the axes exclusively after the switch. The pendant writes `Enable`/`MovePos`/`MoveNeg` of the *selected* axis,
  and only while its enable key is on, and it does that in automatic mode too (#3). Everything else keeps the automatic mode's
  last values.
- **Recommended (option A):** at the switch, stop each moving enabled axis once, through the existing abort stop
  (`bMoveAbortPending` + `FB_MonetStopTransaction`, #84/#96). `SuspendAutomatic` drives that one transaction until it settles,
  then releases `StopAxis` and writes nothing more, so the pendant takes over. Queued automatic moves are dropped and tracking is
  switched off at the switch. If the key goes back to automatic mid-stop, the body continues the same transaction.
- No MAIN change in MONETS/MONETN is needed. BROTLib and HalfBROT do not change.

## 1. Observations

There are none from a telescope. The issue is from the #56 plan (open question 6), which listed it as a non-goal, like the 0.8.5
plan (2.4). The 0.8.5 to 0.9.2 changes did not touch `SuspendAutomatic` beyond `bHwErrorLast` (#86).

## 2. What happens today

### 2.1 The switch, scan by scan (MONETS `MAIN`; MONETN `MAIN` is the same)

- Mode (MONETS `MAIN` 68-73, MONETN 67-71): `KeyLocal` -> `manual` if `HydraulicsControl.Enable`, else `off`; otherwise
  `automatic`. Both `manual` and `off` call `SuspendAutomatic` (MONETS 118-121, MONETN 117-119).
- Order in a scan: `TelescopeControl()` or `SuspendAutomatic()` -> the axis blocks (`ElevationControl`, `AzimuthControl`,
  `DerotatorControl`) -> `PendantControl`. So `SuspendAutomatic`, like the body, sees the axis state of the previous scan, and
  anything it writes to an axis block is read by that block in the same scan, before the pendant writes.
- First scan in manual (`fbStartup.EnterSuspend()` TRUE): `bHwErrorLast := TRUE` (#86); if `bPower OR Initializing`, the
  `_PowerOn` abort (1250-1278): `MoveAxis`/`HomeAxis := FALSE` on all axes, i.e. `CancelMove()`/`CancelHome()`. These only drop a
  request that has not started (BROTLib #63).
- Every scan in manual: `fbStartup.Suspend()` and the READY_STATE heartbeat. Nothing else: no arbitration, no dispatch, no
  `_StopTelescope`, no events except READY_STATE, no telemetry, and `bReady`/`bStopped`/`bError` are not recomputed (they keep
  their last automatic value).
- Back in automatic: `fbStartup.BeginScan()` clears `paused`, and the body continues with whatever flags and `TCS_command`
  were left.

### 2.2 What each running command does after the switch

| Running at the switch | Axis levels left by the automatic code | Result |
|---|---|---|
| goto / slew / gohome (`MC_MoveAbsolute`) | `MoveAxis` TRUE until `MoveDone`, `Position`/`Velocity` of the target | **runs to the old target** |
| track | `Tracking` TRUE, `Position` = last computed target, no longer updated | `MC_ExtSetPointGen` stays enabled and holds the last target: the axis stops following the sky, and `FB_Axis` creeps it to that target at up to 0.009 deg/s |
| park (stage 10-40) | as goto | **runs to the park position**; the park stage machine is frozen, the covers are not closed by it |
| power-on stage 10 (homing) | `HomeAxis` TRUE, `CancelHome()` only | **homing runs to its end** |
| power-on stage 30/40 (move home) | `MoveAxis` TRUE, `CancelMove()` only | **runs to the home position** |
| stop / abort stop (`TCS_command = stop`) | `StopAxis` edges already written; `fbStopTx` frozen | the `MC_Stop` already sent completes (the axis block holds `Execute` until `Done`); an axis that had not been sent its stop yet is not stopped; `bStop`/`bMoveAbortPending` stay TRUE |
| at rest | - | nothing |

And the flags: `bGoto`, `bSlew`, `bGoHome`, `bTrack`, `bPark`, `bParkAfterStop`, `bSupersedeRequest` and the stage machines stay as
they were. On the return to automatic the body resumes them: a goto that was interrupted by manual continues from its stage,
**after the operator may have moved the axes by hand**. That replay is the policy question of #72 ("never resume an aborted startup
or replay queued automatic commands").

Commands that arrive during manual: `fbComm` is called in every mode, so MQTT `Slew()`/`MoveRaDec()`/`Home()` reach the methods.
`F_CommandAcceptance` decides on `bReady`, which is **not recomputed during manual**, so a telescope that was ready at the switch
accepts and queues them. This corrects the #83 plan (3.8), which said the gate sees `bReady` FALSE. `Park()` (MQTT watchdog,
cabinet over-temperature) sets `bPark` as always; it runs after the return.

### 2.3 Who owns the axes after the switch

There is no ownership switch. Three writers act on the same axis blocks:

1. **Last levels of the automatic code** (persist until something overwrites them): `Enable` TRUE on all axes (from the power-on),
   `MoveAxis`, `HomeAxis`, `Tracking`, `StopAxis`, `Position`, `Velocity`.
2. **The pendant** (`FB_MonetPendantControl`), called every scan in every mode. With `IbEnableKey` FALSE it returns at once (81-83)
   and writes nothing (its stale outputs stay, #3). With the key on, for the selected axis only (selector 6/7/8):
   `Enable := IbEnableButton` (128, 140, 151), `MovePos`/`MoveNeg` from the direction buttons (jog, `MC_Jog`). The mode check is an
   empty `IF` (91), and MAIN does not pass `eTelescopeMode`: **the pendant acts the same in automatic mode** (#3).
3. **The cabinet keys, through MAIN**: in `manual`/`off`, MAIN writes `brakeClearing := CabinetControl.AreBreakesCleared()` and
   `HydraulicsControl.bOpenBrake := brakeClearing` every scan (MONETS 75-86). In automatic, `brakeClearing` is TRUE.

Consequences, from the code:
- **The pendant selecting an axis disables it** unless its enable button is held (`Enable := IbEnableButton`). A falling
  elevation `Enable` closes the Az/El brake (`FB_ElevationControl` 50-53, `F_TRIG` -> `CloseBrake()`), and azimuth and elevation
  run `MC_Power` only with `bBrakeOpen` (`FB_AzimuthControl` 56, `FB_ElevationControl` 83). This is #3, not changed here.
- **The clear-brakes key decides whether Az/El stay powered at all in manual.** With `inKeyClearBrakes` FALSE, the switch to local
  resets the brake RS (`FB_HydraulicsControl` 179-182, `RESET1 := ... OR NOT bBrakeClearing`), so `bBrakeOpen` drops, the Az/El
  drives lose `MC_Power`, and the hydraulic brake closes, on axes that may be moving. The derotator has no brake and stays
  powered. Which key position is normal when switching to local is not known from the code (question 6).
- `off` (local key, hydraulics off): the pump is off, so the brake closes and Az/El are unpowered as well.

So the stop of this plan matters for: the derotator always, and Az/El when the clear-brakes key is on. With the key off, Az/El are
already stopped by the brake (today), and the stop transaction counts them as done (disabled, see 2.4).

### 2.4 Can a stop at the switch fight the pendant?

- The stop path writes only `Tracking`, `MoveAxis` (`_StopTelescope` 1646-1651) and `StopAxis` edges (1664-1678). It never writes
  `Enable`, `MovePos` or `MoveNeg`. The pendant never writes `StopAxis`, `MoveAxis`, `HomeAxis` or `Tracking`.
- **But the NC can.** `FB_AxisCommand` holds `MC_Stop.Execute` until `Done`, and Tc2_MC2 rejects other motion commands while
  `MC_Stop` is executing (Beckhoff documentation, not checked on MONET). A pendant jog during the braking would then fail with an
  `MC_Jog` error, which sets the axis `bError` and needs a reset on the pendant. That is the only conflict, and it lasts for the
  braking time plus about 3 scans (question 1).
- The stop transaction sends one stop per axis and never re-arms an axis after `StopDone` (#96). Only `Stop()` (D2, 1985-1987) can
  re-issue, and only while `bStop` is set. So once the transaction has settled and `StopAxis` is released, nothing stops a jog.
- **A stop arriving during manual would fight the pendant.** `Stop()` can be called over MQTT at any time. If `SuspendAutomatic`
  drove every stop, an MQTT `stop` during manual would brake an axis the operator is jogging. So `SuspendAutomatic` must drive only
  the one stop it started at the switch (`bSuspendStop`, 4.1).
- The pendant disabling an axis mid-stop is harmless for the transaction: a disabled axis counts as done and as stopped
  (`FB_MonetStopTransaction` 128, FBMTC 706-708).
- The pendant reset button calls `Reset()` of the selected axis (MC_Reset). On an axis in Stopping this may end the stop without
  `StopDone`; the transaction then settles on the timeout (`bForced`, D1). Inferred from the #96 analysis, not checked.

### 2.5 Interactions

- **Running power-on:** the existing abort stays (bPower FALSE, stage 0, `MarkAborted`). The homing (stage 10) and the move home
  (30/40) now get the stop, as a reset does since 0.9.2 (#56 answer 2). Stopping a homing carries the BROTLib#92 risk
  (`HomeActive` may stick, the next power-on stays at 0.8), the same as for `stop`/`park`/`reset` during homing.
- **Running park:** today it runs on, undriven, to the park position. Recommended: stop it like any other move and keep `bPark`, so
  the park restarts from stage 0 on the return to automatic (question 2). This differs from a reset (#56 answer 1 keeps the park
  running) on purpose: on a reset the automatic code keeps driving the park; in manual nothing would.
- **`bMoveAbortPending` already TRUE (a preemption is stopping) or `bStop` TRUE:** the running transaction continues
  (`bStopTxBegun` stays TRUE, no new `Begin()`), driven by `SuspendAutomatic` instead of the body. The new command it was stopping
  for is dropped with the other move flags. `bParkAfterStop` still becomes `bPark` at settle.
- **Error (`bError` from the last automatic scan):** the body has already disabled the drives (314-333). No stop, as in
  `F_ResetMoveAbort`. An axis error that appears during the suspend stop: the NC goes to ErrorStop by itself; `StopDone` may not
  come, and the transaction ends on the timeout (forced or stuck, D1).
- **Back to automatic mid-stop:** the body runs with `bMoveAbortPending` (or `bStop`) TRUE and `TCS_command = stop`. The arbitration
  selects `stop` again (560-564 or 518-541), `_StopTelescope` does not `Begin()` again (`bStopTxBegun` TRUE), `TCS_command` and
  `eTimeoutCommand` are unchanged, so `tonCommandTimeout` keeps counting the same stop. It settles as an ordinary abort stop.
  A `power=1` right after the return outranks the abort stop, as accepted in the #56 plan (2.2).
- **Reset:** MAIN calls `TelescopeControl.Reset()` only in automatic (MONETS 94-96). A reset during manual does not reach FBMTC;
  `bReset` is not set. D3 (reset of an axis stuck in Stopping) is therefore only available after the return.
- **#86 error park:** unchanged (`bHwErrorLast := TRUE` at the switch).
- **Focus, covers, roof:** not part of this. Focus has its own mode input; a cover move started by a park continues
  (`FB_MonetCoverControl`).

### 2.6 Inferred, not confirmed (needs a telescope)

- That Tc2_MC2 rejects `MC_Jog` while `MC_Stop.Execute` is held, and with which error. Section 2.4 and question 1 depend on it.
- That a tracking axis reports `InMotion` (ContinuousMotion/SynchronizedMotion) and so counts as moving. `bSlewing` (697-699)
  suggests it does, since it excludes `bTracking` explicitly. If it does not, a tracking axis is not stopped, only
  `Tracking := FALSE` (setpoint generator disabled), which also ends the motion.
- What the hydraulic brake closing on moving Az/El does (clear-brakes key off, 2.3), and whether that is how operators switch.
- Braking distances on MONET/S (measured in step 0 of the #56 plan, if that has run by then).
- `HomeActive` after a stop during homing (BROTLib#92).

## 3. Where the fix belongs, and the release chain

Everything is in **MONETcommon** (FBMTC plus one pure function). MONETS/MONETN `MAIN`, BROTLib and HalfBROT do not change.
1. MONETcommon `develop` -> **0.9.3** (`ProjectVersion`, `Global_Version`).
2. MONETS: pin `MONETcommon` 0.9.2 -> 0.9.3 in `MONETSRuntime.plcproj`, bump `ProjectVersion`, `Update-Libraries.ps1`, activate.
3. MONETN: the same pin bump. A short check there too (step B1 only), because the cabinet keys and brakes are separate hardware
   (#72: do not infer MONET/N from MONET/S).
4. IAG50cm: not affected (no MONETcommon, no manual mode).

**Order:** step A (0.9.2, section 7) first. It confirms the gap and shows the brake behaviour of the clear-brakes key before any
code changes.

## 4. Options and recommendation

**Recommended: A, one stop per moving enabled axis at the switch, through the abort stop, then release to the pendant.**

- **A (recommended).** On the first suspended scan, if an enabled axis moves (or a stop is already running) and there is no error:
  set `bMoveAbortPending`, `TCS_command := stop`, and let `SuspendAutomatic` drive `_StopTelescope` and the settle logic every
  scan until the transaction settles. Then release `StopAxis` and stop writing. In the same first scan, drop the queued move flags,
  reset the stage machines and switch tracking off, so that the manual mode starts from defined axis states. Reuses
  `FB_MonetStopTransaction` and its #96 guards (one edge per axis, latched-level re-arm, `StopDone` + 3 settle scans, D1 timeout).
- **B, rejected: write `StopAxis := TRUE` (or call `fbAxis.Stop()`) once in the first scan.** A level latched from an earlier stop
  gives no edge, so nothing would stop (the #84 step 0 finding). Nothing waits for `StopDone`, nothing releases the level, and a
  latched `StopAxis` is left for the pendant phase.
- **C, rejected: keep calling the body in manual, with a "manual" input that idles the arbitration.** Needs MAIN changes on both
  sites. The body writes the axes every scan (`MoveAxis := FALSE`, tracking, `Enable := FALSE` on an error), so it would fight the
  pendant for the whole manual session, and it changes the readiness semantics that #72 is about.
- **D, rejected: disable the drives at the switch (`Enable := FALSE`).** No ramp; a falling elevation `Enable` closes the Az/El brake
  on moving axes (2.3); the pendant cannot jog until the operator re-enables through it.
- **E, considered: no stop, only the defined state** (drop flags, `Tracking := FALSE`). This is the issue's fallback ("at least the
  manual mode takes over with defined axis states"). It leaves a slew or park running to its target while the operator holds the
  pendant: tens of seconds of motion no one commanded in manual.

### 4.1 Changes (A)

1. **New pure function `F_SuspendStop : BOOL`** (no axis access, tested offline):
   - Inputs: `bError`, `bStopRunning` (`TCS_command = stop` from the last automatic scan, i.e. a stop or abort stop was running),
     `bAzMoving`, `bElMoving`, `bDrMoving` (each `enabled AND (InMotion OR InRamp OR Homing)`, as for `F_ResetMoveAbort`).
   - Result: `NOT bError AND (bStopRunning OR bAzMoving OR bElMoving OR bDrMoving)`.
   - Differs from `F_ResetMoveAbort` in two points: a running park is stopped (no `bParkRunning` exemption, question 2), and a
     running stop is continued even if no axis reports motion any more (it is braking: `Stopping`, not `InMotion`).
2. **Extract, unchanged, from the body into private methods** so that body and `SuspendAutomatic` share one implementation:
   - `_StopTxInputs()`: the input block 178-196.
   - `_StopTxMonitor()`: `bStopped := ...` (706-708) and 710-716 (`bStopped`/`bTimeout` into `fbStopTx`, `Monitor()`, forced count).
   - `_StopTxRelease()`: the settle block 752-766 (`bStop`, `bMoveAbortPending`, `StopAxis := FALSE`, `bParkAfterStop`).
   - `_StopEvents()`: `TCSstopEvent`, `TCSCTOEvent`, `MoveAbortEvent`, `StopForcedEvent`, `StopStuckEvent` (827-833, 864-867, 875-890).
   The body calls them at the same places as today; no reordering.
3. **New VARs:** `bSuspendStop : BOOL` (the switch to manual started or took over a stop that `SuspendAutomatic` drives),
   `bSuspendStopStarted : BOOL` (one scan, logged), `SuspendStopEvent : FB_EventLog`.
4. **`SuspendAutomatic`, first scan** (inside `IF fbStartup.EnterSuspend()`, after the existing power-on abort):
   - `_StopTxInputs()`.
   - Drop the queued moves and reset the stage machines, as the reset does (#85): `bGoHome`, `bGoto`, `bSlew`, `bTrack`,
     `bSupersedeRequest := FALSE`; `bInterrupted := TRUE; _HomeTelescope(); _GotoTelescope(); _SlewTelescope(); _ParkTelescope();
     bInterrupted := FALSE`. `bPark` and `bParkAfterStop` are kept (question 2).
   - `bSuspendStop := F_SuspendStop(...)`.
   - If `bSuspendStop`: `IF NOT bStop THEN bMoveAbortPending := TRUE; END_IF`, `TCS_command := E_TCSCommand.stop`,
     `bSuspendStopStarted := TRUE`. Otherwise: `fbAzimuth/fbElevation/fbDerotator.Tracking := FALSE`, `MoveAxis := FALSE`,
     `TCS_command := no_command`.
5. **`SuspendAutomatic`, every scan while `bSuspendStop`:**
   - `_StopTxInputs()`.
   - Timeout as in the body (651-656, 665): reset on a `TCS_command` change, then `tonCommandTimeout(IN := TRUE, PT := T#15S)`.
   - `_StopTelescope()` (clears `Tracking`/`MoveAxis`, `Begin()` only if not begun, one `StopAxis` edge per moving axis).
   - Jog block (question 1): `MovePos := FALSE; MoveNeg := FALSE` on the three axis blocks. The axis blocks read this before the
     pendant writes again (2.1), so no MAIN change is needed.
   - `_StopTxMonitor()`; `_StopEvents()`; `SuspendStopEvent` (HINT, one scan: 'switch to manual under motion: stopping axes').
   - If `fbStopTx.bSettled`: `_StopTxRelease()`, `fbStopTx.Cancel()`, `bStopTxBegun := FALSE`, `TCS_command := no_command`,
     `bSuspendStop := FALSE`. `MoveAbortEvent` logs 'axes stopped' in this scan.
   - If `fbStopTx.bStuck` (15 s, an axis still not at standstill, question 4): `bSuspendStop := FALSE`, `StopAxis := FALSE` on all
     three, `fbStopTx.Cancel()`, `bStopTxBegun := FALSE`. Keep `bMoveAbortPending`/`bStop` TRUE, so the body stops again after the
     return. `StopStuckEvent` has logged the ERROR.
6. **Body, first statement after `BeginScan`:** `bSuspendStop := FALSE` (the suspension is over; the body owns the stop from now).
   Nothing else in the body changes: the continuation in 2.5 works with today's arbitration.
7. **Stops that arrive during manual are not driven.** A `Stop()` over MQTT during manual sets `bStop` as today and is processed after
   the return. `SuspendAutomatic` drives only the stop it started at the switch (2.4).
8. **Comments:** `SuspendAutomatic` header, the `bMoveAbortPending` comment (94-96: "also set at the switch to manual"), and the
   lifecycle note in `_StopTelescope`.

Size: about 45 new lines in FBMTC, about 40 lines moved into four methods, a 10-line function. Risk: medium. It adds motion
control to a path that had none, and it runs in a mode in which the pendant is active.

**Not included:**
- Rejecting moves that arrive during manual (2.2, question 3). Recommended for #72.
- Gating the pendant on the mode, zeroing its stale outputs, disconnect detection (#3).
- Keeping the Az/El brake open until standstill when the clear-brakes key is off (2.3, question 6; HalfBROT/MAIN).

## 5. Backward compatibility, ADS watcher lists, log texts

- **Public interface:** unchanged (methods, properties, `I_AltAzTelescope`). MAIN unchanged. READY_STATE in manual unchanged
  (`fbStartup.Suspend`).
- **New behaviour:**
  - The switch to local while an enabled axis moves (goto, slew, gohome, track, park, power-on stage 10/30/40, or a running stop)
    brakes the moving axes with `MC_Stop` (NC stop deceleration) instead of letting them run on.
  - The pendant cannot jog during that stop (if question 1 is answered yes). After the stop it can.
  - Tracking ends at the switch (`Tracking := FALSE` on all axes), also at rest.
  - Moves queued before the switch are dropped; they no longer resume on the return. `bPark` is kept, so a park restarts.
- **Timing:** braking time plus 3 scans (30 ms) after the switch, at most 15 s (timeout).
- **ADS symbols:** new `bSuspendStop`, `bSuspendStopStarted`; no removals. Watcher and recording lists: add `bSuspendStop`, and for
  the on-site steps `MAIN.eTelescopeMode`, `MAIN.brakeClearing`, `CabinetControl.inKeyClearBrakes`, `PendantControl.IbEnableKey`,
  `PendantControl.nSelection`. Also in the MONET test plan in the IAG repo (`specs/plans/2026-10-12-monet-telescope-test-plan.md`).
- **Log texts:** new HINT 'switch to manual under motion: stopping axes' (question 5). Reused: 'move interrupted, stopping axes' /
  'axes stopped', and on a timeout 'STOP timeout: an axis is not at standstill' / 'A timeout occurred during STOP command!'.
  These now also appear while the key is on local. Check pyobs and log filters.
- **Persistent data:** none new (`nStopForcedCount` may now also count a suspend stop).

## 6. Tests (MONETcommonTests, TcUnit, CI simulation only, currently 102 tests)

`FB_MonetTelescopeControl` still cannot be instantiated in a test (`REFERENCE TO` the HalfBROT axis FBs, #36). `SuspendAutomatic`,
the extracted methods, the jog block and the continuation into automatic are **not testable offline**; section 7 covers them.

**Suite `F_SuspendStop_Tests`** (new, about 7 tests):
- `Moving_Axis_Stops`: Dr moving, no error -> TRUE.
- `Homing_Axis_Stops`: El homing (power-on stage 10) -> TRUE.
- `Park_Is_Stopped`: Az moving, nothing else (the caller passes no park exemption) -> TRUE. Documents the difference to
  `F_ResetMoveAbort`.
- `Running_Stop_Continues`: no axis moving, `bStopRunning` -> TRUE (braking axes report Stopping, not motion).
- `At_Rest_No_Stop`: nothing moving, no stop running -> FALSE.
- `Error_No_Stop`: moving and `bError` -> FALSE.
- `Error_Running_Stop_No_Stop`: `bStopRunning` and `bError` -> FALSE (drives are off).

**`StopLoop_Sim_Tests`** (one new test): `Sim_Transaction_Driver_Change_No_Second_Edge`. Begin a transaction, run the mock axes for a
few scans with the request logic called by "driver 1", then keep calling without `Begin()` ("driver 2", the return to automatic).
Expect one `StopAxis` edge per moving axis in total and a normal settle. This documents the continuation rule of 2.5 at the
transaction level; it does not test FBMTC.

Unchanged: `FB_MonetStopTransaction_Tests`, `F_ResetMoveAbort_Tests`, `F_CommandAcceptance_Tests`.
Add the suite to `MONETcommonTests` `MAIN` and the README table. CI (`tcbuild.yml`) must pass (about 110 tests).

## 7. On-site check, MONET/S (B1 also on MONET/N)

Rules as in the earlier plans: never open the roof, someone watches the covers. Elevation >= 30 deg, within the axis limits.
The key switch is on the cabinet, so **one person at the cabinet**, one at the pendant (pendant enable key OFF unless a step says
otherwise), one at the recorder. Abort on the first axis error.

Far targets: the first target must lie at least 2 x the braking distance (and not less than 10 deg) beyond the position at the
switch. Use the distances of step 0 of the #56 plan; if that has not run, do it first.

**Record** as for #56 (ADS on-change logger at cycle resolution, else sum-read at 20 Hz):
- `MAIN.TelescopeControl`: `TCS_command`, `bSuspendStop` (0.9.3), `bMoveAbortPending`, `bStop`, `bStopped`, `bPark`, `bGoto`, `bSlew`,
  `bGoHome`, `bTrack`, `bPower`, the `_PowerOn` stage, `fReadyState`, `fbStopTx` (issued/done per axis, `bActive`, `bSettled`,
  `bStuck`, `bForced`), `tonCommandTimeout.ET`.
- `MAIN`: `eTelescopeMode`, `brakeClearing`; `CabinetControl.inKeyLocal`, `inKeyClearBrakes`; `HydraulicsControl.bBrakeOpen`,
  `ObBrakeAzEl`; `PendantControl.IbEnableKey`, `nSelection`, `IbEnableButton`, direction buttons.
- Per axis (Az/El/Dr): `bEnable`, `bMoveAxis`, `bStopAxis`, `bTracking`, `bMovePos`, `bMoveNeg`, `fActualPosition`, and from `fbAxis`:
  `StandStill`, `Stopping`, `Homing`, `InMotion`, `InRamp`, `StopCommand.Execute`, `Axis_Stop.Done`, `Axis_Jog.Error`, `HomeActive`,
  `ErrorID`.
- MQTT READY_STATE and the log topic.

**Step A, on 0.9.2 (confirms the gap):**
1. **A0 brake key, no motion.** At rest, automatic, then key to local with clear-brakes ON, then OFF, then back. Note
   `bBrakeOpen`, `ObBrakeAzEl`, Az/El `fbAxis.Ready`. Ask the operators which position is normal when they switch to local.
   This answers 2.3 without moving under a closing brake.
2. **A1 manual during a slew** (clear-brakes ON). Far slew, key to local 3 s in.
   - Expect on 0.9.2: the axes run on to the slew target; no log line. That confirms #118.
   - Key back to automatic: expect the slew stage machine to continue (2.2). `stop=1` if anything moves unexpectedly.
3. **A2 manual during a track** (clear-brakes ON). Track for 1 min, key to local.
   - Expect `bTracking` TRUE on the axes, the axes hold the last target (no longer follow the sky).
   - Note whether the tracking axes report `InMotion` (2.6).
4. Do **not** switch to local under motion with clear-brakes OFF on purpose (hydraulic brake on moving axes). If the operators say
   that is their normal way of switching, stop here and answer question 6 first.

**Step B, on 0.9.3:**
1. **B1 manual during a slew** (clear-brakes ON, pendant key OFF). Repeat A1.
   - Expect in the first suspended scan `TCS_command = stop`, `bSuspendStop` TRUE, one `StopCommand.Execute` edge per moving axis,
     rest short of the target, 'switch to manual under motion: stopping axes', 'move interrupted, stopping axes', 'axes stopped'.
   - Expect `bSuspendStop` FALSE, `StopAxis` FALSE on all axes, `bGoto`/`bSlew` FALSE afterwards.
   - Then pendant key ON, select azimuth, hold enable, jog left and right: expect normal jogs, no error.
   - Key back to automatic: expect no motion, READY_STATE not online until `power=1` (#72 policy unchanged).
   - Repeat 3 times. Also on MONET/N once.
2. **B2 manual during a track.** Repeat A2. Expect `Tracking` FALSE, a stop on the tracking axes (or only the setpoint disable if
   they do not report motion), no setpoint generator error.
3. **B3 jog during the stop** (question 1). As B1, but with the pendant key ON, azimuth selected, enable held, and a direction
   button held from before the switch.
   - Expect `bMovePos`/`bMoveNeg` FALSE at the axis while `bSuspendStop`, no `Axis_Jog.Error`, the jog starting after 'axes stopped'.
   - Without the jog block (if question 1 is answered no): watch for `Axis_Jog.Error`; this tests 2.6.
4. **B4 back to automatic mid-stop.** As B1, key back to automatic within about 1 s.
   - Expect `TCS_command` to stay `stop`, no second edge per axis, the stop settling in automatic, 'axes stopped' once.
5. **B5 manual during the power-on move home** (#66 setup: slew to az 140 / el 55, `reset=1`, `power=1`, key to local 3 s into the
   move home). Expect the stop short of az 175 / el 40, READY_STATE -2.0 in manual. Back to automatic, `power=1` -> 1.0.
6. **B6 manual during a park.** `park=1`, key to local 3 s into the park move.
   - Expect the stop (question 2), `bPark` still TRUE. Covers: note what they do (not changed by this).
   - Key back to automatic: expect 'PARK commencing' again and the park to end parked.
7. **B7 MQTT stop during manual** (2.4). At rest in manual, pendant jogging azimuth, send `stop=1`.
   - Expect no `StopAxis` edge on the jogging axis while in manual; `bStop` stays TRUE and settles after the return.
8. **B8, only on request: manual during the stage 10 homing** (uncalibrated axes, Wake&Shake context). Expect the homing axes to stop;
   watch `HomeActive` (BROTLib#92) and report there.

Comment the results on #118 (and B7/B3 on #3). Close #118 when B1-B4 and B6 pass.

## 8. Not verified

- Nothing in this plan has been compiled or run. No switch to manual under motion has been recorded on either MONET.
- Tc2_MC2 behaviour of `MC_Jog` during `MC_Stop`, of a tracking axis in `MC_ReadStatus`, and of `MC_Stop` during `MC_Home`.
- The physical meaning and normal position of the clear-brakes key, and what the hydraulic brake does to moving Az/El axes.
- That the MONETN cabinet and pendant are wired like MONETS (same MAIN code; the hardware is not checked).
- That the ordering argument of 2.1 (FBMTC before the axis blocks before the pendant) holds after future MAIN edits. It is true in
  `bce44f3` (MONETS) and `dd64349` (MONETN).

## 9. Risks

1. **Braking at the switch.** `MC_Stop` with the NC stop deceleration on moving axes, with the same 0x4550/0x4260 risk as any stop
   (HalfBROT#77). Mitigation: B1 three times, abort on the first axis error.
2. **Jog block hides a pendant command** for up to the braking time (or 15 s on a stuck stop). The pendant shows nothing for it.
   Mitigation: the HINT log line; the release on timeout (question 4).
3. **Stop during homing** (BROTLib#92), as for reset/stop/park.
4. **Code in a mode that had none.** A bug in the suspend path acts while an operator is at the telescope. Mitigation: the path is
   gated on `bSuspendStop`, which is set only in the first suspended scan and cleared by settle, timeout and the body.
5. **Untested offline** (#36). All integration evidence comes from section 7.
6. **Dropping queued moves** changes what happens on the return (they no longer replay). That is wanted (#72), but a client that
   relied on a goto resuming after a short manual interval will see it gone.

## 10. Open questions for the user

1. **Jog during the stop:** block the pendant jog on all three axes until the suspend stop settles (recommended: the NC would
   reject the jog with an axis error anyway, and the block needs no MAIN change), or leave the pendant untouched?
2. **Park running at the switch:** stop it and keep `bPark`, so it restarts on the return to automatic (recommended: in manual
   nothing would drive the park), or let it run on as today, or stop it and drop `bPark`?
3. **Moves that arrive during manual** are accepted and queued today (stale `bReady`, 2.2). Fix in #72 together with the return policy
   (recommended, one concern per change; comment the finding on #72 and correct the #83 plan), or reject them here with a new reason
   `manual_mode` in `F_CommandAcceptance`?
4. **Stop timeout in manual (15 s, an axis still moving):** release the axes to the pendant and keep `bMoveAbortPending`, so the
   body stops again after the return (recommended), or keep holding the stop (and the jog block) until settled?
5. **Log text:** one-scan HINT 'switch to manual under motion: stopping axes', plus the existing 'move interrupted, stopping axes' /
   'axes stopped' (recommended), or only the existing lines?
6. **Clear-brakes key:** which position is normal when the operators switch to local? If OFF, Az/El are braked by the hydraulic brake
   with the drives unpowered today, and this change only adds the derotator stop. Recommended: answer with step A0, and if OFF is
   normal, file a separate issue to keep the brake open (drives powered) until the suspend stop has settled.
7. **Homing at the switch (stage 10):** stop it, as decided for the reset in #56 (recommended), accepting the BROTLib#92 risk?
8. **MONET/N:** run B1 there too before closing (recommended, #72 asks not to infer N from S), or rely on MONET/S?

## 11. Answers of the user

All eight questions: as recommended (1 block the jog until the stop has settled, 2 stop a running park and keep `bPark`, 3 moves arriving during manual go to #72, 4 release on the 15 s timeout and keep the abort stop, 5 one-scan HINT, 6 step A0 decides, 7 stop a homing, 8 B1 on MONET/N too).

Implementation notes: the stop is driven by `SuspendAutomatic` through `bSuspendStop`; the four shared blocks are the private methods `_StopTxInputs`, `_StopTxMonitor`, `_StopTxRelease` and `_StopEvents` (the event calls moved together to the place of `TCSstopEvent`, so their order in the log within a scan can differ). `bStopRunning` of `F_SuspendStop` is `TCS_command = stop AND (bStop OR bMoveAbortPending)`. The simulation test `Sim_Transaction_Continues_After_Driver_Change` documents the rule only; the rig has a single call path, so it does not exercise the hand-over between `SuspendAutomatic` and the body.
