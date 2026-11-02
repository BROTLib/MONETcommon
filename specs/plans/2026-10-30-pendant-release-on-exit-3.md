# Pendant: release the jog on key off, selector change and pulled cable (#3, HalfBROT #5)

Status: **implemented on branch `fix/pendant-release-on-exit-3`, not compiled, not run on a PLC or telescope** (decisions taken: D1 C, D2 separate issue, D3 yes, D4 close only; D5 open, no version bump in the PR; see the PR).
Date: 2026-10-30.
Repo: BROTLib/MONETcommon. Base: `develop` = `9c89a10` (0.11.2), HalfBROT `develop` = `91a782f` (0.10.2), MONETS `develop` =
`cfd37ec`. Line numbers below are from these commits.
Main files: `MONETcommon/MONETcommon/FB_MonetPendantControl.TcPOU` (FBMPC),
`MONETcommonTests/MONETcommonTests/MONETcommonTests/POUs/FB_MonetPendantControl_Tests.TcPOU`.
Issues read: #3, #156, #69, #151, #155, HalfBROT #5. Also read: MONETS/MONETN `MAIN`, `FB_MonetTelescopeControl` (FBMTC)
`SuspendAutomatic` and `_PowerOn`, BROTLib `FB_BaseAxis`, HalfBROT `FB_AxisControl`, `FB_ElevationControl`,
`FB_FocusControl`, the MONETS/MONETN Visu files and `.tsproj` links, HalfBROT plan `2026-10-12-pendant-fixes.md`.

Legend: **[read]** seen in the code at the commits above, **[inferred]** follows from the code but not observed,
**[unknown]** not answerable from the repos.

## 1. Problem

- [read] `nSelection` comes from four BCD inputs (l. 65-68). `bEnable := IbEnableKey; IF NOT bEnable THEN RETURN;`
  (l. 107-110) exits before the `CASE` (l. 122-177). Each branch 5-8 writes `Enable := IbEnableButton` and `MovePos`/`MoveNeg`
  of its own function only. Nothing writes them for a function that is not selected.
- [read] So key off, a selector change, or selection 0 leave `MovePos`/`MoveNeg` and `Enable` of the last function at their
  last value. `bError := (nSelection = 0)` only drives `ObLampError` (l. 71, 84).
- [read] Az/El/Dr pass `bMovePos`/`bMoveNeg` straight to `Jog_Forward`/`Jog_Backwards` of `FB_Axis` (e.g. `FB_ElevationControl`
  l. 95-96), which is `MC_Jog` gated only by Ready, the limit inputs and `NOT ExtActive` (BROTLib `FB_Axis` l. 262-264). Focus
  the same, gated by `bFocusUnlocked` (`FB_FocusControl` l. 258-259). [inferred] A jog that was running keeps running until a
  limit switch, an NC error or another writer drops the level.
- [read] MONETS `MAIN` calls `PendantControl` at l. 171 (MONETN l. 181), after `ElevationControl`/`AzimuthControl`/
  `DerotatorControl`/`FocusControl`, so a pendant write reaches the axis in the next scan. In manual and off mode only
  `TelescopeControl.SuspendAutomatic()` runs. It writes `MovePos`/`MoveNeg` FALSE for Az/El/Dr only inside `IF bSuspendStop`
  (FBMTC l. 1366-1371), i.e. only while the switch-to-manual stop runs. Never for focus.

Your description is correct in these points. Corrections and additions:

1. **A second writer exists: the Visu.** [read] MONETS `Elevation/Azimuth/Derotator/Focus.TcVIS` and the MONETN ones have
   `PushSwitch` elements bound to `MAIN.<Axis>Control.bMovePos`/`bMoveNeg`. This is the strongest reason for your "release once,
   do not zero every scan" (section 3.1): zeroing every scan would kill the Visu jog with the pendant key off.
