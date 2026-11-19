# Safety start-up/reset sequence: make it testable (MONETcommon #61, second half)

**Status: proposed, nothing implemented.** Written 2026-11-19 from `FB_MonetSafetyHandling` on `develop` (MONETcommon 0.28.1). Not compiled, not run.

## Why

The sequence of #6 (connections up, then ErrAck, Restart, STO reset, check) is checked on MONET/N for the normal start only (2026-11-19, cold reset logged over ADS: order ErrAck, Restart, STO reset, about 3 s, no failure). The other cases need a person at the E-stop or a cabinet power cycle. TcUnit cannot drive the block because its inputs are `AT%I*` TwinSAFE info data and the `FB_Info*` decoders, so every case in the list below is untested.

## Today

In `FB_MonetSafetyHandling.Implementation`, one scan does: decode the info data into strings (`sStateGroup1`, `sStateEStop1`, `sStateSafetyIn`, `sStateSafetyOut`), derive `state`, `estop`, `bSafetyReady`, `bConnectionsUp`, run the start-up CASE (`initialStartUpStep` 0, 10, 100), the reset edge, the sequence CASE (`nSeqStep` 10, 20, 30, 40) that is the only writer of `outErrAck`, `outRestart` and the three `outReset*STO`, and the logs. Only the decoding and the `AT` mapping depend on hardware.

## Change

New `FB_SafetySequence` (no `AT`, no `FB_Info*`, no I/O):

- Inputs: `sStateGroup`, `sStateEStop`, `sStateSafetyIn`, `sStateSafetyOut` (STRING), `bResetRequest`, `tConnectionWait` (30 s), `tPulse` (1 s), `tCheck` (5 s).
- Outputs: `bErrAck`, `bRestart`, `bResetSTO` (one bit, wrapper fans it out to the three drives), `bConnectionsUp`, `bStartupDone`, `bSafetyReady`, `bEstop`, `bSeqFailed`, `sSeqResult`, plus `nStartupStep` and `nSeqStep` for the diagnostics that read them today (`MAIN.SafetyHandling.initialStartUpStep` and `nSeqStep` stay readable through the wrapper).
- The code of the start-up CASE, the reset edge and the sequence CASE moves over unchanged; the timers become parameters so tests do not wait 30 s.

`FB_MonetSafetyHandling` stays the thin wrapper: `AT` mapping, `FB_Info*` decoding, instance of `FB_SafetySequence`, the outputs copied to `outErrAck`, `outRestart` and `outReset*STO`, the logs, `stItems`, `StoActive`, `RestartPending`, `Items`. Its public names and behaviour do not change (MONETS and MONETN `MAIN` untouched), so this is a patch release.

## TcUnit cases (MONETcommonTests)

1. Normal start: connections up, then pulses in the order ErrAck, Restart, STO reset, then `bStartupDone`, `bSafetyReady`, no failure.
2. Start with the E-stop pressed: sequence runs, check times out, `bSeqFailed` TRUE, `sSeqResult` names the E-stop state, `bEstop` TRUE.
3. Connections never up: sequence starts after `tConnectionWait`.
4. Check ends early when group and E-stop reach RUN (no wait for `tCheck`).
5. Reset request while a sequence runs is ignored; outputs are never set in two steps at once.
6. Reset request before `bStartupDone` is ignored.
7. Repeated failure: a second reset with the E-stop still pressed sets `bSeqFailed` again (the wrapper's log fires again).
8. `bResetSTO` only in step 30.

## Not covered

The decoding of the TwinSAFE info bytes (`FB_Info*`, Beckhoff library), the `AT` mapping, the real pulse lengths against TwinSAFE and the AX5805 (check 7 of #61), the FSoE connection timing after a power failure (check 3). Those stay hardware checks.

## Steps

1. Extract `FB_SafetySequence` and the wrapper change. Verify on MONET/N against the 2026-11-19 log (same order and timing), cold reset.
2. TcUnit cases 1 to 8, CI build.
3. Patch release, pin on MONETN, check once on hardware as in step 1, comment on #61.

## Open questions

- One `bResetSTO` bit or three outputs (the three drives could differ later)? Proposal: one bit, the wrapper copies it.
- Should the sequence also take a remote reset (MQTT) as an input? Today the caller ORs it into `resetRequest`; unchanged here.
