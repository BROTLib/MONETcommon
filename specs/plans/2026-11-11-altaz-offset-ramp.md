# Elevation/azimuth offset ramp and track-start blend

Date: 2026-11-11. Status: **implemented on `feat/altaz-offset-ramp` (base `origin/develop` = `abdbdf6`, 0.19.2); CI compile and
TcUnit via the PR; not run on a telescope.** Hardware check on MONET/N (and later MONET/S) is Tim's call, section 6.
Repo: BROTLib/MONETcommon. Main files: `MONETcommon/MONETcommon/FB_OffsetRamp.TcPOU` (new),
`FB_MonetTelescopeControl.TcPOU` (`_ComputeTargets`, `_TrackTelescope`, new inputs).
Trigger: MONETN `DriveManager/parameter-lists-2026-11-10/README.md` (step tests 2026-11-11).

Legend: **[read]** seen in the code, **[measured]** from the MONETN step tests, **[inferred]** follows from the code, not observed.

## 1. Problem

- [measured] After the velocity-loop tuning (el S-0-0100 18000, az 45000), a 0.01 deg offset step during tracking rings:
  elevation 64-70 % overshoot at about 2.5 Hz, azimuth 52-57 % at about 1.7 Hz, settled after about 1 s. Changing the drive P/Kv
  barely changes the overshoot. Slews and tracking (ramped motion) do not ring.