2. **The covers branch is not fully guarded.** [read] The hold-to-run block is inside `__ISVALIDREF(fbCovers)` (l. 93), but the
   lamp lines of `CASE 1, 2, 3` (l. 124-126) dereference `fbCovers` without a guard. Same class as Az/El/Dr.
3. **#156 relevance is weaker than it looks.** [read] MONETS and MONETN pass all seven references in the one `PendantControl(...)`
   call that runs every scan, so in production the references are valid from the first body run. [inferred] The guards protect
   tests and other callers; they are not a candidate cause of the #156 page fault.
4. **Automatic mode is worse than "writes the same properties".** [read] FBMTC sets `Enable := TRUE` for Az/El/Dr only in
   `_PowerOn` stage 10 (l. 1220-1222). The pendant runs after FBMTC. [inferred] With the cabinet key on Remote, the pendant key on
   and the selector on 5-8, the pendant writes `Enable := IbEnableButton` (FALSE unless held) every scan: the selected axis is
   disabled while the telescope is online, and FBMTC does not re-enable it. For the elevation the falling edge closes the brake
   (`FB_ElevationControl` l. 53-56) and `F_BrakeMustClose` keeps it closed. This exists today and is not changed by this plan
   (section 4).
5. **Elevation `Enable` stale TRUE is a #121-type state.** [inferred] Key off while the enable button is held leaves the elevation
   enabled with the brake open, with nobody at the pendant. A drive fault in that state does not drop `bEnable`, so the falling
   edge brake close does not run (manual mode, no `F_BrakeMustClose`). This matters for decision D1.
6. **TwinSAFE.** [read] On MONETS the key, enable button and direction inputs are linked to standard terminals
   `Pendant Control IN (EL1008)` / `IN2 (EL1008)` (`MONETS.tsproj` l. 11818-11826). The TwinSAFE group has no alias device or FB
   named for the pendant (names only). [unknown] Whether the key or enable button is also wired into the E-stop or STO chain in
   the cabinet. MONETN terminal names not checked.
