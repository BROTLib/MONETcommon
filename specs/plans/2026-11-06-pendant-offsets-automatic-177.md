# Pendant in automatic mode: no axis writes, selector 5-8 nudge the pointing offsets (#177)

Status: **decided 2026-11-06 (policy section 2; D1-D6 taken as recommended, section 10); values in 3.4 are defaults to confirm on
site; implementation written on `feat/pendant-offsets-automatic-177`, not compiled or run.**
Date: 2026-11-06.
Repo: BROTLib/MONETcommon. Read at the main checkout `bbbbc51` (branch `test/manual-home-restore-after-abort-152`, on top of
`develop` = `e73ddf0`, 0.17.0; `FB_MonetPendantControl` and `FB_MonetTelescopeControl` as in 0.17.0). Pins: BROTLib 0.18.0,
HalfBROT 0.13.0. MONETS `develop` = `83ef132`, MONETN `develop` = `10ef0d5` (both pin MONETcommon 0.17.0). HalfBROT `c50e28e`.
BROTLib read at the main checkout (detached `fb3150a`, members used here unchanged since 0.18.0 [inferred]).
Line numbers below are from these commits; the ones in #177 (MAIN 171/181) are stale.
Main files: `MONETcommon/MONETcommon/FB_MonetPendantControl.TcPOU` (FBMPC), `FB_MonetTelescopeControl.TcPOU` (FBMTC),
`MONETcommonTests/.../POUs/FB_MonetPendantControl_Tests.TcPOU`, `FB_MonetPendantProbe.TcPOU`.
Issues/plans read: #177, #3 plan `2026-10-30-pendant-release-on-exit-3.md`, #72 plan, #168 (issue and the small-alternative
plan `2026-11-06-manual-heartbeat-estop-error-168.md`), step 4c client check `2026-11-02-step-4c-client-check.md`, HalfBROT
plan `2026-10-12-pendant-fixes.md`. Code read: MONETS/MONETN `MAIN`, BROTLib `I_AltAzTelescope`, `I_Telescope`, `I_Focus`,
`I_BaseAxis`, `FB_AltAzTelescopeControl`, `FB_Comm_MQTT_Influx`, HalfBROT `FB_FocusControl`, `FB_PendantControl`,
MONETcommon `F_AzimuthOffsetOnAxis`, `FB_MonetTelescopeTelemetry`, `ST_MonetTelemetry`, `MONETS.tsproj` (task).

Legend: **[read]** seen in the code at the commits above, **[inferred]** follows from the code but not observed,
**[unknown]** not answerable from the repos.

## 1. Findings

- [read] FBMPC has `eTelescopeMode` (l. 6) but neither `MAIN` passes it (MONETS l. 234-240, MONETN l. 240-246), so it is
  `off` (0). The only use is the empty `IF eTelescopeMode = E_TelescopeMode.automatic THEN ; END_IF` (l. 158-160). The
  `CASE nSelectionStable` 5-8 (l. 173-232) writes `Enable := IbEnableButton` and `Move*` in every mode. The #177 hazard
  (selected axis disabled in automatic, elevation brake closes) stands as described in the issue.
