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
| `FB_MonetCoverControl` | Three mirror covers, sequenced open **1→3→2**, close **2→3→1** (`I_MirrorCovers`) |
| `FB_MonetMqttWatchdog` | Parks the telescope and closes the roof after 30 s without an MQTT broker connection (armed after the first connect) |
| `FB_MonetDiagnostics` | `diagnostics/base/*` telemetry: telescope interrupted/ready/busy/stopped, per-axis error/error ID/enable, safety error/estop/state; on change plus a 30 s heartbeat |

### FB_MonetTelescopeControl

The central function block: a complete Alt-Az telescope controller extending
BROTLib's `FB_AltAzTelescopeControl`. It implements the telescope command
interface (`E_TCSCommand`: `PowerOn`, `GoHome`, `GoTo`, `Track`, `Slew`,
`Park`, `Stop`, ...), drives the Az/El/derotator axes through the HalfBROT
`FB_AxisControl`-based blocks, coordinates the mirror covers, focus,
hydraulics and safety, publishes MQTT telemetry and handles error states and
recovery. Each site passes its own pointing model (`fbPointing`,
`fbPointingInverse`) from `MAIN`.

A new axis or focus error on a telescope that is not parked starts the
emergency park by itself (brake closed, covers closed, hydraulics off,
`bIsParked`); not during an E-stop, not when already parked, and not for
command-level (software) errors. `power=1` is refused until the park is
finished, `stop=1` cancels it; recover with `reset=1`, then `power=1`.
`bIsParked` drops with the cover command of a power-on, so a park during a
power-on closes the covers again. Exception: an azimuth/elevation error in the
first power-on after a PLC restart (the Wake&Shake fault) does not park; the
covers stay open for the manual Wake&Shake (a WARN is logged).

### Safety and auxiliary systems

- `FB_MonetSafetyHandling` — handles the TwinSAFE startup sequence and the
  safe-torque-off (STO) reset handshake for all three drive axes.
- `FB_MonetPendantControl` — BCD-selector manual hand pendant.
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
Explicit suspension still aborts readiness; the manual-to-automatic recovery policy
is tracked separately and is not changed by the interlock correction.

Manual-to-automatic recovery policy: [MONETcommon #72](https://github.com/BROTLib/MONETcommon/issues/72).
The current explicit mode-suspension invalidation is retained pending that decision.
