# MONETcommon 0.8.5: interruptible moves and a reset that really resets (#84, #77, #85, #88)

Date: 2026-10-20. Status: plan only, no code changed.
Repo: BROTLib/MONETcommon. Base: `origin/develop` @ `80b80b1` (0.8.4).
Note: branch from `origin/develop`, not from `main`.

Main file: `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU`. All line numbers below are
**file lines of that .TcPOU on develop 80b80b1** (the XML lines, the form the issues use).

Related code outside this repo (read-only references, no change planned here):
- HalfBROT develop `FB_AxisControl.MoveAxis.Set`: FALSE->TRUE calls `fbAxis.MoveTo()`, FALSE only calls
  `fbAxis.CancelMove()`, which "drops a pending MoveTo() that has not started yet. A running move is not stopped."
- HalfBROT `FB_AxisControl.StopAxis.Set`: FALSE->TRUE calls `fbAxis.Stop()`; the axis block clears `bStopAxis` on `StopDone`.
- BROTLib develop `FB_Axis` (0.8.0): `Stop()` cancels pending move/home and requests `MC_Stop` through
  `FB_AxisCommand` (`Allowed := StandStill OR ContinuousMotion OR DiscreteMotion OR SynchronizedMotion OR Homing`).
  `MC_Stop` is called without `Deceleration`/`Jerk`, so it uses the NC axis defaults.

---

## 1. Root causes

### #84: stop / gohome / track do not interrupt a running slew

Three separate defects add up to "the old move runs to its end":

**(a) An interrupt never stops the axes (gohome/goto/track during a slew or goto).**
Arbitration, park branch L350-358, gohome branch L370-376, goto branch L389-393: the lower command's flag is
cleared and its state machine is called once with `bInterrupted := TRUE`, which only resets `nStage`
(`_GotoTelescope` L651-654, `_SlewTelescope` L1209-1212, `_HomeTelescope` L729-731). Nobody touches the axes.
`MoveAxis` stays TRUE in the axis blocks, and the running `MC_MoveAbsolute` continues to its target. The new
command then waits: gohome needs `bStopped` (L379, `_HomeTelescope` L743), goto needs `NOT bBusy` (L395, L658).
Note that `Track()` (L1593-1596) sets `bGoto`, so "track during a slew" is the goto branch. This matches
HalfBROT#52 check 3 / #56 / #63 check 3 exactly.

**(b) The stop path does not reliably reach `MC_Stop`, and it leaves the state machines half-reset.**
- `_StopTelescope` L1282-1316 is a 3-way IF/ELSIF: scan 1 only clears `Tracking`, the next scan only clears
  `MoveAxis` (`CancelMove`, so a running move is not stopped, see above), and only on a later scan
  `StopAxis := TRUE` is set, and only for axes that are `NOT StandStill`. `StopAxis` is a level that the axis
  block clears on `StopDone`. If `bStopAxis` is still TRUE from an earlier stop whose `MC_Stop` never reported
  Done (for example a request that never became `Allowed`, or one dropped by `Reset`), the setter sees no
  FALSE->TRUE edge and **no `fbAxis.Stop()` is issued at all**. That matches run 2 (bStop TRUE for 9.3 s, axes
  still accelerating). The code alone does not prove which of these it was. Step 0 below confirms it with a scope.
- In the stop branch L324-336 the stage methods are called with the flag cleared but **without**
  `bInterrupted`, so their `nStage` is not reset (only park/power reset on `bStop`, L788 / L906). A goto or slew
  stopped at stage 50 stays at stage 50. The next goto/slew starts there: stage 50 waits for a `MoveDone` that
  already happened. In `_SlewTelescope` stage 50 goes to 100 as soon as `NOT bSlewing AND bStopped`, so the
  next slew can "complete" without moving at all. This is a possible second cause of the silent slews in #83.
- It is IF/ELSIF: only one of `bPark/bGoto/bSlew/bGoHome` is cleared per scan.
  On the stop scan this is mostly harmless, because `bStop` stays TRUE until `bStopped`. It does matter when the
  stop clears in the same scan.

