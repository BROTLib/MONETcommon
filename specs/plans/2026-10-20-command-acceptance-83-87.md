# MONETcommon 0.8.6: explicit command acceptance (#83) and park wins over power-on (#87)

Status: proposed. Date: 2026-10-20. Nothing implemented; no PLC code changed. Policy decisions D1-D7 (section 9) were decided on 2026-10-20: all recommendations accepted. Open: D3 needs the roof/pyobs check in 5.7 before implementing.
Repo: BROTLib/MONETcommon. Base: `origin/develop` @ `94332b3`, which is 0.8.5 as released (tag `v0.8.5`, PR #91/#93).
`FB_MonetTelescopeControl.TcPOU` is identical on `develop`, `v0.8.5` and `fix/slew-interrupt-reset-84-77-85-88` (`25459d5`).
0.8.5 itself has not yet been tested on hardware (see [2026-10-20-slew-reset-fixes.md](2026-10-20-slew-reset-fixes.md)). This plan
builds on its `bMoveAbortPending` mechanism, so the 0.8.5 hardware test should pass before 0.8.6 goes onto the telescope.

Main file: `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU`. All line numbers are **file lines of that .TcPOU at
`94332b3`** (the XML file lines, as in the 0.8.5 plan). Lines in other repos are marked with the repo.

Out of scope: #86 (an axis error alone does not start the emergency park). The emergency park path (`bPark AND bError`)
stays as it is; section 5 lists how this plan interacts with it.

Marked **[unverified]**: anything I could not confirm in the code available here (HalfBROT/NC behaviour, pyobs, roof logic).

---

## 1. Root causes

### #83: a slew/goto/track that cannot run is left pending, silently

1. **The command methods never check the state.** `Slew` L1634-1636, `MoveAltAz` L1553-1555, `Track` L1654-1657,
   `TrackRaDec` L1667-1670, `MoveRaDec` L1565-1567 write the target and set the flag unconditionally. `Home` is not
   overridden: BROTLib `FB_BaseTelescopeControl.Home` (BROTLib L316) sets `bGoHome` the same way. (The `LineIds` block at
   L1895-1897 lists a `FB_MonetTelescopeControl.Home` with no method behind it, a leftover.) None of these methods assigns
   its BOOL return value, so it is always FALSE. The MQTT handler ignores it anyway (BROTLib `FB_Comm_MQTT_Influx._ApplyField`
   L74-99).
2. **Dispatch is a gate without an else.** The slew branch L430-434 sets `TCS_command := slew` only when
   `bReady AND NOT bBusy`. The goto branch (L424), the track branch (L437) and the gohome branch (L407, which also needs
   `bStopped`) work the same way. When the condition is FALSE, nothing happens: the flag stays TRUE, `TCS_command` keeps
   its old value and nothing is logged. On a parked telescope `bReady` is FALSE (L509-514: axes disabled, covers closed),
   so the flag just sits there.
3. **Only a few things clear a pending flag**: the stop branch (L345-349), reset (L123-129, 0.8.5), the error block
   (L183-187), the power-on start in `Init()` (L1523-1526, 0.8.5), park stage 0 (L850-853), and the 600 s timeouts
   (L307-309). The timeouts turn the waiting command into a software error: a slew sent to a parked telescope gives
   **0x40 after 10 minutes** (L1250-1257), a goto 0x10 (L667-675). Since 0.8.5 the stale flag no longer runs after the next
   power-on (`Init()` clears it). It is still pending in silence until then, and still raises the timeout error.
4. **Other silent drops of the same kind:**
   - A clock that is not valid turns any goto/slew/track into a STOP (L110-112). The only log line is "STOP commencing".
   - A move sent during a stop is cleared by the stop branch (L345-349) on every scan while `bStop` is TRUE. No log.
   - A **park** sent during a stop is cleared the same way (L346).
   - A move during an error is cleared by the error block (L183-187) on every scan. No log.
5. The T8 observation in the issue body ("`bSlew` cleared within a second") was a test syntax error (space-separated
   fields, see the issue comment). A slew sent to a parked telescope with the correct syntax **has not been tested yet**. The
   code path above says it stays pending until 0x40 or the next `power=1`. Hardware step 1 confirms this on 0.8.5 first.

### #87: a park sent during a power-on runs after the power-on

1. **Arbitration order**: `ELSIF bPower` (L359) is above `ELSIF bPark` (L368), so the power-on keeps running and `bPark`
   waits. Nothing clears `bPark` during the power-on. 0.8.5 deliberately keeps it at the power-on start (`Init()` L1522:
   "park is kept, it is the safe direction").
2. **Park dispatch needs `bReady`** (L388): `IF bReady AND NOT bBusy THEN TCS_command := park`. A park requested while the
   telescope is parked, after a reset, or after an aborted power-on stays pending until the telescope is ready, which is
   **after the next power-on**. So "park, then power" ends with a park after the power-on in every order, not only during
   a power-on.
3. **Why `power=1` was accepted at all in T8 step 4** (goto, park 2.5 s later, power 0.7 s later): `fbStartup.Update(...,
   Parking := bPark AND NOT bPower, ...)` (L526) clears `Initialized` while a park is pending (BROTLib `FB_StartupReadiness`,
   first IF of the body: `IF Aborted OR Error OR Parking THEN ... Initialized := FALSE`). After that, `AcceptInit()`
   returns TRUE and `Init()` starts a full power-on. While `bPower` is TRUE, `Parking` is FALSE again, so the startup runs
   normally. When it ends, the park branch is reached and the park moves the mount (the elevation move in the issue). This
   chain is confirmed in the code. That it was exactly this chain on the telescope is the most likely explanation, not
   proven.
4. Side effect seen in the code, **[unverified]** on site: `FB_MonetCabinetControl` (over-temperature) and
   `FB_MonetMqttWatchdog` call `Park()` **every scan** while they are triggered. On a parked telescope `bPark` therefore
   stays TRUE (point 2), `Parking` is TRUE and READY_STATE reads 0.3 ("parking") instead of 0.0 for as long as they hold it.

---

## 2. Command-acceptance policy

### 2.1 Rules

- **R1 Never queued.** A move command (slew, goto/track, gohome) is decided **when it arrives**: accepted (runs now,
  or after a bounded wait for its own preparation), **rejected** (nothing is written, logged with a reason), or
  **aborts-and-replaces** (the running move is stopped through the 0.8.5 `bMoveAbortPending` path, then the new one runs).
  A rejected command never sets its flag, so it cannot start later, cannot run into a 600 s timeout, and does not overwrite
  the published target.
- **R2 Park is never rejected.** It is the safe direction. Callers like the cabinet guard and the MQTT watchdog rely on it.
  Park wins over a power-on (#87) and over every move. If the axes cannot move, it parks in place (no axis motion).
- **R3 Stop and reset are always accepted** (unchanged).
- **R4 Power-on** keeps the #80 gate (`AcceptInit`) and is additionally refused, with a log, while a stop or a park is in
  progress (park wins in both orders).

### 2.2 Table

States are checked top to bottom and the first match wins. "Moves" means slew, goto/track (`Track()`, `TrackRaDec()`,
`MoveRaDec()` set `bGoto`; `bAutoTrack` then starts tracking) and direct `bTrack`. Reject reasons in quotes are the log text.

| # | State (condition) | slew / goto / track | gohome | park | power | stop |
|---|---|---|---|---|---|---|
| 1 | **E-stop** (`bEstopTriggered`) | rejected "E-stop active" | rejected "E-stop active" | accepted, parks in place | unchanged (`AcceptInit`; `_PowerOn` fails on the error) | accepted |
| 2 | **Error** (`bError`) | rejected "error, send reset" | rejected "error, send reset" | accepted: emergency park (unchanged, L372-374) | unchanged | accepted |
| 3 | **Clock not valid** (`NOT fbTime.bValid`) | rejected "clock not valid" (was a silent STOP, L110) | accepted (no time needed) | accepted | unchanged | accepted |
| 4 | **Stopping** (`bStop`) | rejected "stop in progress" (D5) | rejected "stop in progress" | **deferred**: runs at standstill (`bParkAfterStop`, 3.6) | refused "stop in progress" | accepted (already) |
| 5 | **Powering on** (`bPower OR fbStartup.Initializing`) | rejected "power-on in progress" (D1) | rejected "power-on in progress" | **aborts the power-on and replaces it** (#87, 3.4) | refused (existing #80: "startup already running") | accepted: aborts the power-on (existing #80) |
| 6 | **Parking** (`bPark OR bParkAfterStop`) | rejected "park in progress" | rejected "park in progress" | accepted, no-op (already parking) | refused "park in progress" (D2) | accepted: cancels the park (existing) |
| 7 | **Parked** (`bIsParked`) | rejected "telescope parked, send power=1 first" | rejected "telescope parked" | accepted, completes at once without motion (3.5) | accepted | accepted (no-op) |
| 8 | **Not ready** (`NOT bReady`: after a reset, aborted power-on, covers/brake/drive not up, transient -2) | rejected "not ready" | rejected "not ready" | accepted: motion park if the axes can move, else in place (3.5, D3) | unchanged (`AcceptInit`) | accepted |
| 9 | **Gohome running** (`bGoHome` or `TCS_command = gohome`) | rejected "gohome in progress" (D4) | accepted, no-op | aborts-and-replaces (existing, L376-385) | refused (existing "already online") | accepted |
| 10 | **Moving or tracking** (a slew/goto/track flag set, `TCS_command` in {slew, goto, track}, `bTracking`, or `bMoveAbortPending`) | **aborts-and-replaces** (latest wins, D4, D7) | aborts-and-replaces (existing 0.8.5, L397-404) | aborts-and-replaces (existing 0.8.5) | refused (existing "already online") | accepted |
| 11 | **Ready, idle** | accepted | accepted | accepted | refused (existing "already online") | accepted (no-op) |

Notes:
- "busy" in the sense of `bBusy` from the **focus** alone (L516-519) is not a move. A move is accepted (row 11) and its
  dispatch waits for `NOT bBusy`. That wait is bounded by the focus move and is not queueing in the sense of #83.
- Row 4 rejects moves during a stop. It does not queue them until standstill. The alternative is in D5.
- Row 10 for goto over **tracking** means stop, then goto (via `bMoveAbortPending`). Today the goto stage 10 only drops
  `Tracking` (L707-713). This costs a few seconds of deceleration per retarget (D7).

---

## 3. Code changes (FB_MonetTelescopeControl.TcPOU unless noted)

### 3.1 New types (MONETcommon, testable without hardware)

```st
TYPE E_CommandDecision : (accept := 0, reject, replace, defer, abort_power); END_TYPE

TYPE E_CommandRejectReason : (none := 0, estop, error, clock_invalid, stopping, powering_on,
	parking, parked, not_ready, gohome_active); END_TYPE

TYPE ST_CommandState :
STRUCT
	bEstop, bError, bClockValid, bStop, bPowering, bParkPending, bIsParked, bReady : BOOL;
	bGoHomeActive	: BOOL;	// bGoHome OR TCS_command = gohome
	bMoveActive		: BOOL;	// slew/goto/track flag, TCS_command slew/goto/track, bTracking or bMoveAbortPending
END_STRUCT
END_TYPE
```

`F_CommandAcceptance(eCommand : E_TCSCommand; stState : ST_CommandState; eReason => E_CommandRejectReason) :
E_CommandDecision` implements the table of 2.2 exactly, row by row, in that order. Only the commands slew, goto,
track, gohome, park are passed. Power stays in `Init()` (3.3), stop/reset never pass through it.
`F_CommandRejectText(eReason) : STRING(80)` returns the quoted texts of the table. `F_TCSCommandText(eCommand)` returns
'slew', 'goto', ... (check first whether BROTLib already has one **[unverified]**, then reuse it).

Keeping the decision a pure function is required, not optional: it is the only part of #83 a TcUnit test can reach
(section 6).

### 3.2 Methods: decide before anything is written

New VARs (FB VAR block):

```st
	// #83: command acceptance
	bCommandRejected		: BOOL;			// set by _NoteRejected, logged and cleared by the body
	eLastRejectedCommand	: E_TCSCommand;
	eLastRejectReason		: E_CommandRejectReason;
	nCommandRejectCount		: UDINT;		// for diagnostics / the optional MQTT field
	bSupersedeRequest		: BOOL;			// an accepted move replaces the running one (row 10)
	eSupersedeKeep			: E_TCSCommand;
	bGoHomeChecked, bGotoChecked, bSlewChecked, bTrackChecked : BOOL;	// flag value the gate has already checked (3.2)
	bParkAfterStop			: BOOL;			// #83/#87: park received during a stop, set at standstill
	bPowerAbortedByPark		: BOOL;			// #87: one scan, logged
	PowerAbortEvent			: FB_EventLog;
	tonRejectRepeat			: TON;
	eLoggedCommand			: E_TCSCommand;
	eLoggedReason			: E_CommandRejectReason;
```

New private methods:

- `_CommandState(eExcludeOwn : E_TCSCommand) : ST_CommandState` fills the struct from the live variables.
  `bPowering := bPower OR fbStartup.Initializing`, `bParkPending := bPark OR bParkAfterStop`. For `bMoveActive`,
  the own flag of `eExcludeOwn` is left out. The direct-write gate needs this, because there the flag *is* the new request.
- `_AcceptMove(eCommand, eExcludeOwn : E_TCSCommand) : BOOL`: calls `F_CommandAcceptance`.
  `reject` -> `_NoteRejected(eCommand, eReason)`, return FALSE.
  `replace` -> `bSupersedeRequest := TRUE; eSupersedeKeep := eCommand;` return TRUE. `accept` -> TRUE.
- `_NoteRejected(eCommand, eReason)`: `nCommandRejectCount += 1`, store command/reason, `bCommandRejected := TRUE`.

Command methods (same pattern for all; shown for `Slew`):

```st
METHOD Slew : BOOL
VAR_INPUT
	fAzimuth	: LREAL;
	fAltitude	: LREAL;
END_VAR
// #83: decide first. A rejected slew writes nothing, not even the target (OBJECT.* telemetry stays truthful).
IF NOT _AcceptMove(E_TCSCommand.slew, E_TCSCommand.no_command) THEN
	RETURN;			// Slew = FALSE: rejected, logged by the body
END_IF
THIS^.fElevation	:= fAltitude;
THIS^.fAzimuth		:= fAzimuth;
THIS^.bSlew			:= TRUE;
bSlewChecked		:= TRUE;	// already checked here: the body's direct-write gate must not check it again
Slew := TRUE;
```

- `MoveAltAz`: as `Slew`.
- `Track`, `TrackRaDec`, `MoveRaDec`: as `Slew` with `E_TCSCommand.goto`, `bGoto`, `bGotoChecked`.
- **New override `Home : BOOL`** (BROTLib's base sets `bGoHome` unconditionally): `_AcceptMove(gohome, ...)`, then
  `bGoHome := TRUE; bGoHomeChecked := TRUE; Home := TRUE`. Overriding a base method works the same way as the existing
  `Slew`/`Park`/`Init` overrides **[unverified until compiled]**. Remove the stale `LineIds` entry for `Home` at L1895-1897
  if XAE does not regenerate it.
- `Park` (L1574):

```st
// #83/#87: park is never rejected. During a stop the stop branch clears bPark every scan (L346),
// so keep it aside and set it at standstill (3.6).
IF bStop THEN
	bParkAfterStop := TRUE;
ELSE
	bPark := TRUE;
END_IF
Park := TRUE;
```

Return values: TRUE = accepted, FALSE = rejected. That is new semantics for an `I_Telescope` method. Today all of them
return FALSE and no caller reads them. BROTLib's MQTT handler could log a FALSE, but it is not needed, because the
telescope logs the reason itself (3.7).

### 3.3 `Init()` (L1521-1543): power is refused while a stop or a park is in progress

Insert before `IF fbStartup.AcceptInit() THEN`, and reuse the #80 refusal reporting:

```st
IF bStop OR bPark OR bParkAfterStop THEN
	// #87: park wins in both orders; a stop is not overridden by a power-on either
	IF NOT bInitRefused THEN
		bInitRefused := TRUE;
		nInitRefusedLevel := ADSLOG_MSGTYPE_WARN;
		IF bStop THEN
			sInitRefusedReason := 'stop in progress';
		ELSE
			sInitRefusedReason := 'park in progress (send stop to cancel it)';
		END_IF
	END_IF
ELSIF fbStartup.AcceptInit() THEN
	... unchanged (0.8.5 clears bGoHome/bGoto/bSlew/bTrack) ...
```

This check must come first. Point 1.#87.3 explains why: a pending park clears `Initialized`, so `AcceptInit()` would accept.
Escape hatch: a park that cannot finish (for example covers that never report `Closed`) blocks the power-on. `stop=1` clears
`bPark` (L346), and `bParkAfterStop` is turned into `bPark` at standstill and is then cleared by the next stop. Reset does
**not** clear `bPark` (0.8.5, deliberately). The refusal text says so.

### 3.4 Arbitration: park wins over a power-on (#87)

Insert directly after the stop branch (L357) and **before** `ELSIF bPower` (L359):

```st
ELSIF bPark AND (bPower OR fbStartup.Initializing) THEN
	// #87: park wins over a running power-on. Abort it exactly like the stop branch does (#80), then let the
	// park run: homing or the move home may still be running, so stop the axes first (#84 mechanism).
	fbStartup.MarkAbortedIfStarting(Power := bPower);
	bPower := FALSE;
	bInterrupted := TRUE;
	_PowerOn();				// abort path L951-979: bPower FALSE, MarkAborted, stage 0, Move/HomeAxis FALSE
	bInterrupted := FALSE;	// #26 one-shot
	IF NOT bError THEN
		bMoveAbortPending := TRUE;
	END_IF
	bPowerAbortedByPark := TRUE;
	TCS_command := E_TCSCommand.stop;
```

New arbitration order: `bStop` > **park over power-on** > `bPower` > `bMoveAbortPending` (no error) > `bPark` > `bGoHome` >
`bGoto` > `bSlew` > `bTrack`.

Event (with the other events): `PowerAbortEvent(Trigger := bPowerAbortedByPark, Level := ADSLOG_MSGTYPE_WARN,
FormatString := '%s', OnMessage := 'power-on aborted: park requested', Comm := fbComm);` and
`bPowerAbortedByPark := FALSE;` after it.

After the abort, `Parking := bPark AND NOT bPower` is TRUE, so READY_STATE goes to 0.3 and `Initialized` is cleared.
The branch therefore does not fire again on the next scan. `bMoveAbortPending` dispatches STOP until standstill, and then
the park branch runs (3.5).

Whether clearing `HomeAxis` stops a homing that is already running is **[unverified]** (HalfBROT). The STOP from
`bMoveAbortPending` covers it: BROTLib `FB_Axis.Stop()` cancels a pending home and `MC_Stop` is allowed in `Homing`
(0.8.5 plan, related code). Hardware step 4 checks it.

### 3.5 Park dispatch and park in place

**Dispatch** (park branch, L387-391): also dispatch when the telescope is not ready but at standstill:

```st
ELSE
	// #83/#87: also when not ready (parked, after a reset, after an aborted power-on). It used to wait for
	// bReady, i.e. until after the next power-on.
	IF NOT bBusy AND (bReady OR bStopped) THEN
		bInterrupted := FALSE;
		TCS_command := E_TCSCommand.park;
	END_IF
END_IF
```

- **Parked**: `_ParkTelescope` jumps to stage 100 because of `IF bIsParked THEN nStage := 100` (L845) and clears `bPark`
  in the same scan (L928). No motion, `bIsParked` stays TRUE. This also ends the 0.3 READY_STATE of 1.#87.4: the cabinet
  guard and the watchdog set `bPark`, and it is cleared again before `fbStartup.Update` runs (L523).
- **Not parked, not ready**: stage 0 (L849-864) decides between a motion park and a park in place. Extend the emergency
  condition at L855:

```st
// #87: the axes can only drive to the park position when they are calibrated, enabled and the brake is open
bAxesCanMove := fbElevation.Calibrated AND fbAzimuth.Calibrated AND fbDerotator.Calibrated AND
	fbElevation.bEnable AND fbAzimuth.bEnable AND fbDerotator.bEnable AND fbBrake.BrakeOpen;
IF bError OR NOT bAxesCanMove THEN	// emergency park, or park in place (no axis motion)
	... unchanged: CloseBrake, disable axes, covers close, nStage := 80 ...
```

  (`bAxesCanMove` is a method-local VAR.) Example: covers closed by hand, axes up. `bReady` is FALSE, `bAxesCanMove` is TRUE,
  so this is a normal motion park; stages 10-40 do not check `bReady`. Example: after an aborted power-on with the azimuth
  not calibrated, `bAxesCanMove` is FALSE, so the park happens in place.
- The park in place sets `bIsParked` with the mount **not at the park position**. The emergency park already does that
  today. Whether anything (roof logic, pyobs) assumes the mount is at the park position when READY_STATE is 0 is
  **[unverified]** (D3).

### 3.6 Park received during a stop

At standstill (L539-544):

```st
IF bStopped THEN
	bStop := FALSE;
	IF TCS_command = E_TCSCommand.stop THEN
		bMoveAbortPending := FALSE;
	END_IF
	IF bParkAfterStop AND NOT bStop THEN	// #83: park sent during the stop runs now (from the next scan)
		bPark := TRUE;
		bParkAfterStop := FALSE;
	END_IF
END_IF
```

A direct write of `bPark` during a stop (ADS, a pendant) still goes through the stop branch and is cleared, as it is today.
Only `Park()` defers. Reset keeps `bParkAfterStop`, just as it keeps `bPark`.

### 3.7 Body: direct-write gate, replace, logging

**Direct-write gate.** Flags written straight into the VAR_INPUTs bypass the methods: an XAE/ADS write, or HalfBROT
`FB_PendantControl` mode 12, which writes `fbTelescope.bGoHome/bPark/bGoto/bTrack` every scan (HalfBROT
`FB_PendantControl` L215-222). MONETS/MONETN do not use that pendant (grep: no writes to the telescope flags in MONETS/MONETN).
Insert after `bError` is computed (after L181), before the error block:

```st
// #83: flags set directly (not through a method) are checked once, on their rising edge
IF bGoHome AND NOT bGoHomeChecked AND NOT _AcceptMove(E_TCSCommand.gohome, E_TCSCommand.gohome) THEN bGoHome := FALSE; END_IF
IF bGoto   AND NOT bGotoChecked   AND NOT _AcceptMove(E_TCSCommand.goto,   E_TCSCommand.goto)   THEN bGoto := FALSE;   END_IF
IF bSlew   AND NOT bSlewChecked   AND NOT _AcceptMove(E_TCSCommand.slew,   E_TCSCommand.slew)   THEN bSlew := FALSE;   END_IF
IF bTrack  AND NOT bTrackChecked  AND NOT _AcceptMove(E_TCSCommand.track,  E_TCSCommand.track)  THEN bTrack := FALSE;  END_IF
```

(ST does not short-circuit `AND`. Write it as nested IFs so that `_AcceptMove` only runs on the edge.)
At the **end of the body** (after the events): `bGoHomeChecked := bGoHome; bGotoChecked := bGoto; bSlewChecked := bSlew;
bTrackChecked := bTrack;`. Flags set **inside** the body are set before this snapshot, so the gate never sees them as new:
the horizon guard `bGoHome` (L296), the auto-track `bTrack := bAutoTrack` (L757) and the 12 h `bPark` (L311, park is not
gated anyway).

**Replace.** Insert just before the arbitration (L332):

```st
// #83 row 10: the latest move wins. Drop the others, reset their stage machines, stop the axes (#84 path).
IF bSupersedeRequest THEN
	IF eSupersedeKeep <> E_TCSCommand.gohome THEN bGoHome := FALSE; END_IF
	IF eSupersedeKeep <> E_TCSCommand.goto   THEN bGoto := FALSE;   END_IF
	IF eSupersedeKeep <> E_TCSCommand.slew   THEN bSlew := FALSE;   END_IF
	IF eSupersedeKeep <> E_TCSCommand.track  THEN bTrack := FALSE;  END_IF
	bInterrupted := TRUE;
	_HomeTelescope(); _GotoTelescope(); _SlewTelescope();	// stage reset, also of the kept one: it starts at 0
	bInterrupted := FALSE;	// #26 one-shot
	IF NOT bError THEN
		bMoveAbortPending := TRUE;
	END_IF
	bSupersedeRequest := FALSE;
END_IF
```

A same-type replace (slew over slew) also goes through here. The new target is already written. The stage reset
matters because the abort branch (L362-366) does not call the stage methods, and a goto left at stage 50 would wait for a
`MoveDone` that already happened (0.8.5 plan 1(b)). Reset and the error block clear `bSupersedeRequest` as well (add it next
to `bMoveAbortPending` in L129 and L188).

**Logging.** Not through `FB_EventLog`, which only logs on a rising edge (BROTLib `FB_EventLog` `LogonTrigger`). Two
rejections in consecutive scans would otherwise give one line. With the events:

```st
// #83: every rejection is logged and published (MQTT log topic, e.g. MONETS/Log). An identical rejection repeated
// within 2 s is logged once (a level writer such as a pendant would flood the log); all of them are counted.
tonRejectRepeat(IN := TRUE, PT := T#2S);
IF bCommandRejected THEN
	IF tonRejectRepeat.Q OR eLastRejectedCommand <> eLoggedCommand OR eLastRejectReason <> eLoggedReason THEN
		sRejectMessage := CONCAT(CONCAT(F_TCSCommandText(eLastRejectedCommand), ' rejected: '),
			F_CommandRejectText(eLastRejectReason));
		ADSLOGSTR(ADSLOG_MSGTYPE_WARN OR ADSLOG_MSGTYPE_LOG, '%s', sRejectMessage);
		IF fbComm <> 0 THEN
			fbComm.PublishLog('WARNING', sRejectMessage);
		END_IF
		eLoggedCommand := eLastRejectedCommand;
		eLoggedReason := eLastRejectReason;
		tonRejectRepeat(IN := FALSE);
	END_IF
	bCommandRejected := FALSE;
END_IF
```

If two different rejections happen in one scan (several verbs in one MQTT message), only the last one is logged. Both are
counted. Example lines: `slew rejected: telescope parked, send power=1 first`, `goto rejected: power-on in progress`.

**Clock check (L110-112)**: keep it as the backstop for direct writes and for a clock that becomes invalid during a move.
New commands are rejected by row 3 before that, with a reason.

### 3.8 What stays as it is

- Stop branch (L334-357), `_StopTelescope`, the abort branch, the stop timeout: 0.8.5, unchanged.
- Emergency park (`bPark AND bError`, L372-374), `_ParkTelescope` stages 60-100.
- `Init()` still clears stale move flags at power-on (0.8.5 #85). With the gate it should never find one, but it stays as a
  defence against direct writes.
- `SuspendAutomatic` (L1071-1090): not touched. Commands sent while suspended reach the methods. The gate sees
  `bReady` FALSE (row 8) and rejects moves, so they are no longer queued for the return to automatic.

---

## 4. How the rejection is surfaced

**Now (0.8.6): log line plus MQTT log.** Every rejection gives an ADS log entry (TwinCAT event log, level WARN) and the
same text through `fbComm.PublishLog` on the telescope's MQTT log topic (`MONETS/Log` for MONET/S, `sTopicLog` in MONETS
`MAIN`). The new power refusals go through the existing #80 `InitRefusedEvent` ("power-on ignored: ..."). The #87 abort
logs "power-on aborted: park requested". `nCommandRejectCount` is visible over ADS and in the scope.

Whether pyobs-monet reads the MQTT log topic is **[unverified]**. If it does not, the rejection is visible to operators
(log, Grafana if the log goes to Influx), but not to the pyobs task that sent the command.

**Optional follow-up: an MQTT status field.** For example in `_SendTelemetry`:
`TELESCOPE.COMMAND.REJECT_COUNT` (UDINT, monotonic) and `TELESCOPE.COMMAND.LAST_REJECTED` (for example `slew:parked`).

| Pros | Cons |
|---|---|
| Machine-readable: pyobs can compare the counter before and after a command, without parsing log text | No request ID, so the counter cannot say *which* command was rejected; fine for one client, ambiguous with several (pyobs plus GUI) |
| Cheap: two `Publish` calls on the existing 0.5/1 s telemetry cadence | Telemetry is periodic: up to 1 s of delay; pyobs would have to wait at least that long before it can conclude "accepted" |
| Lands in Influx/Grafana for statistics (how often does pyobs send into a parked telescope) | Schema change: the field has to be added to pybrotlib's telemetry schema (pyBROT: issue only, no code change from here) and to whatever Telegraf/Grafana expects; whether a string field needs quoting in the line protocol through `fbComm.Publish` is **[unverified]** |
| | `TELESCOPE.*` is BROTLib's namespace (`FB_BaseTelescopeControl._PublishTelemetry`); a MONET-only field there is a precedent decision |

Recommendation: ship the log in 0.8.6, then add `REJECT_COUNT` only (numeric, no quoting question) once pyobs actually
needs it, together with a pyBROT issue. A real accept/reject reply needs a request/response topic (BROTLib comm
change), which is a larger, separate item.

---

## 5. Interactions and risks

1. **0.8.5 is the base and is not hardware-tested.** Replace and park-wins both reuse `bMoveAbortPending` and the
   re-armed `_StopTelescope`. If 0.8.5 step 2/3 fails on the telescope, fix that first. Otherwise the two releases
   cannot be told apart on site.
2. **Arbitration order** (3.4). The new branch sits below `bStop`, so a stop still beats everything, and the #80 abort in the
   stop branch is unchanged. The park-over-power branch calls `_PowerOn` with `bInterrupted`, which is an abort by design
   (L951, L966). This is the one place where a park is meant to abort a power-on. A goto during the power-on is rejected
   (row 5) and never reaches the arbitration, so the #63 requirement "a goto during power-on does not abort it" still holds.
3. **`bInterrupted` stays a one-shot** in the two new places (3.4, 3.7). `INTERRUPTED_AUTO_CLEAR_COUNT` must stay at 0.
4. **Emergency park (#86 out of scope).** `bPark AND bError`: the gate never rejects a park, and the error branch of the
   arbitration is unchanged. The park-over-power branch fires with an error as well. That is harmless: `_PowerOn` aborts on
   `bError` by itself, and `bMoveAbortPending` is not set while `bError` is TRUE. The in-place condition only *adds*
   `NOT bAxesCanMove` to the emergency condition, so the emergency path itself is unchanged. If #86 later sets `bPark` on
   an axis error, the park-over-power branch already does the right thing.
5. **Safety callers.** The cabinet guard (`FB_MonetCabinetControl`, before the telescope in MONETS `MAIN`) and the MQTT
   watchdog (after it) call `Park()` every scan. With park-wins they now abort a power-on at once. Before, the power-on
   completed first and the park followed. That is the intended behaviour (#32: "any new move ends parked again"). Depending
   on call order within the scan, `power=1` is either refused ("park in progress") or accepted and aborted in the next scan.
   Both end parked.
6. **Deadlock check.** Power is refused while a park is pending (3.3). A park that can never finish (covers never `Closed`,
   `bBusy` stuck) would block power-on. The escape is `stop=1`, which clears `bPark`. Reset does not clear it. Compare this
   with MONETS#3 (`bInterrupted` latch): here the blocking state is visible (READY_STATE 0.3, "park in progress" in the
   refusal) and clearable. If the team wants reset to clear a normal park too, do it only when `NOT bError` (0.8.5 Risks 3).
7. **Park in place (D3).** `bIsParked` TRUE with the mount not at the park position, and covers closed while the elevation
   may be anywhere. The emergency park already does both. Check **[unverified]** whether the roof logic or pyobs closes the
   roof on READY_STATE 0 assuming the park position, and whether the covers may close at any elevation (they do in the
   emergency park).
8. **Behaviour changes pyobs may notice** **[unverified which ones pyobs relies on]**: a move during power-on is rejected
   (before: ran afterwards); goto from tracking stops first (D7); a slew during a goto replaces it (before: the slew ran after
   the goto); a move during a stop is rejected; `power=1` during a park is refused. T12 step 1 ("no rejected-command
   events" in a normal night) is the regression check. T12 step 6 (BROTLib#34, goto in the first seconds after a PLC start)
   now expects "goto rejected: clock not valid" instead of a STOP event.
9. **Test-plan expectations that change:** #63 check 2 / T8 step 4 ("goto interrupted by park, then power-on at once:
   power-on proceeds") becomes: `bInterrupted` FALSE, **power-on refused with "park in progress"** while the park runs,
   then `power=1` after `bIsParked` works. The 0.8.5 regression step 8 ("goto during power-on: the goto runs after it")
   becomes "the goto is rejected, the power-on completes". Update the IAG test plan
   (`IAG/specs/plans/2026-10-12-monet-telescope-test-plan.md`) when D1/D2 are decided.
10. **Level writers** (HalfBROT pendant mode 12) re-set a rejected flag every scan. The gate clears it every scan, and the
    log is throttled to once per 2 s. Not used on MONET, but `FB_MonetTelescopeControl` can be passed to that pendant.
11. **Slew has no target check** (no `< 2 deg` / `> 89.5 deg` guard like goto L677-698). This is pre-existing and not part
    of #83. Note it for a later issue.
12. **0.8.5 version constant is inconsistent**: `Global_Version.TcGVL` at `v0.8.5` has `iBuild := 4` with
    `sVersion := '0.8.5'`. Fix it in the 0.8.6 bump (section 8).

---

## 6. TcUnit tests (MONETcommonTests)

`FB_MonetTelescopeControl` still cannot be instantiated in a test (REFERENCE TO HalfBROT axis FBs, see
[2026-09-22-add-tcunit-tests.md](2026-09-22-add-tcunit-tests.md)). The deferred 0.8.5 extraction (`F_Arbitrate` and others) is
also still open. Test the new pure functions. One local instance per test, multi-scan tests guard their own state.

1. **`F_CommandAcceptance_Tests`** (required; one test per table row, several asserts each):
   `Estop_Rejects_Moves_Accepts_Park`, `Error_Rejects_Moves_Park_Is_Accepted`, `Clock_Invalid_Rejects_Slew_Goto_Track_Not_GoHome`,
   `Stopping_Rejects_Moves_Defers_Park`, `Powering_Rejects_Moves`, `Powering_Park_Aborts_Power`,
   `Parking_Rejects_Moves`, `Parked_Rejects_Moves_With_Reason_Parked` (#83), `Parked_Park_Is_Accepted`,
   `NotReady_Rejects_Moves_Accepts_Park`, `GoHome_Active_Rejects_Slew_Goto_Track`, `GoHome_Over_GoHome_Accepts`,
   `Moving_Slew_Over_Goto_Replaces`, `Moving_Goto_Over_Slew_Replaces`, `Tracking_Goto_Replaces`,
   `Moving_GoHome_Replaces`, `Abort_Pending_Counts_As_Moving`, `Ready_Idle_Accepts_All`.
   Precedence: `Error_Reason_Beats_Parked`, `Stop_Beats_Powering`, `Powering_Beats_Parked` (power accepted from parked,
   then a goto during it says "power-on in progress", not "parked").
   Reason output: every reject returns a reason other than `none`, and every accept/replace returns `none`.
2. **`F_CommandRejectText_Tests`**: every `E_CommandRejectReason` other than `none` gives a non-empty text that fits STRING(80).
3. **`F_ParkInPlace_Tests`** (if `bAxesCanMove`/the stage-0 choice is extracted as `F_ParkInPlace(bError, cal x3, enable x3,
   brake) : BOOL`): `Error_Parks_In_Place`, `Uncalibrated_Axis_Parks_In_Place`, `Disabled_Axis_Parks_In_Place`,
   `Brake_Closed_Parks_In_Place`, `All_Up_Moves_To_Park` (covers closed alone does not matter).
4. **`F_InitDecision_Tests`** (if 3.3 is extracted as `F_InitDecision(bStop, bParkPending, bAcceptInit) : (accept/refuse +
   reason)`): `Refused_While_Stop`, `Refused_While_Park_Pending`, `Refused_While_Park_After_Stop`,
   `Park_Check_Comes_Before_AcceptInit` (#87: park pending + AcceptInit TRUE -> refused), `Accepted_Otherwise`.
5. **`FB_RejectLogThrottle_Tests`** (if the throttle in 3.7 is a small FB; multi-scan): identical rejection twice within
   2 s logs once and counts twice; a different reason logs immediately; after 2 s it logs again.
6. If the 0.8.5 `F_Arbitrate` extraction is done in the same release, add: `Park_During_Power_Aborts_Power`,
   `Park_During_Initializing_Aborts_Power`, `Stop_Above_Park_Over_Power`, `Emergency_Park_Unchanged`,
   `Park_Dispatched_When_Stopped_Not_Ready`, `Park_Not_Dispatched_While_Busy`. If it is still deferred, say so in the PR.

Add the suites to `MONETcommonTests` `MAIN` and to the README coverage table. Limits: 32 suites / 32 tests / 256 asserts
(README). Suite 1 is about 22 tests and 80-100 asserts, so it fits, but count the total over all suites.

**For Tim (Windows/XAE), after the code exists:**

```powershell
cd <MONETcommon checkout, branch fix/command-acceptance-83-87>
& "C:\Program Files\Industrial Brains B.V\TcBuild\TcBuild.exe" install MONETcommon.sln -x MONETcommon -p MONETcommon -l MONETcommon.library
cd C:\TwinCAT\3.1\Runtimes\UmRT_Default; .\Start.bat
cd <MONETcommon checkout>
& "C:\Program Files\Industrial Brains B.V\TcBuild\TcBuild.exe" build MONETcommonTests.sln
.\MONETcommonTests\tools\Run-Tests.ps1
```

Report back: (a) any build errors or warnings in `FB_MonetTelescopeControl` and the new F_/E_/ST_ objects (especially the
`Home` override and the `ST_CommandState` initialisation), (b) the Run-Tests summary line (suites / tests / failed), and
(c) for each failed test its name and assert message.
Then build MONETS against the installed library in XAE (no activation yet) and report errors or warnings.

---

## 7. Hardware re-test on MONET/S (Tim, after 0.8.6 is installed, built and activated)

**Rules:** **NEVER open the roof.** Covers open and close as part of power-on/park. MQTT fields **comma-separated**
(`command azimuth=150,elevation=50,slew=1`, never spaces, see the #83 comment). Prerequisite: the 0.8.5 hardware test
steps 1-5 (0.8.5 plan section 6) have passed, or run them first on this build.

**Record (TwinCAT Scope, 10-20 Hz; or the MQTT recorder):** `bSlew, bGoto, bTrack, bGoHome, bPark, bParkAfterStop, bPower,
fbStartup.Initializing, bIsParked, bReady, bStopped, bStop, bError, bMoveAbortPending, TCS_command, fReadyState,
nCommandRejectCount, eLastRejectReason, bInterrupted`, axis actual positions, `fbX.bEnable`, `fbX.Calibrated`, covers
opened/closed, `tonSlewTimeout.ET`, `tonGotoTimeout.ET`, plus the ADS log and the `MONETS/Log` topic. Note
`INTERRUPTED_AUTO_CLEAR_COUNT` and `nCommandRejectCount` at the start.

Steps:

0. **Version and baseline on 0.8.5 (optional, before installing 0.8.6, about 2 min):** telescope parked, send
   `command azimuth=150,elevation=50,slew=1`. Expect on 0.8.5: no motion, no log, `bSlew` stays TRUE, `tonSlewTimeout.ET`
   counts up. Then `command stop=1` (clears it). This confirms root cause 1.#83 with the correct syntax.
   After installing 0.8.6: check online that `Global_Version.stLibVersion_MONETcommon.sVersion = '0.8.6'`.
1. **#83 parked** (telescope parked, READY_STATE 0):
   `command azimuth=150,elevation=50,slew=1`, then `command rightascension=<RA>,declination=<Dec>,track=1` (any valid
   target), then `command gohome=1`. Expect for each, at once: a log line "slew rejected: telescope parked, send power=1
   first" (goto / gohome likewise), `bSlew`/`bGoto`/`bGoHome` **never** TRUE, `tonSlewTimeout.ET` stays 0, no motion,
   READY_STATE stays 0, `nCommandRejectCount` +3. `OBJECT.HORIZONTAL.ALT/AZ` in the telemetry do not change to 50/150.
   Also via XAE: write `TelescopeControl.bSlew := TRUE` once. It must be FALSE again in the next scan and logged.
2. **Park while parked, then power-on** (#87 family): `command park=1`. Expect: no motion, `bPark` back to FALSE
   within one scan (the scope may not catch it; READY_STATE must not change and stay 0). Then `command power=1`, wait for
   READY_STATE 1.0, then **watch 60 s: nothing moves**, `bPark` FALSE.
3. **Moves during power-on**: from parked, `command power=1`; about 10 s later
   `command azimuth=150,elevation=50,slew=1`, then `track=1` with an RA/Dec above 20 deg elevation. Expect "rejected:
   power-on in progress" for each, the power-on completes normally (READY_STATE 1.0, `bInterrupted` FALSE), and **no move
   afterwards** (watch 30 s).
4. **#87 park wins**, two variants:
   a. From parked, `command power=1`; during homing (READY_STATE 0.8, about 10-20 s in) `command park=1`. Expect: log
      "power-on aborted: park requested", `bPower` FALSE, READY_STATE 0.3 then 0.0, any axis that was homing comes to
      standstill (report how fast), covers close, `bIsParked` TRUE, **no axis moves to the park position** (park in
      place, axes not calibrated), no axis error.
   b. From parked, `command power=1`; once the axes are calibrated and the move to home has started (stage 30/40, about
      40 s in, elevation moving) `command park=1`. Expect: the same log, the axes stop (STOP via `bMoveAbortPending`), then
      a park. Report **which**: a motion park (axes drive to the park position, `bAxesCanMove` TRUE) or a park in place.
      Both are acceptable. Report which one happened and the elevation at the end.
   After each variant: `command power=1` must give a full, normal power-on.
5. **Power during a park** (new #63 check 2 expectation): online at READY_STATE 1.0, `command azimuth=120,elevation=60,slew=1`;
   2 s later `command park=1`; 0.7 s later `command power=1`. Expect: "power-on ignored: park in progress (send stop to
   cancel it)", the slew stops, the park completes, `bInterrupted` FALSE. Then `command power=1` works and **nothing
   moves after it**.
6. **Latest move wins** (online): `command azimuth=120,elevation=60,slew=1`, 2 s later
   `command azimuth=150,elevation=50,slew=1`. Expect "move interrupted, stopping axes", standstill, then el 50 / az 150
   reached. Repeat with a `track=1` (above 20 deg) 2 s into a slew: stop, goto, tracking. Then while **tracking**
   `track=1` to a second target: stop, goto, tracking. Report how long the stop between the moves takes.
7. **Gohome running**: `command gohome=1` from el 50; 1 s later `command azimuth=150,elevation=50,slew=1`. Expect "slew
   rejected: gohome in progress", the gohome completes, no slew afterwards.
8. **Stop in progress**: slew to el 60 / az 120, 3 s later `command stop=1`, within 1 s
   `command azimuth=150,elevation=50,slew=1`. Expect "slew rejected: stop in progress", the axes stop, no slew afterwards.
   Then the same slew once `bStopped`: must run. Then slew again, `command stop=1` and within 1 s `command park=1`.
   Expect: stop, then the park starts by itself at standstill (`bParkAfterStop` TRUE, then `bPark`). End parked.
9. **Error and not ready**: `command power=1`. Set a target below 2 deg (track to an RA/Dec below the horizon) -> 0x15.
   Then `command azimuth=150,elevation=50,slew=1`: "rejected: error, send reset". `command reset=1` (axes stay disabled
   if they were disabled; READY_STATE -2 or similar). Slew again: "rejected: not ready". `command park=1`: park
   (in place if the axes are disabled; report which), `bIsParked` TRUE. If the axes stay enabled after the 0x15 reset
   (software error only), the slew after the reset may simply work. That is correct, report it.
10. **Regression, short**: from parked, power-on, goto + tracking for 2 min, `command stop=1`, `command gohome=1`,
    `command park=1`. No rejection lines in this normal sequence. `INTERRUPTED_AUTO_CLEAR_COUNT` unchanged.
11. End: `command park=1`, `bIsParked` TRUE.

**Report back per step:** pass/fail, the exact log lines with timestamps (ADS log or `MONETS/Log`), the READY_STATE
sequence, `nCommandRejectCount` before/after, any axis error IDs, the scope export (CSV) of steps 4, 5, 6 and 8, the
answers asked for in steps 4b, 6 and 9, and `INTERRUPTED_AUTO_CLEAR_COUNT` at the end. Forces and XAE writes (step 1)
are done by Tim manually in XAE.

MONET/N gets the same library. A short repeat of steps 1, 2, 4a and 5 there is enough once MONET/S has passed.

---

## 8. Release (0.8.6)

1. `git fetch`; branch `fix/command-acceptance-83-87` from `origin/develop` (`94332b3` or later).
2. Commits: (a) types + `F_CommandAcceptance` + texts + tests; (b) method gates, `Home` override, `Init` refusal,
   logging (#83); (c) park-over-power branch, park dispatch, park in place, `bParkAfterStop` (#87); (d) direct-write
   gate + replace; (e) version bump.
3. Version: `ProjectVersion` 0.8.5 -> **0.8.6** in `MONETcommon/MONETcommon/MONETcommon.plcproj`; `Global_Version.TcGVL`
   `iBuild := 6`, `sVersion := '0.8.6'`. This also fixes 0.8.5's stale `iBuild := 4`.
4. PR into **develop**: "Fixes #83, #87". The PR text names the behaviour changes (5.8) and D1-D7 as decided. CI tests
   must pass.
5. Release PR develop -> main as a **merge commit** (not squash), tag `v0.8.6`, as for 0.8.5.
6. Tim installs the library, updates the reference in MONETS (and MONETN), builds, activates; hardware test section 7.
   Comment the results on #83 and #87; close them after the test passes.
7. Docs: update the arbitration design note `IAG/specs/design/monets-telescopecontrol-command-arbitration.md` (new order,
   acceptance table, park in place), the IAG test plan expectations (5.9), and this plan's Status line.
   Open a pyBROT **issue** (no code change) only if the MQTT field of section 4 is added.

Sizing: about 150-200 lines in `FB_MonetTelescopeControl` plus 3 small types/functions and about 25-30 tests. About
1 day plus 1 hardware session (1.5-2 h, no roof).

---

## 9. Open decisions

| # | Question | Recommendation |
|---|---|---|
| D1 | Slew/goto/track **during a power-on**: reject, or queue until the power-on ends (the 0.8.5 regression step 8 reading of #63)? | **Reject** ("power-on in progress"). #83 asks for it, and a queued move is exactly the #87 pattern. |
| D2 | `power=1` **while a park is pending/running**: refuse (park wins in both orders) or let power cancel the park? | **Refuse**, `stop` is the escape. Changes #63 check 2 / T8 step 4 expectations (5.9). |
| D3 | Park when the axes **cannot move** (aborted power-on, after a reset): park in place (covers close, hydraulics off, `bIsParked` away from the park position) or reject? | **Park in place**, as the emergency park already does. Needs a yes on the roof/pyobs question in 5.7. |
| D4 | Busy: does the **latest move win** (aborts-and-replaces), and are moves **during a gohome** rejected? | Latest wins for slew/goto/track; **reject during gohome** (it may be the horizon guard's recovery). |
| D5 | Moves **during a stop**: reject, or queue until standstill? | **Reject**. A park during a stop is deferred to standstill (it is the safe direction). |
| D6 | MQTT status field now or later? | Later, and only `REJECT_COUNT` first (section 4). |
| D7 | Goto **from tracking**: stop first (consequence of D4), or keep today's "drop tracking and move" at stage 10? | Stop first (one path for all replaces). Revisit if retargeting in a night gets noticeably slower (step 6 measures it). |
