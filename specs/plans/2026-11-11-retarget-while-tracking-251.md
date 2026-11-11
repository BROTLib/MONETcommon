# Plan: retarget while tracking without a one-scan el/az setpoint step (MONETcommon#251)

Date: 2026-11-11. Status: **plan only, nothing changed.** Base: `origin/develop` = `dbd67e1` (0.20.0, includes #250).
(The local `develop` checkout is 190 commits behind; all line numbers below are from `origin/develop`.)
Files: `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU` (FB_MTC), `FB_OffsetRamp.TcPOU`,
tests in `MONETcommonTests/.../POUs/FB_MonetRig_Tests.TcPOU`. The derotator is not changed (#252).

## 1. What the code does today (read, not run)

**The `Track()` path the issue cites does not stay in `Tracking`.** `Track()`/`TrackRaDec()` (FB_MTC 3035-3083) call
`_AcceptMove(goto)`; while a track runs `_CommandState.bMoveActive` is TRUE (3118-3123), so `F_CommandAcceptance` says
`replace` -> `bSupersedeRequest`, `eSupersedeKeep := goto`, `bGotoActive := TRUE`. Next body (BROTLib
`FB_BaseTelescopeControl` 258-283): `bTrackActive := FALSE`, `bMoveAbortPending := TRUE` -> the arbiter dispatches **stop**
-> `_StopTelescope` (2577-2628) drops `Tracking` (one scan, BROTLib#101), then MC_Stop -> standstill -> `_GotoTelescope`
(untrack/start_move/wait_moves/wait_standstill/done, 1160-1218) -> `bTrackActive := bAutoTrack` -> `_TrackTelescope`
move onto the track -> `Tracking` with the #250 track-start blend. `_TrackTelescope` is not called while the stop runs,
so no new target reaches a tracking axis. Cost: a full stop and restart of all three axes for every retarget, also for
a 10 arcsec refinement. `Repeated_Track_Retargets_And_Second_Stop_Works` passes through exactly this (step 30 waits for
`NOT bMoveAbortPending`); it does not check that `Tracking` stayed TRUE.

**The real one-scan jump: a direct write of the inputs `fRightAscension`/`fDeclination` while tracking.** They are
`VAR_INPUT` of the base block and are read every scan by `eq2hor` in `_ComputeTargets` (FB_MTC 2350). The MONETN Visu
binds `MAIN.TelescopeControl.fRightAscension`/`fDeclination` (`MONETN/MONETNRuntime/VISUs/Telescope.TcVIS`; the goto
button then calls `Track()` with the same values, MAIN 163-166), and any ADS client can write them. In that scan
`fElevationCalc`/`fAzimuthCalc` (2380/2382) are the new target, and `_TrackTelescope` writes them to the tracking axes:
**elevation line 2676, azimuth line 2706** (`Position := Calc + TrackBlend.fValue`), derotator line 2711. The axes stay in
`Tracking`; with the AX5125 in mode 11 this is the velocity kick of the old offset step (50-70 % overshoot for 0.01 deg).

**Related hole:** a target change during the move onto the track (the `NOT Tracking` branch, 2653-2672 / 2683-2702):
MC_MoveAbsolute keeps the start target, and at `MoveDone` a gap above `fTrackStartBlendMax` gets `SetValue(0.0)` = the
old one-scan step (#250 kept that on purpose for turn changes/faults).

Step 0 of the implementation: confirm this reading on the rig (section 4, test 1 fails today, test 2 passes today).

## 2. Design

One retarget detector in `_TrackTelescope`, for both sources (direct write and, phase B, `Track()`).

**Detection.** New VAR: `fRaTracked`, `fDecTracked` (the RA/Dec of the last tracking scan), `fElSetLast`, `fAzSetLast`
(setpoints written last scan), `fElVelLast`, `fAzVelLast`, `nTrackScan` (scan counter of the last `_TrackTelescope` call;
a counter `nScan` incremented in `_ScanBegin`). A retarget is: el and az `Tracking`, `nTrackScan = nScan - 1` (so stale
values after a stop/park/suspend/reset/manual are never used, no clearing needed in the 6 places that drop `Tracking`),
and `ABS(fRightAscension - fRaTracked) > 1E-9 OR ABS(fDeclination - fDecTracked) > 1E-9`. Offsets do not trigger it
(they have their own ramp). Updated at the end of every `_TrackTelescope` call.

**Gap.** Per axis, `fGap := (fSetLast + fVelLast * dt) - fCalcNew` (the old setpoint carried on by one scan, so the
sidereal velocity does not drop to 0 in the retarget scan). Azimuth in axis units after `F_AxisTurn` (`bHold` keeps the
turn while tracking, so the gap is never a 360 deg wrap). Derotator: `fDeroGap := fbDerotator.Position - fDerotatorCalc`
is only *read* (guard).

**Small retarget** (`ABS(fElGap) <= fTrackStartBlendMax AND ABS(fAzGap) <= fTrackStartBlendMax AND ABS(fDeroGap) <=
fRetargetDerotatorMax` and `fOffsetAcceleration > 0`): `fb<Axis>TrackBlend.Shift(fGap)` for el and az, stay in
`Tracking`, write `Position := Calc + blend` as now (= the carried-on old setpoint in this scan). `Shift` is a new method
of `FB_OffsetRamp`: `fValue := fValue + fDelta; bDone := FALSE;` **keeping `fRate`**. The issue's
`SetValue(old setpoint - new target)` zeroes the blend velocity; mid-blend that is a velocity step of up to 0.32 deg/s,
so `Shift` is used (at rest both are identical). Threshold: reuse `fTrackStartBlendMax` (0.5 deg, same meaning: "a gap
above this is not blended"), `<=` blends, as at the track start. A 0.5 deg blend at the defaults takes about 3.2 s, peak
0.32 deg/s, at most 0.0032 deg/scan; 0.05 deg takes 1.0 s.

**Large retarget** (any of the three over its limit, or `fOffsetAcceleration = 0`): leave the scan **before any
setpoint is written** (`RETURN` at the top of `_TrackTelescope`: el, az *and derotator* keep last scan's setpoint, so the
derotator code is untouched) and request the replace exactly as `Track()` does: `IF _AcceptMove(goto, no_command) THEN
bGotoActive := TRUE`. Next scan: supersede -> abort stop -> goto -> track start (today's proven path, all three axes).
Offsets: not cleared (the RA/Dec were already written; same as the Visu goto today).

**Derotator guard `fRetargetDerotatorMax`** (new input, default `fDerotatorOffsetRate * tCycle` = 0.005 deg, the step
#236 already allows per scan). In the small case the derotator still gets its new target in one scan; without the
guard a 0.05 deg el/az retarget could step the AX5206 (F324, MS-165/172). Consequence: with the default many small
retargets take the large path (= today's behaviour) until #252 gives the derotator a blend; then raise or drop the guard.

**Phase B: `Track()`/`TrackRaDec()` while tracking** (optional, same PR or after #252). If `TCS_command = track`, el/az/dero
`Tracking`, `NOT bMoveAbortPending`, `bAutoTrack`, and the sky separation old->new RA/Dec <= `fTrackStartBlendMax`:
run `F_CommandAcceptance` (rejections unchanged) but do **not** set `bSupersedeRequest`/`bGotoActive`; write RA/Dec
(+ `_ClearOffsets` as now). The detector then decides small/large with the exact axis gaps (az = sep/COS(el) can still
exceed 0.5 near the zenith -> large path from the body). A larger separation keeps the current replace path immediately,
so `Repeated_Track_Retargets_...` (0,-85 -> 180,-80) is unchanged. `MoveRaDec()` (a goto) is not changed.

**Mid-blend:** `Shift` adds to the running blend, velocity continuous; the gap test uses the full new gap.
**Mid-move onto the track:** change the `MoveDone` branch: if the gap > `fTrackStartBlendMax`, do not set `Tracking`;
drop `MoveAxis` for one scan and move again (new Execute edge latches the new target), at most 3 times, then the old
behaviour with a log entry. **Mid-goto / mid-abort stop:** `_TrackTelescope` does not run; the goto/track move latches the
target it sees, the track start blends the rest (above). **Offsets:** independent; the offset ramp keeps running and is
part of `Calc`, so the gap is measured against the full target. **Stop/park/suspend/reset/error:** they drop `Tracking`
and stop calling `_TrackTelescope`; the scan counter disarms the detector, the blend is re-set at the next track start
(existing `SetValue`). A Stop during a retarget blend is a normal stop from tracking (no change). `fOffsetAcceleration = 0`
disables the small path (all retargets large), the A/B switch of #250.

Telemetry/diagnostics: `eLastRetarget` (none / blend / move) and `nRetargetCount`, for the tests and the scope.

## 3. Risks and behaviour changes

- Direct RA/Dec writes while tracking: before, an instant jump; now a blend (small) or a stop+goto (large). Large retargets
  by direct write now stop the telescope: intended, the Visu goto does the same a moment later.
- Phase B: `Track()` to a nearby target no longer stops the axes; pyobs/brotgui see `bGotoActive` stay FALSE and `TCS_command`
  stay `track`. Check BROTLib comm/pyobs do not wait for a goto state after `track` (read `FB_Comm_MQTT_Influx`, pyobs driver).
- The reported RA/Dec (`hor2eq` of the actual axes) reaches the new target only after the blend (<= 3.2 s at 0.5 deg).
- Restart-the-move at track start replaces a jump that #250 kept for turn changes/faults; bounded to 3 tries.
- `_AcceptMove` called from the body (not a method call): the replace is consumed in the next body, as for a method.
- The 1E-9 RA/Dec compare: a client re-writing the same value does not trigger (as `_ClearOffsets`).

## 4. Rig tests (FB_MonetRig_Tests methods, no new suite; mock with `SetMoveLatchesTarget(TRUE)` like
`Track_Start_Setpoint_Begins_At_The_Axis_Position`)

Common: per scan record `d1 = |pos - posPrev|` and `d2 = |pos - 2 posPrev + posPrev2|` for el and az (setpoint =
`rig.fbElev/fbAzim.fPosition`); pass = `d1 < fOffsetVelocity*dt + 0.0005` (0.0055 deg) and `d2 < 0.0001 deg`
(velocity change < 0.01 deg/s per scan; the ramp gives 0.002, a 0.01 deg step gives d2 = 0.01). No `bError`, no mock NC error.
1. `Retarget_By_Direct_RaDec_Write_Is_Blended`: track (0, -85), 1 s in, write `fbTel.fDeclination := -85.02` directly.
   Pass: d1/d2 limits, `Tracking` stays TRUE every scan, `eLastRetarget = blend`, setpoint within 1E-4 deg of the new
   target after 2 s. Fails on today's code (d2 ~ 0.02). If the derotator guard sends it to the move path, pick a smaller
   delta (check with a Python eq2hor beforehand) or set `fRetargetDerotatorMax` in the test.
2. `Large_Retarget_By_Direct_Write_Goes_Through_The_Move`: write RA 180 / Dec -80. Pass: in the retarget scan all three
   setpoints unchanged, then `bMoveAbortPending`, stop, goto, track of the new target; d1/d2 limits on every tracking
   scan; `eLastRetarget = move`; nothing rejected.
3. `Retarget_Mid_Blend_Keeps_The_Velocity`: direct write -85.02, after 0.3 s (blend running) -85.0. Pass: d2 limit across
   the second retarget, ends at the target, still tracking.
4. `Retarget_During_The_Move_Onto_The_Track`: in step 15 of `Track_Start_Setpoint_Begins_At_The_Axis_Position` (axes moving onto the track) write Dec -85 -> -84 (gap > 0.5 deg).
   Pass: no first-tracking-scan step (gap < 1E-9 as in #250's test), a second move is seen.
5. `Stop_After_Retarget_Settles`: test 1, `Stop()` 0.2 s into the blend. Pass: stop settles in < 400 scans, no tracking,
   no error; next `Track()` starts with a blend from the axis position.
6. Phase B: `Small_Track_Retarget_Stays_Tracking`: `Track(0, -85.02)` while tracking. Pass: `Tracking` never FALSE,
   `bMoveAbortPending` never TRUE, `nCommandRejectCount` unchanged, d1/d2 limits. `Repeated_Track_Retargets_...` unchanged.

## 5. CI

`tcbuild.yml` runs on **push only** (self-hosted runner, never `pull_request`): installs MONETcommon as a library (= the
compile) and runs all TcUnit suites on the user-mode runtime. Nothing can be compiled locally on this PC; every compile
is a push to a branch (e.g. `fix/retarget-blend-251`). Push test-first: tests 1/3/4 must fail on the unchanged code
(proves the rig sees the jump), then the fix. Watch `GVL_Diag.sFailures`; a missing runtime/licence skips the tests with a
warning, so check that they actually ran. New POU method `FB_OffsetRamp.Shift` must be in the `.plcproj` (it is inside the
POU file, no project change).

## 6. Hardware check for Tim on MONET/N (after pinning the release in MONETN, no MAIN change)

Scope (same project as the 2026-11-11 step tests): el/az set position, actual position, following error, derotator set
position and following error, `eLastRetarget`, `Tracking`.
1. Track a star. Visu: change Dec by +0.02 deg (direct write). Expect: no set-position step (largest scan-to-scan change
   about sidereal + < 0.002 deg), blend about 0.6 s, following-error peak far below the 0.01 deg step test, no ring.
2. Visu/MQTT: a target 5 deg away. Expect: stop, goto, track start as before; no NC 0x4257/0x4260, no AX5125/AX5206 F-error.
3. (Phase B) MQTT `track` 0.05 deg away: tracking stays on, blend about 1 s, derotator follows without F324.
4. `Stop` during a blend: normal stop, no NC error.
5. A/B: `fOffsetAcceleration := 0` online = every retarget takes the move path (no blend). Pass for all: no drive or NC
   error, no one-scan set-position step, el/az following error settles within 1 s.
