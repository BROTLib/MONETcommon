# Stop during a power-on that moves the axes never completes (#96)

Date: 2026-10-20. Status: **implemented in the PR `fix/stop-during-poweron-96`, CI result in the PR; not run on a PLC or telescope.** Section 4.1 is implemented as written, except that the stop write does not require `bEnabled` (the 0.9.0 code did not either). The scan model used for the tests (`StopLoop_Sim_Tests`) reproduces repeated MC_Stop restarts and a late release with skewed Done times, not an endless hang: with fixed Done times the old rule always reaches a scan where all axes are at rest, so the endless hang of #96 still rests on the code reading and on IAG50cm#108. #96 and the 0.9.0 part of #84 stay open until
the checks in section 7 pass.
Repo: BROTLib/MONETcommon. Base: `develop` = `origin/develop` = `b413ef0` (0.9.0 plus #86, #113, oil-level tests).
Line numbers below are from `b413ef0`. The #84 analysis comment used the `v0.9.0` tag, so its numbers are about 20 lines lower.

Main file: `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU` (FBMTC below). Also read: BROTLib `FB_Axis` and
`FB_AxisCommand` (BROTLib `develop` = v0.9.0; neither file changed since v0.8.0), HalfBROT `FB_AxisControl`,
`FB_ElevationControl`, `FB_AzimuthControl` and `FB_DerotatorControl` (none changed since v0.5.5), MONETS `MAIN`, and IAG50cm
`FB_StopTransaction` / `FB_TelescopeControl._StopTelescope` (IAG50cm#108, #98).

## 0. The branch `fix/slew-interrupt-reset-84-77-85-88`

This branch does **not** fix #96, and it is not unmerged. `git rev-list` reports it 0 commits ahead of `develop` and 48 behind.
Its tip `25459d5` ("Fix interruptible moves and reset (#84, #77, #85, #88), bump to 0.8.5") is in `v0.8.5` and `v0.9.0`.
That commit introduced the current `_StopTelescope`, i.e. the code on which #96 was observed. Its plan
(`2026-10-20-slew-reset-fixes.md`, lines 284-287) foresaw a second `Stop()` after `StopDone` and called it
"harmless (`MC_Stop` is allowed at standstill)". That assumption is wrong (section 2), and the branch can be deleted.

## 1. Observation (#96, MONETcommon 0.8.5, BROTLib 0.8.0, HalfBROT 0.5.5)

- Start: reset, unpowered, az 137.6 / el 53. `power=1`, `stop=1` about 3 s into the move home. The MONETS home is az 175 / el 40 (`MAIN` line 43).
- The axes stopped at az 150.9 / el 40.0. Azimuth stopped mid-move. Elevation was at its home value.
- After that, `bStop` stayed TRUE and `bStopped` FALSE for more than 6 min. READY_STATE stayed at -2.0. `power=1`, `reset=1` and goto/track
  had no effect.
- Az: `bStopAxis` T, `StopCommand.Pending` T, `StopCommand.Execute` F, `Axis_Stop.Busy` T, `MoveCommand.Execute` T.
- El: `bStopAxis` F, `StopCommand.Execute` T, `Axis_Stop.Busy` T and `Done` T, `MoveCommand.Execute` T.
- Recovery: `MAIN.TelescopeControl.bStop := FALSE` over ADS, then `power=1` reached 1.0 in about 10 s.
- The same hang on 0.9.0 after a stop 4 s into a slew (#84 comment of 2026-10-20 14:55). In that case clearing `bStop` over ADS was
  not enough: `park=1` was accepted but nothing moved.

## 2. Root cause

### 2.1 Confirmed from the code

Scan order on MONET/S: `TelescopeControl` runs before the axis blocks (MONETS `MAIN` 104-117, then 135-149). The telescope therefore
sees the axis outputs from the previous scan.

1. **`_StopTelescope` asks for a stop again after every `StopDone`.** FBMTC 1582-1607: for each axis that is
   `NOT fbAxis.StandStill`, the `ELSE` branch writes `StopAxis := TRUE` on every scan once `bStopIssuedX` is TRUE. The
   `bStopIssued*` guard (1583) only prevents dropping a level that was already latched. It does not prevent a new edge.
2. **The axis block clears `bStopAxis` on `StopDone`** (HalfBROT `FB_ElevationControl` 110-112, `FB_AzimuthControl` 86-88,
   `FB_DerotatorControl` 90-92). The next `StopAxis := TRUE` is then a FALSE->TRUE edge. The setter (`FB_AxisControl` 258-262)
   calls `fbAxis.Stop()`, and `Stop()` calls `StopCommand.Request()` (`FB_Axis` 395-406).
3. **The axis is still not `StandStill` when `StopDone` is seen.** `FB_AxisCommand` releases `Execute` only one call after it sees
   `Done` (`FB_AxisCommand` 35-37; `Done` is the MC block's output from its last call). While `MC_Stop.Execute` is TRUE the axis
   stays in Stopping. The library header says so itself (`FB_AxisCommand` 6-7: "MC_Stop must not be held (it keeps the axis in
   Stopping)"). `StandStill` is `Axis_Status.StandStill`, read at the start of the axis call (`FB_Axis` 118, 356). So in the scan
   after `StopDone`, the telescope sees `StopAxis` FALSE and `StandStill` FALSE together, and point 1 asks for a new stop.
4. **The new request restarts `MC_Stop` on an axis at rest.** `Request()` while `Execute` is TRUE holds `Execute` low for one cycle
   and then raises it again (`FB_AxisCommand` 28-34). `StopAllowed` includes `StandStill` (`FB_Axis` 256-260), so a new `MC_Stop`
   starts, the axis is Stopping again, `Done` comes, `bStopAxis` is cleared, and the cycle repeats. Each axis loops on its own
   period. It is `StandStill` (as the telescope sees it) for about one scan per period.
5. **`bStop` needs all enabled axes at `StandStill` in the same scan.** `bStopped` (644-646) requires every enabled axis to be at
   standstill at once, and `bStop` is cleared only on `bStopped` (683-684). Once two axes are out of phase, that never happens.
   Nothing else clears `bStop`. The 15 s `tonCommandTimeout` (602) only drives the log event, and that event always prints
   `NOCOMMAND` because the call passes `OnMessage := 'NOCOMMAND'` (794), which overrides the text set at 601.
6. **Why every command is ignored.** The stop branch (462-485) comes first in the arbitration. Every scan it clears `bPower`, `bPark`,
   `bGoto`, `bSlew`, `bGoHome` and `bTrack` and sets `bMoveAbortPending`.
   - `reset=1` does not touch `bStop` (174-196).
   - In 0.8.5, `Init()` accepted the request, and the stop branch dropped `bPower` in the next scan. On `develop` and 0.9.0, `Init()`
     refuses with 'power-on ignored: stop in progress' (1755-1760).
   - `Park()` only sets `bParkAfterStop` (1833-1834), which runs only after `bStopped`.
7. **Why clearing `bStop` over ADS helps on #96 but not on #84 (0.9.0).** `bMoveAbortPending` is still TRUE, so the
   `ELSIF bMoveAbortPending` branch (504-508) keeps dispatching `stop`, and `_StopTelescope` keeps looping.
   - `power=1` works because `bPower` ranks above that branch (501). `_StopTelescope` is then no longer called, the loop ends, and
     the power-on runs.
   - `park=1` ranks below that branch (510), so it never starts. That matches both reports.
8. **`MoveCommand.Execute` TRUE is a leftover, not the cause.** `MoveCommand` is released only on `Axis_Move.Done` / `Axis_Modulo.Done`
   (`FB_Axis` 208, `FB_AxisCommand` 35-37). A move aborted by `MC_Stop` ends with `CommandAborted` and never sends `Done`.
   `Stop()` and `CancelMove()` only drop a pending request (`FB_AxisCommand` 53-61). So `Execute` stays TRUE until the next `MoveTo()`,
   which drops it for one cycle (28-30), or until an axis `Reset`. A held `Execute` cannot restart an `MC_MoveAbsolute`, which needs a
   rising edge, and nothing in `FB_Axis` gates the stop channel on `MoveCommand`. The issue's reading "move and stop hold each other"
   is therefore not supported by the code. The stop channel re-triggers itself.
9. **Related latent bug, same root in the library: a stop during homing.** `HomeCommand` has the same problem: an aborted `MC_Home`
   leaves `HomeCommand.Execute` TRUE, so `HomeActive` (`FB_Axis` 244) stays TRUE. `Home()` is then ignored (384-387). If the axis is not
   calibrated after the abort, the next power-on writes `HomeAxis := TRUE` in stage 10 (FBMTC 1222-1228). The homing never starts, and
   the power-on would stay at READY_STATE 0.8. This follows from the code only. It has not been observed.

### 2.2 Inferred, not confirmed on MONET/S

- **That the MONET/S hang is this loop.** The #96 readings fit two phases of it:
  - Elevation shows `StopDone` already cleared `bStopAxis`, with `Execute` and `Done` still TRUE (the scan before the re-request).
  - Azimuth shows a re-request in its cycle with `Execute` held low (`Pending` T, `Execute` F).

  They were probably separate ADS reads, though, not one consistent cycle. So they are consistent with the loop but do not prove it.
- **Strong outside evidence: the same loop on IAG50cm, recorded on hardware.** IAG50cm#108 used the same BROTLib `FB_Axis` and the
  same "`NOT StandStill` -> `StopAxis := TRUE`" rule. A 10 ms edge log showed `StopCommand.Execute` re-triggered about 33 times per
  second on both axes. `MC_Stop` Done was one scan apart between the two axes, they stayed out of phase, and `bStopped` never came.
  It was fixed by one `MC_Stop` per axis per stop transaction (IAG50cm PR #117, `FB_StopTransaction`, in v0.4.0), and the fix
  passed 4 of 4 on the telescope.
- That the phase lock lasts indefinitely (>6 min) on MONET/S rather than resolving by chance. A stop that sometimes completes
  (0.8.5 slew stop: `bStop` cleared after ~7 s) fits a phase-dependent coincidence.
- Why the azimuth showed `Axis_Stop.Busy` TRUE while `Execute` was FALSE (Tc2_MC2 `MC_Stop` internals, not visible in our code).
- Whether elevation's move was aborted just before 40.0 or had finished. `MoveCommand.Execute` TRUE suggests it was aborted.
- The 0.8.5 slew report ("`MC_Stop` accepted but the axes ran to the target") is **not** explained by this. On 0.9.0 the axes did
  decelerate. That observation stays with #84.

## 3. Where the fix belongs, and the release chain

The loop is created by the caller's re-request rule, so the fix goes in **MONETcommon** (`_StopTelescope` and the `bStop` release). BROTLib
`FB_Axis` behaves as documented. IAG50cm fixed the same caller bug in its own application in the same way. HalfBROT needs no change.

Chain for the recommended option:
1. MONETcommon `develop` -> **0.9.1** (`ProjectVersion`, `Global_Version`). No BROTLib/HalfBROT pin change: MONETcommon pins
   BROTLib 0.9.0 / HalfBROT 0.5.5, and its test project pins HalfBROT 0.5.6.
2. MONETS: `MONETcommon, 0.9.0 (IAG)` -> `0.9.1` in `MONETSRuntime.plcproj`, `ProjectVersion` 1.4.2 -> 1.4.3, install via
   `Update-Libraries.ps1`, activate on MONET/S.
3. MONETN: the same pin bump, 0.4.2 -> 0.4.3. MONETN gets the fix with no separate on-site check planned (same code path).
4. IAG50cm: not affected (no MONETcommon; BROTLib 0.8.0; its own fix is #117).

Follow-up, not part of #96: open a **BROTLib issue** for `FB_AxisCommand` releasing `Execute` on `CommandAborted` (point 2.1.9 and the
`MoveCommand` leftover). That would be BROTLib 0.9.1, then pin bumps in HalfBROT, MONETcommon, MONETS and MONETN, and later IAG50cm.
It is a library-wide behaviour change, so it gets its own plan in `BROTLib/specs/plans`.

## 4. Options and recommendation

**Recommended: A, a stop transaction in MONETcommon (port of IAG50cm `FB_StopTransaction`, three axes).**

- **A (recommended).** One `MC_Stop` per axis per stop transaction, never re-requested because of `StopDone`. The transaction ends when
  every axis that was sent a stop has reported `StopDone` (or is disabled) **and** `bStopped` has held for 3 scans. This removes the
  dependence on all axes finishing in the same scan and the restart of `MC_Stop` on an axis at rest. It is proven on IAG50cm hardware,
  and it is the smallest release chain (MONETcommon only).
- **B, rejected for #96.** Make `FB_Axis.Stop()` idempotent while a stop is executing (like `Home()` while `HomeActive`). It would fix
  every caller. But it changes library semantics for IAG50cm, which relies on its own transaction, needs the long BROTLib->HalfBROT->
  MONETcommon->MONETS chain, and still leaves the same-scan `bStopped` requirement. It stays a candidate hardening item for the
  BROTLib follow-up issue.
- **C, rejected as the fix.** Only a timeout that clears `bStop`. It hides the loop and leaves `MC_Stop` restarting at ~30 Hz for 15 s
  under load. It is kept only as a safety net inside A (D1).

### 4.1 Changes (A)

1. **New `FB_MonetStopTransaction`** (pure logic, no axis access), modeled on IAG50cm `FB_StopTransaction`, for Az/El/Dr.
   - **Body (request logic, called from `_StopTelescope`).**
     - Inputs per axis: `bEnabled` (`fbX.bEnable`), `bStandStill` (`fbX.fbAxis.StandStill`), `bStopAxis` (`fbX.StopAxis`).
     - Outputs per axis: `bWriteLow` (one scan) or `bWriteHigh` (one scan).
     - Rule: `IF NOT bIssuedX AND bEnabledX AND NOT bStandStillX`. If `bStopAxisX` is still latched from an earlier stop, write it low
       for one scan first (keeps the 0.8.5 re-arm, #84 step 0). Otherwise write it high once and set `bIssuedX`.
     - After that, never write it high again in the same transaction.
   - **`Monitor()`, called every scan after `bStopped` is computed.**
     - Inputs: `bStopped`, `StopDone` and `bEnabled` per axis.
     - It latches `bDoneX` and counts the scans with `bStopped` TRUE.
     - `bSettled := bStopped for >= 3 scans AND for each axis (NOT bIssuedX OR bDoneX OR NOT bEnabledX)`.
     - Without an active transaction, `bSettled := bStopped`.
   - **Methods.**
     - `Begin()`: in the scan where `TCS_command` becomes `stop`.
     - `Cancel()`: whenever `TCS_command` is not `stop`. This replaces the `bStopIssued*` reset at 628-633.
     - `Restart()`: see D2.
2. **`_StopTelescope`** (1562-1608) keeps the `Tracking` and `MoveAxis := FALSE` lines. It writes `StopAxis` only from the FB outputs.
   On `bSettled` it writes all three `StopAxis := FALSE`. The `bStopIssuedAz/El/Dr` variables are removed.
3. **Body:** 683 `IF bStopped THEN bStop := FALSE` becomes `IF fbStopTx.bSettled THEN`. The `bMoveAbortPending` clear (685-687)
   follows the same condition, so the interrupt path (#83/#84) uses the same transaction.
4. **D1, safety net for a missing `Done`.** On `tonCommandTimeout.Q` with `TCS_command = stop`:
   - If `bStopped` holds but some issued axis never reported `Done`: settle anyway, clear `bStop` and `bMoveAbortPending`, log
     WARN 'STOP completed without StopDone from <axes>', and count it in `nStopForcedCount` (VAR PERSISTENT, like
     `nInterruptedAutoClearCount`). That count should stay 0.
   - If an axis is still not at standstill: log ERROR 'STOP timeout: <axes> not at standstill' once and keep the stop. Do not drop the
     enables: dropping them under motion caused the 0x4260/0x4550 errors in #84.
5. **D2, a repeated `stop=1` restarts the transaction only for moving axes.** `Stop()` while `bStop` is already TRUE calls `Restart()`.
   That clears `bIssuedX` only for axes with `fbAxis.InMotion OR fbAxis.InRamp` and NOT `fbAxis.Stopping`. A client that repeats
   `stop` cannot recreate the loop, and an operator can retry a stop that was dropped while the axis kept moving.
6. **D3, `reset=1` recovers a stuck stop without ADS.** On `rtrigReset.Q` with `bStop` TRUE and no enabled axis in motion
   (`NOT InMotion AND NOT InRamp`), call `fbX.Reset()` on the axes that are still `Stopping`. The axis `Reset` releases
   `StopCommand.Execute` and drops pending requests (`FB_Axis` 262). Then `Cancel()` the transaction, and `bStop` clears at
   `bStopped`. Never reset an axis in motion (#84 side finding: a reset 5 s into a park was followed by 0x4260).
7. **Log text:** remove `OnMessage := 'NOCOMMAND'` from the `TCSCTOEvent` call (794). The text set per command at 598-626 then
   applies, giving 'A timeout occurred during STOP command!'.

Size: about 120 lines in FBMTC plus a 90-line FB. Risk: medium. It touches the arbitration's stop release, and that is covered by
the tests in section 6.

## 5. Backward compatibility

- **Public interface of `FB_MonetTelescopeControl`:** unchanged (methods, properties, `I_AltAzTelescope`). MQTT topics and
  READY_STATE values are unchanged. After a stop during a power-on from a non-parked state, READY_STATE is -2.0 (as on 0.8.5). After
  #113 the same holds from parked, because the covers are already opening.
- **Timing:** `bStop` now clears at least 3 scans after `bStopped`, and only after the `StopDone` of every axis that was sent a stop.
  A stop of a resting telescope (stage 0 of #80) still completes within a few scans.
- **ADS symbols:** `bStopIssuedAz/El/Dr` are removed, and `fbStopTx.*` and `nStopForcedCount` are added. Update ADS watcher/recording
  lists, including the MONET test plan in the IAG repo (`specs/plans/2026-10-12-monet-telescope-test-plan.md`, which lists these
  names).
- **Log text:** 'A timeout occurred during NOCOMMAND command!' becomes '... during STOP command!' (and POWER/PARK/... for the
  others). Check client log filters (pyobs).
- **Persistent data:** one new VAR PERSISTENT. A download needs the usual persistent-data check, and the counter starts at 0.
- **New behaviour:**
  - D2: a repeated `stop` can re-issue a stop to an axis that is moving again.
  - D3: `reset=1` may call `Reset()` on axes that are stuck in Stopping.
  - Both happen only in states that hang today.
- **Not changed:** BROTLib, HalfBROT, IAG50cm.

## 6. Tests (MONETcommonTests, TcUnit, no hardware)

`FB_MonetTelescopeControl` cannot be instantiated in a test: it takes `REFERENCE TO` the HalfBROT axis FBs on `AXIS_REF`
(`2026-09-22-add-tcunit-tests.md`). The tests therefore follow the pattern already in use: pure logic plus a scan loop inside one test
method. All instances are local method VARs, so each call starts fresh (TcUnit calls every test every cycle). One loop iteration is
one PLC scan, in MONETS order: telescope part first, then the axis part. Limits per README: 32 tests / 256 asserts per suite.

**Test helpers (test project only):**
- **`FB_MockStopAxis`** = HalfBROT `StopAxis` setter semantics (edge -> `Stop()`), plus the **real** `BROTLib.FB_AxisCommand` as the
  stop channel, plus an `MC_Stop`/PLCopen state model:
  - States: Moving, Stopping, StandStill.
  - A rising `Execute` goes to Stopping. `Done` comes after `nDoneScans` and stays TRUE while `Execute` is TRUE.
  - The state stays Stopping while `Execute` is TRUE, and goes to StandStill when `Execute` is FALSE after `Done`.
  - If `Execute` falls before `Done`, the stop finishes anyway and pulses `Done` once.
  - `StopAllowed` = StandStill or Moving. `StandStill`/`Stopping` are sampled at the start of the call. `StopDone` clears `bStopAxis`.
  - Counters: `nExecuteEdges`, `nStopCalls`.
- **`F_StopWrite_v090`**: a verbatim copy of the 0.9.0 rule (FBMTC 1582-1607), used only to show that the mock reproduces the hang.

**Suite `StopLoop_Sim_Tests`** (reproduces #96 / IAG50cm#108 without hardware):
1. **`Sim_v090_Rule_Loops_With_Skewed_Done`** (characterization).
   - Setup: Az Moving with `nDoneScans` 3, El Moving with 5, Dr StandStill, `bStop` set at scan 1, 1000 scans.
   - Each scan: `bStopped` from the mocks' outputs; `F_StopWrite_v090` writes `StopAxis`; the mocks run.
   - Asserts: Az and El `nExecuteEdges` > 10; `bStopped` never TRUE after scan 20.
   - If this test does not fail the old rule, the mock is wrong.
2. **`Sim_v090_Rule_Settles_With_Equal_Done`**: the same with 3/3 and the same start scan. Assert that `bStopped` comes, which shows the
   dependence on phase that explains the occasional pass.
3. **`Sim_Transaction_One_Edge_Per_Axis`**: the setup of 1 with `FB_MonetStopTransaction`. Asserts:
   - `nExecuteEdges` = 1 for Az and El, 0 for Dr.
   - `bSettled` by scan `max(nDoneScans) + 6`.
   - `bSettled` stays TRUE for 50 more scans.
4. **`Sim_Transaction_Latched_StopAxis`**: Az `bStopAxis` TRUE at the start. Asserts: one scan written low, then one edge; `nStopCalls` = 1.
5. **`Sim_Transaction_Done_Skew_10_Scans`**: Az 3, El 13. Asserts: settled only after El's `Done`; one edge each.

**Suite `FB_MonetStopTransaction_Tests`** (pure FB, inputs set per step):
- `Moving_Axis_Gets_One_Stop`
- `Standstill_Axis_Gets_No_Stop`
- `Latched_StopAxis_Low_Then_High`
- **`No_Second_Stop_While_Stopping_After_Done`** (the #96 guard): after the write, `StopDone` pulses and `bStopAxis` is FALSE with
  `StandStill` FALSE for 5 scans. Assert that `bWriteHigh` stays FALSE.
- `First_Standstill_Sample_Does_Not_Settle`: `bStopped` TRUE before `Done` -> not settled (IAG50cm#98).
- `Settles_After_Done_And_Three_Scans`
- `Done_In_Different_Scans_Settles`
- `Disabled_Axis_Counts_As_Done`
- `No_Transaction_Settles_On_Standstill`: stop at stage 0 without motion (#80).
- `Cancel_Rearms_Next_Transaction`: a stop cut off by a reset or a power-on sends again on the next stop.
- `Restart_Resends_Only_To_Moving_Axes` (D2)
- `Missing_Done_Settles_Only_On_Timeout` (D1, if the timeout decision is put in the FB as a `bTimeout` input; keep it there so it can be tested).

**BROTLibTests (follow-up issue, not this release):**
- `FB_AxisCommand_Tests.Aborted_Keeps_Execute` pins today's behaviour (the `MoveCommand.Execute` leftover of #96).
- After the BROTLib change it is replaced by `Aborted_Releases_Execute`.

Add both suites to `MONETcommonTests` `MAIN` and the README table. CI (`tcbuild.yml`) must pass.

## 7. On-site check, MONET/S (after 0.9.1 is installed and activated)

Rules as in the earlier plans: never open the roof. Someone watches the covers.

**Record** with an ADS on-change logger at cycle resolution, as for IAG50cm#108. If that is not available, use a sum-read at 20 Hz so
that the values come from one cycle.
- **`MAIN.TelescopeControl`:** `bStop`, `bStopped`, `bMoveAbortPending`, `TCS_command`, `bPower`, `bReset`, `bParkAfterStop`, the
  `_PowerOn` stage, `fReadyState`, `nStopForcedCount`, and `fbStopTx` (issued/done per axis, `bSettled`).
- **Per axis (Az/El/Dr):** `bEnable`, `bStopAxis`, `bMoveAxis`, `bHomeAxis`, `fActualPosition`, and from `fbAxis`: `StandStill`,
  `Stopping`, `StopDone`, `StopCommand.Pending/Execute`, `Axis_Stop.Busy/Done/Error`, `MoveCommand.Execute`,
  `HomeCommand.Execute`, `HomeActive`, `ErrorID`.
- **Also:** MQTT READY_STATE and the log topic.

**Per run, note:**
- the time of `stop=1` and the time `bStop` goes FALSE
- the number of `StopCommand.Execute` rising edges per axis
- positions at the stop and at rest
- the READY_STATE sequence
- log lines (expect 'STOP commencing' / 'STOP completed', no timeout)
- for the following `power=1`: the time to 1.0 and any error

**Step 0 (optional, on 0.9.0 before the update, confirms 2.2).** Run step 1 once with the logger. Expected if the cause is right:
`StopCommand.Execute` edges repeating every few scans on at least one axis, and `bStopped` never TRUE. Recovery: ADS
`MAIN.TelescopeControl.bStop := FALSE`, then `power=1`. **Not** `park`: on 0.9.0 the park stays behind `bMoveAbortPending`.

1. **Stop 3 s into the move home.**
   - Setup: online, slew to az 140 / el 55 and wait for standstill. Then `reset=1`: READY_STATE goes to -2.0 (MarkAborted) and the
     axes stay enabled. Then `power=1`: covers are open, so stage 10 is passed at once, and stage 30/40 moves to az 175 / el 40.
   - About 3 s after the axes start moving, send `stop=1`.
   - Expect:
     - both axes decelerate, az stops short of 175
     - exactly one `StopCommand.Execute` edge per moving axis
     - `bStop` FALSE within the deceleration time + 0.1 s
     - READY_STATE -2.0, no 'timeout during STOP'
     - `bStopAxis` FALSE on all axes afterwards
   - Then `power=1`: 0.7 -> 1.0, ending at az 175 / el 40.
2. **Stop at stage 0 without motion (#80 check on 0.9.1).** Parked, `power=1`, `stop=1` within 2 s while the covers open.
   - Expect: no `StopCommand` edge, `bStop` FALSE within 3 scans, READY_STATE -2.0 (#113), covers keep opening.
   - Then `power=1`: 1.0. Then `park=1` to return to parked.
3. **Repeat after reset.** After step 1: `reset=1`, then step 1 again. Repeat until there are **3 runs** of step 1 and 2 runs of step 2.
   The bug depends on phase, so a single pass proves little.
   - Expect the same values each time, `bStopAxis` FALSE at idle before each run, and `nStopForcedCount` 0.
4. **Regression for #84 (slew).** Slew az 150 / el 45 from az 120 / el 60, `stop=1` 3 s in, 3 times. Expect one edge per axis,
   `bStop` clears, and the next slew is accepted. Then `park=1` 2 s into a slew: 'move interrupted', then the park runs.
5. **Only on request (needs uncalibrated axes, i.e. after a TwinCAT restart, Wake&Shake context):** stop during the stage 10 homing
   sweep, then `power=1`. This watches for the latent `HomeActive` bug (2.1.9): `HomeCommand.Execute` and `HomeActive` stay TRUE
   after the stop, and stage 10 stays at 0.8. If it shows, report it on the BROTLib follow-up issue. Recover with an axis `Reset`.

Comment the results on #96 and #84. Close #96 after steps 1-3 pass.

## 8. Not verified

- Nothing in this plan has been compiled or run.
- That the MONET/S hang is the re-request loop: inferred from the #96 readings (probably not one cycle) and from IAG50cm#108. Step 0
  would confirm it.
- Tc2_MC2 details: Stopping held while `MC_Stop.Execute` is TRUE (stated in `FB_AxisCommand`, seen on IAG50cm); `Busy` after `Execute`
  falls; `MC_Home`/`MC_MoveAbsolute` give `CommandAborted` and no `Done` when stopped; `ConstantVelocity`/`InRamp` FALSE at rest
  (used by D2/D3).
- Whether the NC motion lock (IAG50cm#98, StateDWord bit 29) outlasts the PLCopen Stopping state on MONET/S. The 3 settle scans are
  taken from IAG50cm and are not measured here.
- The derotator state during #96 (not recorded).
- The stage 10 homing hang (2.1.9): code only, never observed.
- MONETN: no on-site check planned.

## 9. Open questions (answered 2026-10-20)

1. D2, answered: pyobs sends `stop` once, but the tests so far were run without pyobs (ADS/MQTT by hand), so the pyobs behaviour is not known. D2 stays limited to moving axes as written.
2. D3, answered: yes, an automatic axis `Reset()` is acceptable because it is limited to axes without motion and the axes stand still afterwards.
3. BROTLib follow-up, answered: filed as [BROTLib#92](https://github.com/BROTLib/BROTLib/issues/92), with the step 5 check noted as not run yet.
4. Not known: whether the #96 ADS readings were taken in one cycle or one by one (the reporter does not remember). The mechanism in section 2 does not depend on it.