- [read] The #3 release (l. 123-145) uses `nDriven := SEL(IbEnableKey AND tonSelectionStable.Q, 0, nSelection)`; `_DropEnable`
  fires `tEnableRelease` (T#2S) after release, in any mode.
- [read] **Offsets.** `fElevationOffset`, `fAzimuthOffset`, `fDerotatorOffset` are VAR_INPUTs of BROTLib
  `FB_AltAzTelescopeControl` (l. 6-8), exposed as read/write properties `AltitudeOffset`, `AzimuthOffset`, `DerotatorOffset` of
  `BROTLib.I_AltAzTelescope` (FBMTC `IMPLEMENTS I_AltAzTelescope`, l. 4). FBMTC also has `SetAltitudeOffset`/
  `SetAzimuthOffset`/`SetDerotatorOffset` (l. 2849-2877, plain assignments). MQTT writes the properties:
  `elevationoffset`/`azimuthoffset` in degrees, range +-10, `derotator` **and** `derotatoroffset` both to `DerotatorOffset`,
  range +-360 (`FB_Comm_MQTT_Influx` l. 269-284).
- [read] **Use.** Tracking/goto: `_ComputeTargets` adds `fElevationOffset` to the target elevation and
  `F_AzimuthOffsetOnAxis(fAzimuthOffset, fElevationCalc)` to the azimuth (l. 2280-2282), `fDerotatorOffset` to the derotator
  (l. 2289), every scan; `_TrackTelescope` writes these targets to the axes every scan (l. 2551, 2570, 2589). So an offset
  change moves a tracking telescope at once [inferred]. Slew: offsets applied once at `start_move` (l. 1784-1806), with the
  #212 goto elevation window check (l. 1787). Current sky position: subtracted again for `hor2eq` (l. 2331-2332).
  `fAzimuthOffset` is a sky (cross-elevation) offset; `F_AzimuthOffsetOnAxis` divides by `MAX(COS(el), COS(89.5))` and limits
  the axis value to +-10 deg.
- [read] **Offsets are never reset.** No assignment to the three offsets in FBMTC except the setters; none in `MoveRaDec`,
  `Track`, `TrackRaDec`, `Slew`, `MoveAltAz` (l. 2712-2958) or the input-edge acceptance (l. 580-594). A new target inherits
  the previous offsets today. (HalfBROT `FB_PendantControl` l. 273-280 is the only other writer: +-10 arcsec per press, not
  used at MONET.)
- [read] **No focus offset exists.** `I_Focus` has `Position` (target), `MinPosition`, `MaxPosition`; MQTT `focus` sets the
  absolute target, only at READY_STATE 1 (`_FocusAllowed`, BROTLib#74), rejected outside the travel (MONETcommon#122). HalfBROT
  `FB_FocusControl`: unit mm, MONETS/MONETN travel 0.5..91.5 mm, `fVelocity` 0.2 mm/s, `fTolerance` 0.005 mm, `fHoldBand`
  0.02 mm; in automatic a follower moves to `fPosition`, "a new target during a move restarts it" (l. 123-137), and the
  block owns its own drive enable in automatic (`fbDriveEnable`, l. 196-205). So the pendant writing `fbFocus.Enable` in
  automatic fights the focus block too.
- [read] **Telemetry.** ALT/AZ offsets are published (`FB_MonetTelescopeTelemetry` l. 105, 142:
  `POSITION.INSTRUMENTAL.ALT.OFFSET`, `.AZ.OFFSET`; also BROTLib base l. 41-42). The derotator offset is not published
  (no match in `FB_MonetTelescopeTelemetry`/`ST_MonetTelemetry`).
- [read] **Reset.** Both `MAIN`s already call `TelescopeControl.Reset()` in automatic on `PendantControl.IbResetButton`
  (MONETS l. 112-119), without the pendant key, and feed it into `SafetyHandling` (l. 307). The pendant's own branch 5-8
  additionally calls `fb<Axis>.Reset()` of the selected axis.
- [read] **Order in MAIN.** `TelescopeControl(...)` (automatic) or `SuspendAutomatic(...)`, then the axes and focus, then
  `PendantControl`, then `fbComm` (MQTT). One PLC task (`PlcTask`, `CycleTime="100000"` = 10 ms, `MONETS.tsproj` l. 2149,
  MONETN l. 2591). A pendant write to an offset in scan N is used by FBMTC in scan N+1; an MQTT write later in scan N wins over
  the pendant's write of that scan. No concurrency beyond that ordering.
- [read] **Modes.** MONETS/MONETN: key Remote = `automatic`; key Local = `manual` if `HydraulicsControl.Enable`, else `off`
  (MONETS l. 90-95). So "off" is Local with hydraulics off; today the pendant can jog focus, derotator and azimuth there.
- [read] **Client.** pyobs-brot re-sends absolute setpoints (offsets, focus) until settled (step 4c plan l. 50,
  `brottelescope.py:286-291, 415-429, 472-487` as quoted there). [unknown] Whether it sends offsets before `track` and whether
  it guides through these offsets.

## 2. Policy (decided by Tim, not re-opened)

| Mode | Covers 1-3 | Selector 5-8 | Enable / Move* of axes and focus |
|---|---|---|---|
| manual | hold-to-run | jog as today | written by the pendant (unchanged, #3 release) |
| automatic | hold-to-run | nudge offsets: 7 elevation, 8 azimuth (left/right, sky frame), 6 derotator, 5 focus | **never** written |
| off | hold-to-run | inert | never written (release of what the pendant owned still runs) |

A nudge needs key on, selector stable and the enable button held. Offsets are cleared on a new target. Leaving manual releases
`Move*`; `_DropEnable` drops only an Enable the pendant set and never in automatic; entering automatic from manual must not
leave Enable low. Nudges add to the value read each scan, azimuth in the sky frame, accumulated offset clamped, goto elevation
window respected.

## 3. Proposed change

### 3.1 FBMPC inputs and MAIN wiring

New VAR_INPUTs:

```
iTelescope	: BROTLib.I_AltAzTelescope;		// #177: offsets (read+write) and ReadyState; 0 = no nudges
stMotion	: BROTLib.ST_MotionConfig;		// #177: goto elevation window for the elevation nudge (defaults = library defaults)
tCycle		: TIME := T#10MS;				// #177: PLC task cycle time (`_TaskInfo` is not visible in a library, CI build 2026-11-06)
```

Why `I_AltAzTelescope` and not a new interface: it already carries get/set of the three offsets plus `ReadyState`
(`I_Telescope`), FBMTC implements it, `MAIN` already passes `TelescopeControl` as `AltAzTelescope` to `fbComm`. No BROTLib
release. Cost: the test stub implements all of `I_AltAzTelescope` (about 20 trivial members, 5.1).

`MAIN` (MONETS and MONETN), in the `PendantControl(...)` call:

```
	eTelescopeMode	:= eTelescopeMode,		// MONETcommon#177
	iTelescope		:= TelescopeControl,
	stMotion		:= stTelescopeMotion,	// MONETS only; MONETN does not pass stMotion to TelescopeControl either
```

### 3.2 Gating in FBMPC

- `bManual := eTelescopeMode = manual`, `bAuto := eTelescopeMode = automatic`.
- Ownership (#3 logic): `nDriven := SEL(bManual AND IbEnableKey AND tonSelectionStable.Q, 0, nSelection)`. Leaving manual is then
  one more release trigger: `Move*` FALSE once, no redesign.
- `_DropEnable` guard: on expiry of `tonEnableRelease`, drop only if `NOT bAuto`; on the first scan with `bAuto`,
  `nEnablePending := 0` (the drop is discarded, not deferred). So manual -> automatic never drops an Enable later (FBMTC owns it,
  `_PowerOn` stage 10 after `power=1`, #72); manual -> off still drops it after `tEnableRelease` (as #3).
- `CASE nSelectionStable` 5-8: the existing jog bodies run only `IF bManual`. `IF bAuto` the nudge (3.3) runs instead. In `off`
  nothing (lamps off, error lamp `bError`). Covers 1-3 unchanged in all modes. The empty `IF ... automatic THEN ; END_IF` goes.
- Reset in 5-8: the `fb<Axis>.Reset()` / `fbFocus.Reset()` calls only in manual (3.6).

### 3.3 Nudge engine (automatic, selector 5-8)

Gate `bNudgeOk := bAuto AND IbEnableKey AND tonSelectionStable.Q AND IbEnableButton AND iTelescope <> 0 AND
iTelescope.ReadyState = 1.0` (online, the same gate as MQTT `focus`, BROTLib#74; covers RETURNING/INIT_REQUIRED after manual
until `power=1`, D4). Exactly one direction of the selected function pressed (`up XOR down` for 5/7, `left XOR right` for 6/8).

Arming: `bNudgeArmed` goes FALSE whenever `bNudgeOk` is FALSE or the selector changes, and TRUE only in a scan where
`bNudgeOk` and no direction button is pressed. A direction held through a mode change, key on or enable press does nothing
until released (3.8).

Per scan (dt in seconds, cycle-time independent):

```
fDt := MAX(TIME_TO_LREAL(tCycle) * 1.0E-3, 1.0E-6);
IF bNudgeArmed AND nDir <> 0 THEN
	IF tHeld = 0 THEN fDelta := nDir * fStep;                         // rising edge: one step (a tap)
	ELSIF tHeld >= tSlowAfter AND tHeld < tFastAfter THEN fDelta := nDir * fRateSlow * fDt;
	ELSIF tHeld >= tFastAfter THEN fDelta := nDir * fRateFast * fDt;
	ELSE fDelta := 0; END_IF
	tHeld := tHeld + fDt;                                             // accumulated, not a TON: deterministic in TcUnit
ELSE tHeld := 0; END_IF
```

`_TaskInfo`/`GETCURTASKINDEXEX` are Tc2_System [inferred available, check at first build]. Integrating the nominal cycle time
under-counts a scan overrun slightly: harmless (slower, never faster). A measured `LTIME()` delta was rejected: it does not
advance between calls in one TcUnit cycle.

Apply (only when `fDelta <> 0`, read-modify-write in the same scan, so an MQTT write of the last scan is the base):

- 7: `fNew := iTelescope.AltitudeOffset + fDelta`; refused if `fbElevation.Position + fDelta` (the axis target, which while
  tracking already contains the offset) leaves `stMotion.fGotoMinElevation..fGotoMaxElevation` (defaults 2.0/89.5).
- 8: `iTelescope.AzimuthOffset + fDelta`, sky frame; FBMTC applies `F_AzimuthOffsetOnAxis` (cos(el), bounded). Right = +.
- 6: `iTelescope.DerotatorOffset + fDelta`; refused if `fbDerotator.Position + fDelta` leaves `fMinPosition..fMaxPosition`.
- 5: `fbFocus.Position + fDelta` within `fbFocus.MinPosition..MaxPosition` and within the pendant focus range (3.4) around
  `fFocusBase`; `fFocusBase` := `fbFocus.Position` whenever the target differs from the last value the pendant wrote (a new
  MQTT focus re-bases). Focus is stepped, not ramped (3.4), and does not write `Enable` (`FB_FocusControl` enables itself).
- Clamp, all axes: write `fNew` only if `ABS(fNew) <= fMax` **or** `ABS(fNew) < ABS(fOld)` (an offset set beyond the pendant
  maximum by MQTT can be reduced, never increased).

### 3.4 Suggested values (suggestions, confirm on site; VAR CONSTANT or VAR_INPUT with these defaults, D1)

| Function | Sel | Buttons | Tap step | Held 0.5-3 s | Held > 3 s | Max |
|---|---|---|---|---|---|---|
| Elevation offset | 7 | up +, down - | 5 arcsec | 10 arcsec/s | 60 arcsec/s | abs 600 arcsec (10 arcmin) |
| Azimuth offset (sky) | 8 | right +, left - | 5 arcsec | 10 arcsec/s | 60 arcsec/s | abs 600 arcsec |
| Derotator offset | 6 | right +, left - | 0.02 deg | 0.1 deg/s | 1.0 deg/s | abs 90 deg and derotator axis limits |
| Focus target | 5 | up +, down - | 0.02 mm | one 0.02 mm step every 0.5 s after 1 s held (0.04 mm/s) | no fast stage | +-1.0 mm from `fFocusBase`, and the travel |

Reasoning:
- Tap 5 arcsec: a few times typical seeing, fine enough to centre a star; half the HalfBROT precedent (10 arcsec per press).
- 10 arcsec/s is below the sidereal rate (15 arcsec/s), so the tracking axes see a small extra rate; 60 arcsec/s = 0.6 arcsec
  per 10 ms scan, still far below the move velocities (deg/s). Reaching the maximum from zero: 5 + 25 + 9.5 s x 60 = about 12.5 s
  of holding.
- 600 arcsec: well beyond any pointing-model residual we should accept (if 10 arcmin is not enough, the model is the problem)
  and far inside the MQTT +-10 deg; at el 85 the azimuth axis offset is 600/cos(85) = 1.9 deg, at 89.5 the cap of
  `F_AzimuthOffsetOnAxis` (10 deg) applies. [unknown] MONET camera field of view and plate scale: if the field is smaller than
  about 20 arcmin, set the max to half the field.
- Derotator 0.02 deg moves a star 5 arcmin off-axis by about 0.1 arcsec: fine alignment; 1 deg/s for gross changes. 90 deg
  instead of the MQTT 360: a larger pendant rotation is more likely a mistake than an intent; the axis limit check (MONETS
  -90..360, MONETN -170..300) prevents a target the axis would clamp or refuse (#2 `F_AxisTurn`).
- Focus: 0.02 mm = `fHoldBand`, so each step is a real move [inferred]. Stepped, not ramped, because a target change during a
  move restarts it (`FB_FocusControl` l. 123); a 0.02 mm move at 0.2 mm/s ends before the next step 0.5 s later. +-1 mm:
  [unknown] the usable focus range; confirm against a focus series on site.
- Hold stages 0.5 s / 3 s: a tap never ramps; the operator gets fine control first.

### 3.5 FBMTC: clear offsets on a new target

New private `_ClearOffsets()`: `fElevationOffset := 0.0; fAzimuthOffset := 0.0; fDerotatorOffset := 0.0;` plus one HINT log line
when any was non-zero. Called only after an accepted target (after `_AcceptMove` returned TRUE): in `MoveRaDec`, `Track`,
`TrackRaDec`, `Slew`, `MoveAltAz`, and in the input-edge branches for `bGoto`, `bSlew`, `bTrack` (l. 580-594). Not on a
rejected command (the old target and its offsets stay), not on Home/Park/Init (D6). `bGoHome` edges: no.

Caveats (section 8): the MQTT data pass runs before the verbs (BROTLib#39), so an offset sent in the same message as `track`
would be cleared; and `derotator` is the client's derotator setpoint (same property), so clearing it changes what a client
sees after `track` (D2).

### 3.6 Reset button in automatic

The pendant does not call any axis or focus `Reset()` in automatic or off. `MAIN` keeps resetting `TelescopeControl`,
covers, hydraulics and `SafetyHandling` on the button (unchanged; it does not need the pendant key, O1). No "clear offset"
function on the pendant: a reset press would also reset the telescope. Offsets are cleared by a new target or by MQTT.

### 3.7 Lamps in automatic

- 5-8: direction lamp on while a step in that direction would be accepted (`bNudgeOk`, not at the clamp, not at the elevation
  window / derotator limit / focus range). Enable button not held or not online: direction lamps off.
- Error lamp: `bError` (selector 0), or `iTelescope.ReadyState < 0` (telescope error).
- 1-3: as today. Off mode 5-8: all off except the error lamp.

### 3.8 Mode change, key off, enable release mid-nudge

- Key off, enable released, selector moved, not online any more: `bNudgeOk` FALSE, the nudge stops in that scan, the offset
  keeps the value reached (no roll-back), `bNudgeArmed` FALSE.
- Automatic -> manual (cabinet key to Local with a direction held): the nudge stops; offsets stay (FBMTC suspended, cleared by
  the next target after `power=1`). With D3 the manual jog needs the direction released once after mode entry.
- Manual -> automatic mid-jog: `nDriven := 0` releases `Move*` once; a pending Enable drop is discarded (3.2), Enable stays as
  the operator left it; nudges need `power=1` (READY_STATE 1) and a fresh press.
- Manual -> off: release as #3 including the delayed Enable drop.

### 3.9 Telemetry and visibility

- ALT/AZ offsets: already published; the pendant changes appear there with no change.
- Add `stTelemetry.fDerotatorOffset` and publish `POSITION.INSTRUMENTAL.DEROT.OFFSET` in `FB_MonetTelescopeTelemetry` (name to
  align with pybrotlib, O3).
- One HINT log line per nudge at release: `pendant: elevation offset +35.0 arcsec, now 120.0 arcsec` (function, change, new
  value), so a pendant change is visible in the log next to MQTT writes.
- `nNudge` (function nudged in this scan, 0 none) as an FBMPC VAR for ADS online view; no new MQTT topic.

## 4. Interaction with #168 (mode state machine)

- The pendant reads the cabinet-key mode from `MAIN`, not an FBMTC mode state. With #168, the SUSPENDING phase (the #118 stop) is
  still `manual` for the pendant; today `SuspendAutomatic` runs before the axes and writes `Move*` FALSE every scan during the
  stop, so it wins over a pendant jog written in the previous scan [inferred]. Unchanged by this plan.
- RETURNING / INIT_REQUIRED are covered by the online gate (READY_STATE 1).
- If #168 lands: FBMTC should expose its mode state and the pendant gate should use "manual handover complete" for jogs and
  "AUTOMATIC and READY" for nudges. One input change, no redesign. #168's rule "always-on reset must not write to an axis the
  operator jogs" is helped by 3.6 (the pendant no longer resets axes outside manual).

## 5. Tests (TcUnit, `FB_MonetPendantControl_Tests`; no new suite)

### 5.1 Seams

- Existing `Scan`/`Hold` helpers get `eMode : E_TelescopeMode` and pass `eTelescopeMode`; **all existing #3 cases pass
  `manual`** (the default `off` would make them fail), `tCycle := T#10MS`, `iTelescope`.
- New `FB_AltAzTelescopeStub IMPLEMENTS BROTLib.I_AltAzTelescope` in the tests: the three offset properties as plain fields,
  `ReadyState` settable, the rest empty.
- `FB_FocusMock`: `Position` get/set and `MinPosition`/`MaxPosition` must work unscanned [read: the mock follows `fPosition`];
  a reset counter on `FB_ElevationMock` for the reset case (or assert via a latched error that stays).

### 5.2 Cases

1. `Automatic_Selector_5_To_8_Writes_No_Enable_Or_Move` (the #177 bug): test sets `Enable` TRUE on all four, auto, key, sel 5..8,
   button released and pressed, directions pressed -> `Enable` TRUE and `Move*` FALSE on all.
2. `Automatic_Elevation_Tap_Adds_One_Step`: up for one scan -> `AltitudeOffset` = 5/3600.
3. `Automatic_Nudge_Requires_Enable_Button`.
4. `Automatic_Nudge_Requires_Online`: `ReadyState` 0.0 / -1.0 -> no change.
5. `Automatic_Hold_Rate_Is_Cycle_Time_Independent`: 300 scans at 10 ms vs 30 scans at 100 ms in the slow stage -> same offset
   (within 1e-9).
6. `Automatic_Hold_Ramps_Slow_Then_Fast`: offsets at 0.4 s, 2 s, 5 s held match 3.4.
7. `Automatic_Nudge_Adds_To_Concurrent_Write`: between scans the test writes `AltitudeOffset` (stands for MQTT) -> next value is
   that plus the delta.
8. `Automatic_Azimuth_Is_Sky_Frame`: right tap -> `AzimuthOffset` +5/3600 regardless of `fbElevation.Position` (no cos in the
   pendant).
9. `Automatic_Offset_Clamped_At_Max` and `Automatic_Offset_Beyond_Max_Only_Decreases`.
10. `Automatic_Elevation_Nudge_Respects_Goto_Window`: `fbElevation.Position` 89.499 -> up refused, lamp up off, down accepted.
11. `Automatic_Derotator_Nudge_Respects_Axis_Limit`.
12. `Automatic_Focus_Steps_And_Repeats` and `Automatic_Focus_Nudge_Limited_To_Range`; `Automatic_Focus_Rebases_On_New_Target`.
13. `Manual_To_Automatic_Releases_Move_Keeps_Enable`: manual jog sel 7 held, mode auto, `tEnableRelease` T#0S -> `Move*` FALSE,
    `Enable` TRUE after several scans, `nEnablePending` 0.
14. `Manual_To_Off_Drops_Enable` (the #3 behaviour stays outside automatic).
15. `Off_Mode_Is_Inert_For_Axes_And_Focus` (covers hold-to-run still works: `Covers_Hold_To_Run_In_All_Modes` with
    `FB_CoversMock` if the concrete cover type allows, else skipped as in #3).
16. `Held_Direction_Needs_Release_After_Mode_Change` (auto entry; and manual entry if D3).
17. `Key_Off_Mid_Nudge_Stops_And_Keeps_Offset`.
18. `Automatic_Reset_Does_Not_Reset_Axes`.
19. `Unwired_Telescope_Does_Not_Fault`: auto, `iTelescope` 0, sel 5-8 -> no write, no page fault.

FBMTC (in the existing rig suite `FB_MonetRig_Tests`, no new suite): `New_Track_Clears_Offsets`, `New_Slew_Clears_Offsets`,
`Goto_Edge_Clears_Offsets`, `Rejected_Target_Keeps_Offsets`, and a telemetry line for `DEROT.OFFSET` in the recording-comm test.

Run as in #3 plan 5.3 (CI on push, or `tctool` locally). Ask before any local XAE/tctool run on this machine (it is the CI
runner).

## 6. Rollout

0. **Wiring first, on 0.17.0**: MONETS and MONETN PRs adding only `eTelescopeMode := eTelescopeMode` to `PendantControl(...)`.
   No behaviour change (the input is unused in 0.17.0). This guarantees the gate never lands unwired.
1. MONETcommon PR: FBMPC (3.1-3.3, 3.6-3.8), FBMTC `_ClearOffsets` and telemetry (3.5, 3.9), tests, README pendant section,
   this plan's status and index line. Version **0.18.0** (new inputs, behaviour change) (D5). CI green.
2. MONETcommon release 0.18.0.
3. MONETS pin PR: pin 0.18.0 + `iTelescope := TelescopeControl`, `stMotion := stTelescopeMotion`.
4. MONETN pin PR: pin 0.18.0 + `iTelescope := TelescopeControl`.
5. On-site check (section 7) per site, then close #177. Do not merge a pin PR without step 0 merged on that site.

If step 0 is skipped, step 3/4 must contain the wiring in the same commit as the pin.

## 7. On-site hardware check (MONET/S and MONET/N separately; not inferred from one site)

Before: roof closed, covers closed, someone at the E-stop and one at the cabinet key. Online in XAE: `PendantControl.nSelectionStable`,
`nOwned`, `nEnablePending`, `nNudge`, `TelescopeControl.fElevationOffset/fAzimuthOffset/fDerotatorOffset`, `ReadyState`,
`<Axis>Control.bEnable/bMovePos/bMoveNeg/Position`, `FocusControl.fPosition`, `HydraulicsControl.bBrakeOpen`.

1. After step 0 (0.17.0 + wiring): nothing changes; still do not turn the pendant key on with sel 5-8 in automatic.
2. Automatic, online, not tracking: key on, sel 7, 8, 6, 5 in turn, enable released and pressed, directions pressed: no axis
   `bEnable` drops, no `Move*`, brake stays open (the #177 bug gone).
3. Tracking a target at el ~50 (roof closed is fine): sel 7 tap up -> `ALT.OFFSET` +5 arcsec, `ElevationControl.Position` +5 arcsec;
   hold 2 s and 6 s, compare with 3.4; watch following error / tracking state. Same for sel 8 at el ~50 and ~80 (axis change
   = sky change / cos el).
4. Hold until the 600 arcsec clamp: stops; reverse works.
5. MQTT `elevationoffset` during a held nudge: the sum continues from the MQTT value.
6. New `track`: all offsets 0 (and the HINT log line).
7. Sel 6 derotator nudge; sel 5 focus: steps of 0.02 mm, auto-repeat, focus brake behaviour with the 30 s brake delay
   (MONETS `ptBrakeDelay`).
8. Mid-nudge: key off; enable released; cabinet key to Local with a direction still held: nudge stops, no jog until released
   (D3).
9. Manual: #3 checks A1-A4 and B7 short repeat (jog unchanged). Manual jog of el with enable held, key to Remote: `Move*` FALSE,
   `bEnable` stays TRUE, no drop after 2 s; `power=1` works; nudges only after READY_STATE 1.
10. Off (Local, hydraulics off): sel 5-8 do nothing (focus/derotator jog without hydraulics is gone, intended); covers 1-3 work.
11. Reset button in automatic: telescope reset as before, no axis reset from the pendant.
12. With pyobs running: an acquisition after a pendant nudge (does pyobs overwrite it, O2).

Fail if: any `bEnable`/`Move*` write in automatic or off; a nudge without the enable button; rate depends on how long the
scan is; an offset survives a new target; any motion in 10.

## 8. Risks and open questions

- **pyobs re-sends absolute offsets** until settled: a pendant nudge during a pyobs offset settle is overwritten; if pyobs guides
  through these offsets, the pendant fights the guider (O2). Operators should not nudge while pyobs acquires or guides.
- **Clear on new target**: an offset sent in the same MQTT message as `track` (data pass first) or just before it is cleared.
  [unknown] whether pyobs-brot or brotgui do that (O2).
- **Derotator**: `derotator` and `derotatoroffset` are the same property; a client that sets the instrument angle before `track`
  loses it on the clear (D2).
- **Focus** has no offset; the pendant moves the absolute target, MQTT `focus` re-sends overwrite it, and a re-base follows.
- **Tracking response** to target steps (5 arcsec tap) and to an extra ramp rate: [unknown] how `FB_AxisControl` tracking
  absorbs them (following error, overshoot). Check 7.3.
- **Near zenith** the azimuth axis change is up to 115x the sky change (capped at 10 deg); elevation window 89.5 limits it;
  consider refusing azimuth nudges above 85 deg (O4).
- **Off mode inert** removes today's focus/derotator/azimuth jog with hydraulics off: operating procedures change.
- **Covers hold-to-run in automatic** can stop a power-on/park cover sequence (policy keeps it; pre-existing).
- `_TaskInfo`/`GETCURTASKINDEXEX` reference: fallback a `tCycle` input set by `MAIN` (T#10MS).
- O1: `MAIN` resets the telescope on the pendant reset button without the pendant key (pre-existing). Gate it with the key?
- O3: telemetry topic name for the derotator offset.
- O4: azimuth nudge cut-off near zenith.

## 9. Until then

Do not turn the pendant key on with the selector on 5 to 8 while the cabinet key is on Remote (automatic). The wiring PR
(step 0) alone does not change this; only the 0.18.0 pin does. The #3 on-site check keeps the pendant key off in automatic.

## 10. Decisions for Tim

Taken 2026-11-06: all as recommended. D2 yes, with the O2 check (pyobs-brot, brotgui) still open before release; off mode stays
inert (the loss of the hydraulics-off jogs is accepted, ask the site technicians before release).

- **D1** Values of 3.4 (step, rates, hold stages, maxima): as suggested, final after 7.3-7.7.
- **D2** Clear the derotator offset on a new target too (as decided), or El/Az only because `derotator` is a client setpoint.
  Recommended: as decided, after confirming no client sends `derotator` before `track` (O2).
- **D3** Manual jog also armed only after a release at mode entry (small change to manual). Recommended yes.
- **D4** Nudges only at READY_STATE 1. Recommended yes.
- **D5** Version 0.18.0. Recommended.
- **D6** Clear offsets also on Park/Home/Init. Recommended no (new target only).

## 11. Steps

1. MONETS/MONETN wiring PRs on 0.17.0 (section 6 step 0).
2. FBMPC: inputs (3.1), gating and `_DropEnable` guard (3.2), nudge engine with arming and clamps (3.3/3.4), reset (3.6), lamps
   (3.7), log line and `nNudge` (3.9). File: `FB_MonetPendantControl.TcPOU`.
3. FBMTC: `_ClearOffsets` at the eight accept points (3.5), derotator offset telemetry (`FB_MonetTelescopeTelemetry`,
   `ST_MonetTelemetry`).
4. Tests: helpers with mode/tCycle/iTelescope, `FB_AltAzTelescopeStub`, mock additions, cases 5.2; README rows.
5. README pendant section (modes table of section 2), this plan's status, `index.md` line.
6. Push, CI, fix until green; release 0.18.0; pin PRs MONETS and MONETN (section 6).
7. On-site check section 7 on both sites; comment on #177 and close.
