# Design docs

Living design docs for MONETcommon, one per topic, updated in place as the code changes; kept after landing, not deleted.

- [monetcommon-safe-states.md](monetcommon-safe-states.md): the safe state per error class (trigger, what
  `FB_MonetTelescopeControl` does today with file:line, the D4 target, reset path, test). **final for step 7** (#24 aim 6,
  step 0 of [2026-11-02-reference-implementation-issue-24.md](../plans/2026-11-02-reference-implementation-issue-24.md);
  ramp rule D8 and roof rule D9 decided 2026-11-02, step 7 PR order in the doc; PR 1, the `bCloseRoofRequest`
  signal, written on `feat/close-roof-request`).
