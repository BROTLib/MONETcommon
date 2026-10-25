# Wake&Shake after a PLC restart: keep the brake open in a bounded window (#151, groundwork for #76)

Date: 2026-10-25. Status: **implemented in a PR (window, brake rule, tests, README). Not compiled in XAE, not run on a PLC or telescope. Version bump left to the release.**
Repos: BROTLib/MONETcommon (main). Base: `develop` = `2bbda83` (0.9.9), HalfBROT 0.7.2.
Line numbers below are from `FB_MonetTelescopeControl` (FBMTC) at that commit.

Files read: `F_BrakeMustClose`, `F_WakeShakeErrorExempt`, `FB_MonetTelescopeControl` (error park 440-460, brake close
643-648, `_PowerOn` 1060-1167), plans #121 and #113.
Issues read: #151, #76 (with the 2026-10-19 comment), #113.

## 1. Problem

Since 0.9.4 FBMTC calls `fbBrake.CloseBrake()` in every scan in which `fbElevation.bEnable` is FALSE (`F_BrakeMustClose`, #121,
automatic only). The Wake&Shake of the AX5125 drives (`P-0-0166`, then `P-0-0160`, #76) must run with the covers open and the
brake released, but with the axes **not** enabled by the PLC. The drive runs the command itself. After a PLC restart the first
power-on faults on azimuth/elevation (expected, #113), and the operator then cannot hold the brake open from the Visu: the
`bOpenBrake` latch is closed again within one scan. Only manual mode (key Local) works, which needs a person at the cabinet.

## 2. Wider picture

Three things already exist and one is missing:

| Piece | State |
|---|---|
| `F_WakeShakeErrorExempt` (#113) | No emergency park for the first-power-on Az/El error, so the covers stay open. |
| `F_BrakeMustClose` (#121) | Closes the brake whenever the elevation is disabled. Knows nothing about Wake&Shake. |
| `_PowerOn` stage 0 -> 10 | Opens the covers, then enables the axes. No Wake&Shake stage, does not touch the brake. |
| #76 | Wanted: a PLC stage that runs both commands. Blocked on the drive password (S-0-0267, write protection of the IDNs). |

The interlock "the brake may be open with the elevation disabled" is the same one in all three. So the fix names it **once**, as
a *Wake&Shake window*, and both consumers use it:

1. **Now (#151):** the operator opens the brake in the Visu and starts the commands in XAE. The window only keeps the PLC from
   closing the brake again.
2. **Later (#76):** a new `_PowerOn` stage between 0 and 10 sets the same window itself, opens the brake, runs the SoE commands
   and closes the window. Nothing in the interlock changes, only who opens the brake.

## 3. Design

### 3.1 The window is a latch, not the exemption condition

`F_WakeShakeErrorExempt` is evaluated only at the one-scan edge of a new hardware error (`F_ErrorParkRequest`, FBMTC 451-456).
By then the power-on has already aborted (`bPower` FALSE, `nStage` 0, lines 1061-1073), so `bPowering` is no longer true in the
following scans. Using it directly in `F_BrakeMustClose` would give the window for exactly one scan.

Therefore FBMTC keeps `bWakeShakeWindow : BOOL` (not persistent, FALSE after a PLC restart):

- **Set** in the scan in which `bErrorParkExempt` is TRUE (the #113 condition: first power-on only, Az/El error only, no
  focus/derotator error). `tonWakeShakeWindow` is started.
- **Cleared**, whichever comes first:
  - `tonWakeShakeWindow` elapsed (see 3.3);
  - a park starts (`bPark`, error park, watchdog park), or `bIsParked` becomes TRUE;
  - the covers are not open any more (a Wake&Shake needs them open anyway);
  - a new power-on starts (`bPower` rising). Stage 10 then enables the elevation, so `F_BrakeMustClose` is FALSE by itself, and
    a repeated failure sets the window again through the exemption;
  - an E-stop;
  - a focus or derotator error (the exemption would not have applied to it).
- **Not cleared** by `reset`: the drive errors have to be reset before `P-0-0166`/`P-0-0160` (#76 step 2).

### 3.2 `F_BrakeMustClose` takes the window

New input `bWakeShakeWindow`. `F_BrakeMustClose := NOT bElevationEnabled AND NOT bWakeShakeWindow`.

While the window is set the brake is neither commanded closed nor opened by the PLC. The operator's latch from the Visu works
again, and the brake still closes by itself when the window ends (3.1), because the call then returns TRUE again and
`CloseBrake()` clears the latch (#121 C2).

### 3.3 Timeout

Bounded by 15 min instead of the 12 h idle park: an open brake on an unbalanced telescope swings (40 to 66 deg in #121), and a
Wake&Shake needs a few minutes. The value is a constant in FBMTC, open question 1.

When the window ends by timeout the brake closes and a WARN is logged ('Wake&Shake window ended: brake closed'). The covers stay
as they are (#113). The operator repeats `power=1` (which fails again at the first scan of the window and sets it again) or sends
`park=1`.

### 3.4 Logging

`FB_EventLog` messages, one per transition, so the sequence can be reconstructed from the log:
- 'Wake&Shake window open (brake may stay open, N min)';
- 'Wake&Shake window closed: <timeout | park | covers | power-on | estop | other error>'.

### 3.5 What stays as it is

- `F_WakeShakeErrorExempt` and `F_ErrorParkRequest` keep their signatures and behaviour (#113 tests unchanged).
- Manual mode: unchanged (`SuspendAutomatic` does not call this code).
- The elevation `bEnable` is still the only enable of the safety rule. In the window the PLC does **not** enable the axes.
- `bFirstPowerOnPending` stays the "commutation not determined" proxy. For #76 it should become the drive state (3.6).

### 3.6 How #76 builds on it (not part of this change)

- New `_PowerOn` stage (for example 5) after the covers are open: `bWakeShakeWindow := TRUE`, `fbBrake.OpenBrake()`, wait for the
  brake, drives disabled and reset, `S-0-0267` unlock, `P-0-0166` and success, `P-0-0160` and success, then window closed and
  stage 10. Timeouts and error path per #76.
- Skip the stage when the commutation is already determined, read from the drive instead of `bFirstPowerOnPending`
  (command state, `P-0-0167`). That also covers a drive swapped without a PLC restart.
- The window latch, its timeout and its end conditions are reused. The operator path of this issue stays as the fallback until
  #76 is proven.
- Open for #76: the password source (a persistent variable entered once in XAE, no secret in git) and re-locking.

## 4. Files

| File | Change |
|---|---|
| `F_BrakeMustClose.TcPOU` | New input, new condition, comment (the exception and why). |
| `F_WakeShakeWindow.TcPOU` (new, pure) | `F_WakeShakeWindow(bWindow, bSet, bTimeout, bParkActive, bCoversOpen, bPowerOnStart, bEstop, bOtherHwError) : BOOL`, the latch decision, so the clear conditions are testable offline (FBMTC itself cannot run in a test, #36). |
| `FB_MonetTelescopeControl.TcPOU` | `bWakeShakeWindow`, `tonWakeShakeWindow`, two `FB_EventLog`, the call after the error-park block (456) and before the brake close (646), the new input there. |
| `MONETcommon.plcproj` | Add the new POU. Version bump to 0.9.10 per the release routine. |
| `MONETcommonTests` | `F_WakeShakeWindow_Tests`, additions to `F_BrakeMustClose_Tests`, suite registration in `MAIN`, README table. |
| `README.md`, `specs/plans/index.md` | Behaviour note and the index line. |

Not touched: HalfBROT, MONETS/MONETN `MAIN`. The mock POUs with uncommitted changes in the working tree are unrelated and stay
untouched.

Order in the scan: the window decision must run after the `bErrorParkExempt` line (453) and before the `F_BrakeMustClose` call
(646). The brake close stays after the dispatch (#121 risk 2).

## 5. Tests (TcUnit, CI simulation)

`F_BrakeMustClose_Tests` (extend):
- `Disabled_NoWindow_Closes`: elevation disabled, no window -> TRUE (existing).
- `Disabled_Window_Keeps`: elevation disabled, window -> FALSE.
- `Enabled_Window_Keeps`: elevation enabled, window -> FALSE.

`F_WakeShakeWindow_Tests` (new):
- `Sets_On_Exempt_Error`: not set, `bSet` -> TRUE.
- `Stays_Set_Without_Condition`: set, `bSet` FALSE, nothing else -> TRUE (the one-scan edge problem of 3.1).
- One test per clear condition: timeout, park active, covers not open, power-on start, E-stop, other hardware error -> FALSE.
- `Clear_Wins_Over_Set`: both in the same scan -> FALSE.
- `Not_Set_Stays_Not_Set`.

Not testable offline: the FBMTC wiring and the timer. Both are covered on site (6).

## 6. On-site check, MONET/S

The roof stays closed and someone at the telescope has the E-stop at hand. The telescope should be near balance (el 40).

1. **W1, the #151 sequence.** PLC cold restart, `power=1`, wait for the expected Az/El error and 'no emergency park' (#113).
   `reset`, open the brake in the Visu. Pass: `bBrakeOpen` stays TRUE for more than 1 min while the elevation `bEnable` is FALSE;
   log shows 'Wake&Shake window open'; `diagnostics/WakeShakeWindow` goes TRUE with it and stays TRUE while the brake is open.
   Run the Wake&Shake from XAE, then `power=1`: `diagnostics/WakeShakeWindow` goes FALSE, READY_STATE reaches 1.0.
2. **W2, timeout.** Same start, leave the brake open. Pass: the brake closes after the timeout, log shows the window closed with
   the reason, the covers stay open. `diagnostics/WakeShakeWindow` goes FALSE in the same moment as the brake closes.
3. **W3, park ends the window.** In the window with the brake open, `park=1`. Pass: the brake closes at once, covers close,
   `diagnostics/WakeShakeWindow` FALSE.
4. **W4, no window outside the case.**
   - After a completed power-on, a later Az/El error: emergency park as before (#86), brake closed.
   - An ADS write of `HydraulicsControl.bOpenBrake := TRUE` while parked and no window: the brake closes again (#121 B3).
   - A focus or derotator error in the first power-on: emergency park, no window.
   - In all three, `diagnostics/WakeShakeWindow` stays FALSE.
5. **W5, regression.** Power-on from parked, park from ready, manual mode with the clear-brakes key: unchanged.

Close #151 when W1-W4 pass. Report the result in the issue and in this plan.

## 7. Risks

1. **The window is the only path that keeps an open brake with the elevation disabled.** An unbalanced telescope can swing. The
   bounds are as narrow as #113's (first power-on, Az/El error only), plus the timeout and the clears above. The sequence still
   depends on the operator running the Wake&Shake before the timeout.
2. **A latch that is never cleared would defeat #121.** Every clear condition has a test (5), and the 15 min timeout is the
   backstop.
3. **The set condition is the #113 exemption.** If the drive does not fault in the first power-on (commutation already
   determined) there is no window, and no need for one.
4. **Operator surprise.** A brake that closes by itself after the timeout in the middle of a Wake&Shake would be seen as a fault.
   The warning in the log and the documented timeout are the mitigation. A longer timeout is a trade-off against 1.

## 8. Not verified

- Nothing has been compiled or run.
- That `bPower` is FALSE and the stage is 0 after the Wake&Shake fault, as read from lines 1061-1073. The scan order and the
  state of `fbStartup.Initializing` after `MarkAborted()` are read from the code only.
- That 15 min is enough for the operator to run both commands from XAE (the commands' duration is open in #76).
- That the hydraulics latch is not also cleared elsewhere in the window (pump stop, oil pressure loss still clear it, as before).

## 9. Open questions for the user

1. Is 15 min the right window? Fixed constant, or a configurable value in `telescopeConfig`?
2. At the end of the window by timeout: only close the brake (recommended), or also run the emergency park that #113 skipped?
3. Should `reset` extend the window (restart the timer), so a long XAE session is not cut off, or is the timer fixed from the error?
4. Decided 2026-10-25: yes, `diagnostics/WakeShakeWindow`, in the PR. Questions 1-3 decided as proposed (fixed 15 min, timeout only closes the brake, reset does not extend).
