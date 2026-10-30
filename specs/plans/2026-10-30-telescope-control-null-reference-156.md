# Telescope control: no page fault in manual after a cold reset (#156)

Status: implemented in a PR (D1 together with #3 as 0.11.3, D2 no, D3 no); not compiled, not run on a PLC or telescope
Date: 2026-10-30.
Repo: BROTLib/MONETcommon. Base: `develop` = `94a25b3` (0.11.2 plus the unreleased #3 pendant merge), MONETS `develop` =
`cfd37ec`, MONETN `develop` = `9ae07bb` (both pin MONETcommon 0.11.2). Line numbers below are from these commits.
Main file: `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU` (FBMTC).
Issues read: #156. Also read: MONETS/MONETN `MAIN`, FBMTC (`SuspendAutomatic`, `_ReadAxes`, `_StopTxInputs`, all public
methods and properties), `FB_MonetPendantControl` (FBMPC), `FB_MonetMqttWatchdog`, `FB_MonetDiagnostics`, BROTLib
`FB_BaseTelescopeControl`, `FB_AltAzTelescopeControl`, `FB_StartupControl`, `FB_Comm_MQTT_Influx`, `FB_MonetRig`,
`MonetSuspendUnderMotion_Tests`, `SuspendAutomatic` at tags v0.9.2, v0.9.3, v0.9.10.

Legend: **[read]** seen in the code at the commits above, **[inferred]** follows from the code but not observed,
**[unknown]** not answerable from the repos.

## 1. Problem

- [read] MONETS `MAIN` l. 103-123 (MONETN l. 114-133): the full `TelescopeControl(...)` call, the only place that assigns
  `fbElevation`, `fbAzimuth`, `fbDerotator` (`REFERENCE TO`, FBMTC l. 10-12) and the interfaces `fbFocus`, `fbCovers`,
  `fbBrake`, `fbHydraulics` (l. 6-9), runs only in automatic. With the cabinet key on Local `MAIN` calls only
  `TelescopeControl.SuspendAutomatic()`.
- [read] `SuspendAutomatic` (l. 1298) starts with `_ReadAxes()` (l. 1308, reads `fbAzimuth.fbAxis.StandStill` etc., l. 1890)
  and `_StopTxInputs()` (reads `fbAzimuth.bEnable`, l. 1925). No `__ISVALIDREF` anywhere in FBMTC.
- [inferred] After a cold reset the references are null, so the first manual scan dereferences null: page fault in
  `PlcTask`, as in the issue.
- [read] The bisect fits: at v0.9.2 `SuspendAutomatic` touched no axis. v0.9.3 (#118) added `_StopTxInputs()`, later
  `_ReadAxes()` was added in front of it (present at v0.9.10). So every release from 0.9.3 on has it, including the 0.11.2
  that MONETS and MONETN pin today. [inferred] The missing run (HalfBROT 0.7.2 + MONETcommon 0.9.10, forced) would crash too;
  HalfBROT 0.7.3 is not needed for it.
- [unknown] That offset 0x10eda3 is in `_ReadAxes`. The dump would confirm it (section 7, optional).

Your analysis is correct. Additions:

1. **Not only a forced input.** [inferred] Any start of the PLC with empty references and the key on Local faults: cold reset,
   a new download, and a controller boot after a power cut with the physical key on Local. A warm restart or stop/start after
   an automatic period does not, because the references keep their last value. The power-cut case matters most: the PLC then
   fails at boot with nobody on site.
2. **A second path: `Stop()`.** [read] `Stop()` (l. 2405) calls `fbFocus.Stop()` when `bStop` is already TRUE (l. 2412, #122).
   In manual `bStop` is not released (only the suspend stop releases it). [inferred] Two MQTT `stop` commands in manual after a
   cold reset call a method on a null interface: page fault too. Same cause, different trigger.
3. **Everything else is safe.** [read] The other public members only read or write FBMTC's own variables or call
   `_AcceptMove`/`fbStartup`: `Init`, `Park`, `Reset`, `Home`, `Slew`, `Track`, `TrackRaDec`, `MoveAltAz`, `MoveRaDec`,
   `Set*Offset`, all properties (FBMTC and BROTLib base classes). The other callers in `MAIN` use only those:
   `CabinetControl` and `MqttWatchdog` call `Park()`, `fbComm` calls the commands and properties above plus `Stop()`,
   `Diagnostics` reads outputs. `Stop()` is the only one with a dereference.
4. **FBMPC is not involved.** [read] MONETS/MONETN pass all references to `PendantControl` every scan (see the #3 plan, 1.3).

## 2. What stays as it is

- The body of FBMTC: it runs only in the call that assigns the references.
- MONETS and MONETN `MAIN`: no change needed (section 4).
- `SuspendAutomatic` behaviour once the references are valid (after the first automatic scan): unchanged.

## 3. Proposed change (FBMTC only)

### 3.1 Guard in `SuspendAutomatic`

New VAR `bAxesValid : BOOL;` (or a local). At the start:

```
bAxesValid := __ISVALIDREF(fbAzimuth) AND __ISVALIDREF(fbElevation) AND __ISVALIDREF(fbDerotator);
IF bAxesValid THEN
	// l. 1308-1392 as today: _ReadAxes, _StopTxInputs, EnterSuspend block, bSuspendStop block
END_IF
// l. 1393-1418 as today: SuspendStopEvent, ParkedInPlaceEvent, _StopEvents, timeout reset, fbStartup.Suspend, heartbeat
```

Why wrap and not `RETURN` at the top (your proposal): an early return also skips `fbStartup.Suspend` and the READY_STATE
heartbeat, so after a cold start in manual pyobs gets no READY_STATE at all. v0.9.2 published it. [read] The tail uses no
reference: `fbStartup.Suspend` is independent of `EnterSuspend`, the events use FBMTC variables, `fbComm` is checked
(`CommValid := fbComm <> 0`).

Notes:
- With invalid references `EnterSuspend` is not called, so `bInitRequired` is not set. [inferred] Harmless: after a cold start
  the telescope is not initialized, `bReady` is FALSE and the first automatic scan needs an Init anyway. Nothing is moving
  that would need the suspend stop.
- The interfaces are not checked in the guard: the axis part touches them only through `_PowerOn` (only if `bPower` or
  `fbStartup.Initializing`, both FALSE after a cold reset [inferred]) and the stage methods called with `bInterrupted`. All of
  them can only have been set by a body run, which assigned the interfaces. [inferred] If you want to be strict, add
  `fbFocus <> 0 AND fbCovers <> 0` to `bAxesValid`. Decision D2.

### 3.2 Guard in `Stop()`

```
IF bStop THEN
	fbStopTx.Restart();
	IF fbFocus <> 0 THEN
		fbFocus.Stop();
	END_IF
END_IF
```

Interfaces are checked with `<> 0`, as `FB_MonetMqttWatchdog` does. `__ISVALIDREF` is for references only.

### 3.3 Not done here

- No guards in the body or the private methods: they run only after the assignment. Adding them would hide a wiring error
  instead of faulting on it.
- No early `RETURN` with a log line. [inferred] A log every manual scan would flood; a one-shot "axes not wired" hint is
  possible but not needed for the fix. Decision D3.

## 4. MAIN side: not needed

The guard is in the library, so both sites are fixed by the pin alone. A `MAIN` change (for example the full call in every
mode, or a one-time assignment) would need two pin PRs and leave the trap for the next caller. Not recommended.

## 5. Tests (TcUnit)

### 5.1 Pattern

[read] `FB_MonetRig` holds a fresh FBMTC on mocks, `SetManual(TRUE)` makes `Scan()` call only `SuspendAutomatic`. So a rig
with `SetManual(TRUE)` before the first `Scan()` is exactly the cold start on Local. No new seam, no new suite (38 suites in
`MAIN`). Add the cases to `MonetSuspendUnderMotion_Tests`. A failure is a page fault, CI shows a crashed run, as for
`Unassigned_References_Do_Not_Fault` in FBMPC.

### 5.2 Test cases

1. `Manual_From_Cold_Start_Does_Not_Fault`: rig, `SetManual(TRUE)`, 20 scans. Assert `fbTel.ReadyState` is the suspended value
   (-2.0, as `Suspend_While_Slewing_Stops_Each_Moving_Axis_Once` expects) and no axis `StopAxis`/`MoveAxis` set. Faults today.
2. `Stop_Twice_In_Manual_From_Cold_Start_Does_Not_Fault`: rig, `SetManual(TRUE)`, scan, `fbTel.Stop()`, scan, `fbTel.Stop()`,
   scan. Faults today on the second `Stop()`.
3. `Manual_From_Cold_Start_Then_Automatic_Requires_Init`: after 1, `SetManual(FALSE)`, wait for the clock, slew rejected,
   `Init()` reaches online. Checks the note in 3.1 (no stale state after the skipped `EnterSuspend`).

Order: write 1 and 2 first and run them on `develop` without the fix, to see the crash in CI (confirms the test reaches the
fault). Then the fix.

### 5.3 What you run (Windows, XAE; this Linux machine cannot build)

Push the branch: `tcbuild.yml` builds `MONETcommonTests` and runs TcUnit on the self-hosted runner. Or locally, as in
`MONETcommonTests/README.md`:

```powershell
tctool MONETcommon.sln library --xae MONETcommon --plc MONETcommon --out MONETcommon.library --install
cd C:\TwinCAT\3.1\Runtimes\UmRT_Default; .\Start.bat
tctool MONETcommonTests.sln build run --require-valid-license
```

Report back: the result of the run with tests only (expected: crash, which test was running), then with the fix (all green,
new case count, `Failed checks` line if any). In XAE: no new warning on FBMTC, let XAE regenerate `<LineIds>`.

## 6. Hardware check (MONET/S, supervised)

Before: telescope parked and unpowered, roof closed, someone at the E-stop, the build with the pinned new MONETcommon
downloaded. Online: `MAIN.TelescopeControl.bAxesValid`, `fReadyState`, `MAIN.eTelescopeMode`.

1. `inKeyLocal` forced TRUE (or the key on Local), PLC cold reset, PLC start. Pass: no exception in the XAE log for 2 min, no
   EtherCAT OP -> SAFEOP, `bAxesValid` FALSE, READY_STATE -2 arrives in MQTT/Influx.
2. Same state, send `stop` twice from pyobs or MQTT. Pass: no exception.
3. Release the force (Remote), `power=1`: startup completes, `bAxesValid` TRUE. Then Local again: the #118 suspend runs as before
   (log, no stop at rest).
4. Repeat 1 once more (the issue saw it twice in a row).

If it faults: note offset and time, do not repeat, roll back (section 9). Then MONET/N with steps 1 and 3 at the next visit.

## 7. Optional: confirm the offset

[unknown] Whether the dump of 2026-10-26 can still be mapped: it needs the exact compiled project of MONETS 1.4.5. If you have
it, open the dump in XAE (PLC project, "Load core dump") and check that the faulting line is in `_ReadAxes`. Not needed for the
fix, but it closes the "Not known" item of the issue.

## 8. Release and pins

1. MONETcommon patch release 0.11.3. It also carries the unreleased #3 pendant merge (`94a25b3`); the #3 on-site check then
   runs on the same build. If you want them apart, release #3 first as 0.11.3 and this as 0.11.4. Decision D1.
2. MONETS pin PR, hardware check (section 6).
3. MONETN pin PR.
4. Comment on #156 with the cause and the release, close after the MONET/S check.

## 9. Risks and rollback

- [inferred] The guard skips the suspend stop if the references are invalid. They can only be invalid before any automatic
  scan, when nothing was commanded, so nothing can be moving under FBMTC control.
- A wrong guard (always FALSE) would disable the #118 stop in manual. Caught by the existing `MonetSuspendUnderMotion` tests and
  hardware step 3.
- Rollback: load the previous boot project (MONETS pin to MONETcommon 0.11.2). Until the fix is deployed: do not cold reset,
  download or power cycle the controller with the key on Local; after a power cut, turn the key to Remote before the PLC starts
  if possible [unknown whether the boot is fast enough for that on site].

## 10. Not verified

Nothing compiled or run. Unknown: the instruction at 0x10eda3; whether a null `REFERENCE TO` read faults in the first or a later
manual scan (expected first); behaviour on MONET/N (same code, not tested); whether a controller power cut with the key on
Local has happened since 0.9.3.

## 11. Decisions for Tim

- **D1** Release: #3 and #156 together as 0.11.3 (recommended), or separately.
- **D2** Guard also on `fbFocus`/`fbCovers` in `bAxesValid`: not needed (recommended no), stricter if yes.
- **D3** One-shot log hint "axes not wired" when the guard skips: recommended no.

## 12. Steps

1. Tests 5.2 cases 1 and 2 in `MonetSuspendUnderMotion_Tests.TcPOU`; push, see CI crash.
2. FBMTC: `bAxesValid` and the wrap in `SuspendAutomatic` (3.1), `fbFocus <> 0` in `Stop()` (3.2). Test case 3.
3. CI green (5.3); this plan's status, index line.
4. Release (D1), MONETS pin, hardware check 6, MONETN pin; comment and close #156.
