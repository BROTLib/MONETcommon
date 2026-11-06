# Manual-mode READY_STATE heartbeat: -1.0 on E-stop / safety error (small alternative to #168)

Date: 2026-11-06. Status: **implemented in a PR on `develop` (decisions D1-D5 as recommended: D1 include, D2 not now, D3 replace, D4 `*In` names, D5 alone). Not compiled, TcUnit not run, not run on a PLC or telescope; the MONETS/MONETN `MAIN` pin PRs and hardware H1-H3 are open.**
Repo: BROTLib/MONETcommon. Base: `main` = `b725c40` (0.16.0). `origin/develop` = `2f36862` (0.16.1, release PR #216 open;
it adds `FB_JogRequest_Tests` and pins HalfBROT 0.12.0, `FB_MonetTelescopeControl` unchanged). MONETS `develop` = `e7247a4`,
MONETN `develop` = `b2f9d29` (both pin MONETcommon 0.16.0, BROTLib 0.18.0). BROTLib read at `v0.18.0`.
Line numbers below are from these commits; the ones in #168 are stale.
Main file: `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU` (FBMTC).
Background: #168 (full mode state machine, deprioritised, status comment of 2026-11-06), #72 plan step 8 / finding F8
(`2026-10-27-manual-to-auto-readiness-policy-72.md`), #156 (`__ISVALIDREF` guard), #177 (pendant `eTelescopeMode`).

Legend: **[read]** seen in the code at the commits above, **[inferred]** follows from the code but not observed,
**[unknown]** not answerable from the repos.

## 1. Goal and non-goals

Goal: while the cabinet key is on Local (`manual` or `off`), the READY_STATE heartbeat shows -1.0 when the E-stop is pressed or
the safety group is in error (and on a power failure), instead of the -2.0 / 0.0 of the last automatic scan.

