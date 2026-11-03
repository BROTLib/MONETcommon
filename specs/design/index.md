# Design docs

Living design docs for MONETcommon, one per topic, updated in place as the code changes; kept after landing, not deleted.

- [monetcommon-safe-states.md](monetcommon-safe-states.md): the safe state per error class (trigger, what
  `FB_MonetTelescopeControl` does today with file:line, the D4 target, reset path, test). **final for step 7** (#24 aim 6,
  step 0 of [2026-11-02-reference-implementation-issue-24.md](../plans/2026-11-02-reference-implementation-issue-24.md);
  ramp rule D8 and roof rule D9 decided 2026-11-02, step 7 PR order in the doc; PR 1, the `bCloseRoofRequest`
  signal, written on `feat/close-roof-request`; PR 3, the hydraulics error reaction (row implemented), written on
  `feat/hydraulics-error-reaction`; PR 5, the state-machine timeouts ramp, then park (row implemented), written on
  `feat/timeout-error-reaction`;
  PR 4, the cover error reaction (row implemented: park on the error, the park ends without closed covers with software error 16#21, the power-on fails before the cover command), written on `feat/cover-error-reaction`;
  PR 7, the power failure reaction (row implemented), written on
  `feat/power-failure-reaction`;
  PR 8, the focus error reaction (row implemented), written on
  `feat/focus-error-reaction`;
  PR 6, the E-stop / safety error reaction (rows implemented), written on
  `feat/estop-safety-reaction`).
