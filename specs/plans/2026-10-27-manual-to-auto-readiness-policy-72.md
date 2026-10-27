# Manual -> automatic readiness recovery policy for MONET/N and MONET/S (#72)

Date: 2026-10-27. Status: **steps 1-3, 5 and 9 implemented in a PR (policy: always Init, dedicated latch, Q1 as recommended; step 6 unchanged = Q2 A). Steps 4 (dispatch defence), 7 (pose check, Q4) and 8 (manual telemetry, Q5) are not done. Nothing compiled locally, run on a PLC or run on a telescope; hardware H0-H8 not run.** Redesign: #168.
Repo: BROTLib/MONETcommon. Base: `develop` = `origin/develop` = `648f2dc` (0.10.0). Line numbers below are from `648f2dc`.
Background: #71 (readiness pending through startup), MONETN#18 / MONETS#22 (MAIN calls `SuspendAutomatic()`), #118 (switch to
manual under motion, plan `2026-10-21-suspend-automatic-stops-axes-118.md`, decisions in its section 11), #86/#113 (error park,
Wake&Shake exemption), #158 (`bIsParked` = secured).

Main file: `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU` (FBMTC). Also read: `F_CommandAcceptance`, `ST_CommandState`,
`E_CommandRejectReason`, `F_SuspendStop`, `F_ErrorParkRequest`, `F_WakeShakeErrorExempt`, BROTLib `FB_StartupControl` /
`FB_StartupReadiness` (`e6c0858`, 0.14.0), MONETS `MAIN` (`ffdf845`), MONETN `MAIN` (`1b97697`), `MONETcommonTests` (`FB_MonetRig`,
`MonetSuspendUnderMotion_Tests`). Everything in section 3 is **read from the code only**.

## 1. Goal and non-goals

Goal: an agreed, documented and tested operator/API behaviour for automatic -> manual -> automatic on both sites, with no path
on which the telescope moves after the return unless the operator has sent a command that is allowed in that state.

