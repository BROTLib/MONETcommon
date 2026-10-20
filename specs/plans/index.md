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