7. **Selector not debounced** (HalfBROT #5). [inferred] Turning 7 -> 8 (0111 -> 1000) can read other codes for a scan or more.
   With a direction held, an intermediate function can get a short jog command. Pre-existing.

## 2. What stays as it is

- Covers hold-to-run (`bCoverJogActive`, l. 89-104): unchanged, it already stops on release, key off and selector leaving 1-3.
- The `Enable := IbEnableButton` level while a function is selected and the key is on: unchanged.
- FBMTC, HalfBROT, MONETS/MONETN `MAIN`: not touched.

## 3. Proposed change (FBMPC only)

### 3.1 Ownership and release once (your item 1, accepted)

New VARs:

```
nOwned			: UINT;		// function 5-8 whose Move*/Enable the pendant wrote in the last scan, 0 = none (#3)
bOwnedEnable	: BOOL;		// the last Enable the pendant wrote for nOwned
```

Before the key-off `RETURN` (after the covers block):

```
nDriven := SEL(IbEnableKey, 0, nSelection);	// what the pendant drives in this scan
IF nOwned <> 0 AND nOwned <> nDriven THEN
	_Release(nFunction := nOwned, bDropEnable := bOwnedEnable);	// once: Move* FALSE, Enable per D1
	nOwned := 0;
	bOwnedEnable := FALSE;
END_IF
```

In branches 5-8, inside the reference guard: `nOwned := <5..8>; bOwnedEnable := IbEnableButton;`.

`_Release` (new private method): `CASE nFunction OF 5: focus, 6: derotator, 7: elevation, 8: azimuth`, each inside
`__ISVALIDREF`, writes `MovePos := FALSE; MoveNeg := FALSE;` and the Enable part of D1.

Why this and not "zero everything every scan" (the fix text in #3):
- With the key off the axes belong to FBMTC (automatic) or to the Visu/SuspendAutomatic (manual). Zeroing `Enable` every scan
  would disable a powered telescope in automatic; zeroing `Move*` every scan would kill the Visu jog (1.1).
- Release once is a transition, so with nothing owned the pendant writes nothing. That is testable (5.2).

Release triggers covered: key off (`nDriven = 0`), selector to any other value (0, 1-4, 9-15, another axis), pulled cable
([inferred] key and selector read FALSE/0 with NO contacts, [unknown] for the real wiring). Mode change is not a trigger
(section 4).

### 3.2 Enable on release (your item 2): decision D1

| Option | What happens on release | For | Against |
|---|---|---|---|
| A | `Move*` FALSE only | Smallest change. In automatic nothing but `Move*` is touched. | Elevation can stay enabled with the brake open, unattended (1.5). Az/Dr stay powered. |
| B | `Move*` FALSE and, if `bOwnedEnable`, `Enable := FALSE` in the same scan | Same as releasing the enable button, which the operator can do today mid-jog. Brake closes on the falling edge. | Jog stop and power off in the same scan. [unknown] AX5000 reaction to enable loss under motion (ramp, coast, brake). |
| C | `Move*` FALSE now; if `bOwnedEnable`, `Enable := FALSE` after a fixed delay (`tEnableRelease`, VAR_INPUT, default T#2S), cancelled if the same function is owned again before it | The jog decelerates under `MC_Jog` first, then the axis is disabled and the brake closes. | One TON and a pending state. The delayed write could land after the cabinet key went to Remote ([inferred] harmless: power-on needs `power=1` after manual, #72, and stage 0 opens covers first). |

Notes for D1:
- `bOwnedEnable` makes the Enable drop conditional: the pendant only drops an Enable it set TRUE itself. If its last write was
  FALSE, writing FALSE again changes nothing, so it is skipped.
- Wake&Shake (#151/#76) runs with the axes **not** enabled by the PLC, so no falling edge happens and the brake is not touched.
  [read] `F_BrakeMustClose` and the #155 brake rules are in the FBMTC body (automatic only), so they do not act in manual.
- In automatic, B and C drop an Enable only if the operator held the enable button with the pendant key on, i.e. the pendant
  already overrode FBMTC (1.4).
- No `StandStill` property on `FB_BaseAxis` [read], so C uses a delay, not standstill.

**Recommendation: C.** It removes the unattended-enabled-elevation state (1.5) without cutting power under motion. If you want
the smallest change for this release, A now and C as a follow-up, but then #3 is not fully fixed.

### 3.3 Reference guards (your item 3, extended)

Wrap branches 6, 7, 8 and the cover lamp lines of branch 1-3 in `__ISVALIDREF` like branch 5. The release (3.1) uses the same
guards. Low production value (1.3), needed by the tests (the test cannot pass a `HalfBROT.FB_CoverControl`, `FB_CoversMock`
only implements `I_MirrorCovers`).

### 3.4 Optional: debounce the selection (decision D3)

Take ownership only after `nSelection` has been stable for 100 ms (TON), release immediately. Removes the short jog on an
intermediate function (1.7) for a 100 ms delay when changing function. Cheap, but it is a behaviour change the on-site test
must cover.

### 3.5 Not changed here

`fbStartTrigger` is computed and never read; the R_TRIGs after the `RETURN` do not run with the key off. Cosmetic, left alone.

## 4. Automatic mode and mode gating: out of scope

[read] Neither MONETS nor MONETN `MAIN` passes `eTelescopeMode` to `PendantControl` (default `off` = 0), and the only use is the
empty `IF ... THEN ; END_IF` (l. 117-119).

Out of scope for this plan, because:
- It needs `MAIN` changes in MONETS and MONETN (wire `eTelescopeMode`), so it cannot ship in the library alone. A gate on
  "manual only" with the input unwired would make the pendant dead after the pin (default `off`).
- It needs a policy first: what may the pendant do in automatic (nothing, lamps only, covers, reset)? That is #72/#168
  territory (manual/automatic handover).
- The release of 3.1 is correct with or without a gate. When the gate comes, "mode leaves manual" becomes one more release
  trigger (`nDriven := 0` when not manual), no redesign.

Proposal: split it into a new issue ("pendant: no axis writes in automatic, wire eTelescopeMode") referencing 1.4, and keep #3
for the stale outputs only. Decision D2. Until then: the on-site check (section 7) does not turn the pendant key on with the
selector on 5-8 in automatic.

## 5. Tests (TcUnit)

### 5.1 How the inputs are driven

[read] The selector, key and buttons are `AT%I*` VARs inside FBMPC, so the existing test (`Unlinked_Pendant_Lights_Error_Lamp`)
covers only the unlinked state. [read] Reading an FB's VAR from outside compiles (MONETS `MAIN` l. 91 reads
`PendantControl.IbResetButton`, FBMPC reads `fbElevation.inDigitalInputs`). Writing it from outside is not expected to compile.

Proposal, the same pattern as the axis mocks (`FB_ElevationMock EXTENDS HalfBROT.FB_ElevationControl`):
- New `FB_MonetPendantProbe EXTENDS MONETcommon.FB_MonetPendantControl`, VAR_INPUT `bKey`, `bEnableButton`, `nSel : UINT`,
  `bUp`, `bDown`, `bLeft`, `bRight`. Body: write `IbEnableKey`, `IbEnableButton`, `IbBCD0..8` (from `nSel.0..3`),
  `IbDirection*`, then `SUPER^()`. [inferred] A derived FB may write the base's `AT%I*` VARs; the first CI build confirms it.
  Fallback if it does not compile: `ADR()` of the VAR and a pointer write in the test.
- The El/Az branches AND the direction with `InDigitalInputs.0/.1` (limit inputs, TRUE = free). On a test target they read 0, so
  `MovePos` would stay FALSE and every release test would pass without testing anything. Add `SetDigitalInputs(nInputs : WORD)`
  to `FB_ElevationMock` and `FB_AzimuthMock` (writes the inherited `InDigitalInputs`), and every test asserts `Move*` TRUE before
  the event (precondition).
- The tests assert the command level (`MovePos`/`MoveNeg`/`Enable` getters of `FB_BaseAxis`), not motion. The mocks are not
  scanned; they do not consume `bMovePos` anyway (no `SUPER^()`).

### 5.2 Test cases (all in `FB_MonetPendantControl_Tests`, no new suite: 38 of 40 suites used)

Names must not start with `Disabled_` (TcUnit skips those).

1. `Elevation_Key_Off_Releases_Jog`: sel 7, key, enable button, up, inputs 16#3 -> precondition `MovePos`, `Enable` TRUE; key
   off -> `MovePos`/`MoveNeg` FALSE; `Enable` per D1 (A: TRUE; B: FALSE; C: TRUE, then FALSE after `tEnableRelease`, set to T#0S
   or a few scans in the test).
2. `Azimuth_Selector_Change_Releases_Jog`: sel 8, right held -> `MovePos` TRUE; sel 7 with right still held -> azimuth `Move*`
   FALSE.
3. `Derotator_Selector_Zero_Releases_Jog`: sel 6, left -> `MoveNeg` TRUE; sel 0 -> FALSE, `ObLampError` TRUE.
4. `Focus_Key_Off_Releases_Jog`: sel 5, up -> `fbFocus.MovePos` TRUE; key off -> FALSE.
5. `Key_Off_Without_Ownership_Writes_Nothing`: key off for several scans, the test sets elevation `MovePos` and `Enable` TRUE
   (stands for the Visu/FBMTC) -> still TRUE.
6. `Release_Is_Once`: after test 1's release, the test sets `MovePos` TRUE again with the key still off -> still TRUE next scan.
7. `Reselect_Cancels_Enable_Drop` (only for C): release, then the same function selected again before the delay -> `Enable`
   stays as the button says.
8. `Unassigned_References_Do_Not_Fault`: probe with no references, key on, sel 1, 6, 7, 8 for one scan each. A failure here is a
   runtime page fault, not a failed assert: CI shows it as a crashed run.

Keep `Unlinked_Pendant_Lights_Error_Lamp` as is.

### 5.3 What you run (Windows, XAE; this Linux machine cannot build)

Either push the branch: `tcbuild.yml` runs on every push to any branch on the self-hosted runner, installs the checked-out
MONETcommon as a library, builds `MONETcommonTests` and runs TcUnit. Check in the log that the tests ran (not "skipped" for a
missing license or stopped runtime) and the new case count.

Or locally, as in `MONETcommonTests/README.md`:

```powershell
tctool MONETcommon.sln library --xae MONETcommon --plc MONETcommon --out MONETcommon.library --install
cd C:\TwinCAT\3.1\Runtimes\UmRT_Default; .\Start.bat
tctool MONETcommonTests.sln build run --require-valid-license
```

In XAE also: build MONETcommon and check there is no new warning on FBMPC; let XAE regenerate the `<LineIds>` and the GUID of
the new method (not edited by hand, as in the HalfBROT pendant plan).

## 6. HalfBROT twin (#5)

[read] HalfBROT `FB_PendantControl` has the same pattern (l. 177-263). [read, HalfBROT plan 2026-10-12] no IAG telescope
instantiates it; IAG50cm has its own `FB_PendantControl`; MONETS/MONETN use FBMPC. [read] It also writes `bStop`, `bPark`,
`bGoto`, `bGoHome` on `fbTelescope`, so a port is not a copy of 3.1.

Recommendation: do not port. Close HalfBROT #5 with a link to the MONETcommon fix and "block not used at IAG". Optionally mark
the POU `{attribute 'obsolete' := 'stale outputs, HalfBROT#5; use MONETcommon.FB_MonetPendantControl'}`: any user then gets a
compile warning. That is a small HalfBROT change and release. Decision D4. Note: HalfBROT plan `2026-10-11-commands-state-machines-io.md`
still lists #5 as covered by its TcUnit work; update that line if #5 is closed.

## 7. On-site check (add to #69, MONETS and MONETN)

Before: roof closed, covers closed, telescope near balance (el ~40), cabinet key Local (manual), someone at the E-stop. Online in
XAE: `MAIN.PendantControl.nSelection`, `nOwned`, `bOwnedEnable`, `MAIN.ElevationControl.bMovePos/bMoveNeg/bEnable`, the same for
azimuth, derotator, focus, and `HydraulicsControl.bBrakeOpen`.

A. Axes disabled (do not press the enable button, so nothing can move; the jog level is still visible online):
1. Key on, sel 7, hold up: `ElevationControl.bMovePos` TRUE, `nOwned` 7. Key off while holding: `bMovePos` FALSE, `nOwned` 0.
2. Same with sel 8 / right, then turn to 7 while holding: `AzimuthControl.bMovePos` FALSE.
3. Sel 6 / left, turn to 0: `DerotatorControl.bMoveNeg` FALSE, error lamp on.
4. Sel 5 / up, key off: `FocusControl.bMovePos` FALSE.
5. Pendant unplugged during 1 (only if allowed): note `nSelection`, `IbEnableKey` and whether anything else reacts (E-stop,
   TwinSAFE). `bMovePos` FALSE.
6. Key off: set `ElevationControl.bMovePos` with the Visu jog button: it works and is not reset by the pendant.

B. Axes enabled, short jogs at the default jog velocity, azimuth and derotator first, elevation last:
7. Sel 8, enable held, jog right, key off mid-jog: the azimuth stops (time it). `Enable` per D1 (C: drops after ~2 s).
8. Sel 6, jog, turn the selector mid-jog: the derotator stops.
9. Sel 7, jog up a few degrees, key off mid-jog: the elevation stops; with C the elevation is disabled after the delay and
   `bBrakeOpen` goes FALSE. Watch for any drop or swing at the disable.
10. If D3: turn the selector 7 -> 8 with a direction held: no motion on another function.

C. Automatic, non-regression (pendant key stays off): cabinet key Remote, `power=1`, short slew, park. `nOwned` stays 0, no
pendant write. Do not turn the pendant key on with sel 5-8 in automatic (1.4, D2).

Fail if: any axis keeps moving after key off, selector change or unplug; the Visu jog stops working with the key off; anything
moves in A.

Version note from #69: a branch build with the same `ProjectVersion` as an installed release overwrites it on the engineering
PC. Bump the branch or back up the installed `.library`.

## 8. Release and pins (not part of this plan)

1. MONETcommon release (0.11.3, or 0.12.0 if you count the pendant behaviour change as minor) after CI and the on-site check.
2. MONETS pin PR to the new MONETcommon.
3. MONETN pin PR to the new MONETcommon.
4. If D4 with the `obsolete` attribute: HalfBROT release, then the BROTLib/HalfBROT pin in MONETcommon (as PR #174).
5. If D2 is a new issue: its `MAIN` changes come with their own pins later.

## 9. Risks

- The release runs once. If a write is lost (online change resetting `nOwned`, or the reference invalid in that scan) there is no
  retry. [inferred] Online change keeps VAR values; a cold/warm reset clears the axes' `bMovePos` too.
- C's delayed disable can hit an axis the operator meanwhile jogs from the Visu (key off). Safe direction (stops it), but
  surprising. Same for the release of `Move*` while a Visu jog runs on the same axis.
- A pulled cable may not read as key off if the key is wired NC or through another path [unknown].
- Disable under motion (B) or after a too-short delay (C): drive and brake reaction [unknown], checked in 7.B.
- The probe FB may not compile (5.1): fallback with `ADR()`, or the tests shrink to the unlinked state again.
- The automatic-mode hazard of 1.4 stays until D2 is done.

## 10. Not verified

Nothing compiled or run. Unknown: TwinSAFE wiring of key and enable button (1.6); whether the enable button is momentary or a
three-position enabling switch; AX5000 behaviour on enable loss; `MC_Jog` deceleration on level drop (expected per Beckhoff
`MC_Jog`, not checked); the real effect of 1.4 on the telescope; MONETN pendant terminal types.

## 11. Decisions for Tim

- **D1** Enable on release: A, B or C. Recommended C (delay T#2S).
- **D2** Mode gating: split into a new issue (recommended) or include here with `MAIN` changes in MONETS/MONETN.
- **D3** Debounce the selector (100 ms) now: recommended yes, small.
- **D4** HalfBROT #5: close only (recommended), or close and mark the POU `obsolete`, or port.
- **D5** Version: 0.11.3 or 0.12.0.

## 12. Steps

1. FBMPC: `nOwned`, `bOwnedEnable`, `nDriven`, `_Release` method, the release block before the `RETURN`, ownership in 5-8,
   reference guards on 1-3 lamps and 6-8 (3.1, 3.3). D1 variant (and D3 if chosen). Files: `FB_MonetPendantControl.TcPOU`.
2. Test seams: `FB_MonetPendantProbe.TcPOU` (new), `SetDigitalInputs` in `FB_ElevationMock.TcPOU`, `FB_AzimuthMock.TcPOU`,
   `MONETcommonTests.plcproj` (new POU).
3. Tests 5.2 in `FB_MonetPendantControl_Tests.TcPOU`; row in `MONETcommonTests/README.md` (the "only this unlinked state can be
   tested" sentence goes).
4. `README.md` (pendant behaviour line 94-95: release on key off / selector change / selector 0), this plan's status, index line.
5. Push the branch, CI build and TcUnit (5.3); fix until green.
6. Comment on #69 with section 7; on-site A, B, C on MONETS and MONETN.
7. Release and pins (section 8); comment and close #3 (or keep open for D2) and HalfBROT #5 (D4).
