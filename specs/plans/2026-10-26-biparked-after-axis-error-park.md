# `bIsParked` TRUE after an emergency park that never reached the park position (#122 note, #86, #113)

Date: 2026-10-26. Status: **implemented in a PR (#158), not compiled here, not run on a PLC or telescope.**
Decisions (Tim, 2026-10-26): D1 option B, D2 keep READY_STATE 0.0, D3 MONETcommon only, Q3 `bParkedInPlace` FALSE after a restart,
Q5 `park=1` on a parked telescope stays the silent jump to stage 100, Q6 new issue #158. Implemented: tests extend `MonetErrorPark_Tests`
(in place after an elevation or focus error, not in place after a normal park); the other tests of section 7 are not written.
Repo: BROTLib/MONETcommon. Base: `develop` = `66e86b3` (0.9.10). Line numbers below are from `FB_MonetTelescopeControl`
(FBMTC) at that commit.

Files read: FBMTC (error park 463-474, arbitration 476-609, `fbStartup.Update` 704-709, `_ParkTelescope` 944-1052,
`_PowerOn` 1068-1191), `F_ErrorParkRequest`, `F_CommandAcceptance`, BROTLib `FB_StartupReadiness`, `MonetErrorPark_Tests`,
`FB_MonetRig`, MONETS/MONETN `MAIN` (telescope config), plans #86, #113, #121, #151, README, IAG test plan
(`IAG/specs/plans/2026-10-12-monet-telescope-test-plan.md`, T8 step 7).
Issues read: #122 (with comments), #113, #86.

No issue tracks this point yet: #122 was closed on 2026-10-26 for the focus part, its last note ("after an emergency park
caused by an axis error, `bIsParked` is TRUE although the telescope never reached the park position") was not addressed.

## 1. Problem (from #122, MONET/S, 2026-10-21, MONETcommon 0.9.2 / BROTLib 0.9.0 / HalfBROT 0.5.6)

Telescope online and idle. The focus drive was disabled in XAE at 75.2 mm: NC error 0x4260, `emergency park started: axis
error`. The telescope stayed at el 39.9 with `bIsParked` TRUE and the error set. The MONET/S park position is el 60 / az 175 /
derotator 0 / focus 36 (`MAIN.TcPOU:42`).

## 2. Why `bIsParked` is TRUE (confirmed from the code)

It is not a stale flag and not an aborted park. The emergency park **completed as designed** and sets `bIsParked` at its end:

1. `F_ErrorParkRequest` (FBMTC 465) sees the rising edge of `bHwError` (the focus counts, line 247), not parked, no E-stop -> `Park()`.
2. Arbitration `ELSIF bPark` with `bError` (530-532): `TCS_command := park`, no `bReady` gate.
3. `_ParkTelescope` stage 0 (974-980): `bError` -> **park in place**: brake closed, the three axes disabled, covers closed,
   stage 80. No axis is moved (only `fbFocus.Position := focusPark` is written, line 967; the focus is in error and cannot move).
4. Stage 80 (1034-1044): brake closed -> hydraulics off; covers closed -> stage 100.
5. Stage 100 (1045-1049): `bIsParked := TRUE`, `bPark := FALSE`.

The telescope was at the power-on home position, `elevationHome := 40` (`MAIN.TcPOU:43`), which explains el 39.9.

The same happens in the other park-in-place branch, `NOT bAxesCanMove` (#87, line 974): a park after a reset or an aborted
power-on (seen in #121 step 4 at el 40).

So in the code `bIsParked` means **"secured"** (covers closed, brake closed, hydraulics off, axes disabled), not "at the park
position". This was intended in #86 (plan section 3, "the emergency park ends with `bIsParked` TRUE, so the persisting error
cannot trigger again") and is pinned by `MonetErrorPark_Tests` (`Elevation_Error_...`, `Persisting_Error_...`), which check
`bIsParked` but no position. The flag's comment at line 957 says "parked means: nothing opened since the last park".

## 3. Who reads `bIsParked`

| Consumer | Line | Needs "secured" or "at park position"? |
|---|---|---|
| `F_ErrorParkRequest` (no second error park) | 465 | secured |
| `_ParkTelescope` jump to stage 100 | 958 | secured (nothing to close) |
| `F_CommandAcceptance` reject `parked` ('send power=1 first') | `F_CommandAcceptance:49` | secured (axes disabled either way) |
| 12 h idle park timer | 401, 409 | secured |
| `F_WakeShakeWindow` `bParkActive` (#151) | 663 | secured |
| `fbStartup.Update(Parked)` -> READY_STATE 0.0 | 706, BROTLib `FB_StartupReadiness:62` | **ambiguous**: clients read 0.0 as "parked" |
| `_CommandState.bIsParked` | 2326 | secured |
| MONETS Visu (`Telescope.TcVIS:4830`) | | display; operator reads "parked" |

Only the public READY_STATE (and the Visu lamp) can be misread. While the error is set READY_STATE is -1.0 (error wins,
`FB_StartupReadiness:52`); after `reset=1` it becomes **0.0 (parked)** with the telescope at el 40. Whether pyobs, the roof or
the dome act on 0.0 as "at the park pose" is **not verified** (pyobs and MONETRoof not read; the MONETS watchdog park and roof close
are commented out, `MAIN.TcPOU:209-210`).

## 4. Correct semantics (proposal)

Keep `bIsParked` = secured. Every internal consumer needs exactly that, and #86/#113/#151 are built on it. Add the missing
information instead of changing the flag:

- `bParkedInPlace : BOOL` (not persistent, FALSE after a PLC restart), TRUE when a park ended through the park-in-place branch
  (emergency or `NOT bAxesCanMove`), FALSE when a park ended through stages 10-60 (axes at the park position) and whenever
  `bIsParked` drops (#113 cover command, completed power-on).
- After a PLC restart `bIsParked` is TRUE by default (#113) without knowing the pose; `bParkedInPlace` stays FALSE there
  (open question Q3).

## 5. Fix options

| Option | What | Pro | Con |
|---|---|---|---|
| **A** Document only | README/comment: `bIsParked` = secured, may be off the park position after an emergency park; log the position | no code risk | READY_STATE 0.0 stays ambiguous for clients; nothing to test |
| **B** (recommended) `bParkedInPlace` + log + telemetry | set in stage 0 branch, cleared in the normal path and with `bIsParked := FALSE`; one INFO/WARN 'parked in place at el %s az %s (not the park position)'; `diagnostics/ParkedInPlace` in `FB_MonetDiagnostics` | no change to #86/#113/#151 gates; clients and operator can tell; small | READY_STATE unchanged unless D2 says otherwise |
| **C** Split the flag | `bIsParked` only at the park position; new `bSecured` for the internal gates | READY_STATE honest without a client change | touches every row of section 3, the #86/#113/#151 tests and BROTLib's copies (`FB_BaseTelescopeControl`, `F_ErrorParkRequest`, `F_CommandAcceptance`); after a reset READY_STATE would be -2.0 ("not parked, not online") |
| **D** Finish the park later | after `reset=1`, a `park=1` on a parked-in-place telescope powers the axes and drives to the park position | ends in the real park pose | needs a power-on (covers open, brake open) inside a park; new state machine; outside this issue |

Option B variant (B2, needs D2): READY_STATE stays 0.0 but a new value or a separate MQTT field tells "parked in place". A new
READY_STATE value is a protocol change for pyobs; a separate field is not.

Option B, position check (B3, optional): instead of (or in addition to) the branch flag, compute `bAtParkPosition` from
`fElevationCurrent`/`fAzimuthCurrent`/derotator against `telescopeConfig.*Park` with a tolerance. Catches a pendant move after a
park too, but needs a tolerance per axis and the azimuth wrap (#2). Not proposed for the first step.

## 6. Change (option B)

| File | Change |
|---|---|
| `FB_MonetTelescopeControl.TcPOU` | `bParkedInPlace` (VAR, readable like `bIsParked`); `_ParkTelescope` stage 0 park-in-place branch sets an instance flag `bInPlace := TRUE`, the `ELSE` branch `FALSE`; stage 100 copies it to `bParkedInPlace`; `bParkedInPlace := FALSE` next to both `bIsParked := FALSE` (1111, 1187). One `FB_EventLog` 'parked in place at el %s az %s' on the rising edge. Comment at 957 and 1046: "parked = secured, not necessarily at the park position, see `bParkedInPlace`". |
| `FB_MonetDiagnostics.TcPOU` | `diagnostics/base/ParkedInPlace`, on change plus heartbeat, the same pattern as `WakeShakeWindow` (`FB_MonetDiagnostics:117-119`). |
| `README.md` | Error-park paragraph: the emergency park secures the telescope where it stands; `bIsParked` = secured, `bParkedInPlace` tells the pose. |
| `specs/plans/index.md` | index line (done with this plan). |

Not touched: `F_ErrorParkRequest`, `F_CommandAcceptance`, `F_WakeShakeWindow` and their tests, HalfBROT, MONETS/MONETN `MAIN`,
BROTLib (unless D3). The mock POUs with uncommitted changes in the working tree are unrelated and stay untouched.

## 7. Tests (TcUnit, `MONETcommonTests`, `FB_MonetRig`, existing style: one rig per method, `nStep`/`nCycles`, `F_Note`)

`MonetErrorPark_Tests` (extend):
- `Emergency_Park_Is_Parked_In_Place`: online (rig is at `elevationHome` 40), `InjectElevationError`, wait for `bIsParked`.
  Assert `bParkedInPlace` TRUE, `rig.fbElev.fActualPosition` within 0.5 of 40 and not 60, covers closed, `bError` TRUE.
- `Focus_Error_Park_Is_Parked_In_Place`: the #122 case, same with `InjectFocusError`.
- `Reset_After_Emergency_Park_Keeps_In_Place`: after the park, clear the mock error and `Reset()`. Assert `bError` FALSE,
  `bIsParked` and `bParkedInPlace` TRUE, READY_STATE as decided in D2 (today 0.0). The mock error stays "until the telescope
  resets it" (`FB_ElevationMock.InjectError` comment); the reset path in the mock was not read in detail.
- `PowerOn_After_Emergency_Park_Clears_In_Place`: after the reset `Init()`; at the first cover open call `bIsParked` and
  `bParkedInPlace` are FALSE; online again.

`MonetParkDuringPowerOn_Tests` or a new `MonetPark_Tests`:
- `Normal_Park_Is_Not_In_Place`: online, `Park()`, wait for `bIsParked`. Assert `bParkedInPlace` FALSE, elevation at 60.
- `Park_After_Reset_Is_In_Place`: the #121 step 4 path (`NOT bAxesCanMove`): online, inject and clear an error, `Reset()`
  (axes disabled), `Park()`. Assert `bParkedInPlace` TRUE, no axis moved.
- `Restart_Default_Is_Not_In_Place`: fresh rig, a few scans: `bIsParked` TRUE, `bParkedInPlace` FALSE (pins Q3).

Existing tests must stay green unchanged (they check `bIsParked` only).

## 8. Hardware check (IAG test plan, T8 step 7, new item after (f); MONET/S, someone watching the covers)

- **(g) Parked in place.** Telescope online at home (el 40). Disable the focus drive in XAE (the #122 case) or write
  `MAIN.ElevationControl.bEnable := FALSE` over ADS. Pass: 'emergency park started: axis error', covers closed, hydraulics off,
  `bIsParked` TRUE, `bParkedInPlace` TRUE, the log line 'parked in place at el 40 ...', `diagnostics/base/ParkedInPlace` TRUE,
  elevation unchanged. `reset=1` (re-enable the drive first): READY_STATE as decided in D2, `bParkedInPlace` still TRUE.
  `power=1`: `bParkedInPlace` FALSE with the cover command, ready. `park=1`: el 60 / az 175, `bParkedInPlace` FALSE.
- Record at 10-20 Hz: `bError`, `bPark`, `bIsParked`, `bParkedInPlace`, park stage, `fElevationCurrent`, `fAzimuthCurrent`,
  READY_STATE, covers, hydraulics `bEnable`.
- Read `Global_Version` first (MS-65 ran 0.9.2 against a 0.9.8 pin).

## 9. Risks

1. A client that waits for READY_STATE 0.0 before closing a roof/dome gets 0.0 at el 40 today and after option B too (unless D2).
   Whether el 40 clears the roof is **not verified**.
2. `bParkedInPlace` is only as good as the branch it is set in: a pendant move after a normal park leaves it FALSE (B3 would catch it).

## 10. Not verified

- Nothing compiled or run.
- How pyobs / the observatory software and MONETRoof use READY_STATE 0.0 (not read).
- Whether the covers actually closed in the #122 run (the issue does not say; the code says they should).
- Whether IAG50cm (BROTLib `FB_BaseTelescopeControl`, BROTLib copies of `F_ErrorParkRequest`/`F_CommandAcceptance`) has the
  same park-in-place path and wants the same flag.
- That the mock clears an injected error on `Reset()` (section 7).

## 11. Open questions / DECISION NEEDED (Tim)

1. **DECISION NEEDED D1:** option A, B (recommended), C or D? Is "`bIsParked` = secured" the intended meaning?
2. **DECISION NEEDED D2:** READY_STATE after `reset=1` on a parked-in-place telescope: keep 0.0 (recommended, plus the separate
   `ParkedInPlace` field), or a distinct value (pyobs change)?
3. **DECISION NEEDED D3:** MONETcommon only, or the same flag in BROTLib for IAG50cm?
4. Q3: after a PLC restart the pose is unknown. Keep `bParkedInPlace` FALSE (recommended, no change to restart behaviour), or TRUE
   / computed from the positions (B3)?
5. Should a `park=1` on a parked-in-place telescope be refused with a hint ('parked in place, send power=1 then park=1'), or
   stay the silent jump to stage 100 of today?
6. File a new issue for this (recommended, #122 is closed), or reopen #122?
