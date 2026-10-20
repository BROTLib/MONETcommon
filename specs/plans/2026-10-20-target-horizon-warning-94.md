# Target-below-horizon warning only for a requested goto/track (#94)

Date: 2026-10-20. Status: implemented in code (function F_TargetBelowHorizonWarning + TcUnit suite), not yet compiled or tested on hardware.
Repo: BROTLib/MONETcommon. Base: `origin/develop` @ `94332b3`.

Main file: `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU`.

## 1. Problem
`Target below safe horizon` is logged on a parked, idle telescope (MONET/S, 2026-10-20 04:40:53).
`TCStargetHorizonEvent` triggers on `fElevationCalc < fHorizon` with no check of the telescope state.
`fElevationCalc` follows the last RA/Dec target, so a stale target drops below the horizon by itself as the
sky turns. `FB_EventLog` logs `OnMessage` on every rising edge (and nothing on the falling edge, because
`OffMessage` is empty), so each crossing produces one false warning.

History: cde38bb (#21, code review) removed the gate `bPower AND ...` on purpose, because `bPower` only
covers the power-on phase. The new gate must be about a running command, not about power.

## 2. Fix
- Gate on `(bGoto OR bTrack)`. Leave `bSlew` out: a slew is alt/az (`_SlewTelescope` stage 25 uses
  `fElevation`/`fAzimuth`), so a stale RA/Dec must not warn during it.
- `bPower AND NOT bIsParked` is not used: it still fires for an idle, unparked telescope with a stale target.
- Ordering: do not gate in place at the event (~line 612). `_GotoTelescope` clears `bGoto` (error 0x15, target
  below 2 deg) earlier in the same scan, and `_TrackTelescope` ends below 5 deg, so the gate would almost never
  be TRUE there. Evaluate it once, right after `fElevationCalc` is final (~line 224), before arbitration
  and dispatch, and keep it in a member.

Edit:
1. Declaration: `bTargetBelowHorizonWarn : BOOL; // #94: goto/track requested with the target below fHorizon, taken before the dispatch can clear bGoto`
2. After `fAzimuthCalc := fAzimuthCalc + fAzimuthPointingOffset + F_AzimuthOffsetOnAxis(...)` (~line 226):
   `bTargetBelowHorizonWarn := (bGoto OR bTrack) AND fElevationCalc < fHorizon;`
   with a comment that a stale RA/Dec target on a parked/idle telescope drifts below the horizon by itself.
3. Event (~line 612): `Trigger := bTargetBelowHorizonWarn`, other arguments unchanged.
4. Leave LineIds alone (XAE regenerates them on save).

Optional (decision 4): extract `F_TargetBelowHorizonWarning(bGoto, bTrack, fElevationCalc, fHorizon) : BOOL`
like `F_HomeBelowHorizon` / `F_CommandAcceptance`, for a TcUnit test.

## 3. Out of scope
- #77: already fixed by #91 (0.8.5, on develop; `_GotoTelescope` returns before the 0x15 check when
  `bReset OR bError OR bInterrupted OR NOT bGoto`). Still open only because #91 merged into develop. Close it
  after the on-site reset check, with a note in the #94 PR.
- MONETS / MONETN: no vendored copy (they reference the installed library; the event only appears in the
  generated `.tmc`). Reinstall the library and rebuild.
- IAG50cm `FB_TelescopeControl` has the same event: separate issue.

## 4. Files
- `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU`
- `MONETcommon.plcproj` (`ProjectVersion`) and `Global_Version.TcGVL` (`stLibVersion_MONETcommon`), see decision 1
- This plan and its entry in `specs/plans/index.md`
- If the function is extracted: `F_TargetBelowHorizonWarning_Tests.TcPOU`, tests `MAIN.TcPOU`,
  `MONETcommonTests.plcproj`, coverage table in `MONETcommonTests/README.md`, plus a `<Compile>` entry in `MONETcommon.plcproj`

## 5. Tests
`FB_MonetTelescopeControl` cannot be unit tested (needs `REFERENCE TO` HalfBROT axis FBs). With the function
extracted: idle below horizon -> FALSE; goto below -> TRUE; goto above -> FALSE; track below -> TRUE;
exactly at `fHorizon` -> FALSE (strict `<`).

## 6. Verification
1. XAE: build MONETcommon (no errors, no new warnings), save as library, install, rebuild MONETS.
2. MONET/S:
   - Parked with a stale RA/Dec below the horizon: no warning in the XAE or MQTT log.
   - Goto to a target below the horizon: one WARN "Target below safe horizon", plus the 0x15 error. Reset
     clears it (also verifies #77).
   - Normal goto/track above the horizon: no warning.
3. Overnight error log on a parked night: no entry like 04:40:53.

## 7. Risks
- Intended behaviour change: no warning for a stale below-horizon target without a command.
  `OBJECT.HORIZONTAL.ALT` still publishes `fElevationCalc`.
- Open PR #95 (0.8.6) touches the same file in other regions: whichever merges second rebases.
- Warning (`fHorizon` 1.0) and 0x15 (2.0) thresholds differ: targets between 1 and 2 deg only get 0x15. Not new.
- On develop `iBuild := 4` does not match `sVersion := '0.8.5'`; #95 sets 6 / 0.8.6.

## 8. Decisions (2026-10-20)
1. Version: no own bump; the fix goes into develop and ships with the next release.
2. `bTrack` stays in the gate: a track request to a target below the horizon warns once at its rising edge.
3. IAG50cm `FB_TelescopeControl`: separate issue (its event uses `rtrigHorizonError`, different logic).
4. The condition is a testable function, `F_TargetBelowHorizonWarning`, with `F_TargetBelowHorizonWarning_Tests`.

Branch: `fix/target-horizon-warning-94` from `origin/develop`.
