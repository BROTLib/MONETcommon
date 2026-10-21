# A preempted move keeps moving to its old target (#56, and part 1 of #66)

Date: 2026-10-21. Status: **plan agreed (section 11), implemented in the PR (0.9.2 after the release). Nothing run on a PLC or telescope.**
Repo: BROTLib/MONETcommon. Base: `develop` = `origin/develop` = `2ce22ef` (0.9.1: #96 stop transaction, #86, #113).
Line numbers below are from `2ce22ef`.

Main file: `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU` (FBMTC below). Also read: `FB_MonetStopTransaction`,
`F_CommandAcceptance`, BROTLib `FB_Axis` (`develop` = 0.9.0), HalfBROT `FB_AxisControl` (0.5.6, `MoveAxis`/`StopAxis` setters),
MONETS `MAIN` (reset button, line 91-97), `FB_MonetMqttWatchdog` / `FB_MonetCabinetControl` (both call `Park()`), IAG50cm
`FB_TelescopeControl` (`develop`, and tag `v0.3.11` for IAG50cm#101), and the IAG plan `IAG/specs/plans/2026-10-14-iag50cm-slew-replacement.md`.
Issues read: #56, #66, #84 (comments), BROTLib#92, IAG50cm#43, IAG50cm#101.

## 0. The issue text of #56 is out of date

#56 was written against code from before 0.8.5. It says the preemption branches "only reset their stage". That is no longer the case:

- 0.8.5 (#84) added `bMoveAbortPending`. Every preemption branch sets it, and the arbitration then dispatches `stop` before the
  new command (FBMTC 544-548).
- 0.8.6 (#83) routes every accepted move over a running one through the supersede block (463-485), which also sets it.
- 0.9.1 (#96) makes that stop one `MC_Stop` per moving axis (`FB_MonetStopTransaction`). The new command starts only after the
  transaction has settled (729-737).

So the main case of #56 (Park/GoHome/Goto/Track over a running goto/slew/home) is handled **in the code**, but it has **never
passed on the telescope**.
- On 0.8.4 and 0.8.5, `gohome=1` 1.5 s into a slew ran the slew to its end (#84 comments of 2026-10-19 and 2026-10-20). On 0.8.4
  the cause was a `bStopAxis` latched from an earlier stop. On 0.8.5 `MC_Stop` was accepted but the axes still ran to the
  target, which is unexplained (the #96 plan leaves it with #84).
- On 0.9.0 an explicit `stop` 4 s into a slew did brake the axes, then hung (#96).
- No preemption has been run on MONET hardware with 0.9.x.

What is still missing in the code is smaller than #56 says: mainly the **reset** path, which is also part 1 of #66 (section 2.2).

## 1. Observations

- **IAG50cm#43 (2026-10-12, v0.3.2):** park 2 s into a goto, the axes ran on to the goto target, about 8 s of unwanted motion.
  The same for goto over slew, about 10 s. It was fixed with `bStopping` (IAG50cm PR #45) and verified on 2026-10-13 (T9 steps 1, 2, 7).
- **IAG50cm#101 (2026-10-14, v0.3.11):** slew over slew. `TCS_command` 6 -> 5 and `dec.bMoveAxis` FALSE, but Dec did not brake
  and ran 9.3 s to the first target. Still open, and its cause is not established (IAG plan above, "Hypotheses to distinguish").
- **MONET/S #84 (0.8.4/0.8.5):** `gohome=1` during a slew: the slew ran to its end, then the move home started and ended in
  0x4260/0x4550 (HalfBROT#77). Side finding: a `reset=1` sent 5 s into a park move was followed by elevation 0x4260.
- **#66 (code reading 2026-10-11, no hardware):** a reset or interrupt during power-on stage 10/30 leaves the axes moving.
  Part 2 of #66 (false `bPowerFailure` on the stop path) is fixed: `_PowerOn` now checks the abort first (1225-1240).

## 2. Root cause and remaining gaps

### 2.1 Confirmed from the code: which preemption stops the axes

Scan order on MONET/S: `TelescopeControl` runs before the axis blocks, so the body sees the axis state of the previous scan.
`MoveAxis := FALSE` calls `FB_Axis.CancelMove()`, which drops only a move that has not started (`FB_Axis` 409-418). Only a
FALSE->TRUE edge on `StopAxis` calls `FB_Axis.Stop()` (HalfBROT `FB_AxisControl`, `StopAxis` setter).

| Preemption | Path in FBMTC | Axes stopped? |
|---|---|---|
| Slew/MoveAltAz/Goto/Track/TrackRaDec/Home over a running move or over tracking, including slew over slew and goto over goto | `_AcceptMove` -> `replace` (last row of `F_CommandAcceptance`; `_CommandState` counts the own flag because the methods pass `eExcludeOwn := no_command`, 2038-2043) -> supersede block 463-485 sets `bMoveAbortPending` -> 544-548 dispatches `stop` in the same scan -> `_StopTelescope` 1621-1656 | yes, one `MC_Stop` per moving axis |
| `Park()` (also MQTT watchdog, cabinet over-temperature) over goto/slew/gohome | park branch 558-567 | yes, the same path, one scan later |
| `Park()` over a running power-on | 527-539 (#87) | yes |
| `stop=1`, horizon guard (421-427), invalid clock (169-171) | stop branch 502-525 | yes |
| Axis or focus error | 304-323: drives disabled, emergency park | no `MC_Stop`, the drives are switched off (unchanged, emergency path) |
| **`reset=1` (MQTT, cabinet or pendant button) during goto/slew/gohome/track** | reset block 214-236: flags dropped, stages reset, **`bMoveAbortPending := FALSE` (226)** | **no** |
| **`reset=1` during power-on stage 10/30/40** | reset block, then `_PowerOn` abort 1242-1255: only `MoveAxis/HomeAxis := FALSE` (cancel only) | **no** (#66 part 1) |
| `SuspendAutomatic` (switch to manual) during a power-on or a move | 1355-1363: `_PowerOn` abort only, the body is no longer called | no (non-goal, as in 0.8.5) |
| `Park()` over tracking (no goto/slew/gohome flag) | park branch 569-575: waits for `NOT bBusy AND (bReady OR bStopped)`, then park stage 10 ends tracking | not through the stop. There is no "old target" here, so it is not a #56 case (see 2.3) |

### 2.2 The gaps

**G1: a reset under motion lets the axes run to the old target (#56 for the reset, #66 part 1).**
- The reset block clears the move flags (#85) and resets the stages. It explicitly clears `bMoveAbortPending` (226, from the
  0.8.5 plan: "Clearing it makes sure reset never blocks the next power-on"). Nothing writes `StopAxis`.
- In the next scan the arbitration falls to the `ELSE` branch (625-632), and `MoveAxis := FALSE` is again only `CancelMove()`.
- With `bPower` TRUE, `_PowerOn` runs its abort block in the same scan: `HomeAxis := FALSE` is `CancelHome()`, so a homing
  sweep (stage 10) and the move home (stage 30/40) keep running.
- A park kept by the reset (bPark survives a reset on purpose, emergency park) then waits in the park branch for
  `NOT bBusy`, so **an emergency park after a reset under motion is delayed by the rest of the old move.**

**G2: the command timeout carries over from one command into the next, so the abort stop "times out" at once.**
- `tonCommandTimeout` is called with `IN := TCS_command = <cmd>` inside the branch of that command (635-667), so `IN` is TRUE
  in every branch. `IN` only falls in a scan with `no_command`. A preemption goes from slew/goto/track straight to `stop` and
  then straight to the new command, so `IN` never falls.
- `ET` keeps counting from the start of the first command, and only `PT` changes (15 s for stop, 2 min for slew/goto/park/gohome).
- Consequences, read from the code:
  - A stop that preempts anything that ran longer than 15 s (any track, most gotos) has `fbStopTx.bTimeout` TRUE from its first
    scan (689). In that scan the axes are not at rest, so `Monitor()` raises `bStuck` and the body logs ERROR 'STOP timeout:
    an axis is not at standstill'. `TCSCTOEvent` adds WARN 'A timeout occurred during STOP command!'.
  - If an issued axis has not reported `StopDone` by the third scan at rest, the transaction settles as `bForced` (WARN and
    `nStopForcedCount`+1). That bypasses the `StopDone` guard of #96, but only in this case.
  - The command after the stop inherits the same `ET`. For example, a park after a long track logs a park timeout at once.
- The same applies to an explicit `stop=1` during a track. This is the IAG50cm #40 bug, which IAG fixed with
  `eLastTimeoutCommand` (IAG `FB_TelescopeControl` 740-748). MONETcommon never got that fix.
- Inferred, not checked against logs: some of the 'timeout during NOCOMMAND' lines seen with 0.8.x/0.9.0 may come from this.

**G3: a replacement does not restart the 600 s state-machine timeout.**
- `tonSlewTimeout(IN := bSlew)` and `tonGotoTimeout(IN := bGoto)` (436-438) keep running when slew replaces slew or goto
  replaces goto, because the flag stays TRUE.
- A client that keeps sending new targets for 10 min gets 0x40/0x10 on a slew that has just started. This is minor. IAG50cm
  restarts them (IAG 701, 718).

**Accepted, not a gap: a power-on during the reset stop.** `bPower` outranks `bMoveAbortPending` (541 vs 544), so `power=1` takes
over from the abort stop. If the stop was dispatched for at least one scan, the `MC_Stop` already sent keeps braking: the axis
block holds `StopCommand.Execute` until `Done`. Power-on stage 10->30 waits for `bStopped` anyway (1303-1304). The abort flag stays
set through the power-on, and a no-op stop follows it (0.8.5 lifecycle). Unchanged.

### 2.3 IAG50cm: what to copy and what not

- **The #43 fix (`bStopping`, IAG 96, 544-551, 632-646, 660-669, 688-695)** works the same way as MONET's `bMoveAbortPending`.
  One difference: IAG stops only `IF TCS_command = <the preempted command>`, while MONET stops whenever a lower flag was set.
  - IAG has that condition so that a Slew()/Track() arriving during a park does not stop the park. In MONET such a move is
    rejected as `parking` before it can get there (`F_CommandAcceptance`).
  - When nothing is moving, MONET's extra stop settles in 3 scans (#80, stage 0).
  - **Do not port the condition.** It would only add a path where a flag is dropped while `TCS_command` is still the old value.
- **IAG50cm#101 (slew over slew did not brake).** The IAG plan lists candidate causes. None of them has a counterpart in MONET 0.9.1:
  1. *Same-scan re-entry*: IAG's `bRestartSlew` branch does not set `TCS_command := stop`, so the dispatcher can call
     `_SlewTelescope` again in that scan. In MONET the supersede block (463) runs before the arbitration, and the
     `bMoveAbortPending` branch sets `stop` in the same scan. Even if it were re-entered, `_SlewTelescope` stage 0 waits for
     `bStopped` (1544).
  2. *Three-step stop*: IAG v0.3.11 `_StopTelescope` cleared Tracking, then MoveAxis, then wrote `StopAxis`, each on a later
     scan, and it had no re-arm for a latched `StopAxis`. MONET clears `MoveAxis` and writes `StopAxis` in the same call, and
     re-arms a latched level (`abWriteLow`, #84 step 0).
  3. *Stop released early*: MONET releases on `fbStopTx.bSettled` (#96, IAG50cm#98).

  Because the cause of #101 is unknown, this comparison does **not** prove that MONET is safe. Slew over slew, with a first target
  beyond the stopping distance, is a required on-site step (7, A2).
- **Also do not port** IAG's `Slew()` writing `bRestartSlew := bSlew` (IAG 2277). MONET's `_AcceptMove` -> supersede already covers
  replacement of the same command. A third target during the stop updates the pending target and does not restart the
  transaction (`bStopTxBegun` stays TRUE while `TCS_command` stays `stop`). That is the "latest target wins, no queue" rule the
  IAG plan recommends.
- **Do port** the per-command timeout restart (IAG #40) for G2, and the timer restart on replacement for G3.

### 2.4 Inferred, not confirmed

- That `Axis_Status.StandStill` is FALSE and `Homing` is TRUE during an `MC_Home` sweep (PLCopen state model, Tc2_MC2). G1 relies
  on it for stage 10.
- That `MC_ExtSetPointGenEnable.Busy` is short, so `bBusy` is FALSE while the axes track. If not, `Park()` during tracking (watchdog
  at night) would wait in the park branch for good. This is not part of this fix, but it is checked on site (A5).
- That the 3 settle scans are enough when a replacement arrives in the scan in which the old move was just requested. The
  pending `MoveTo` is cancelled, but if `MC_MoveAbsolute` had already started in that axis cycle, the axis only shows motion one
  scan later. The transaction stays active until 3 scans at rest, so it should still issue the stop. Not measured.
- Why 0.8.5 accepted `MC_Stop` and still ran to the target (#84). If that comes back on 0.9.x, it shows up in step A1.
- The G2 log lines on site: from the code only. 0.9.1 has not been run with a preemption.

## 3. Where the fix belongs, and the release chain

Everything is in **MONETcommon** (FBMTC plus one pure function). BROTLib and HalfBROT do not change.
1. MONETcommon `develop` -> **0.9.2** (`ProjectVersion`, `Global_Version`). The BROTLib 0.9.0 / HalfBROT pins stay as they are.
2. MONETS: `MONETcommon, 0.9.1 (IAG)` -> `0.9.2` in `MONETSRuntime.plcproj`, `ProjectVersion` 1.4.2 -> 1.4.3, install with
   `Update-Libraries.ps1`, activate on MONET/S.
3. MONETN: the same pin bump, 0.4.2 -> 0.4.3. No separate on-site check planned (same code path).
4. IAG50cm: not affected (no MONETcommon). #101 stays an IAG issue.

**Order:** run the #96 on-site checks and step A of section 7 on **0.9.1 first**. Step A shows whether the main case of #56 already
works and confirms G1/G2 before the code changes.

## 4. Options and recommendation

**Recommended: A, close the reset gap through the existing abort stop, and fix the two timers.**

- **A (recommended).** On a reset, an axis that is still moving is stopped by the same path a preemption uses: keep
  `bMoveAbortPending` TRUE, so the `#96` transaction sends one `MC_Stop` per moving axis. Two exceptions:
  - a park is running: its motion already goes where the kept `bPark` wants;
  - there is an error: the drives are already off.

  For each axis this ends in a `StopAxis` edge -> `FB_AxisControl` setter -> `fbAxis.Stop()`, which is the "Stop per axis"
  that #66 asks for, without `bStop`. Other parts of A:
  - restart the command timeout on every `TCS_command` change (G2);
  - restart the slew/goto/home timeout on a replacement (G3).
- **B, rejected: call `fbX.fbAxis.Stop()` (or write `StopAxis`) directly in the `_PowerOn` abort block and the reset block**, as the
  #66 text suggests.
  - Nothing would wait for the stop to settle. A park kept by the reset would start into decelerating axes: the park branch only
    checks `NOT bBusy`, and `bBusy` does not include `MC_Stop`.
  - It bypasses the latched-`StopAxis` re-arm and the `StopDone`/settle guard of #96.
  - It would also only cover the power-on, not goto/slew/gohome.
- **C, rejected: set `bStop` on a reset under motion.**
  - It shows up as an operator stop: 'STOP commencing' is logged.
  - `Init()` refuses with 'stop in progress'.
  - `Park()` (watchdog, over-temperature) is deferred into `bParkAfterStop`.
  - The stop branch clears the kept `bPark` every scan (514). That breaks the rule that a reset keeps the emergency park.
- **D, rejected: copy IAG's "only if the preempted command is the one moving" condition.** See 2.3.
- **E, considered: only verify, no code change.** It covers the main case of #56 if step A passes, but it leaves G1-G3.

### 4.1 Changes (A)

1. **New pure function `F_ResetMoveAbort : BOOL`** (no axis access, tested offline).
   - Inputs: `bError`, `bParkRunning` (`TCS_command = park` from the previous scan), and per axis
     `abMoving[1..3]` (`bEnable AND (InMotion OR InRamp OR Homing)`).
   - Result: `NOT bError AND NOT bParkRunning AND (abMoving[1] OR abMoving[2] OR abMoving[3])`.
   - `NOT StandStill` is not used, on purpose. An axis stuck in Stopping without motion (the #96 hang) must not re-arm the
     abort on every reset. Today a reset gets out of such a hung abort stop by clearing the flag, and that has to stay possible.
2. **Reset block (226):** `bMoveAbortPending := FALSE;` becomes
   `bMoveAbortPending := F_ResetMoveAbort(bError := bError, bParkRunning := TCS_command = E_TCSCommand.park, abMoving := ...);`.
   - `bSupersedeRequest := FALSE`, the flag clearing (#85) and the stage resets stay.
   - In the reset scan, `_PowerOn` aborts as today (bPower FALSE).
   - In the next scan the `bMoveAbortPending` branch dispatches `stop`. `_StopTelescope` clears `MoveAxis`/`HomeAxis`
     (cancel) and the transaction stops each moving axis once.
   - At settle the flag clears, and a kept `bPark` runs.
   - A reset during an abort stop that is still braking keeps it going (it continues the same transaction).
3. **`abMoving` gains `Homing`** (189-191: `... OR fbX.fbAxis.Homing`), so that the stage 10 homing counts as motion for
   `F_ResetMoveAbort`, for D2 (`Restart` after a repeated stop) and for D3 (no axis `Reset()` on a homing axis). Both of those
   are wanted.
4. **Command timeout per command (G2).**
   - New VAR `eTimeoutCommand : E_TCSCommand`.
   - Just before the dispatch (635): `IF TCS_command <> eTimeoutCommand THEN tonCommandTimeout(IN := FALSE); eTimeoutCommand := TCS_command; END_IF`.
   - Each command's timeout then counts from its own dispatch, and the 15 s stop timeout counts from the start of the stop.
5. **Replacement restarts its 600 s timeout (G3).** In the supersede block (463-485), call `tonSlewTimeout(IN := FALSE)`,
   `tonGotoTimeout(IN := FALSE)` or `tonHomeTimeout(IN := FALSE)` according to `eSupersedeKeep`. In the next scan `IN` is the
   flag again, so the timer counts from the new request.
6. **Log:** no new text. A reset under motion now logs 'move interrupted, stopping axes' / 'axes stopped' (`MoveAbortEvent`,
   852-857), not 'STOP commencing'. A separate text is open question 3.
7. **Comments:** update the `bMoveAbortPending` comment (94-95) and the lifecycle comment in the reset block. The 0.8.5
   statement "reset clears it" changes to "a reset keeps it while an axis is moving".

Size: about 30 lines in FBMTC, plus a 15-line function. Risk: low-medium. It changes what a reset does under motion (sections 5 and 9).

**Not included (open questions 4-5):**
- D3 extended to `bMoveAbortPending` (reset an axis stuck in Stopping on an abort stop, not only on `bStop`).
- `SuspendAutomatic` under motion.

## 5. Backward compatibility, ADS watcher lists, log texts

- **Public interface:** unchanged (methods, properties, `I_AltAzTelescope`). MQTT topics and READY_STATE are unchanged. After a
  reset, READY_STATE is -2.0 as before (`MarkAborted`).
- **New behaviour:**
  - A reset while an enabled axis moves (goto, slew, gohome, track, power-on stage 10/30/40) brakes the axes with `MC_Stop`
    instead of letting them run to the old target. The braking uses the NC stop deceleration, as for any stop (risk 1).
  - A reset during a running park does not stop it.
  - A park kept by a reset under motion now starts after the deceleration, not after the old move.
- **Timing:** a slew/goto/gohome replacement now gets the full 600 s from the new request. The 2 min/15 s command timeouts count per
  command.
- **ADS symbols:**
  - New: `eTimeoutCommand`.
  - Changed meaning: `fbStopTx.abMoving[*]` now includes `Homing`.
  - Removed: none.
  - Watcher and recording lists: add `eTimeoutCommand`, and for the on-site steps also `fbX.fbAxis.Homing` and `tonCommandTimeout.ET`.
    This includes the MONET test plan in the IAG repo (`specs/plans/2026-10-12-monet-telescope-test-plan.md`).
- **Log texts:**
  - The texts themselves are unchanged.
  - Lines that disappear: 'A timeout occurred during STOP command!' and 'STOP timeout: an axis is not at standstill' right at the
    start of a stop that follows a long command, and the inherited timeouts of the command after it.
  - Lines that newly appear: 'move interrupted, stopping axes' / 'axes stopped' after a reset under motion.
  - Check pyobs or other log filters that might count timeouts.
- **Persistent data:** none new.
- **Not changed:** BROTLib, HalfBROT, IAG50cm, `F_CommandAcceptance`, the stop transaction FB.

## 6. Tests (MONETcommonTests, TcUnit, CI simulation only, currently 95 tests)

`FB_MonetTelescopeControl` still cannot be instantiated in a test: it takes `REFERENCE TO` the HalfBROT axis FBs (#36). The
arbitration (supersede -> abort stop -> new command), the reset block and the timer changes are therefore **not testable
offline**. They are covered only by section 7. Offline coverage is limited to the pure decision.

**Suite `F_ResetMoveAbort_Tests`** (new, about 7 tests):
- `Moving_Axis_Stops`: Az moving, no error, no park -> TRUE.
- `Homing_Axis_Stops`: El in Homing (stage 10) -> TRUE.
- `All_At_Rest_No_Stop`: no axis moving -> FALSE (a reset at rest changes nothing).
- `Error_No_Stop`: moving but `bError` -> FALSE (drives off, emergency park not delayed).
- `Park_Running_No_Stop`: moving and `bParkRunning` -> FALSE.
- `Stuck_Stopping_No_Rearm`: an axis in Stopping with no motion flag -> FALSE (reset must still release a hung abort stop).
- `Disabled_Axis_Ignored`: `abMoving` is built from `bEnable AND ...`. Test the helper that builds it if it is a function, or
  document that it is built in the caller.

**Existing suites:** `StopLoop_Sim_Tests` and `FB_MonetStopTransaction_Tests` are unchanged. The abort stop uses the same
transaction, and its one-edge-per-axis and settle behaviour is already covered (`Sim_Transaction_One_Edge_Per_Axis`,
`Sim_Transaction_Latched_StopAxis`). Not added: a test of `tonCommandTimeout`. It would need multi-cycle timing, and the test
pattern here is one call per test (local VARs). The change is 3 lines and is checked on site by the log (step B1).

Add the suite to `MONETcommonTests` `MAIN` and the README table. CI (`tcbuild.yml`) must pass (about 102 tests).

## 7. On-site check, MONET/S

Rules as in the earlier plans: never open the roof, and someone watches the covers. Choose coordinates within the MONET/S axis
limits, with elevation >= 30 deg.

For the preemption steps, the first target must lie **farther away than the braking distance plus a margin**. On IAG50cm the
T9.4 run on 2026-10-13 was not discriminating because of this. The MONET/S braking distance at 10 deg/s azimuth / 5 deg/s
elevation is not measured: do step 0 first. The coordinates for steps A and B are chosen on site from that measurement.

**Record** with an ADS on-change logger at cycle resolution, as for #96. If that is not available, use a sum-read at 20 Hz.
- **`MAIN.TelescopeControl`:** `TCS_command`, `bMoveAbortPending`, `bSupersedeRequest`, `bStop`, `bStopped`, `bReset`,
  `bPark`, `bGoto`, `bSlew`, `bGoHome`, `bTrack`, `bPower`, the `_PowerOn` stage, `fReadyState`, `fbStopTx` (issued/done per
  axis, `bSettled`, `bStuck`, `bForced`), `tonCommandTimeout.ET/Q`, `eTimeoutCommand` (0.9.2 only), `nStopForcedCount`.
- **Per axis (Az/El/Dr):** `bEnable`, `bMoveAxis`, `bStopAxis`, `bHomeAxis`, `fActualPosition`, and from `fbAxis`:
  `StandStill`, `Stopping`, `Homing`, `InMotion`, `InRamp`, `StopCommand.Execute`, `Axis_Stop.Done`, `MoveCommand.Execute`,
  `HomeActive`, `ErrorID`.
- **Also:** MQTT READY_STATE and the log topic.

**Per run, note:**
- the time of the command and the times `TCS_command` changes
- the position at the preemption, the position at rest, and the old target
- the number of `StopCommand.Execute` edges per axis
- the log lines and any axis error

**Step 0, before step A: measure the braking distance (answers question 7).** Without it, A1-A3 and B1 can pass for the wrong
reason: a first target that lies within the braking distance is reached anyway, even if the preemption does nothing.
1. Choose a long slew with elevation >= 30 deg all the way, and at least 60 deg of azimuth travel (for example az 100 -> 175 at
   el 45).
2. About 3 s into the slew (the axes at full speed) send one plain `stop`.
3. Note, per axis, the position at the `stop` command, the position at rest, and the time until `StandStill`. The braking distance
   is the difference. Repeat once in the other direction.
4. Fix the first target of every preemption step so that it lies at least **2 x the measured braking distance** (and not less than
   10 deg) beyond the position where the preempting command is sent. Elevation >= 30 deg, within the axis limits, roof closed.
5. Write the measured distances and the chosen coordinates into the run notes of the steps below. If a step shows the axes
   stopping short of the target, check against this distance that the step discriminated.

**Step A, on 0.9.1 (before this change; it also gives the #56 baseline):**
1. **A1 goto over slew.** Slew to a far target, then 3 s later `goto` (RA/Dec) to a target in another direction.
   - Expect `TCS_command` slew -> stop -> goto, one edge per moving axis, rest short of the slew target, then the goto.
   - If the axes run to the slew target, #56 is not fixed on 0.9.x: stop and analyse before step B.
2. **A2 slew over slew** (IAG50cm#101 analogue). Slew az 175 -> 100 at el 45, and 2-3 s later slew to az 160 / el 50.
   - Expect braking before az 100 and no visit to the first target.
3. **A3 park over goto.** `park=1` 3 s into a long goto. The watchdog and over-temperature parks call the same `Park()`.
   - Expect the stop, then the park.
4. **A4 gohome over track (G2).** Track for at least 1 min, then `gohome=1`.
   - On 0.9.1, expect 'STOP timeout: an axis is not at standstill' (ERROR) and 'A timeout occurred during STOP command!' at
     the start of the stop. If they come, G2 is confirmed.
   - Expect the home move to start after the stop either way.
5. **A5 park over track.** Track, then `park=1`.
   - Expect the park to start within a few scans (2.4: `bBusy` while tracking).
   - Expect no 'move interrupted' line: this path goes through park stage 10, not the stop.
6. **A6 reset during a slew (G1).** `reset=1` 3 s into a far slew.
   - On 0.9.1, expect the axes to run on to the target. That confirms G1.
   - Then `power=1`.

**Step B, on 0.9.2:**
1. **B1:** repeat A1-A4.
   - Expect the same motion as in A, if A passed.
   - In A4, expect **no** timeout lines.
   - Expect `tonCommandTimeout.ET` near 0 at each `TCS_command` change.
2. **B2 reset during a slew.** Repeat A6.
   - Expect `TCS_command` -> stop one scan after the reset, one edge per moving axis, rest short of the target.
   - Expect 'move interrupted, stopping axes' / 'axes stopped', **no** 'STOP commencing', READY_STATE -2.0, no axis error.
   - Then `power=1` -> 1.0.
   - Repeat 3 times. A reset under motion was followed by 0x4260 on 0.8.5; that was during a park, which is not stopped here.
3. **B3 reset during the power-on move home (#66, stage 30/40).**
   - Setup as in #96 step 1: slew to az 140 / el 55, `reset=1`, `power=1`, then `reset=1` about 3 s into the move home.
   - Expect a stop short of az 175 / el 40, READY_STATE -2.0, then `power=1` -> 1.0.
4. **B4 reset during a park.** `park=1`, then `reset=1` about 3 s into the park move.
   - Expect the park to continue to the park position without a stop and without a 'move interrupted' line, and end parked.
   - Watch for 0x4260 as in #84 (not expected to change).
5. **B5 park after a reset under motion** (emergency park timing).
   - `reset=1` 3 s into a far slew, and immediately `park=1`.
   - Expect the stop, then the park, without waiting for the slew target.
6. **B6, only on request (needs uncalibrated axes, Wake&Shake context):** `reset=1` during the stage 10 homing sweep, then
   `power=1`.
   - Expect the homing axes to stop.
   - Watch `HomeActive` and the power-on stage. If the power-on stays at READY_STATE 0.8, that is BROTLib#92, and it
     already applies to `stop` and `park` during homing. Recover with an axis reset at rest.
   - Report the result on BROTLib#92.

Comment the results on #56, #66 and #84.
- Close #56 when A1-A3 (or B1) pass and B2/B5 pass.
- Close #66 when B3 passes. Its part 2 is already done.

## 8. Not verified

- Nothing in this plan has been compiled or run, and no preemption has run on MONET hardware with 0.9.x.
- That the main case of #56 works on 0.9.1. It is read from the code, and step A checks it.
- The IAG50cm#101 comparison: the candidate causes are absent in MONET, but #101's cause is unknown.
- Tc2_MC2 details: StandStill/Homing during `MC_Home`, `MC_Stop` allowed in Homing (`StopAllowed` includes it, `FB_Axis`
  256-260), and how long `MC_ExtSetPointGenEnable.Busy` lasts.
- G2 is read from the code with standard TON semantics: `ET` counts from the rising edge of `IN`, and a new `PT` applies at once.
  It has not been observed on site.
- The NC stop deceleration on MONET/S from slew speed (HalfBROT#77).

## 9. Risks

1. **Braking on reset.** A reset under motion now ends in `MC_Stop` with the NC stop deceleration. On 0.8.4/0.8.5, stops and
   reversals with high elevation torque gave 0x4550/0x4260 (HalfBROT#77). The risk is the same as for an explicit stop. The
   moving axes are not reset, per the #84 finding. Mitigation: B2 three times, abort on the first axis error.
2. **Stop during homing (BROTLib#92).** A reset in stage 10 now stops the homing. If `HomeActive` sticks, the next power-on stays at
   0.8. That is already true for `stop` and `park` during homing. Open question 2.
3. **Reset no longer always clears `bMoveAbortPending`.** It stays TRUE while an axis really moves. A hung abort stop with no
   motion is still released by a reset (`Stuck_Stopping_No_Rearm`). A hung stop on an axis that reports motion forever would
   need the D1 timeout (logged, `bStuck`), as for `bStop`.
4. **Timer change.** Timeouts that fired spuriously no longer fire. A command that really hangs gets its full timeout from its
   own start. This is intended, but it changes which log lines operators see.
5. **Arbitration not tested offline** (#36). All integration evidence comes from section 7.

## 10. Open questions for the user

1. **Reset during a running park:** keep the park running (recommended), or stop it and restart it from stage 0?
2. **Reset during the stage 10 homing:** stop the homing axes, as your constraint says and as `stop`/`park` already do, and accept
   the unverified BROTLib#92 risk? Or let a homing finish on reset until BROTLib#92 is fixed?
3. **Log text:** reuse 'move interrupted, stopping axes' for the reset stop (recommended, no new text for log filters), or add a
   separate line such as 'reset: axes moving, stopping them'?
4. **G2/G3 (timers):** fix them in this release (recommended, because they directly affect the abort stop: false ERROR and
   possible forced settle), or file a separate issue?
5. **D3 for abort stops:** should `reset=1` also reset axes stuck in Stopping when only `bMoveAbortPending` is set (today D3
   requires `bStop`)? It is not needed for #56.
6. **`SuspendAutomatic` under motion:** a non-goal here, as in 0.8.5. Should it get its own issue?
7. **Coordinates for step A/B** at MONET/S, and whether to measure the stopping distance first with one explicit `stop` from slew speed.
8. **Rescope #56:** comment on #56 that its main case is handled in code since 0.8.5/0.8.6/0.9.1 and is pending step A, and that
   the reset case is tracked with #66 part 1?

## 11. Answers of the user (2026-10-21)

1. Reset during a running park: **keep the park running** (as recommended).
2. Reset during the stage 10 homing: **stop the homing axes**, accepting the unverified BROTLib#92 risk. B6 stays "only on request".
3. Log text: **reuse** 'move interrupted, stopping axes' / 'axes stopped'.
4. G2/G3: **fix them in this release**.
5. D3 for abort stops: **yes**. `reset=1` also resets axes stuck in Stopping when only `bMoveAbortPending` is set (same rule as for
   `bStop`: only if no enabled axis is moving).
6. `SuspendAutomatic` under motion: **separate issue** (filed, see the PR).
7. Coordinates for step A/B: not answered (asked what it means). Default: measure the stopping distance first with one explicit
   `stop` from slew speed, then choose the first target at least that distance plus a margin away, elevation >= 30 deg, to be
   agreed on site.
8. Rescope #56: **yes**, comment on #56; the reset case is tracked with #66 part 1.
