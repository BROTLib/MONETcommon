# Plans

Dated `YYYY-MM-DD-<slug>.md` investigation and work logs, one per unit of work.

- [2026-09-20-code-review.md](2026-09-20-code-review.md): full code review of MONETcommon
  (bugs, security, design, CI). **draft**, review finished; #4, #5 and part of #9 fixed on `develop`. Nothing run on a PLC or a telescope.
- [2026-09-22-ut1-utc-correction.md](2026-09-22-ut1-utc-correction.md): wire UT1-UTC (`dut1`)
  correction into MONET pointing via the existing MQTT weather-bridge pattern. **Planned, not
  started.**
- [2026-09-22-add-tcunit-tests.md](2026-09-22-add-tcunit-tests.md): add a TcUnit test suite,
  mirroring AstroBROT's setup. **Planned, not started** — gated on a spike to confirm
  MONETcommon's FBs can run in a TwinCAT test target at all.
- [2026-10-20-slew-reset-fixes.md](2026-10-20-slew-reset-fixes.md): 0.8.5 fixes for #84 (stop/gohome/track
  do not interrupt a slew), #77, #85 and #88 (reset). **Implemented in code (0.8.5), not yet compiled or tested on hardware; unit-test extraction deferred.** Step 0 on 0.8.4 showed
  `bStopAxis` latched TRUE, so `_StopTelescope` needs an edge re-arm.
- [2026-10-20-command-acceptance-83-87.md](2026-10-20-command-acceptance-83-87.md): 0.8.6 plan for #83 (slew/goto/track
  that cannot run is rejected with a log, never queued) and #87 (park wins over a power-on, park in place when the axes
  cannot move). Builds on 0.8.5's `bMoveAbortPending`. **implemented in code (PR #95), not compiled or hardware-tested**; policy decisions D1-D7 decided.
- [2026-10-20-target-horizon-warning-94.md](2026-10-20-target-horizon-warning-94.md): #94, 'Target below safe
  horizon' only for a requested goto/track, evaluated before the dispatch can clear `bGoto`.
  **Merged to develop (PR #97), not released; XAE compile and on-site check pending.**
- [2026-10-20-wrap-limits-2.md](2026-10-20-wrap-limits-2.md): #2, azimuth/derotator wrap from the per-site axis
  limits instead of MONETN literals; refuse instead of clamp. **Planned, not started**, four open questions.
- [2026-10-20-error-park-86.md](2026-10-20-error-park-86.md): #86, a new axis/focus error on a telescope that is not
  parked starts the emergency park by itself (`F_ErrorParkRequest`). **Implemented in code, not compiled or hardware-tested.**
- [2026-10-20-unpark-on-poweron-113.md](2026-10-20-unpark-on-poweron-113.md): #113, `bIsParked` drops with the cover command of
  a power-on so a park during it closes the covers; Wake&Shake exemption from the #86 park. **Implemented in code, not compiled or hardware-tested.**
- [2026-10-20-stop-during-poweron-hang-96.md](2026-10-20-stop-during-poweron-hang-96.md): #96 (and the 0.9.0 part of #84), stop never completes because `_StopTelescope` restarts `MC_Stop` after every `StopDone`; one stop per axis and transaction (port of IAG50cm#108), 0.9.1. **Implemented in a PR, not run on a PLC or telescope.**
- [2026-10-21-preempted-move-keeps-moving-56.md](2026-10-21-preempted-move-keeps-moving-56.md): #56 and #66 part 1. The main case (a preempted goto/slew/home keeps moving) has been handled in the code since 0.8.5/0.8.6/0.9.1 but never passed on hardware. Still open: a reset under motion does not stop the axes (also power-on stage 10/30/40), and the command timeout carries over into the abort stop (false 'STOP timeout'). Proposed for 0.9.2. **Implemented in a PR, not run on a PLC or telescope.**
- [2026-10-21-park-brake-121.md](2026-10-21-park-brake-121.md): #121, the Az/El brake open with the elevation disabled. The warning is cosmetic on every disable, the brake close is edge-only and `bOpenBrake` is a latch; who opened it in step 4 is not identified (H1-H6, Influx read-back before 2026-10-24). Fix: level-triggered close in automatic (MONETcommon 0.9.3), elevation `bReady` from the enable and a debounced warning (HalfBROT 0.5.7). **Plan, not started.**
- [2026-10-25-wake-shake-window-151.md](2026-10-25-wake-shake-window-151.md): #151, the brake cannot stay open for the Wake&Shake after a PLC restart because #121 closes it while the elevation is disabled. A latched, bounded Wake&Shake window (set by the #113 exemption, 15 min, cleared by park/covers/power-on/E-stop) that `F_BrakeMustClose` respects; the same window is the hook for the automatic stage of #76. **Implemented in a PR, not compiled or run on a PLC or telescope.**
- [2026-10-21-suspend-automatic-stops-axes-118.md](2026-10-21-suspend-automatic-stops-axes-118.md): #118, the switch to manual (`SuspendAutomatic`) leaves moving axes running to their old target. Proposed for 0.9.3: one stop per moving axis through the abort stop, driven by `SuspendAutomatic` until settled, then released to the pendant; queued moves dropped. Pendant handover, brake key and return to automatic analysed from the code only. **Implemented in a PR, not run on a PLC or telescope.**
- [2026-10-26-biparked-after-axis-error-park.md](2026-10-26-biparked-after-axis-error-park.md): #158 (from a #122 note, #86, #113), `bIsParked` TRUE at el 40 after an emergency park. Not a stale flag: the park-in-place branch completes and sets it, so `bIsParked` means "secured", not "at the park position". Decided: option B, keep the flag and add `bParkedInPlace` with a log line and `diagnostics/base/ParkedInPlace`; READY_STATE unchanged, MONETcommon only, FALSE after a restart. **Implemented in a PR, not run on a PLC or telescope.**
- [2026-10-27-brake-open-with-latched-error.md](2026-10-27-brake-open-with-latched-error.md): #155, the Visu cannot open the brake while a power-on elevation error is latched (`reset=1` needed first). Cause: the level `CloseBrake()` in the `fbElevation.bError` block runs every scan and ignores the #151 window. Fix (option A): close once on the error edge, and `F_BrakeMustClose` takes the level blocks E-stop, hydraulics error, safety error, covers not open inside the window; new input `bSafetyError` (to be wired in MONETS/MONETN `MAIN`). **Implemented in a PR, not run on a PLC or telescope.**
