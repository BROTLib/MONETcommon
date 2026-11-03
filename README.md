# MONETcommon

MONETcommon is the **common control library for the MONET-class telescopes** of
the BROT project: it contains the MONET-specific TwinCAT 3 (IEC 61131-3
Structured Text) function blocks shared by the MONET North (MONETN) and MONET
South (MONETS) telescope applications. It extends the BROTLib core and the
HalfBROT hardware layer with the full MONET telescope-control logic: the
complete Alt-Az telescope lifecycle, TwinSAFE safety handling, pendant
control, cabinet I/O, power monitoring and mirror-cover control.

MONETcommon is the **reference implementation** of the MONET control code.
MONETS and MONETN both reference the library directly (placeholder resolution
`MONETcommon, * (IAG)`); MONETN dropped its former local copies of these blocks
in MONETN PR #9 (2026-10-08). Hydraulics and focus are not part of this
library: both sites drive them straight from HalfBROT's
`FB_HydraulicsControl`/`FB_FocusControl`.

The library is versioned in the project file and tagged on release (current:
see `Global_Version.TcGVL` / the repository's release tags) and is built by
the Institut für Astrophysik Göttingen (company field `IAG`).

---

## Repository layout

```
MONETcommon/
├── MONETcommon.sln               # TwinCAT solution
├── MONETcommon/
│   ├── MONETcommon.tspproj       # TwinCAT library project
│   └── MONETcommon/
│       ├── MONETcommon.plcproj   # PLC library project
│       ├── Global_Version.TcGVL  # Library version (kept in sync with the release tag)
│       ├── E_ModeLanguage.TcDUT  # Unused in this library today; only MONETN's own copy is referenced
│       └── FB_*.TcPOU            # Function blocks (see below)
└── README.md
```

Note: `FB_MonetHydraulicsControl` and `FB_MonetFocusControl` were removed
(2026-09-22) — neither MONETN nor MONETS ever instantiated them (both sites
use HalfBROT's `FB_HydraulicsControl`/`FB_FocusControl` directly), so they
were orphaned forks of HalfBROT's actively-maintained versions.

## Function blocks

| Function block | Description |
|---|---|
| `FB_MonetTelescopeControl` | Full Alt-Az telescope lifecycle — extends `FB_AltAzTelescopeControl` (BROTLib) |
| `FB_MonetSafetyHandling` | TwinSAFE startup/safety handling — E-stop, STO reset for all three axes |
| `FB_MonetPendantControl` | Manual hand pendant (BCD selector + buttons) |
| `FB_MonetCabinetControl` | Cabinet physical I/O — buttons, switches, lamps, temperature |
| `FB_MonetPowerMonitoring` | Three-phase power-quality monitoring |
| `FB_MonetPendantControl` covers | Selection 1-3 with the key on: hold-to-run open/close of all covers via `HalfBROT.FB_CoverControl` (one copy since 0.10.0, needs HalfBROT with `Stop()`) |
| `FB_MonetMqttWatchdog` | Parks the telescope and closes the roof after 30 s without an MQTT broker connection (armed after the first connect) |
| `FB_MonetDiagnostics` | `diagnostics/base/*` telemetry: telescope interrupted/ready/busy/stopped, per-axis error/error ID/enable, safety error/estop/state; on change plus a 30 s heartbeat |
| `FB_MonetTelescopeTelemetry` | The `telescope/dome` telemetry record of `FB_MonetTelescopeControl` from an `ST_MonetTelemetry` snapshot (used internally) |

### FB_MonetTelescopeControl

The central function block: a complete Alt-Az telescope controller extending
BROTLib's `FB_AltAzTelescopeControl`. It implements the telescope command
interface (`E_TCSCommand`: `PowerOn`, `GoHome`, `GoTo`, `Track`, `Slew`,
`Park`, `Stop`, ...), drives the Az/El/derotator axes through the HalfBROT
`FB_AxisControl`-based blocks, coordinates the mirror covers, focus,
hydraulics and safety, publishes MQTT telemetry and handles error states and
recovery. Each site passes its own pointing model (`fbPointing`,
`fbPointingInverse`) from `MAIN`.

**Command inputs are edge-triggered and read-only (#24 step 5, breaking).** The inputs inherited from BROTLib's
`FB_BaseTelescopeControl` (`bPower`, `bGoHome`, `bPark`, `bTrack`, `bGoto`, `bSlew`, `bStop`, `bReset`) are never written
by the block. Their rising edge is a command, exactly like the method (`Init`, `Home`, `Park`, `Track`/`TrackRaDec`/
`MoveRaDec` for `bGoto`, `Slew`/`MoveAltAz`, `Stop`, `Reset`; `bTrack` tracks the target inputs): a held input starts its
command once, the input falling aborts nothing, and a method call and an input edge in the same scan are one command. A
method call (or a new edge) while the same command runs is still a command (a new `Track` retargets, a second `Stop`
restarts the stop, #96). The requests are the read-only outputs `bPowerActive`, `bGoHomeActive`, `bParkActive`,
`bTrackActive`, `bGotoActive`, `bSlewActive`, `bStopActive`, `bResetActive` (requested or running, cleared by the block
where it used to clear the inputs). Until 0.14.x the block wrote the inputs: a writer could write `TRUE` again after a
command ended and get a new one; now it must write `FALSE` first. MQTT clients (pybrotlib, pyobs-brot, brotgui) are not
affected, the PLC turns their verbs into method calls. The MONET Visu `TOGGLE` switches on these inputs need TAP mode or
method calls (plan `specs/plans/2026-11-02-step-4c-client-check.md`), and status lamps should show the `b*Active` outputs.

A new axis, focus or hydraulics error or a power failure on a telescope that is not parked starts the
emergency park by itself (brake closed, covers closed, hydraulics off,
`bIsParked`); not during an E-stop (the E-stop holds its own park until the release, below), not when already parked, and not for
command-level (software) errors other than the state-machine timeouts (below). `power=1` is refused until the park is
finished, `stop=1` cancels it; recover with `reset=1`, then `power=1`.
`bIsParked` drops with the cover command of a power-on, so a park during a
power-on closes the covers again. Exception: an azimuth/elevation error in the
first power-on after a PLC restart (the Wake&Shake fault) does not park; the
covers stay open for the manual Wake&Shake (a WARN is logged). From then on a
15 min Wake&Shake window lets the brake stay open with the elevation disabled
(otherwise it is closed whenever the elevation is disabled, #121). The window
closes, and the brake with it, on park, E-stop, a focus/derotator error, closed
covers, a new `power=1` or the timeout (#151).

**Hydraulics error (#24 step 7 PR 3, D8).** `fbHydraulics.Error` is a telescope error like an axis error (`bError`,
READY_STATE -1.0, the emergency park in place on its rising edge, with the same exemptions: not parked, not in E-stop;
never treated as the Wake&Shake fault). Ramp first: if the brake is still open, the oil pressure is there and an axis
moves, the axes get a controlled stop in the same scan (the normal stop, one MC_Stop per axis) and stay enabled until it
has settled, then the park in place disables them, closes the brake and the covers and switches the hydraulics off. With
the pressure already gone (brake locked), or with an E-stop, a safety error or any other error, there is no ramp: the
lock is the stop, as for an axis error. Recovery: reset the hydraulics (HalfBROT `Reset()`, the `MAIN` reset button);
`bError` then goes by itself, `reset=1` clears the roof request, `power=1` powers on again. A `reset=1` while the
hydraulics error is still there does not clear it and is not held (it would abort the park).

**State-machine timeouts (#24 step 7 PR 5, D8).** A goto (16#10), home (16#30) or slew (16#40) still running after
`stMotion.tStateMachineTimeout` (600 s) is a software error as before (`bError`, READY_STATE -1.0, latched until
`reset=1`), but the axes are no longer disabled under motion: the drives are still controllable, so a moving axis gets a
controlled stop (the normal stop, one MC_Stop per axis) and then the telescope parks by itself, driving to the park
position when the axes can move (else in place), closing the brake and the covers and switching the hydraulics off. Not
when already parked, and not with an E-stop, a safety error or a hardware error (those keep their own reaction; one that
appears during the stop or the park drive disables the axes and the park ends in place). No roof close request (D9:
recoverable). `stop=1` cancels the park (the axes are then disabled); `reset=1` clears the error, `power=1` powers on again.

**Cover error (#24 step 7 PR 4).** `fbCovers.Error` (HalfBROT `FB_CoverControl`, latched there; no cover moves while it is
set) is not a telescope error (`bError` stays FALSE, the drives are controllable) and does not request the roof close. A new
cover error on a telescope that is not parked and not in E-stop starts a normal park: a running move is stopped first, then
the park drives to the park position (in place if the axes cannot move). A park whose covers are in error no longer waits for
them: once the axes are disabled, the brake is closed and the hydraulics are off it ends with `bIsParked` and the covers as
they are, and raises **software error 16#21** (READY_STATE -1.0, log `Internal command error, ID: 33`). A power-on with the
cover error fails at once, before it commands the covers (power-on failure, READY_STATE -1.0, the telescope stays parked).
Recovery: the `MAIN` reset button (HalfBROT cover `Reset()` and, in automatic, `reset=1`) clears both; then `power=1` goes
online (the covers open) and `park=1` closes them. `reset=1` alone clears 16#21, but `power=1` keeps failing while the cover
error is there.

**Power failure (#24 step 7 PR 7, D8).** `bMainReady` FALSE (wired to `PowerMonitoring.phaseOK` in `MAIN`: any guard
warning or error of the power monitor, not debounced) is a telescope error too, handled like the hydraulics error: `bError`,
READY_STATE -1.0 (until 0.13.0 it only gated online with -2.0), moves and `power=1` refused, the emergency park in place on
its falling edge (not when parked, not in E-stop, never the Wake&Shake fault). Ramp first while the drives can still ramp
(brake open with oil pressure, no axis error, an axis moves), else the lock is the stop; then park in place, covers closed
as far as they still move, hydraulics off. It is not latched: `bError` goes by itself when the supply is back; then
`reset=1` clears the roof request and `power=1` powers on again (the completed startup is gone). Even a short dip of the
phase monitor parks the telescope; if the power monitor's warnings blip on site, filter them in `MAIN` /
`FB_MonetPowerMonitoring`.

**Focus error (#24 step 7 PR 8, D8).** A focus error alone (no axis, hydraulics or software error, no power failure, no E-stop or safety error)
leaves the axes controllable, so its emergency park drives to the park position instead of parking in place, if the axes
can drive (calibrated, enabled, brake open, oil pressure there): a moving axis gets the controlled stop first, then the park
drives, disables the axes, closes the brake and the covers and switches the hydraulics off (`bParkedInPlace` FALSE). The
faulty focus is not sent to its park position and the park does not wait for it. Otherwise (another error at the same time or
during the drive, an E-stop, axes that cannot drive) it parks in place as before; after a switch to manual the pending park
parks in place. No roof close request (D9). Recovery: `reset=1` (resets the focus; during this park a reset is one scan and
restarts the park once), then `power=1`.

**E-stop and safety error (#24 step 7 PR 6, D8).** An E-stop (`bEstopTriggered`) or a safety error without E-stop
(`bSafetyError`) cuts the drives (STO): the brake lock is the stop. The axes are disabled in every scan (until 0.13.0 only
for the E-stop), and on the rising edge the hydraulics are switched off at once (the hydraulic brake locks) and every move
ends (no controlled stop, no MC_Stop). If the telescope is not parked, a park in place is held while the E-stop or the
safety error is there (nothing parks and the covers are not commanded to open or close meanwhile, also not after a `reset=1`)
and runs as soon as both are gone: covers closed, hydraulics off, `bIsParked`, `bParkedInPlace` (WARN "E-stop / safety error
released: park in place"). The covers are not on the E-stop circuit (they keep their supply), and an E-stop means stop: on
the edge and in every scan while the E-stop or the safety error is there the block calls `fbCovers.Stop()`, so a running
open or close (a power-on, a park, the pendant in automatic) ends where it is; a power-on or a park waits in its stage and the
covers close only after the release, by the held park (PR 6 follow-up; `fbCovers` is `REFERENCE TO HalfBROT.FB_CoverControl`
for that, `MAIN` unchanged). Not for a safety error alone while the
Wake&Shake window is open (the window survives it, #155). Recovery: release the E-stop and run the safety reset (the
`MAIN` reset button), wait for the park, `reset=1` clears the roof request, `power=1`. A safety error is still not
`bError`, does not reject moves and is not part of the READY_STATE -1.0 condition (the E-stop is, as before); online
it drops out of READY_STATE 1.0 because the axes are disabled.

The brake can be opened in the window with the elevation error latched, without `reset=1` first: the error closes
the brake once, and while the window is open the brake is closed again only by an E-stop, a hydraulics error, a safety
error (input `bSafetyError`, wire it to `SafetyHandling.error`; FALSE = not enforced) or covers that are not open (#155).

What the block does for each error class (axis, focus, software, hydraulics, covers, E-stop, power, clock, MQTT,
cabinet temperature, manual mode, PLC restart), with the code line, the target safe state and the test that pins it:
[specs/design/monetcommon-safe-states.md](specs/design/monetcommon-safe-states.md).

**Roof close request (`bCloseRoofRequest`, #24 step 7, D9).** An output that says the roof should be closed, set only by the
serious error classes: an azimuth/elevation/derotator axis error, a hydraulics error (`fbHydraulics.Error`), the E-stop
(`bEstopTriggered`), a safety error (`bSafetyError`) or a power failure (`bMainReady` FALSE). Not by a focus error, the
software errors (timeouts, config, goto target) or `bError`/READY_STATE -1.0 in general. Latched on the rising edge, so a
short E-stop or phase dip still requests the close; `reset=1` clears it only once no cause is active (an axis error and the
request go with one reset; a hydraulics error needs the hydraulics reset first). A cause already present at the PLC start
sets it too. It is a signal only: the library does not act on it, the site `MAIN` closes the roof on it (behind its enable
switch, only in automatic; not wired yet, step 7 PR 2). It is updated in every automatic scan; in manual
(`SuspendAutomatic`) it keeps its last value. The MQTT watchdog closes the roof on its own, as before.

**Motion limits (`stMotion`, #24).** The input `stMotion : BROTLib.ST_MotionConfig` (BROTLib >= 0.17.0, alt-az only)
holds the move velocities (azimuth/derotator 10.0, elevation 5.0 deg/s), the timeouts (goto/home/slew state machine
600 s, park/gohome/goto/slew command 2 min, idle park, power-on and track 12 h, stop and suspend stop 15 s), the goto
(2.0..89.5 deg) and track (5.0..89.5 deg) elevation windows, the horizon margin (0.2 deg) and the azimuth/derotator track
margin (1.0 deg) and track time cap (14400 s). The defaults are the values that were built in up to 0.11.x, so a `MAIN`
that does not set `stMotion` behaves as before. Internal timings (tracking delay, Wake&Shake window, interrupted
auto-clear, MQTT cadence) stay constants. `BROTLib.F_MotionConfigValid` checks it every scan: an invalid configuration
is **software error 16#32** (like 16#31 for a home/park position below the horizon), the defaults stay in use and a reset
does not clear it until the configuration is valid.
Deprecated for one release (removed in the next minor): `fElevationMoveVelocity`, `fAzimuthTrackMargin`,
`fDerotatorTrackMargin`, `fAzimuthTrackTimeCap`, `fDerotatorTrackTimeCap`. One set to a value other than its old default
(5.0, 1.0, 1.0, 14400.0, 14400.0) wins over the `stMotion` field and is logged once (WARN `deprecated inputs override
stMotion ...`). MONET/S passes `fElevationMoveVelocity := 3.0` and needs no change; move it to
`stMotion.fElevationMoveVelocity` at the next re-pin.

**Stage machines (#24 step 3).** Goto, slew, park and power-on step through the enumerations `E_GotoStage`,
`E_SlewStage`, `E_ParkStage` and `E_PowerOnStage`, whose values are the old stage numbers (the progress log, e.g.
`GoTo progress: 25`, reads as before); the read-only properties `GotoStage`, `SlewStage`, `ParkStage` and `PowerOnStage`
return the current stage. Home (`_HomeTelescope`) has no stage. When a machine aborts is one table, `F_StageAbort` (goto/slew: reset,
error, interrupt or the flag dropped; home: reset, error, interrupt; park: reset, stop, interrupt, not an error, which is
the emergency park; power-on: all of them). The software errors of the machines (16#10, 16#15/16#16, 16#21, 16#30, 16#40) are
raised in one place (`_RaiseSoftwareError`, text in `sSoftwareErrorText`) and latched until `reset=1` as before.

**Telemetry (#24 step 4a).** The MONET fields of the telescope telemetry (`TELESCOPE.READY_STATE`/`MOTION_STATE`/
`INTERRUPTED_AUTO_CLEAR_COUNT`, `WEATHER.REFRACTION.*`, `OBJECT.*`, `POINTING.*`, `POSITION.LOCAL/INSTRUMENTAL/HORIZONTAL/EQUATORIAL.*`,
44 publishes after BROTLib's `_PublishTelemetry`, every 500 ms while moving or tracking, otherwise every second) are published by
`FB_MonetTelescopeTelemetry` from the snapshot `ST_MonetTelemetry` (instance `stTelemetry`, ADS-readable), which the telescope
fills once per publish. Topics, order, formats and conditions are those of 0.12.0.

**Targets (#24 step 4b).** Once per automatic scan, after the error reaction and before the horizon guard, the private
`_ComputeTargets` computes the refraction weather, the commanded target (eq2hor, pointing model, offsets, derotator,
velocities, the turn inside the axis limits) and the reported position (inverse pointing model, hor2eq). The code is that of
0.12.0, moved unchanged out of the body.

`bIsParked` means *secured* (covers closed, brake closed, hydraulics off, axes disabled; with a cover error the covers as far
as they got, reported as 16#21), not *at the park
position*: the emergency park, and a park after a reset or an aborted power-on, secure the telescope where it stands.
`bParkedInPlace` (MQTT `diagnostics/base/ParkedInPlace`, WARN log "parked in place at el .. az ..") is TRUE then; it drops
with the cover command of a power-on and is FALSE after a normal park and after a PLC restart (#158).

### Safety and auxiliary systems

- `FB_MonetSafetyHandling` — handles the TwinSAFE startup sequence and the
  safe-torque-off (STO) reset handshake for all three drive axes.
- `FB_MonetPendantControl` — BCD-selector manual hand pendant. The error lamp shows the selected function's error;
  selector 0 (pendant disconnected) lights it and does nothing else. The horn variable is not wired (nothing sounds).
  The pendant lets go of a function it drives (key off, selector change, selector 0, cable pulled): `MovePos`/`MoveNeg` of that
  axis or focus are set FALSE once, and its `Enable` after `tEnableRelease` (2 s) if the pendant had set it (#3). It writes
  nothing for a function it does not own, so a jog from the Visu keeps working with the pendant key off. The selector must read
  the same for `tSelectionDebounce` (100 ms) before the pendant drives that function.
  Inputs since #24 step 6 (breaking): `fbCovers` (`REFERENCE TO HalfBROT.FB_CoverControl`), `fbFocus : BROTLib.I_Focus`,
  `fbElevation`/`fbAzimuth`/`fbDerotator` (`REFERENCE TO` the HalfBROT axis blocks); `fbBrake` and `fbHydraulics` are gone (never
  read), so the site `MAIN` drops those two lines of the `PendantControl(...)` call. The focus takes the interface because the
  pendant only uses `I_Focus` members; the covers stay the concrete block because the pendant also calls `Stop()` and reads
  `bError`, which `I_MirrorCovers` does not have.
- `FB_MonetCabinetControl` — cabinet buttons/switches/lamps and temperature
  monitoring.
- `FB_MonetPowerMonitoring` — monitors the three-phase supply quality.
- `FB_MonetMqttWatchdog` — safe state when the broker is gone: after `tGrace`
  (30 s) without `bConnected` it calls `Park()` on the telescope and `Close()`
  on the roof every cycle until the connection is back, and publishes
  `electronics/base/MQTTWatchdog=TRIGGERED`. Armed only after the first
  successful connect, so a boot without broker never trips it.
- `FB_MonetDiagnostics` — on-change diagnostics telemetry with a full snapshot
  every `tHeartbeat` (30 s); the measurement is `diagnostics`.

Both are called from each site's `MAIN` (`FB_init` takes `comm`, like the other
blocks).

Hydraulics and focus are not part of this library — both MONETN and MONETS
drive them directly from HalfBROT's `FB_HydraulicsControl` and
`FB_FocusControl`.

---

## Relationship to the other BROT projects

```
BROTLib  ──►  HalfBROT  ──►  MONETcommon  ──►  MONETN / MONETS
 (core)        (hardware)     (MONET logic)     (site applications)
```

- **BROTLib** provides the core telescope abstractions
  (`FB_AltAzTelescopeControl`, `I_Axis`, communication, ...).
- **HalfBROT** provides the Halfmann-mount hardware blocks the MONET axes are
  built on.
- **MONETcommon** adds the MONET-specific control logic and is consumed
  directly by **MONETS** and **MONETN**. Only each site's `MAIN` (location,
  calibration, limits, pointing model, broker, site-only sensors) differs; a fix
  to shared behaviour belongs here, not in a site repository.

**Why the axes are `REFERENCE TO` the concrete HalfBROT blocks (#24, D6).** `FB_MonetTelescopeControl` takes
`fbElevation`/`fbAzimuth`/`fbDerotator` as `REFERENCE TO` HalfBROT's `FB_ElevationControl`/`FB_AzimuthControl`/
`FB_DerotatorControl`, not as BROTLib's `I_Axis`, and `I_Axis` is not widened:
- M needs members `I_Axis` does not carry: the axis limits `fMinPosition`/`fMaxPosition` (azimuth/derotator turn,
  `F_AxisTurn`), `fActualPosition`, and the NC status of the nested `fbAxis` (`InMotion`, `InRamp`, `Homing`, `Stopping`,
  `StopDone`, `StandStill`, `Busy`, `isTracking`) that the stop transaction and the error reactions read.
- Widening `I_Axis` with these would put HalfBROT (Halfmann mount) details into BROTLib, and BROTLib cannot reference
  HalfBROT (the dependency goes the other way).
- Tests do not need an interface: the mock rig of `MONETcommonTests` subclasses the concrete HalfBROT blocks
  (`FB_ElevationMock` etc., body replaced by a motion model), so the real M runs on mocks as it is.

Interfaces are used where the consumer only needs interface members (the pendant's focus, `I_Focus`).

**Pure functions from BROTLib (#24 step 6).** `F_ErrorParkRequest`, `F_GotoTargetCheck`, `F_HomeBelowHorizon`,
`F_TargetBelowHorizonWarning`, `F_TCSCommandText`, `F_ResetMoveAbort` and `F_SuspendStop` are BROTLib's (`BROTLib.F_*`,
`Telescope/Command/`, BROTLib >= 0.17.0, tested in `BROTLibTests`); MONETcommon's copies were removed. BROTLib's
`F_ResetMoveAbort`/`F_SuspendStop` take one `bAnyAxisMoving` (M ORs the three axes). MONETcommon keeps its own
`F_CommandAcceptance`, `F_CommandRejectText`, `E_CommandRejectReason`, `ST_CommandState` and `E_CommandDecision`: they have the
#72 `init_required` row, which BROTLib 0.17.0 does not.

## Dependencies

- **BROTLib**, **AstroBROT**, **HalfBROT** (BROT namespace).
- Beckhoff system libraries: `Tc2_MC2`, `Tc2_Standard`, `Tc2_System`,
  `Tc2_Utilities`.

## Building and deployment

The library is built with TwinCAT 3.1 Build 4024 in TwinCAT XAE as a TwinCAT
library (`.tspproj`, AmsPort 851; solution platforms Debug/Release × TwinCAT
RT (x64/x86), TwinCAT CE7 (ARMV7), TwinCAT OS (ARMT2)) and referenced from the
MONETN / MONETS application projects.

**CI.** `.github/workflows/tcbuild.yml` builds `MONETcommon.sln` with [`tctool`](https://github.com/BROTLib/TcTool) on every push (self-hosted runner, never on pull requests) and runs the TcUnit tests of `MONETcommonTests` on the user-mode runtime (see `MONETcommonTests/README.md`). A green run means the project compiles and the tests passed, or that they were skipped with a warning because the TwinCAT license or runtime is missing; `tctool` exit code 1 (built with warnings) counts as success.

## Startup readiness (READY_STATE)

Startup preparation reports 0.7, calibration and automatic startup positioning
report 0.8, and 1.0 means the complete startup has finished. Internal mechanical
readiness remains separate. An aborted partial startup is not online (-2 remains
the existing unknown/not-operational value); errors and parking retain their
existing values. Normal later slew/tracking remains online.

This change requires BROTLib with `FB_StartupReadiness`. MONET/N and MONET/S
MAIN must call `SuspendAutomatic()` while the cyclic controller is disabled, so
manual mode cannot replay a frozen ONLINE readiness. Preserve their distinct
site calibration/home settings. Validate both sites before rollout.

Publish readiness on changes and at the existing stationary cadence. The new
state uses the existing telemetry field and power-on commands. Update Python
consumers for CALIBRATING/0.8 before firmware rollout. Hardware acceptance and
firmware deployment are separate from building these changes.

Readiness uses the same error predicate in automatic and suspended/manual mode:
axis/controller error, latched software error, power-on failure, or E-stop. A
power-on failure remains visible through reset until the next clean power-on
attempt clears `bPowerFailure`; it does not wait for that startup to complete.
A transient operational interlock gates online without erasing completed startup.
Explicit suspension still aborts readiness and is not changed by the interlock correction.

### Manual to automatic ([#72](https://github.com/BROTLib/MONETcommon/issues/72))

After **any** manual interval (`manual` or `off`) a new explicit Init (`power=1`) is required before the telescope moves
again. There is no silent recovery, a startup is never resumed and nothing queued replays. The first manual scan sets
`InitRequired` (read-only property, `bInitRequired`); only a completed startup clears it (not reset, stop, park, E-stop or an
aborted startup). While it is set, slew, goto, track and gohome are rejected with the log text
`init required after manual mode, send power=1 first` (`E_CommandRejectReason.init_required`), also when they were sent
during manual (the direct flag-write gate uses the same check), and the horizon guard does not start a home. Stop, park, reset and Init stay
accepted. READY_STATE stays -2.0 (0.0 if secured, -1.0 on error) until the Init completes.

The Init **homes the telescope**: it opens the covers, enables the drives, homes only the axes that are not calibrated
and drives elevation, azimuth and derotator to their configured home positions. It does not park. Park is a separate
command (`bIsParked` means secured). A park that was pending at the switch is kept (#118) and resumes when automatic
comes back; this is logged as a warning. Telemetry other than READY_STATE (1 s heartbeat) pauses in manual.

Not part of this change, see the plan `specs/plans/2026-10-27-manual-to-auto-readiness-policy-72.md`: a pose check before
the move to home (Q4), error and E-stop state in manual (Q5, needs a MAIN line per site) and the redesign of the mode
handling ([#168](https://github.com/BROTLib/MONETcommon/issues/168)). Hardware acceptance on MONET/N and MONET/S is separate
(IAG repo, MONET telescope test plan).