Non-goals: the stop at the switch (#118, done); pendant/control interlocking (#3); the operational interlock that gates ONLINE
without erasing a completed startup (#71: must not change); physical abort work (#66); firmware deployment or telescope movement.

## 2. Policy (proposed, as agreed in the conversation)

- **After any manual interval a fresh, explicit Init (power-on) is always required.** No silent recovery, no resumed startup,
  no replayed automatic move. Applies equally to `manual` and `off` (both call `SuspendAutomatic`, MONETS MAIN 69-72/120-122,
  MONETN 68-71/131-133).
- Why: Init is already light. Stage 10 homes only axes that are **not** calibrated (FBMTC 1219-1239), so after a short manual
  interval it re-enables the drives, re-opens covers/brake and drives the axes to HOME. It re-establishes enable, brake, covers and
  calibration state that the controller cannot verify from position tracking alone (the pendant can disable axes, the
  clear-brakes key can drop the brake and `MC_Power`, the operator may close covers or stop the hydraulics).
- Power-on ends at **HOME** (stage 30, `telescopeConfig.*Home`, 1246-1255), not at the park position. Park is a separate
  "secured" command; `bIsParked` means secured, not "at park position" (#158, FBMTC 137-142).
- `bPower := FALSE` at the switch is **not** a fix and needs no change: `bPower` only means "power-on sequence running" and is
  already cleared on completion (1278) and on suspend (1303, and in `_PowerOn` 1181). Public readiness is `fbStartup.Initialized`.

**Operator / API behaviour after the change**

| Situation | What the operator sees | What the operator must do |
|---|---|---|
| In manual | READY_STATE -2.0 (0.0 if secured, -1.0 on error), heartbeat 1 s; no other telescope telemetry | nothing; moves sent now are rejected |
| Back in automatic, before Init | READY_STATE stays -2.0 / 0.0; goto/slew/track/home rejected with WARN `... rejected: init required after manual mode`; nothing moves | send `power=1` (Init) |
| Init after manual | 0.7 -> 0.8 -> 1.0; covers open, drives on, uncalibrated axes re-home, **the telescope homes** (drives to HOME) | wait for 1.0 |
| Always accepted | stop, park, reset, Init | - |

## 3. Findings (read from the code)

**On the agreed facts:**
- Correct: Init homes only uncalibrated axes (1219-1239); power-on ends at HOME (1246-1285); `bPower` cleared at 1278/1303.
- Correction: `fbStartup.EnterSuspend()` only latches `paused` and returns TRUE on the first manual scan
  (FB_StartupControl 110-113). The completed-startup latch is cleared by `Suspend()`, which runs `FB_StartupReadiness` with
  `Aborted := TRUE` in every manual scan (118-129), and `AcceptInit()` refuses while `paused` (142). `BeginScan` clears `paused` (38).
  On return `Update(...)` keeps `Initialized` FALSE, so READY_STATE is -2.0 (0.0 if `bIsParked`) until a new startup completes.

**Reported gaps:**

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| G1 | Moves gated on mechanical `bReady`, not `Initialized` | **Confirmed** | `_CommandState.bReady := bReady` (2442), `F_CommandAcceptance` row `not_ready` (54) is the only readiness row; dispatch also on `bReady` (640, 657, 665, 670); `bReady` = calibrated + covers open + enabled + brake open + no error (780-785). After return with drives still enabled a goto is accepted and runs without Init. Worse during manual: `bReady` is not recomputed (no assignment in `SuspendAutomatic` 1298-1403), so a move sent in manual is accepted on the stale value and stays queued; the first-scan drop (1310-1314) does not catch it. |
| G2 | Pending `bPark` survives manual and restarts by itself | **Confirmed, but decided** | `SuspendAutomatic` keeps `bPark` on purpose (1294, 1319; #118 answer 2; test `Suspend_During_Park_Stops_The_Park_And_Keeps_bPark`). `Park()` during manual (MQTT watchdog, cabinet over-temperature) also sets it (2286-2290). On return the park branch dispatches at standstill (599-625); `F_ParkAxesCanMove` (1061-1064) drives to the park position from wherever the pendant left the axes if they are calibrated/enabled/brake open, else parks in place. `Init()` is refused while it is pending (2208-2216). |
| G3 | Horizon guard sets `bGoHome` before Init | **Confirmed (conditional)** | 464-470: `fbElevation.Calibrated AND fElevationCurrent < fHorizon + 0.2` sets `bStop`, then `bGoHome`. Set inside the body, so the direct-write gate never checks it (841-846, 284-288). Dispatch needs `bReady AND bStopped` (640): with drives enabled the telescope homes without Init. Whether the pendant can place el that low is unverifiable from code (NC soft limits). |
| G4 | Stage 30 drives all three axes to HOME at once, no path/limit/wrap check | **Confirmed for "no pose check"; wrap partly refuted** | 1246-1255 writes all three `MoveAxis` in one scan. Same pattern in `_HomeTelescope` (989-998) and park stage 20 (1086-1095), so not new. `MC_MoveAbsolute` moves along the axis coordinate, so it does not cross a wrap limit if the start pose is inside `fMinPosition/fMaxPosition`; nothing checks that start pose. New after manual: the start pose is arbitrary (normally Init starts from the park pose). |
| G5 | Stage 0 opens covers unconditionally | **Confirmed, by design** | 1201-1207 and the `CoverAutoOpen` setter (2182-2186): Wake&Shake needs open covers. Not specific to manual; no change proposed (Q3). |
| G6 | `bFirstPowerOnPending` FALSE after manual, so an Az/El error in re-Init parks | **Confirmed (code); relevance unverifiable** | Cleared only at 1277, never set again; exemption 540-543 needs it. Error park is the safe direction. Whether the Wake&Shake fault recurs after manual (e.g. `off` mode, hydraulics stopped) needs hardware (H6). |
| G7 | `bHwErrorLast := TRUE` hides an error from manual | **Partly refuted** | 1301 only suppresses the #86 emergency park (by design, test `Back_To_Automatic_With_An_Error_Does_Not_Park`). The error itself is recomputed on return (276-280): READY_STATE -1.0, moves rejected (`error`), Init fails (`_PowerOn` 1171-1174). |

**New findings:**
- F8: In manual `bError` and `bEstopTriggered` are stale. `SuspendAutomatic` passes them to `Suspend()` (1397) but never recomputes
  `bError`, and MAIN passes `bEstopTriggered` only in the body call (MONETS MAIN 115, MONETN 126). So an axis error or E-stop that
  appears in manual is not in the manual READY_STATE heartbeat. Relevant to the issue's telemetry item.
- F9: rejection and Init-refused log lines are emitted only by `_LogEvents` (1774-1792), i.e. in automatic. A move or Init sent in
  manual is logged only after the return. Init in manual is refused with 'automatic mode off' (2236-2238), not queued.
- F10: a reset in manual does not reach FBMTC (MAIN `Reset()` only in automatic, MONETS 94-96, MONETN 93-95).
- F11: a reset does **not** clear a pending park (`_HandleReset` keeps `bPark`, 1832-1833); only stop does (563).

## 4. Code changes (MONETcommon only; ordered)

Choice of gate: gating on `fbStartup.Initialized` directly would also require Init after every reset, aborted startup and E-stop
(all clear `Initialized`: `_HandleReset` 1827, `MarkAborted*`, `AbortExtra`), and would make the existing reset/stop tests fail.
That is broader than #72. **Recommended: a dedicated latch** (Q1).

1. **Latch.** New VAR `bInitRequired : BOOL` (not persistent; after a PLC restart the axes are uncalibrated anyway) and
   `InitRequiredEvent : FB_EventLog`. Set TRUE in the `EnterSuspend()` branch (1300). Cleared only in `_PowerOn` stage 100 next to
   `MarkCompleted()` (1276). Not cleared by reset, stop, park, E-stop or an aborted startup. Read-only property `InitRequired` for ADS/Visu.
2. **Acceptance.** `ST_CommandState.bInitRequired` (filled in `_CommandState`, 2435-2449); new enum value
   `E_CommandRejectReason.init_required` **appended at the end** (keeps existing numeric values); new row in `F_CommandAcceptance`
   after `parked` (51) and before `not_ready` (54), so estop/error/clock/stop/powering/parking/parked keep their reasons. Park returns
   early (21-29) and stays never-rejected. Covers the methods (`Slew`, `MoveAltAz`, `MoveRaDec`, `Track`, `TrackRaDec`, `Home`) and the
   direct-write gate (284-303). Because the latch is set in the first manual scan, moves sent **during** manual are rejected too
   (fixes G1 including the stale-`bReady` case). Reject text via `F_CommandRejectText`: 'init required after manual mode'.
3. **Horizon guard (G3).** 467: `ELSIF NOT bHomeBelowHorizon AND NOT bInitRequired THEN bGoHome := TRUE;`. The stop part stays
   (stopping is always safe). Init then brings the telescope to HOME anyway.
4. **Dispatch defence.** In the `bGoHome`/`bGoto`/`bSlew`/`bTrack` branches (627-672) require `NOT bInitRequired`; a flag that is
   still set is dropped with the reject log. Only internal setter today is the horizon guard (grep of `:= TRUE`), so this is a guard
   against future setters; drop it if review prefers one gate.
5. **Return log.** On the first automatic scan after manual (`BeginScan` after `paused`; detect with a member `bWasSuspended`) log
   HINT 'automatic mode: init required (manual mode since last startup)'; if `bPark` is pending, WARN 'automatic mode: pending park
   resumes' (Q2).
6. **Pending park (G2)** - per Q2. Recommended: unchanged (#118 decision), plus the WARN of step 5.
7. **Pose check before stage 30 (G4)** - per Q4. Recommended minimum: pure function `F_PowerOnPoseCheck` (each axis inside
   `fMinPosition..fMaxPosition`, el above `fHorizon`), evaluated when stage 10 -> 30; on failure `bPowerFailure`, message
   'power-on: axis outside its limits, move it in manual'. Sequencing (el first) only if the operators ask for it.
8. **Manual telemetry (F8)** - per Q5. Recommended: compute the error predicate (276-280, no axis writes) in `SuspendAutomatic`
   before `Suspend()`; E-stop needs a MAIN line in both apps (`TelescopeControl.bEstopTriggered := SafetyHandling.estop;` before
   `SuspendAutomatic()`), so it goes with the pin PRs.
9. **Comments/docs in code:** `SuspendAutomatic` header (1291-1295), `_CommandState`, `F_CommandAcceptance` header, `bInitRequired`.

Unchanged on purpose: `bFirstPowerOnPending` (G6, Q6), cover opening (G5, Q3), `bHwErrorLast := TRUE` (G7), interlock handling (#71),
`Init()` precedence (stop/park refuse Init, 2208-2216), BROTLib `FB_StartupControl`.

**Exceptions (must stay accepted with `bInitRequired` TRUE):** `Stop()`, `Park()` (including the 12 h park 482-484 and the #86
error park), `Reset()`, `Init()` (via `AcceptInit`), offsets (`Set*Offset`), focus commands (own mode input).

## 5. Tests (offline, MONETcommonTests, TcUnit on the user-mode runtime)

Location: `MONETcommonTests/MONETcommonTests/MONETcommonTests/POUs/`; suites declared in `MAIN.TcPOU`; table in
`MONETcommonTests/README.md`. The rig `FB_MonetRig` runs the real FBMTC on mock axes and already has `SetManual`, `SetEstop`,
`JogElevation`, `Inject*Error`. **Rig additions:** `DecalibrateAxis(n)` (mock core `bCalibrated := FALSE`), `DisableAxis(n)`
(pendant enable released), `SetElevationPosition(f)` (for the horizon case); per-axis home/move counters if not present.
Check the TcUnit capacity first: the README says 32 suites, `MAIN` already declares 38 (parameter not found in the repo).

`F_CommandAcceptance_Tests` (pure): `init_required` rejects slew/goto/track/gohome; park not rejected; estop/error/stopping/powering/
parking/parked keep precedence over it; `bInitRequired` FALSE changes nothing (regression for all existing rows).

New suite `MonetManualReturn_Tests` (each test: clock valid, Init, online unless stated):

| Id | Scenario (issue item) | Expect |
|---|---|---|
| R1 | completed startup -> manual at rest -> automatic | READY -2.0, `bInitRequired`, no MoveAxis/HomeAxis edge for 200 scans |
| R2 | R1 then Slew/MoveRaDec/Track/Home and a direct `bGoto := TRUE` (**move without Init**) | each rejected `init_required`, targets unchanged, no motion |
| R3 | Slew sent **during** manual, then return | rejected, nothing queued, no motion after return |
| R4 | R1 then Init | 0.7 -> 0.8 -> 1.0; no homing on calibrated axes; one move to HOME per axis; latch FALSE; Slew then accepted |
| R5 | **interrupted startup**: manual in stage 10, and separately in stage 40 | on return no resume (bPower FALSE, stage 0, no edges); Init restarts from stage 0; R5 vs R1 differ in home counts |
| R6 | **manual movement**: `JogElevation` in manual, return, Init | stage 30 starts from the jog pose, ends at HOME, 1.0 |
| R7 | **loss of calibration**: `DecalibrateAxis(az)` in manual, return | moves rejected; Init re-homes az only, then 1.0 |
| R8 | **E-stop/reset**: E-stop in manual, release, return | READY -1.0 while pressed after return; `reset=1`; latch still TRUE; moves rejected; Init -> 1.0 |
| R9 | reset after return, no Init | latch stays TRUE, moves still rejected |
| R10 | **park**: park running at switch (existing test keeps `bPark`), return | per Q2 (recommended: park resumes, ends secured, Init refused 'park in progress' meanwhile); also `Park()` sent during manual |
| R11 | **repeated transitions**: 3 x automatic/manual without Init, then Init; and manual again during that Init | latch set after each; every startup completes only after its own Init |
| R12 | horizon guard: el below `fHorizon + 0.2` in manual, return | no `bGoHome`, no motion; Init homes |
| R13 | exceptions with latch TRUE: stop, park, reset, Init | all accepted |
| R14 | interlock regression (#71): transient Operational loss without manual | `Initialized` kept, latch FALSE, back to 1.0 without Init |
| R15 | G6 characterization: Az error during the Init after manual | emergency park (no exemption), READY -1.0 |

Unchanged and must stay green: `MonetSuspendUnderMotion_Tests`, `MonetResetUnderMotion_Tests`, `MonetParkDuringPowerOn_Tests`,
`MonetErrorPark_Tests`, `MonetTimeouts_Tests`, `F_SuspendStop_Tests`. **Running tests needs the user's go-ahead** (this machine is the
CI runner; `tctool`/XAE not started by this plan).

## 6. Documentation

- `README.md` "Startup readiness" (147-174): replace "pending that decision" with the policy, the operator table of section 2, the
  new reject reason, and "telemetry other than READY_STATE pauses in manual" (plus F8 outcome).
- `MONETcommonTests/README.md`: new suite row, rig methods.
- `specs/plans/index.md`: entry for this plan. Command-acceptance plan (`2026-10-20-command-acceptance-83-87.md` 2.2 table): new row.
  #118 plan 2.2: note that moves in manual are now rejected.
- IAG repo `specs/plans/2026-10-12-monet-telescope-test-plan.md`: operator procedure "after manual: power=1 before any move".
- pyobs-brot (#91): new reject text; READY_STATE -2.0 after manual until Init; confirm which manual-mode telemetry it needs (Q5).
- Comment the findings (G1-G7, F8-F11) and the decision on #72.

## 7. Open questions for the user / operators

1. **Gate:** dedicated `bInitRequired` latch (recommended, only manual) or gate on `fbStartup.Initialized` (also after reset,
   aborted startup, E-stop)?
2. **Pending park at the return:** (A) keep, it resumes and drives to park from the pendant pose (#118 decision; recommended + WARN),
   (B) drop `bPark` at the switch and log, (C) keep but always park in place after manual (no axis motion).
3. **Covers:** keep stage 0 opening them unconditionally after manual (recommended, Wake&Shake), or honour `bCoverAutoOpen` / roof state?
4. **Pose before stage 30:** none (today), limit/horizon check (recommended), or sequenced move (el first)? Operators: is there a
   collision/cable risk moving all three axes at once from an arbitrary pendant pose?
5. **Manual telemetry:** recompute the error predicate in manual and wire E-stop in MAIN (recommended), and which other telemetry
   (position, cabinet) pyobs needs in manual?
6. **Wake&Shake after manual:** keep the error park on an Az/El fault in the re-Init (recommended until H6 shows the fault recurs)?
7. **Init during manual:** keep refusing (recommended) or remember it and start on return (contradicts "explicit Init")?

## 8. Hardware acceptance (separately for MONET/S and MONET/N)

**Requires the user's explicit go-ahead for each site and session. This plan authorizes no firmware deployment and no telescope
movement.** Do not infer N from S: cabinet keys, brakes, home/park configuration and NC limits differ per site (MAIN `telescopeConfig`).
Rules as in the #118 plan: roof closed, someone at the cabinet key, one at the pendant, one at the recorder; el >= 30 deg; abort on the
first axis error. Record `fReadyState`, `bInitRequired`, `bPower`, `_PowerOn` stage, `bPark`, `bGoHome/bGoto/bSlew/bTrack`, `bReady`,
`bError`, per-axis Enable/Calibrated/MoveAxis/HomeAxis/position, `eTelescopeMode`, `brakeClearing`, MQTT READY_STATE and log topic.

- H0 (installed firmware, no motion): parked, key local and back; note READY_STATE, Init refusal text when sent in manual.
- H1 completed startup -> manual at rest -> automatic: moves rejected 'init required', nothing moves; `power=1` homes; 1.0.
- H2 pendant jog of az and el by a few degrees in manual -> automatic -> `power=1`: no re-homing, all axes to HOME, watch the path (Q4).
- H3 key to local during stage 10 and during stage 40 of a power-on: no resume on return; `power=1` restarts.
- H4 calibration loss (only with an operator-approved method, e.g. `off` mode with hydraulics stopped): Init re-homes that axis.
- H5 E-stop pressed in manual, released, `reset=1`, `power=1`.
- H6 `off` mode (hydraulics off) then automatic and `power=1`: does the Wake&Shake fault recur (G6)?
- H7 park running at the switch, return: behaviour per Q2.
- H8 three automatic/manual cycles in a row, then `power=1`.
Comment results per site on #72; close only when both sites pass H1, H2, H3, H5, H7, H8.

## 9. Risks and rollout

Risks: (1) a client that sent goto right after the return now gets a rejection (intended; pyobs must send `power=1`); (2) the latch is
invisible state - mitigated by the property, logs and README; (3) G4 motion from an arbitrary pose remains until Q4 is answered;
(4) a stale `bInitRequired` would block observing - it is cleared only by a completed startup, which is always reachable from automatic;
(5) offline mocks do not model brakes/pendant, so hardware steps carry the real evidence.

Order: answer Q1-Q7 -> MONETcommon PR (steps 1-5, 9 and the chosen 6-8) with tests -> CI (with go-ahead) -> release 0.11.0 (new enum
value) -> MONETS/MONETN pin PRs (plus MAIN E-stop line if Q5), merged but **not activated** -> H0-H8 on MONET/S, then MONET/N, each
with go-ahead -> pyobs-brot update before activation -> docs and close #72.