**(c) The 2 s stop timeout is too short.** L428 `tonCommandTimeout(PT := T#2S)` for STOP. From about 9 deg/s
the deceleration plus settling takes longer than 2 s, so the timeout fires during every real stop. The timeout
itself only logs (`TCSCTOEvent` L585), it does not disable drives. The axis errors in run 1 (0x4550 following
error on elevation, then 0x4260 when L175-179 disables all axes on an elevation error) come from the mechanics
or the NC (HalfBROT#77: a hard reversal or deceleration with high elevation torque), not from the timeout.
`MC_Stop` with the NC default deceleration may be harder than the move's own ramp. That is a BROTLib/NC
parameter question (see Risks).

### #77: reset cannot clear 0x15 / 0x16

`_GotoTelescope` L618-649: the timeout check (0x10), `fElevationCalc<2.0` (0x15) and `fElevationCalc>89.5`
(0x16) all run **before** the `IF bReset OR bError OR bInterrupted` branch at L651. The body's reset block
L107-119 first clears `bSoftwareError` (L110) and then calls `_GotoTelescope()` (L115), which sets it again
from the stale target. `bReset` is only cleared when `NOT bError` (L508), so it never clears. The checks also run
on calls where no goto was requested at all (reset, park/gohome interrupts, the stop path).
`_SlewTelescope` (L1200-1208, timeout before the reset branch) has the same ordering for 0x40. It is harmless
today, because the timer resets when `bSlew` drops, but it should be fixed the same way.

### #85: reset does not clear a pending bGoHome

- The reset block L107-119 resets the stages only; no command flag is cleared.
- The error block L164-170 clears `bTrack/bGoto/bSlew` but **not `bGoHome`** (and not `bPark`, which is
  intended: emergency park).
- `_HomeTelescope` L729-731 returns early on reset/interrupt without clearing `bGoHome`. `bGoHome` is only
  cleared when the move finishes (L758), on the home timeout (L738), in the stop branch (L334), or in park
  stage 0 (L805).
- After power-on `bReady` becomes TRUE and the arbitration (L367-383) starts the stale home move.

### #88: nSoftwareErrorID / bSoftwareError stay set after reset

The reset block L110 clears `bSoftwareError` but not `nSoftwareErrorID`. `nErrorID` is cleared by L172
whenever `NOT bError`, but `nSoftwareErrorID` is only overwritten by the next software error.
(`bSoftwareError` itself then stays set only through the #77 re-raise.)

### For assessment only: #83, #86, #87

- **#83 (slew while parked)**: the slew branch L401-405 only dispatches with `bReady AND NOT bBusy`.
  Otherwise `bSlew` stays pending, without a log, until the 600 s slew timeout (0x40) or the next power-on
  (the same "stale command" class as #85/#87). The observed fast clear of `bSlew` was a test-command syntax
  error. The stage-50 leftover from 1(b) is a possible second route.
- **#86 (axis error alone does not emergency-park)**: the emergency branch (`_ParkTelescope` stage 0 L810-816)
  is reached only through `bPark` (L342-348). `bError` never sets `bPark`.
- **#87 (park during power-on)**: `bPower` (L339) outranks `bPark` (L342). `bPark` stays TRUE and is
  dispatched as soon as the power-on ends.

---

## 2. Proposed changes (all in FB_MonetTelescopeControl.TcPOU unless noted)

### 2.1 #77 + #88: guards after the reset branch, reset clears the software error completely

`_GotoTelescope`: move the reset/interrupt branch to the top, and run the target checks only for a real goto:

```st
_GotoTelescope := nStage;
// #77: a reset/interrupt/stop call only resets the stage; it must never raise an error from a stale target
IF bReset OR bError OR bInterrupted OR NOT bGoto THEN
	nStage := 0;
	RETURN;
END_IF
IF tonGotoTimeout.Q THEN ... (unchanged, 0x10) END_IF
IF fElevationCalc < 2.0 THEN ... (unchanged, 0x15) END_IF
IF fElevationCalc > 89.5 THEN ... (unchanged, 0x16) END_IF
CASE nStage OF ...
```

`NOT bGoto` also covers the stop-branch call (L329, flag already FALSE), so the stage resets there too
(this is part of the #84 fix (b)). The dispatcher only calls `_GotoTelescope` while `bGoto` is TRUE, so normal
operation is unchanged.

`_SlewTelescope`: same reorder (`IF bReset OR bError OR bInterrupted OR NOT bSlew THEN nStage := 0; RETURN;`
before the 0x40 timeout check).
`_HomeTelescope`: the reset branch is already first. Add the `NOT bGoHome` handling as in 2.3.

Body reset block (L107-119):

```st
IF bReset THEN
	fbStartup.MarkAborted();
	bHorizonError := FALSE;
	bSoftwareError := FALSE;
	nSoftwareErrorID := 0;			// #88
	...
```

`bHomeBelowHorizon` (L123-128) keeps re-raising 0x31 every scan, as intended (configuration error).

### 2.2 #84: interrupts and stop really stop the axes

**New VAR** (in the FB's VAR block):

```st
	// #84: a running move was interrupted (gohome/goto/park/track over a slew or goto): stop the axes
	// before the new command may start. Cleared at standstill, on reset and on an error.
	bMoveAbortPending	: BOOL;
	MoveAbortEvent		: FB_EventLog;
```

**Interrupt branches** (L350-358, L370-376, L389-393): after the one-shot `bInterrupted` calls, add
`bMoveAbortPending := TRUE;`. Also set it in the stop branch, so that a stop and an interrupt both use the
same "stop until standstill" mechanism.

**Arbitration**: insert directly after `ELSIF bPower` and before `ELSIF bPark`, but let the emergency park pass:

```st
ELSIF bMoveAbortPending AND NOT bError THEN
	// #84: the interrupted move is still running; stop it before the next command is dispatched.
	// Flags of the new command (bGoHome/bGoto/bPark) stay set and run once bStopped.
	bTrack := FALSE;	// a pending track would otherwise start on the moving axes
	TCS_command := E_TCSCommand.stop;
```

Leave `bPark AND bError` unaffected: on an error the axes are disabled by L164-170 anyway, so the abort has
nothing to wait for. The error clear also drops it: in the `IF bError THEN` block (L164) add
`bMoveAbortPending := FALSE;`.

Lifecycle of `bMoveAbortPending` (summary of the rules below):
- **Set** by the interrupt branches and by the stop branch.
- **Set while `bPower` outranks it** (an interrupt in the scan before a power-on is accepted, or a power-on
  accepted while an interrupted move is still decelerating): the flag stays set for the whole power-on, because
  the `bPower` branch is above it and nothing in the power-on clears it. When the power-on ends, the abort branch
  dispatches STOP once; at standstill it clears and the pending command runs. This is accepted: the power-on is
  never blocked, and the extra STOP after it only stops axes that should not be moving anyway.
- **Cleared** only (a) at standstill **and** after STOP was dispatched (refinement below), (b) in the error
  block, (c) in the reset block. Nothing else clears it.

Clear it at standstill, next to `bStop` (L504):

```st
IF bStopped THEN
	bStop := FALSE;
	bMoveAbortPending := FALSE;	// #84
END_IF
```

Edge case: an interrupt in the very scan the old move starts (axes still at standstill, `MoveTo` pending).
Here `bStopped` is TRUE and the abort would clear before the axes start. `_StopTelescope` (below) clears
`MoveAxis`, which cancels the pending `MoveTo`, in the same scan. The reset of the abort flag has to happen
**after** the stop branch has run at least once, so the flag may only be cleared on a scan where
`TCS_command = stop` was already dispatched:

```st
IF bStopped AND (TCS_command = E_TCSCommand.stop) THEN ...	// instead of plain bStopped
```

(The same refinement is safe for `bStop`. Leave `bStop` as is unless the unit test shows the same race.)

Reset block (L107-119): add `bMoveAbortPending := FALSE;`. Reset disables nothing by itself, but after a reset
the stop path is re-entered by any new stop. Clearing it makes sure reset never blocks the next power-on.

**Stop branch** (L313-337): reset all stage machines, not just one:

```st
IF bStop THEN
	fbStartup.MarkAbortedIfStarting(Power := bPower);
	bTrack := FALSE;
	IF bPower OR fbStartup.Initializing THEN
		bPower := FALSE;
		_PowerOn();
	END_IF
	// #84: clear every pending move and reset every stage machine (was IF/ELSIF, and without bInterrupted)
	bPark := FALSE; bGoto := FALSE; bSlew := FALSE; bGoHome := FALSE;
	bInterrupted := TRUE;
	_ParkTelescope(); _GotoTelescope(); _SlewTelescope(); _HomeTelescope();
	bInterrupted := FALSE;	// #26 one-shot
	bMoveAbortPending := TRUE;
	TCS_command := E_TCSCommand.stop;
```

Interaction: the stop branch must keep the power-on abort first (#80). `_PowerOn` itself does not look at
`bInterrupted` here, because `bStop` is already TRUE.

**`_StopTelescope`** (L1282-1316): issue everything in the first scan, re-arm the edge, release at standstill.

New VARs (FB VAR block):

```st
	// #84: this stop request has written its own FALSE->TRUE edge on StopAxis (per axis).
	// Cleared at standstill and whenever no STOP is dispatched.
	bStopIssuedAz	: BOOL;
	bStopIssuedEl	: BOOL;
	bStopIssuedDr	: BOOL;
```

Method body, written out per axis:

```st
// #84: one scan does it all. fbAxis.Stop() cancels a pending MoveTo itself; clearing MoveAxis
// alone (CancelMove) does not stop a running move.
fbAzimuth.Tracking := FALSE;  fbElevation.Tracking := FALSE;  fbDerotator.Tracking := FALSE;
fbAzimuth.MoveAxis := FALSE;  fbElevation.MoveAxis := FALSE;  fbDerotator.MoveAxis := FALSE;

IF bStopped THEN
	fbAzimuth.StopAxis := FALSE;
	fbElevation.StopAxis := FALSE;
	fbDerotator.StopAxis := FALSE;
	bStopIssuedAz := FALSE;
	bStopIssuedEl := FALSE;
	bStopIssuedDr := FALSE;
	_StopTelescope := FALSE;	// unchanged polarity, see below
ELSE
	// Re-arm: a StopAxis level still latched from an earlier stop (step 0) is dropped for one scan,
	// so that the next TRUE is a fresh FALSE->TRUE edge and the setter really calls fbAxis.Stop().
	IF NOT fbAzimuth.fbAxis.StandStill THEN
		IF fbAzimuth.StopAxis AND NOT bStopIssuedAz THEN
			fbAzimuth.StopAxis := FALSE;	// latched from before -> one scan low
		ELSE
			fbAzimuth.StopAxis := TRUE;
			bStopIssuedAz := TRUE;
		END_IF
	END_IF

	IF NOT fbElevation.fbAxis.StandStill THEN
		IF fbElevation.StopAxis AND NOT bStopIssuedEl THEN
			fbElevation.StopAxis := FALSE;
		ELSE
			fbElevation.StopAxis := TRUE;
			bStopIssuedEl := TRUE;
		END_IF
	END_IF

	IF NOT fbDerotator.fbAxis.StandStill THEN
		IF fbDerotator.StopAxis AND NOT bStopIssuedDr THEN
			fbDerotator.StopAxis := FALSE;
		ELSE
			fbDerotator.StopAxis := TRUE;
			bStopIssuedDr := TRUE;
		END_IF
	END_IF
END_IF
```

Sequence for a latched axis: scan 1 `StopAxis` TRUE->FALSE, scan 2 FALSE->TRUE (edge, `fbAxis.Stop()`),
`bStopIssuedX` TRUE, later scans keep writing TRUE. An axis that was not latched gets the edge in scan 1.
If the axis block clears `StopAxis` on `StopDone` before `StandStill` is seen here, the next TRUE is another
edge and a second `Stop()`; that is harmless (`MC_Stop` is allowed at standstill).

`bStopIssued*` must not survive a STOP that ended without standstill (TCS_command changed by reset, error or a
new command): otherwise the next stop sees `StopAxis` latched AND `bStopIssued` TRUE and writes TRUE without an
edge, the original bug. So clear all three in the body whenever STOP is not dispatched, next to the call at L426:

```st
ELSIF TCS_command = E_TCSCommand.stop THEN
	_StopTelescope();
	...
END_IF
IF TCS_command <> E_TCSCommand.stop THEN
	bStopIssuedAz := FALSE; bStopIssuedEl := FALSE; bStopIssuedDr := FALSE;	// #84: re-arm for the next stop
END_IF
```

Return value: keep the current polarity (`_StopTelescope := FALSE` when `bStopped`, L1314). Its sole caller
(L427) ignores it. Alternatively drop the return value (`METHOD PRIVATE _StopTelescope`) in the same commit.
Do not change it to TRUE.

Step 0 showed `bStopAxis` latched TRUE at idle, so the re-arm is required (the plain "if not standstill,
StopAxis := TRUE" is not enough).
Tracking, MoveAxis and Stop in one scan is fine: `FB_Axis` already blocks the tracking request while a stop
is pending or executing (BROTLib FB_Axis L178).

Test note (unit test of the extracted logic, section 5, and the scope in hardware step 2): with `StopAxis`
latched TRUE before the stop, `StopAxis` must go **low for exactly one scan and then high** (a fresh
FALSE->TRUE edge), and `fbAzimuth.fbAxis.StopCommand.Execute` / `Axis_Stop.Busy` must follow.

**Stop timeout** L428: `T#2S` -> `T#15S`, **provisional**. 15 s is an estimate (deceleration from 10 deg/s
plus settle), not a measurement: step 0 only covered the latched `bStopAxis`, no `MC_Stop` ran. Measure the
time from `stop=1` to `bStopped` at the highest slew speed on 0.8.5 (hardware step 2) and set the final value
to that time plus a margin before the release. The 'NOCOMMAND' text in the event is misleading too: set
`TCSCTOEvent.OnMessage` before the TON call so that the message names STOP.
Check: with `bMoveAbortPending` dispatching STOP, the timeout now also applies to interrupts. The same value
is fine there too.

**Event**: `MoveAbortEvent(Trigger := bMoveAbortPending AND NOT bStop, Level := HINT, FormatString := '%s',
OnMessage := 'move interrupted, stopping axes', OffMessage := 'axes stopped', Comm := fbComm);`

### 2.3 #85: reset and error drop stale move commands

Reset block (L107-119), before the stage resets:

```st
	// #85: a reset drops every pending move command; nothing may move later because of a command sent
	// before the reset. bPark is kept on purpose (safe direction, emergency park, see Risks).
	bGoHome := FALSE;
	bGoto := FALSE;
	bSlew := FALSE;
	bTrack := FALSE;
	bMoveAbortPending := FALSE;
```

Error block (L164-170): add `bGoHome := FALSE;`. The horizon guard (L271-277) runs later in the same scan and
sets `bGoHome` again if the telescope is really below the horizon, so the guard keeps working. While
`bError` is TRUE it is dispatched only after the reset, which then clears it again. The guard re-sets it on
the next scan if the condition still holds, which is what we want: the horizon guard is a live condition,
not a stale command.

Power-on start (`Init()` L1465, accepted branch): drop stale move commands as well (this also covers the
#87 family for goto/slew/gohome; park is decided separately, see 4.):

```st
IF fbStartup.AcceptInit() THEN
	bGoHome := FALSE; bGoto := FALSE; bSlew := FALSE; bTrack := FALSE;	// #85: nothing stale after power-on
	bPower := TRUE;
	fReadyState := 0.7;
```

Note: `Init()` is the only power-on entry for MQTT `power=1`. Check MONETS/MONETN `MAIN` and the MQTT
handler: if anything sets `bPower` directly, put the same clear in the body under `fbStartup` "accepted"
instead (for example on the rising edge of `fbStartup.Initializing`).

### 2.4 Optional small cleanup in the same branch

`fbElevation.MoveAxis` etc. are left TRUE after an interrupt only until `_StopTelescope` runs. No further change.

**Non-goal: `SuspendAutomatic`** (L1026-1045). It sets `bInterrupted` only around `_PowerOn()` (L1033-1035),
to abort a running power-on when automatic control is disabled. It does not stop a running goto/slew, does not
call `_StopTelescope`, and does not set `bMoveAbortPending`. Changing that is out of scope for 0.8.5.

---

## 3. Interactions and risks

1. **Arbitration order** after the change: `bStop` > `bPower` > `bMoveAbortPending (no error)` > `bPark` >
   `bGoHome` > `bGoto` > `bSlew` > `bTrack`. The emergency park (`bPark AND bError`) is not delayed: with
   `bError` the abort branch is skipped, and the error block clears the abort flag. A normal park over a slew
   now stops first and then parks. Previously the park also waited (`bReady AND NOT bBusy`), so no regression.
2. **bInterrupted** stays a one-shot (#26). The new stop-branch call sets and clears it in the same scan.
   `bInterruptedIdle`/auto-clear (L297-303) must stay at 0: check `INTERRUPTED_AUTO_CLEAR_COUNT` after the tests.
   `_PowerOn` treats `bInterrupted` as an abort (L906, L921). The stop branch already aborts the power-on before
   setting `bInterrupted`. Never set `bInterrupted` in a path where `bPower` should survive
   (#63 check 2: goto during power-on must not abort it; unchanged, since the goto branch is below `bPower`).
3. **Park / emergency park**: `_ParkTelescope` resets on `bStop OR bInterrupted` (L788). The stop branch now
   always calls it with `bInterrupted`. That is equivalent to today, because `bStop` was already resetting it.
   Reset keeps `bPark`: a park pending at the time of an error remains and becomes the emergency park.
   If the team wants reset to also drop a normal park, do it only when `NOT bError` (never drop an emergency park).
4. **Power-on stages**: clearing flags in `Init()` happens before stage 0. The stop branch keeps
   `MarkAbortedIfStarting` first (#80). `bMoveAbortPending` is below `bPower`, so a power-on is never blocked
   by a pending abort. Reset clears it.
5. **#77 reorder**: `_GotoTelescope` now returns early on `NOT bGoto`. Any caller relying on the old
   "call with bGoto FALSE runs the CASE" behaviour? Only the stop branch (L329), which wanted a reset anyway.
   `bAutoTrack` (L712) is unaffected.
6. **Mechanics (HalfBROT#77)**: a real stop decelerates with the NC default deceleration and is followed by a
   reversal (gohome). Run 1 suggests that can hit the 5.0 deg following-error limit on elevation with high torque.
   This plan does not change NC parameters. If the hardware test shows 0x4550 on stop, open a BROTLib issue for an
   explicit `MC_Stop.Deceleration`/`Jerk` (and check the elevation following-error window and decel in the
   MONETS .tsproj, done manually in XAE). Do not tune that from this branch.
7. **Command timeout**: STOP 15 s now also covers interrupts. A stuck abort (axis never reaches standstill)
   logs a timeout but still blocks the new command. Acceptable: a stop/reset/emergency park still works.
   The 15 s value is provisional until the deceleration time is measured (2.2, hardware step 2).
8. **Suspend (non-goal)**: `SuspendAutomatic` does not stop a moving telescope and is not changed (see 2.4).

---

## 4. Assessment of #83, #86, #87

| Issue | Same release? | Reason |
|---|---|---|
| #83 slew while parked: reject explicitly | **Partly**. The "never queued" part is mostly fixed by 2.3 (power-on clears stale goto/slew) and by 2.1/2.2 (no stage-50 leftover). The explicit rejection with a log/event is **0.8.6** | Rejection = new behaviour visible to pyobs/MQTT (log text, maybe a status field). Small but an API decision: which states reject (parked, not powered, powering on, error), and is there an MQTT status? Do it together with #87, same "command acceptance policy". |
| #87 park during power-on delayed | **0.8.6** | Needs a policy decision: park wins (aborts the power-on, like a stop) or the park is dropped with a log. 2.3 already drops stale goto/slew/gohome on power-on. Park is the safe direction and is deliberately not dropped here. Recommendation for 0.8.6: a park received while `bPower` aborts the power-on and parks (park wins), documented in the arbitration design doc. |
| #86 axis error does not emergency-park | **Separate release (0.8.7 or with its own safety review)** | A behaviour change of the safety path: an automatic emergency park on a rising `bError` closes covers and switches off hydraulics. It must be limited to axis errors (not software errors 0x15/0x16/0x31, not a focus error?), only while powered/not parked, and needs its own hardware test (#64 check 4). Mixing it with 0.8.5 would make it hard to tell which change caused what on the telescope. If it is not implemented, the hardware test plan (emergency park step) and the pyobs/watchdog expectations must say that an explicit `park` is needed after an error. |

---

## 5. Unit tests to add (MONETcommonTests, TcUnit)

`FB_MonetTelescopeControl` cannot be instantiated in a test today: it takes `REFERENCE TO` the concrete
HalfBROT axis FBs on `AXIS_REF` (open question in `specs/plans/2026-09-22-add-tcunit-tests.md`). The
pattern that works (BROTLibTests `FB_StartupControl_Tests` from BROTLib PR #77; MONETcommonTests
`F_HomeBelowHorizon_Tests`, `F_AzimuthOffsetOnAxis_Tests`) is to **pull the pure decision logic out into a
small FB/function without hardware** and test it scan by scan with plain ST calls and `Assert*`.
Each test method uses its own local instances, multi-scan tests guard their own state
(TcUnit calls every test every cycle).

Proposed extraction (internal to MONETcommon, used by FB_MonetTelescopeControl):

1. `F_GotoTargetError(bRequested, bReset, bInterrupted, bError, bTimeout, fElevationCalc) : UINT`
   returns 0 / 16#10 / 16#15 / 16#16 with the new order. Tests `F_GotoTargetError_Tests`:
   - `Reset_Never_Raises_Below_2deg` (#77): bReset TRUE, el -36 -> 0.
   - `Reset_Never_Raises_Above_89_5` (#77): bReset TRUE, el 89.9 -> 0.
   - `Interrupt_Or_No_Request_Never_Raises`: bInterrupted TRUE or bRequested FALSE -> 0.
   - `Goto_Below_2deg_Raises_15`, `Goto_Above_89_5_Raises_16`, boundaries 2.0 / 89.5.
   - `Timeout_Raises_10_Only_When_Requested`.
2. `FB_MoveCommandState` (pure flags: bGoHome/bGoto/bSlew/bTrack/bPark/bMoveAbortPending, with methods
   `Reset()`, `PowerAccepted()`, `ErrorScan()`, `Interrupt()`, `Standstill(bStopDispatched)`).
   Tests `FB_MoveCommandState_Tests`:
   - `Reset_Clears_Pending_GoHome` (#85), `Reset_Clears_Goto_Slew_Track`, `Reset_Keeps_Park`.
   - `Error_Clears_GoHome` (#85), `Error_Keeps_Park` (emergency park).
   - `Power_Accepted_Drops_Stale_Moves`.
   - `Interrupt_Sets_Abort_Pending`, `Abort_Clears_Only_After_Stop_Dispatched_And_Standstill` (the edge case in 2.2),
     `Error_Drops_Abort_Pending`, `Reset_Drops_Abort_Pending`.
   - `Abort_Set_During_Power_On_Survives_Until_Power_On_Ends` (lifecycle in 2.2): abort set, `bPower` TRUE for
     several scans -> flag stays TRUE, not cleared by standstill while STOP is not dispatched; after `bPower`
     drops, STOP is dispatched, then the flag clears at standstill.
   - `Stop_Rearms_Latched_StopAxis` (if the per-axis stop logic is extracted, e.g. `FB_StopAxisEdge`): StopAxis
     latched TRUE, axis moving -> scan 1 FALSE, scan 2 TRUE, later scans TRUE; not latched -> TRUE in scan 1;
     a STOP left before standstill re-arms on the next one.
3. `F_SoftwareErrorAfterReset` or test it inside 2.: `Reset_Clears_Software_Error_ID` (#88): after Reset(),
   nSoftwareErrorID = 0 and bSoftwareError FALSE when no error source is active; 0x31 comes back when
   bHomeBelowHorizon is TRUE.
4. `F_Arbitrate(...) : E_TCSCommand` (**required**): pure precedence function with bStop, bPower,
   bMoveAbortPending, bPark, bError, bGoHome, bGoto, bSlew, bTrack, bReady, bBusy, bStopped as inputs. The
   stop branch / arbitration change is the riskiest part of 0.8.5, so it must be covered by unit tests, not only
   by the hardware test. Tests:
   `Emergency_Park_Not_Delayed_By_Abort`, `Abort_Before_GoHome`, `Power_Above_Abort`, `Stop_Above_All`,
   `Goto_During_Power_On_Waits` (#63 check 2 regression). Size note: it is the largest extraction (the
   body's IF/ELSIF chain L313-420 becomes a function call plus the per-branch side effects). Keep the function
   to the precedence decision only; the flag clearing and the stage calls stay in the body.
5. HalfBROT/BROTLib side, optional: a test in BROTLibTests that `FB_Axis` / `FB_AxisCommand` re-issues a stop
   after a latched level (the edge re-arm), if step 0 shows that was the cause. That would be a BROTLib change,
   not part of this release.

Add the new suites to `MONETcommonTests` `MAIN` and to the README coverage table. Check the TcUnit limits
(32 suites / 32 tests / 256 asserts per README). CI (`.github/workflows/tests.yml`) must pass on the PR.

---

## 6. Hardware re-test on MONET/S (after the library is installed and activated)

Rules: **NEVER open the roof** (no roof command, no night run as part of this test). Covers/mirror covers are part
of the power-on as usual. MQTT fields are **comma-separated**
(`command azimuth=120,elevation=60,slew=1`, never spaces, see #83 correction). Record at 10-20 Hz
(TwinCAT Scope or the MQTT recorder): `bStop, bMoveAbortPending, bGoHome, bGoto, bSlew, bTrack, bPark,
bPower, bError, nErrorID, nSoftwareErrorID, bSoftwareError, bReset, bStopped, TCS_command`, axis actual
positions/velocities, `fbX.StopAxis`, `fbX.MoveAxis`, elevation torque, plus the event log.

**Step 0 (DONE 2026-10-20: `bStopAxis` is latched TRUE at idle, no `MC_Stop` is ever requested, so the edge re-arm of 2.2 is required; see #84 comment. Step 0 did not measure a deceleration time, because no
`MC_Stop` ran; that measurement is part of step 2 below) (before coding, on 0.8.4, optional but recommended for #84)**: scope `fbAzimuth.bStopAxis`,
`fbAzimuth.fbAxis.StopCommand.Pending/Execute`, `Axis_Stop.Busy/Done/Error`, `MoveCommand.Execute`,
`_SlewTelescope.nStage` during one `stop=1` 3 s into a slew to el 60 / az 120. This tells whether the stop
request is never made (latched level, 2.2 re-arm needed) or made and ignored by the NC. That needs the
telescope in the state of the earlier stop test; to be run only on request.

Then on 0.8.5:

1. `command reset=1`, `command power=1`. Wait for READY_STATE 1.0. Check `nSoftwareErrorID` = 0.
2. **#84 stop**: `command azimuth=120,elevation=60,slew=1`; 3 s later `command stop=1`. Expect: axes decelerate
   within a few seconds and stop short of the target, 'STOP completed', no 'timeout during STOP', no axis error.
   Then `command azimuth=150,elevation=50,slew=1`: must reach el 50 / az 150.
   Scope `fbX.StopAxis` per axis: if it was latched TRUE, it must go low for one scan, then high (fresh edge).
   **Deceleration-time measurement** (for the provisional STOP timeout in 2.2): repeat the stop at the highest
   slew speed (long slew, for example az +180 deg, `stop=1` once the axes are at full speed) and record the time
   from `stop=1` to `bStopped` per axis. Report the maximum; the final timeout is that plus a margin.
3. **#84 gohome interrupts**: slew to el 60 / az 120, 1.5 s later `command gohome=1`. Expect: 'move interrupted,
   stopping axes', standstill short of the target, then the move home, `bGoHome` FALSE at the end. Watch for
   0x4550 at the reversal (HalfBROT#77). If it appears, stop the test, `reset=1`, and report: this is the NC
   deceleration/following error item in Risks 6, not a 0.8.5 regression.
4. **#84 track interrupts**: slew, 2 s later `command track=1` with an RA/Dec above 20 deg elevation
   (same fields as in the earlier tests, comma-separated). Expect: stop, then goto, then tracking.
5. **#84 park interrupts**: slew, 2 s later `command park=1`. Expect: stop, then park, `bIsParked` TRUE.
   Then `command power=1` again.
6. **#77/#88**: set a target below 2 deg (track to an RA/Dec below the horizon) -> 0x15. `command reset=1` ->
   `bError` FALSE, `nErrorID` 0, `nSoftwareErrorID` 0, 'Internal command error ... cleared', no `fElevationOffset`
   workaround needed. Repeat with a target above 89.5 deg -> 0x16 -> reset clears it.
7. **#85** (stale gohome must not survive a reset):
   a. Telescope powered and idle at READY_STATE 1.0, away from home (for example after step 2 at el 50).
   b. Reproduce the stale flag the way it showed up in testing: `command park=1`, then while the park is running
      `command gohome=1` (it stays pending behind the park), then `command reset=1` (or an axis error)
      before the park ends. Check `bGoHome` over ADS (read-only).
   c. After `reset=1`: `bGoHome` FALSE, `bError` FALSE. Let the park finish (`bIsParked` TRUE).
   d. `command power=1`, wait for READY_STATE 1.0, then watch 30 s: the telescope must NOT start a move home
      (axes at standstill, `bGoHome` FALSE). On 0.8.4 this is where the stale gohome started by itself.
   Record `bGoHome`, `bReady`, axis positions at 10 Hz for the whole step.
   (A plain `stop=1` during a gohome does not reproduce #85: the stop branch clears `bGoHome` itself.)
8. **Regression**: power-on, then goto during power-on (#63 check 2: power-on completes, `bInterrupted` stays
   FALSE, the goto runs after it), stop during power-on stage 0 (#80), `park=1` from tracking.
   `INTERRUPTED_AUTO_CLEAR_COUNT` stays 0.
9. End: `command park=1`, wait for `bIsParked` TRUE.

Forces, online changes, writes over ADS (e.g. forcing an axis error) are done manually in XAE.

---

## 7. Release process

1. `git fetch`; branch from `origin/develop`: `fix/slew-interrupt-reset-84-77-85-88`.
2. Commits (one per issue makes the PR reviewable): #77+#88 guard order and reset; #85 flag clearing;
   #84 interrupts/stop; tests; version bump `ProjectVersion` 0.8.4 -> **0.8.5** in
   `MONETcommon/MONETcommon/MONETcommon.plcproj` (and wherever 0.8.4 is listed: README/CHANGELOG, library
   properties).
3. PR into **develop** ("Fixes #84, #77, #85, #88"; mention HalfBROT#77 and that #83/#86/#87 are separate).
   CI tests must pass. Review by the maintainer.
4. Release PR **develop -> main**, merged as a **merge commit (not squash)**, like the earlier releases.
   Tag/release v0.8.5 if that is the convention.
5. The maintainer installs the library (`Update-Libraries.ps1`), updates the reference in MONETS (and MONETN),
   builds and activates. Forces and online changes are done manually.
6. Hardware re-test (section 6). Comment the results on each issue; close after the test passes.
   Update HalfBROT#77 with the stop/reversal result.
7. Update the command-arbitration design notes with the new `bMoveAbortPending`
   priority and the reset/power-on flag rules.

---

## 8. Ordering and sizing

| Order | Item | Size | Risk |
|---|---|---|---|
| 1 | #77 + #88 (guard order, nSoftwareErrorID) | S (about 20 lines) | low |
| 2 | #85 (reset/error/power-on flag clearing) | S | low-medium (keep bPark) |
| 3 | #84 (abort-pending, stop branch, _StopTelescope, timeout) | M (about 80-120 lines) | medium: arbitration change, needs step 0 and the full hardware test |
| 4 | Test extraction + suites | M | low (pure functions) |
| - | 0.8.6: #83 rejection + #87 park-vs-power policy | S-M | decide policy first |
| - | Later, own safety review: #86 auto emergency park | M | high (safety path) |

Recommendation: do 1-2 first and test them on the hardware alone if time is short (they unblock the
recovery from errors). #84 is the larger change; step 0 is done and showed that the edge re-arm is needed. One branch, one 0.8.5 release, about 1-1.5 days including tests,
plus one hardware session.
