# bIsParked drops with the cover command of a power-on (#113)

Date: 2026-10-20. Status: implemented in code (PR on `fix/unpark-on-poweron-113`), not compiled in XAE, not tested on hardware: #113 stays open until the checks in section 5 pass.
Repo: BROTLib/MONETcommon. Base: `origin/develop` after PR #112 (#86).

Main file: `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU`.

## 1. Problem
`bIsParked` is TRUE after every PLC restart and goes FALSE only when a power-on completes (`_PowerOn` stage 100). `_PowerOn` stage 0 opens
the covers, so a park during a power-on from parked takes the `_ParkTelescope` shortcut (`IF bIsParked THEN nStage := 100`), only clears
`bPark` and leaves the covers open. Same for the MQTT watchdog and cabinet over-temperature park, and #86 never fires.

## 2. Fix
- `bIsParked := FALSE` in `_PowerOn` stage 0, just before `fbCovers.Open()` (the command, not the cover feedback or the end of stage 0: the covers
  are already fully open when stage 0 ends). A power-on that fails before the CASE (brake clearing, `bError`, stop/reset/interrupt returns) has
  opened nothing and stays parked. IAG50cm does the same at its power-on stage 0.
- A park after that is a normal park (park in place if the axes cannot move): brake closed, covers closed, hydraulics off, `bIsParked` TRUE.
- Wake&Shake: the first power-on after a PLC restart usually faults on azimuth/elevation, and the Wake&Shake (P-0-0166/P-0-0160) is then run
  by hand with the covers open. With the flag dropping at the cover command, #86 would close the covers under it. `F_WakeShakeErrorExempt`
  skips the #86 park, with one WARN, when `bFirstPowerOnPending` (TRUE from the PLC start, cleared at power-on stage 100) AND a power-on is
  running (`bPower OR fbStartup.Initializing`) AND the error is azimuth/elevation only. Manual, MQTT, watchdog and cabinet parks still close the
  covers. Bounded by the 12 h idle park (`tonPowerTimeout`).

## 3. READY_STATE
Start-up 0.7/0.8 unchanged. A stop (or manual key) during a power-on from parked now ends at -2.0 instead of 0.0 (the covers keep opening; `Open()`
latches), which is the truthful value; it changes the #80 result verified on 0.8.4. A park during start-up: 0.3, then 0.0 once the covers are closed.
Moves after an aborted start are rejected as "not ready" instead of "parked". Check how pyobs handles -2.

## 4. Tests
`F_WakeShakeErrorExempt_Tests` (TcUnit, pure function). The `bIsParked` flip is inside `FB_MonetTelescopeControl` and is checked on site.

## 5. On-site check (MONET/S, someone watching the covers)
Record at 10-20 Hz: `bIsParked`, `bPower`, power-on and park stages, covers, brake, pump, `bError`, `READY_STATE`, `bFirstPowerOnPending`.
1. Parked, `power=1`, `park=1` while the covers move: `bIsParked` FALSE at the command, 'power-on aborted: park requested', covers reverse and close,
   brake closed, pump off, `bIsParked` TRUE, READY_STATE 0.7 -> 0.3 -> 0.
2. Same, park in stage 10 after the brake opens and before the sweep; never park during a sweep.
3. `power=1`, `stop=1` during the cover move: READY_STATE -2, covers stay open, `park=1` closes them.
4. MQTT watchdog: broker off during a power-on from parked: covers close.
5. First power-on after a real TwinCAT restart: Wake&Shake fault, exemption WARN, no park, covers open; reset, Wake&Shake, `power=1` -> 1.0.
   Then a provoked axis error (#86 step 1) parks.
6. Regression: a normal park from online; a power-on with `bBrakeClearing` FALSE stays parked.

## 6. Risks
- The covers can now close while the drives are in the homing sweep after a park during a power-on; the axes are stopped first (#84 path).
- After an unattended restart plus a power-on from pyobs that faults, the covers stay open for up to 12 h.
- A Reset Cold counts as a restart (harmless).