- [read] Path of an offset: MQTT `elevationoffset`/`azimuthoffset` (BROTLib `FB_Comm_MQTT_Influx`) and the pendant (#177)
  write `AltitudeOffset`/`AzimuthOffset` = `fElevationOffset`/`fAzimuthOffset`. `_ComputeTargets` added them to the target every
  scan (`fElevationCalc + fElevationOffset`, `F_AzimuthOffsetOnAxis(fAzimuthOffset, el)`), `_TrackTelescope` writes the target to
  `fbElevation/fbAzimuth.Position` every scan, and BROTLib `FB_Axis` feeds it as `TrackPosition` to `MC_ExtSetPointGenFeed`
  (the fed velocity is the track rate, clamped to 0.009 deg/s, independent of the jump).
- [inferred] So a 0.01 deg offset is a 0.01 deg jump of the NC set position in one 10 ms cycle. The AX5125 in mode 11
  (position control lag-less) derives its velocity feedforward from the position command: the jump is a velocity kick of the
  order of 1 deg/s, which explains why the loop gains hardly matter.
- [read] Track start: `_TrackTelescope` first moves each axis with `MoveAxis` (MC_MoveAbsolute takes the position at the rising
  Execute), then sets `Tracking`. The setpoint of the first tracking scan is the target *now*; the axis stands where the target
  was at the start of the move. [measured] About 0.008 deg on the azimuth after a slew into tracking: the same kind of kick.
- [read] The derotator already has a rate limit for its offset (#236/#246, `fDerotatorOffsetRate`); untouched here.

## 2. Design

`FB_OffsetRamp`: a follower of a requested value with the velocity limited (`fVelocity`) and the change of velocity limited
(`fAcceleration`); time-optimal bang-bang profile with the discrete braking law
`v = a*dt*(SQRT(0.25 + 2|e|/(a*dt^2)) - 0.5)`, so it does not overshoot by the last step. The applied velocity is continuous
(changes by at most about `a*dt` per scan); only the acceleration has steps. A new request during a ramp continues from the current
value and velocity. `bEnable` FALSE or a limit <= 0: follows at once. `SetValue()` puts the value at rest (blend start).

Use in `FB_MonetTelescopeControl`:

1. **Offsets.** `fElevationOffsetApplied`/`fAzimuthOffsetApplied` (sky frame) follow `fElevationOffset`/`fAzimuthOffset` through
   one ramp each, enabled while `bTrackActive OR fb<Axis>.Tracking` (as #246 for the derotator). They replace the requested offsets
   in the target (`fElevationCalc`, `F_AzimuthOffsetOnAxis(fAzimuthOffsetApplied, el)`) and in the first `hor2eq` (the reported
   RA/Dec subtracts what the axes actually carry). Outside a track the applied offset is the requested one at once: goto/slew are
   moves with their own profile and are unchanged (the slew still reads `fElevationOffset`/`fAzimuthOffset` directly). The
   derotator target follows the smooth el/az automatically. Telemetry keeps publishing the requested offsets.
2. **Track-start blend.** In the scan `_TrackTelescope` sets `Tracking := TRUE` for the elevation/azimuth, the blend is set to
   `fActualPosition - target` (at rest) and the setpoint written that scan is the axis position. From then on
   `Position := target + blend`, and the blend ramps to 0 with the same limits. The blend is in axis units after `F_AxisTurn`; a
   gap above `fTrackStartBlendMax` (0.5 deg: a turn change or a fault) is not blended (old behaviour). The derotator is not blended
   (its #246 lock/hold logic stays as it is; see section 7).

New inputs (both sites use the defaults unless their `MAIN` sets them; nothing to change in MONETN/MONETS `MAIN`):

| Input | Default | Meaning |
|---|---|---|
| `fOffsetAcceleration` | 0.2 deg/s^2 | change of velocity of the offset ramp and the blend; **0 = old behaviour (steps), both** |
| `fOffsetVelocity` | 0.5 deg/s | velocity cap (same as `fDerotatorOffsetRate`) |
| `fTrackStartBlendMax` | 0.5 deg | larger track-start gaps are not blended |

Profile at the defaults (10 ms scan, simulated in Python and pinned by the unit tests):

| Offset | Time to reach | Max per scan | Peak velocity |
|---|---|---|---|
| 5 arcsec (pendant tap) | 0.15 s | 0.58 arcsec | 0.016 deg/s |
| 0.01 deg (the step test) | 0.43 s | 1.6 arcsec | 0.044 deg/s |
| 1 arcmin | 0.56 s | 2.1 arcsec | 0.058 deg/s |
| 10 arcmin (pendant limit) | 1.8 s | 6.6 arcsec | 0.18 deg/s |
| 2 deg | 6.5 s | 18 arcsec | 0.5 deg/s (cap) |

Velocity change per scan <= 0.0023 deg/s in all cases (the old step: about 1 deg/s in one cycle). Pendant held at 60 arcsec/s: lag
2.2 arcsec, none after release.

Why this shape and not a fixed-time S-curve: a fixed duration would make large offsets fast; the rate/acceleration limits scale
with the offset and the follower retargets without a velocity step (MQTT offsets in quick succession, pendant). Why not a filter on
the whole target: it would lag the sidereal motion; the ramp only acts on the offset/gap, the target itself is passed through, so
**tracking accuracy is unchanged once the ramp is over** (the offset/gap is fully applied, no steady-state error).

Tuning note [inferred]: the ramp time of a 0.01 deg step (0.43 s) is about one period of the elevation mode (0.4 s) and 0.7 of
the azimuth one (0.6 s). If a residual ring shows, lower `fOffsetAcceleration` (0.1 -> 0.6 s for 0.01 deg) before touching the
drives; it can be set per site in `MAIN` (`TelescopeControl.fOffsetAcceleration`) and online.

## 3. Interactions checked [read]

- MONET/S: same library, same defaults; nothing in its `MAIN` needs to change. Its AX5206 derotator path is unchanged (#236/#246).
- `_ClearOffsets` (#177) on a new target: the requested offsets drop to 0; during a running track the applied ones ramp to 0.
- Stop/park/suspend: `Tracking` drops, the applied offsets equal the requested at once (the axes stop anyway), the blend is set
  again at the next track start.
- `F_TargetBelowHorizonWarning`, `F_GotoTargetCheck`, `F_AxisTurn`: use `fElevationCalc`/`fAzimuthCalc` with the applied offsets;
  outside a track identical to before.
- Pendant (#177) reads/writes the requested offsets (`AltitudeOffset`), unchanged; its window check uses `fbElevation.Position`.

## 4. Tests (MONETcommonTests, no new suite: 35 of 35 slots used)

- `F_AxisTurn_Tests` (the tracking setpoint helpers), `FB_OffsetRamp`: `OffsetRamp_Step_Is_Ramped` (0.01 deg: 0.3-0.5 s,
  < 0.0005 deg/scan, velocity change < 0.003 deg/s/scan, no overshoot, ends at rest), `OffsetRamp_Disabled_Follows_At_Once`
  (bEnable FALSE, a = 0, v = 0), `OffsetRamp_Retarget_Keeps_The_Velocity_Continuous` (+0.01 then -0.01 after 0.15 s; 2 deg capped
  at 0.5 deg/s), `OffsetRamp_SetValue_Blends_To_The_Target` (0.008 deg gap to 0, no crossing, < 1 s).
- `FB_MonetRig_Tests` on the mock rig: `AltAz_Offsets_Set_During_A_Track_Are_Ramped` (0.01 deg el and az during a track: el
  setpoint < 0.001 deg/scan, az < 0.002 deg/scan, whole offset incl. 1/COS(el) after 1.5 s, still tracking, no error);
  `Track_Start_Setpoint_Begins_At_The_Axis_Position` (mock moves latch their start target, `FB_MonetRig.SetMoveLatchesTarget`;
  0.005 deg offsets written during the move onto the track make a gap; the first tracking setpoint equals the axis position, then
  < 0.001 deg/scan, gap closed). Both rig tests fail on the old code (step of 0.01 deg / jump at track start).
- Mock change: `FB_AxisMockCore.bLatchTarget` (default FALSE, the old model, so the other suites are unchanged).

## 5. What Tim runs in XAE (or CI on the PR)

CI (`tcbuild.yml`, on push) installs the library and runs all suites. Locally, if wanted:
1. Open `MONETcommon.sln`, Build -> Rebuild `MONETcommon`; expect 0 errors (new POU `FB_OffsetRamp` is in the `.plcproj`).
2. Save and install as library (as `release-library.yml` / `tools/Invoke-XaeBuild.ps1 install`), then open
   `MONETcommonTests/MONETcommonTests.sln`, rebuild, activate **on the user-mode runtime of the test target only**, run.
3. Report: build errors/warnings; TcUnit totals and any failed test among the six above (with `GVL_Diag.sFailures`).

## 6. Hardware check (Tim's call, not done)

On MONET/N after pinning the release in MONETN (no MAIN change):
1. Track a star; scope the elevation and azimuth set position, actual position and following error (same Scope project as the
   2026-11-11 step tests).
2. `elevationoffset` +0.01 deg, then back to 0; same for `azimuthoffset`. Expect the set position to ramp over about 0.43 s and
   the overshoot to drop well below the 50-70 % of the step; report overshoot, settling time and the peak following error.
3. Slew into tracking: report the set-position jump at track start (was about 0.008 deg on azimuth) and the following error.
4. Comparison: `TelescopeControl.fOffsetAcceleration := 0` online restores the old steps (A/B in the same night); optionally
   0.1 if a residual ring remains.
5. Tracking accuracy: following error and guiding residuals during a quiet track unchanged.

## 7. Open / follow-ups

- Retarget while tracking (`Track()` with a new RA/Dec while the axes track, `Repeated_Track_Retargets_...`): the target itself jumps
  and the axes stay in tracking. Not changed here; the same blend could take it (`SetValue(old setpoint - new target)`).
- Derotator: its track-start gap (#246, the axis 0.08 deg behind at track start) could use the same blend instead of the hold.
- BROTLib `FB_Axis` feeds the track velocity clamped to 0.009 deg/s and its direction from the sign of the position difference,
  not the velocity of the fed position; with the ramp the fed velocity is wrong by up to the ramp velocity [inferred, harmless in
  mode 11 where the drive differentiates the position]. Worth a look in BROTLib.
