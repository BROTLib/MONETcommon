# Brake cannot be opened from the Visu while a power-on error is latched (#155, #151, #121)

Date: 2026-10-27. Status: **decisions D1-D5 taken (2026-10-27, Tim: "go with your recommendations"); implemented in a PR, not run on a PLC or telescope.**
Repo: BROTLib/MONETcommon. Base: `develop` = `66e86b3` (0.9.10), HalfBROT 0.7.3. Line numbers below are from
`FB_MonetTelescopeControl` (FBMTC) at that commit. Written by an Opus planning agent from a read-only pass over the code;
the findings are not yet checked on a PLC.

## 1. Problem (from #155, MONET/S, remote, key on Remote, `eTelescopeMode` = 2)

2026-10-26 12:04, MONETS 1.4.5 / MONETcommon 0.9.10 / HalfBROT 0.7.3: `power=1` after a restart, the **elevation** faulted
within 11 s (NC 0x4650, PLC error 61703). Covers open, hydraulics on, brake clearing TRUE, the #151 window open, but
`HydraulicsControl.bBrakeOpen` FALSE: the Visu could not open the brake. After `reset=1` it could. Same on 2026-10-25.

## 2. Cause (confirmed from the code)

FBMTC 288-293:

    IF fbElevation.bError THEN
        fbElevation.Enable := FALSE; fbAzimuth.Enable := FALSE; fbDerotator.Enable := FALSE;
        fbBrake.CloseBrake();
    END_IF