Non-goals: no axis, brake, hydraulics or cover write in manual from the new inputs; no reset forwarding (section 5); no change
to the automatic body; no pendant gating (#177); not the #168 redesign; no firmware deployment, no telescope movement.

## 2. Findings (read from the code)

- [read] `SuspendAutomatic` (FBMTC 1511-1643) computes the heartbeat at 1635:
  `fReadyState := fbStartup.Suspend(Error := bError OR bSoftwareError OR bPowerFailure OR bEstopTriggered, Parked := bIsParked);`
  then publishes it every 1 s and on change (1637-1641). `bSoftwareError` is redundant (already in `bError`, 446).
- [read] **All four terms are stale in manual.** `bError` is assigned only in the body (446, from `bHwError` 440-445: El/Az/Der/focus
  axis error, hydraulics error, `NOT bMainReady`). `bEstopTriggered` and `bMainReady` are BROTLib `FB_BaseTelescopeControl`
  inputs (v0.18.0 l. 43-44, `bMainReady` default TRUE), `bSafetyError` an FBMTC input (28); MAIN passes all three only in the
  automatic call (MONETS `MAIN` 174-177, MONETN 181-183). `SuspendAutomatic()` is called with no arguments (MONETS 181, MONETN 187).
  So an E-stop pressed in manual is not in the heartbeat, and one released in manual stays -1.0 until the return. Confirms #72 F8.
- [read] `bPowerFailure` is the power-on failure latch (`_PowerOn` 1371-1390). In manual it changes only in the first scan
  (`_PowerOn` with `bInterrupted` clears it if a power-on was running, 1533-1537). Keeping it is right: it is a real
  "Init failed" state that is still true in manual.
- [read] BROTLib `FB_StartupControl.Suspend` (v0.18.0) runs `FB_StartupReadiness` with `Aborted := TRUE`; `Error` then only selects
  the value: -1.0 wins over Parked 0.0 and over -2.0 (`FB_StartupReadiness` 52-68). **In manual, adding a term to `Error` has no
  side effect beyond the published value** (startup is already aborted every manual scan). [inferred]
- [read] **Asymmetry with automatic.** The automatic heartbeat (`fbStartup.Update`, 965-970) uses
  `Error := bError OR bSoftwareError OR bPowerFailure OR bEstopTriggered`: E-stop and power failure (via `bError`) give -1.0, a
  **safety error alone does not** (it disables the axes, 994, so `bReady` and `Operational` drop: -2.0). There `Error` is not
  value-only: it clears `Initialized` (`FB_StartupReadiness` 27-31). Adding the safety error to the manual predicate only makes
  manual stricter than automatic and contradicts README 327 ("the same error predicate in automatic and suspended/manual mode").
  Decision D1.
- [read] `SafetyHandling(...)` runs after the telescope call in both `MAIN`s (MONETS 303, MONETN 296), so `estop`/`error` are
  one scan old in both modes. Same as automatic today, no change. `SafetyHandling.estop` = `sStateEStop1 <> 'RUN'`,
  `.error` = `inError` (`FB_MonetSafetyHandling` 71-105).
- [read] Callers of `SuspendAutomatic`: MONETS `MAIN`, MONETN `MAIN`, `FB_MonetRig.Scan` (103). Nothing else in the BROT repos.
- [read] Method inputs with defaults exist in the libraries this builds on (BROTLib `FB_Comm_MQTT.PublishLREAL` etc.,
  `domain : STRING(255) := 'environment'`). FBMTC's own methods have none. Whether an existing `SuspendAutomatic()` call with
  empty parentheses still compiles after inputs are added is **[unknown]** (expected yes for a formal call with defaults in
  TwinCAT 3.1.4024; all three callers are changed anyway, so it only matters for an out-of-tree caller).

## 3. Proposed change

### 3.1 FBMTC `SuspendAutomatic`

New optional inputs. **Not** named `bEstopTriggered` / `bSafetyError` / `bMainReady` as first proposed: those are already FB
inputs (BROTLib base 43-44, FBMTC 28) and a method input of the same name shadows them inside the method (compiler
behaviour [unknown]: warning or error; `THIS^.` would be needed to reach the member). Proposed names, as in the rig:

```
METHOD SuspendAutomatic
VAR_INPUT
	// #168 small alternative: the live safety state for the manual READY_STATE heartbeat only. Never used for an axis,
	// brake, hydraulics or cover write. Defaults = no E-stop, no safety error, power present (heartbeat as before).
	bEstopIn		: BOOL := FALSE;	// MAIN: SafetyHandling.estop
	bSafetyErrorIn	: BOOL := FALSE;	// MAIN: SafetyHandling.error
	bMainReadyIn	: BOOL := TRUE;		// MAIN: PowerMonitoring.phaseOK (MONETS ready, MONETN mainReady)
END_VAR
```

Line 1635 becomes (only use of the inputs):

```
fReadyState := fbStartup.Suspend(
	Error := bError OR bPowerFailure OR bEstopIn OR bSafetyErrorIn OR NOT bMainReadyIn,
	Parked := bIsParked);
```

- `bError` stays (stale, value of the last automatic scan). A live hardware term (axis/focus/hydraulics `Error` read inside the
  `__ISVALIDREF` guard, local variable only) would also show an axis error that appears in manual, but adds reads of the
  references and interfaces in manual; not in this change (D2).
- The stale member `bEstopTriggered` is replaced by `bEstopIn` (D3): with it, an E-stop released in manual drops to -2.0/0.0 at
  once. Every caller passes the input after this change.
- `bSoftwareError` dropped as redundant (in `bError`). `bMainReadyIn` also covers a power failure that starts in manual (in
  automatic it is already -1.0 via `bHwError`).
- Header comment of the method (1512-1518): one line on the new inputs and that they are telemetry only.
- Not touched: the `__ISVALIDREF` block (1523-1616), `F_SuspendStop`, `_StopTelescope`, the `EnterSuspend` branch, events.
  [inferred] Code review must confirm with a grep that the three inputs appear only in the `Suspend(...)` call.

### 3.2 MONETS and MONETN `MAIN` (ELSE branch)

```
ELSE
    // Invalidate startup readiness and publish its heartbeat without running automatic motion in manual mode.
    // MONETcommon#168: E-stop, safety error and power failure reach the manual READY_STATE (telemetry only).
    TelescopeControl.SuspendAutomatic(
        bEstopIn		:= SafetyHandling.estop,
        bSafetyErrorIn	:= SafetyHandling.error,
        bMainReadyIn	:= ready);		// MONETN: mainReady (without the L3 warning, as in its automatic call)
END_IF
```

MONETS `MAIN` 179-182, MONETN `MAIN` 185-188. Same signals as the automatic call of each site.

## 4. What the heartbeat then shows in manual

| State in manual | Today | After |
|---|---|---|
| at rest, not parked / parked | -2.0 / 0.0 | -2.0 / 0.0 |
| E-stop pressed in manual | -2.0 / 0.0 | -1.0 |
| E-stop pressed in automatic, released in manual | -1.0 until the return | -2.0 / 0.0 after the release (D3) |
| safety error alone | -2.0 / 0.0 | -1.0 (D1) |
| power failure (`NOT bMainReady`) starting in manual | -2.0 / 0.0 | -1.0 |
| axis error starting in manual | -2.0 / 0.0 | unchanged (D2) |
| error / power-on failure from the last automatic scan | -1.0 | -1.0 |

Return to automatic unchanged: the body recomputes everything; `bInitRequired` (#72) still demands `power=1`.

## 5. Reset in manual: unchanged (decided)

`TelescopeControl.Reset()` stays automatic-only in `MAIN` (MONETS 112-118, MONETN 107-113). Checked:

- [read] `Reset()` (2738-2743) only sets `bResetChecked` and `bResetActive := TRUE`. `bResetActive` is cleared only in the body
  (989-991, when no axis/focus/software error is active, or during the focus-error park). `SuspendAutomatic` never clears it.
  So a forwarded reset **latches `bResetActive` through the whole manual interval** and runs on the first automatic scan:
  `_HandleReset` (2019-2085: `MarkAborted`, drops moves, `F_ResetMoveAbort`, stage reset, axis `Reset()` of stopping axes) and
  the axis reset block (408-428, `fbAzimuth/fbElevation/fbDerotator/fbFocus.Reset()` on an error). Claim confirmed.
- [read] **The same latch is already reachable today:** an MQTT `reset` calls `Telescope.Reset()` without a mode check
  (BROTLib `FB_Comm_MQTT_Influx` 161-162, same in v0.18.0). So forwarding the cabinet/pendant button would add a second path to an
  existing behaviour, not a new one. Worth a note on #168; not changed here.
- [unknown] What `HalfBROT` axis `Reset()` (`MC_Reset`) does to an axis the pendant is jogging. Not verified, which is why the
  reset is not forwarded. What a manual reset needs (axis errors from the pendant) is part of #168/#177.

## 6. Tests (MONETcommonTests, TcUnit; running them needs the user's go-ahead)

Correction to the request: there is **no `MonetManualReturn_Tests`** suite. The #72 cases went into
`MonetSuspendUnderMotion_Tests` (`Back_To_Automatic_Requires_Init_Before_Any_Move`, 384). Add the new cases there.

Capacity [read]: `MONETcommonTests.plcproj` 230-242: `MAXNUMBEROFTESTSUITES` 41, `MAXNUMBEROFTESTSFOREACHTESTSUITE` 32,
`MAXNUMBEROFASSERTSFOREACHTESTSUITE` 256. The "32" in the README is tests **per suite**, not suites. `MAIN` declares 33 suites on
`main` (34 on `develop` with `FB_JogRequest_Tests`); its "suite 35 of 41" comment is off by the removed suites. The suite has 9
tests and ~50 assert calls, so 3 more tests (~15 asserts) fit; no new suite.

**Rig** (`FB_MonetRig.Scan`, 103): pass the existing fields, `fbTel.SuspendAutomatic(bEstopIn := bEstopIn, bSafetyErrorIn := bSafetyIn,
bMainReadyIn := bMainReadyIn);`. `SetEstop`, `SetSafetyError`, `SetMainReady`, `JogElevation` exist (202, 214, 262, 274).
Mock counters available: `nStopEdges`, `nResetEdges` (axes), `nStopCalls`/`nOpenCalls`/`nCloseCalls` (covers),
`nDisableCalls` (hydraulics).

New tests in `MonetSuspendUnderMotion_Tests` (each: clock valid, Init, online, then `SetManual(TRUE)`, 30 scans):

| Id | Scenario | Expect |
|---|---|---|
| T1 `Manual_Estop_Shows_Error_In_ReadyState` | `SetEstop(TRUE)` in manual; later `SetEstop(FALSE)` | `ReadyState = -1.0` within 2 scans; back to -2.0 after the release |
| T2 `Manual_SafetyError_And_Power_Failure_Show_Error` | `SetSafetyError(TRUE)`, release; `SetMainReady(FALSE)`, restore | -1.0 each, back to -2.0 (per D1) |
| T3 `Manual_Estop_Does_Not_Command_The_Axes` | `JogElevation(TRUE)` in manual (axes enabled), then E-stop and safety error for 50 scans | counters and levels unchanged from before the E-stop: `nStopEdges`, `nResetEdges` of all axes, `bEnable` and `MovePos` of the elevation still TRUE, covers `nStopCalls`/`nCloseCalls`, hydraulics `nDisableCalls`; `bSuspendStop`, `bMoveAbortPending`, `bResetActive` FALSE |
| T4 (extend R8 of #72 if present, else in T1) | E-stop in manual, return to automatic with it pressed | first automatic scan: existing E-stop reaction (-1.0, hydraulics off), no regression |

The cold-start case (`Manual_From_Cold_Start_Does_Not_Fault`, 488) must stay green with the new call (rig passes the inputs).

Must be re-run (whole `MONETcommonTests` run in CI, `tcbuild.yml`): `MonetSuspendUnderMotion_Tests`, `MonetStopWhileTracking_Tests`
(switch to manual while tracking), `MonetErrorPark_Tests`, `MonetResetUnderMotion_Tests`, `MonetTimeouts_Tests`, `MonetSafeState_Tests`
(E-stop / safety / power rows), `FB_MonetRig_Tests`, `F_CommandAcceptance_Tests`. `F_SuspendStop_Tests` no longer lives here: it
moved to `BROTLibTests` (#24 step 6) and `F_SuspendStop` is not changed, so no re-run needed.

## 7. Release and pins

1. MONETcommon PR on `develop` (after #216 / 0.16.1 is merged): FBMTC 3.1, rig, T1-T4, docs (below).
2. CI green with the user's go-ahead (this machine is the CI runner; no `tctool`/XAE run by this plan).
3. Release **0.17.0** (minor: public method signature) via the `release.yml` workflow_dispatch (bump on `develop`, PR to `main`,
   `tag-release.yml` tags), as 0.16.0 (`f9c7d5d`, bot commit to `Global_Version.TcGVL` + `.plcproj`).
4. Pin PRs, one per site, like MONETS `d68d7ec` / MONETN #68: `<DefaultResolution>MONETcommon, 0.17.0 (BROT)</DefaultResolution>`
   in `MONETSRuntime.plcproj` / `MONETNRuntime.plcproj` **plus** the `MAIN` change of 3.2 in the same PR (the call does not need
   the inputs to compile [unknown], but without them nothing changes). CI installs the pinned release with
   `tools/Install-PinnedLibraries.ps1`; on the site PC `tools/Update-Libraries.ps1` (elevated, manual).
5. No CHANGELOG exists in MONETcommon, MONETS or MONETN; release notes are the GitHub release text.

Docs in the MONETcommon PR: `README.md` "Startup readiness" 327-331 (predicate in manual, the D1 asymmetry) and 349-351 (remove
"error and E-stop state in manual (Q5 ...)" from "Not part of this change"); `MONETcommonTests/README.md` row 33
(`MonetSuspendUnderMotion_Tests`); the #72 plan step 8 status line; this plan's status; `specs/plans/index.md`.
pyobs-brot: READY_STATE -1.0 can now appear in manual (it could already after an automatic error); check no client treats it
as "start an error recovery" while the key is on Local.

## 8. Ordering with #177

#177 changes the `PendantControl(...)` call (MONETS `MAIN` 230, MONETN 236) and `FB_MonetPendantControl`; this changes the ELSE
branch at MONETS 179-182 / MONETN 185-188. Different hunks, no code dependency [inferred]. Both need a MONETcommon release, a pin
PR with a `MAIN` change per site and a manual-mode check on site. Recommended: if the #177 policy decision is taken before this PR
is released, ship both in 0.17.0 with one pin PR and one on-site session per site; otherwise ship this alone (it needs no
policy decision) and rebase #177's `MAIN` change on it.

## 9. Hardware acceptance (separately for MONET/S and MONET/N)

**Each site and session needs the owner's explicit go-ahead. This plan authorizes no firmware deployment and no telescope
movement.** Do not infer N from S (different `MAIN`, `ready` vs `mainReady`, safety wiring).
Setup: firmware with the pin downloaded by the owner, roof closed, telescope parked, key on Local, nobody jogs. Record
MQTT `TELESCOPE.READY_STATE`, `SafetyHandling.estop/.error`, `MAIN.eTelescopeMode`, per-axis `bEnable`/`bError`.

- H1 key Local, E-stop pressed: READY_STATE -1.0 within ~1 s. Released and safety reset (cabinet button): back to 0.0 (parked).
- H2 (only if D1 = include, and only with a method the owner approves for raising a safety error) safety error: -1.0.
- H3 key back to Remote: unchanged automatic behaviour, `power=1` still required (#72).
- No jog test with the E-stop on site: T3 covers "no command from FBMTC"; the physical STO path is not changed by this plan.

Comment results per site on #168 (or a new issue split from it); close the item when both sites pass H1 and H3.

## 10. Risks

- Shadowing / signature (3.1): avoided by the `*In` names; empty-parentheses call compatibility [unknown] until compiled.
- D1: a safety error alone gives -1.0 in manual but -2.0 in automatic. Documented; aligning automatic would change
  `Initialized` handling there (not value-only) and the Wake&Shake window (#155), so it is out of scope.
- D3: dropping the stale `bEstopTriggered` member relies on every caller passing `bEstopIn`; a caller that does not would show
  -2.0 instead of -1.0 for an E-stop carried over from automatic. All known callers are changed.
- Mocks do not model STO or the pendant; T3 only proves FBMTC writes nothing. Real evidence is H1.
- Rollback: pin MONETcommon 0.16.x again and revert the `MAIN` lines (the call without arguments compiles if [unknown] holds,
  otherwise revert both together).

## 11. Not verified

Nothing compiled, nothing run (no XAE, `tctool`, TcUnit, PLC or telescope). Not checked: compiler behaviour for method inputs
that shadow FB inputs and for `SuspendAutomatic()` with defaulted inputs; whether `SafetyHandling.estop` is TRUE in `off` mode
with the hydraulics off (expected yes, independent circuit); HalfBROT `Reset()` under a pendant jog; what pyobs-brot does with -1.0
in manual. Line numbers are from the commits in the header and will move with 0.16.1/#177.

## 12. Decisions

- **D1** Safety error in the manual predicate: include (recommended; the heartbeat is the only manual telemetry and value-only
  there) and document the automatic asymmetry, or leave out for symmetry.
- **D2** Live hardware error (axis/focus/hydraulics) in manual: not now (recommended), or a read-only local term inside the guard.
- **D3** Replace the stale `bEstopTriggered` with `bEstopIn` (recommended) or OR both.
- **D4** Input names `bEstopIn`/`bSafetyErrorIn`/`bMainReadyIn` (recommended) instead of the shadowing names.
- **D5** Ship with #177 in one 0.17.0 or alone (section 8).

## 13. Steps

1. Answer D1-D5.
2. MONETcommon branch from `develop` (after 0.16.1): rig call, T1-T3 (T1/T2 fail on today's code), then 3.1, T4, docs.
3. CI run with the go-ahead; fix; merge.
4. Release 0.17.0; MONETS and MONETN pin PRs with 3.2 (merged, not deployed).
5. H1-H3 on MONET/S, then MONET/N, each with the owner's go-ahead; comment on #168.