runs in **every scan** while the elevation error is latched, before and independent of the window (661-672). The Visu
writes `MAIN.HydraulicsControl.bOpenBrake := TRUE` (MONETS `Hydraulics.TcVIS`, DipSwitch). HalfBROT `CloseBrake` sets
`bCloseBrake`, and the hydraulics body (called after FBMTC in `MAIN`) clears `bOpenBrake` in the same scan
(`FB_HydraulicsControl` 170-172). `reset=1` clears `fbElevation.bError` (FBMTC 223-227), the block stops, the window is
still set (a reset does not clear it, #151 3.1), and the next Visu click works: exactly the observed sequence.

With an azimuth-only error the block does not run and the #151 window already works; #151's W1 was never run with an
elevation error. `F_BrakeMustClose` (670) is not the blocker: the window was open, so it returned FALSE.

Nothing else closes the brake in this state: `_ParkTelescope` does not run (exempt, no park), HalfBROT
`FB_ElevationControl` closes only on the falling edge of the enable (51-55).

## 3. Agreed rule (with the user, 2026-10-27)

With the elevation drive disabled, the brake may be opened unless one of these holds: **E-stop, hydraulics not OK,
safety error, covers closed.** A latched axis error alone must not block it.

### 3.1 Signals in FBMTC

| Condition | Signal | Source | Note |
|---|---|---|---|
| E-stop | `bEstopTriggered` | BROTLib `FB_BaseTelescopeControl` VAR_INPUT, `SafetyHandling.estop` (MONETS `MAIN`:115, MONETN `MAIN`:113) | already used by #86/#151 |
| Hydraulics not OK | `fbHydraulics.Error` (`I_Hydraulics`) | HalfBROT `bError`, latched: pressure/suction/main-pump failure, oil low | `PumpRunning`/`OilPressureOK` also exist, see D3 |
| Safety error | **none** | `FB_MonetSafetyHandling.error`, `.state`, `.bSafetyReady` live only in `MAIN` | new VAR_INPUT `bSafetyError`, D2 |
| Covers closed | `fbCovers.Opened` (`I_MirrorCovers`) | | "not open" also blocks while moving |

## 4. Options (D1)

| Option | What | Pro | Con |
|---|---|---|---|
| **A** (recommended) Window-scoped | The drive-disabled exception stays the #151 window. The elevation-error block no longer closes in every scan; the four conditions close the brake inside the window as level blocks | fixes the issue case; #121 unchanged outside the window; small, testable | not "any state": after the window timeout, after `reset=1` with covers open, or after a non-first power-on failure the brake still closes |
| **B** Global | `F_BrakeMustClose := NOT bElevationEnabled AND (bEstop OR bHydError OR bSafetyError OR NOT bCoversOpen)` | "any state" as the issue asks | #121 protection shrinks to "covers closed": a brake opened by an unknown actor (#121 H1-H6) with the covers open and drives disabled stays open with no time limit. The #121 step 4 case had the covers closed, so B would still have caught that one |
| **C** Operator release | A new FBMTC method/input (`ReleaseBrake`) the Visu calls instead of writing `bOpenBrake`; opens a bounded "brake release" window (same latch and timeout as #151) if none of the four blocks hold | "any state", keeps #121 against unexplained opens, the window has an owner | Visu change in MONETS and MONETN, new latch, more tests; follow-up |

The rest of this plan is option A. C can be added later on the same `F_BrakeMustClose` inputs.

## 5. Design (option A)

### 5.1 Elevation error: close once, not every scan (D4)

Replace the level close at 292 with a close on the rising edge of the elevation error:

    rtrigElevationError(CLK := fbElevation.bError);
    IF fbElevation.bError THEN
        ... (enables FALSE as before)
        IF rtrigElevationError.Q THEN
            fbBrake.CloseBrake();	// #155: once at the fault; the operator may open it again in the #151 window
        END_IF
    END_IF

Why not just delete it: outside the window `F_BrakeMustClose` closes anyway (the elevation is disabled in this scan), but in
the scan that sets the window a brake opened by the power-on would only be closed by HalfBROT's falling-edge close.
`FB_ElevationMock` does not model that edge, so the rig would show the brake staying open. The edge close keeps "a new
elevation fault closes the brake" in MONETcommon and testable. The R_TRIG is not called in manual (FBMTC body not called);
on the switch back to automatic with the error present it fires once, which is safe.

### 5.2 `F_BrakeMustClose` takes the blocks

    F_BrakeMustClose := NOT bElevationEnabled AND
        (NOT bWakeShakeWindow OR bEstop OR bHydraulicsError OR bSafetyError OR NOT bCoversOpen);

The blocks are **level** conditions: they close the brake while they hold but do not end the window (D5). A safety error
that is acknowledged, or a hydraulics error reset from the Visu, does not cost the operator the Wake&Shake window.
E-stop and covers not open still end the window in `F_WakeShakeWindow` (unchanged), so for them the input is defence in
depth. An enabled elevation is not affected (E-stop disables the axes at 726 anyway, one scan later).

### 5.3 New input `bSafetyError`

`bSafetyError : BOOL := FALSE` in FBMTC VAR_INPUT. MONETS and MONETN `MAIN`: `bSafetyError := SafetyHandling.error`
(D2) in the `TelescopeControl(...)` call, separate PRs after the MONETcommon release, pinned like 0.9.10.
The default FALSE means the block is not enforced until MAIN wires it (the M5 story of `bEstopTriggered`); B-checks below
include it.

### 5.4 Logging

One `FB_EventLog` WARN on the scan in which a block closes an open brake inside the window:
'brake closed in the Wake&Shake window: <E-stop | hydraulics error | safety error | covers not open>'.
The window open/close logs of #151 stay.

### 5.5 What stays as it is

`F_WakeShakeWindow`, `F_WakeShakeErrorExempt`, `F_ErrorParkRequest`, the #86 emergency park (stage 0 closes the brake and
the park clears the window, so `F_BrakeMustClose` closes it every scan), #87 park in place, manual mode, HalfBROT.

## 6. Interaction with other issues

| Issue | Effect |
|---|---|
| #121 | Unchanged outside the window. Inside the window the exposure is the same as for an azimuth fault today (all axes are disabled on any `bError`, 281-283); only the elevation-error special case goes |
| #151 | Completes it for an elevation fault; W1 of #151 now passes without `reset=1` |
| #113 | Unchanged (the exemption sets the window as before) |
| #86 | Unchanged: an error after a completed power-on parks, the park clears the window, the brake is closed every scan; covers closed |
| #76 | The automatic Wake&Shake stage can open the brake in the window even with an elevation fault |

## 7. Files

| File | Change |
|---|---|
| `FB_MonetTelescopeControl.TcPOU` | `rtrigElevationError`, edge close at 292; `bSafetyError` VAR_INPUT; new inputs at the `F_BrakeMustClose` call (670); one `FB_EventLog` |
| `F_BrakeMustClose.TcPOU` | inputs `bEstop`, `bHydraulicsError`, `bSafetyError`, `bCoversOpen`; condition and comment |
| `MONETcommonTests`: `F_BrakeMustClose_Tests`, new `MonetBrakeRelease_Tests`, `FB_MonetRig` (`SetSafetyError`, pass `bSafetyError`), `MAIN` registration, test README table | see 8 |
| `README.md` (77-79) | the window also allows opening with an elevation fault; the four blocks |
| `specs/plans/index.md` | index line |
| MONETS, MONETN `MAIN` | `bSafetyError := SafetyHandling.error` (separate PRs) |

Not touched: HalfBROT, `F_WakeShakeWindow`, the mock POUs with uncommitted changes in the working tree.

## 8. Tests (TcUnit, `FB_MonetRig`, existing style: one rig per method, `nStep`/`nCycles`, `F_Note`)

`F_BrakeMustClose_Tests` (existing tests get the new inputs explicitly, `bCoversOpen := TRUE`, otherwise
`Disabled_Window_Keeps` would flip to TRUE):
- `Window_NoBlock_Keeps`; `Window_Estop_Closes`; `Window_HydraulicsError_Closes`; `Window_SafetyError_Closes`;
  `Window_CoversNotOpen_Closes`; `Enabled_Block_Keeps` (elevation enabled + any block -> FALSE);
  `Disabled_NoWindow_Closes` unchanged.

`MonetBrakeRelease_Tests` (new):
- `First_PowerOn_Elevation_Error_Brake_Can_Be_Opened` (the #155 case): `Init()`, wait `rig.fbCovers.bOpened`,
  `InjectElevationError`, 60 scans; window TRUE, no park; `rig.fbBrakeHyd.OpenBrake()` (operator), 20 scans:
  `BrakeOpen` TRUE, `bError` TRUE, `rig.fbElev.nResetEdges = 0` (no reset).
- `Elevation_Error_Closes_The_Brake_Once`: brake open before the error -> closed in the error scan;
  `nCloseCalls` does not grow on later scans while the window holds.
- `Window_Brake_Closes_On_Estop` / `_On_Safety_Error` / `_On_Hydraulics_Error` (`fbBrakeHyd.bFailed := TRUE`):
  opened in the window, block set -> closed; safety/hydraulics block cleared -> window still TRUE, open works again.
- `Error_After_Completed_PowerOn_Still_Closes` (#86 regression): online, `InjectElevationError` -> park, operator open ->
  closed again within one scan.
- `Window_Timeout_Then_Open_Is_Closed` (`SetTimeDivisor`): pins option A (no window -> closed).

Existing `MonetErrorPark_Tests`, `F_WakeShakeWindow_Tests` stay green unchanged.

## 9. Hardware check (MONET/S, roof closed, someone at the E-stop, telescope near balance at el 40)

- **B1, #155.** PLC cold restart, `power=1`, wait for the elevation fault and 'no emergency park'. **No reset.** Open the
  brake in the Visu: `bBrakeOpen` TRUE for > 1 min, `bError` TRUE, `diagnostics/WakeShakeWindow` TRUE.
- **B2, safety/hydraulics blocks.** In B1 with the brake open: trigger a safety error (if feasible) / stop the pump:
  brake closes, log line with the reason, window stays; after ack the Visu opens it again.
- **B3, E-stop / covers.** E-stop or `covers` close: brake closes, window ends (#151 W-checks).
- **B4, #86.** Online, disable the elevation drive in XAE: emergency park, brake closed, Visu open is undone at once.
- **B5, timeout and hydraulics restart.** Leave the window to time out: brake closes, Visu open is undone (option A). Wait 300 s with the brake closed (HalfBROT switches the hydraulics off), restart the hydraulics from the Visu with the elevation error still latched, wait for the pressure, open the brake again.
- **B6, wiring.** Read `bSafetyError` in ADS: follows `SafetyHandling.error`. Read `Global_Version` first.
- Record at 10-20 Hz: `fbElevation.bError`, `bWakeShakeWindow`, `HydraulicsControl.bOpenBrake`, `bBrakeOpen`, `bCloseBrake`,
  `bEstopTriggered`, `bSafetyError`, `HydraulicsControl.bError`, `PumpRunning`, `CoverControl.Opened`.

## 10. Risks

1. An open brake with all drives disabled and a latched elevation fault can let an unbalanced telescope swing. Same exposure
   as the #151 window with an azimuth fault; bounded by the 15 min window and the four blocks.
2. `bSafetyError` defaults to FALSE: the safety block is not enforced until MONETS/MONETN `MAIN` wire it.
3. HalfBROT switches the hydraulics off 300 s after the brake closed (`tonHydraulicsOffTimer`). After a failed power-on the
   operator has about 5 min before the pump stops and the Visu click does nothing until the hydraulics are restarted.
4. If "hydraulics not OK" includes `PumpRunning`/`OilPressureOK` in FBMTC, a Visu click during the 3 s pressure build-up
   would be wiped by `CloseBrake` (D3).
5. A new elevation fault during the Wake&Shake closes the brake once (5.1); this aborts the Wake&Shake. Intended.

## 11. Not verified

- Nothing compiled or run.
- That `fbElevation.Enable := FALSE` updates `fbElevation.bEnable` in the same scan (assumed as in #121/#151).
- The Visu writes `bOpenBrake` in automatic only through the DipSwitch; other writers (pendant in automatic) not checked.
- The MONETN Visu brake element (only grep hit, not read).
- Whether the drive error can be reset in XAE without `reset=1` (would clear `fbElevation.bError` too).

## 12. Decisions (2026-10-27, Tim: go with the recommendations)

1. **D1:** option A (window-scoped). C (operator release) stays a follow-up on the same `F_BrakeMustClose` inputs if A is too restrictive on site.
2. **D2:** safety error = `SafetyHandling.error` (not `NOT bSafetyReady`, which would block for ~30 s after every restart).
3. **D3:** hydraulics not OK = `fbHydraulics.Error` only (HalfBROT handles pump and pressure; a check here would wipe a Visu click during the 3 s pressure build-up).
4. **D4:** close once on the rising edge of the elevation error (the mock does not model HalfBROT's close on the enable edge).
5. **D5:** a block closes the brake while it holds and does not end the window (E-stop and covers not open end it anyway in `F_WakeShakeWindow`).
6. The 15 min window stays. The 300 s hydraulics shut-off of HalfBROT is not changed; B5 below checks that the hydraulics can be restarted from the Visu while the elevation error is latched and the brake opened again afterwards.

## 13. Implementation notes

- Block log: `WakeShakeBlockEvent` logs once (rising edge) when a block closes an open brake in the window, with the reason.
- Rig: `FB_MonetRig.SetSafetyError`, new suite `MonetBrakeRelease_Tests`; the window timeout test uses `SetTimeDivisor`.
- Not run locally (no TwinCAT in this session); CI on the PR is the first run. The `bWakeShakeWindow` assertions in the rig suite are unverified guesses until then.
- Version bump follows in the release commit, as for 0.9.x.
- **Hardware checks** are in the MONET test plan (IAG repo, R3), not here.
