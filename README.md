# MONETcommon

MONETcommon is the **common control library for the MONET-class telescopes** of
the BROT project: it contains the MONET-specific TwinCAT 3 (IEC 61131-3
Structured Text) function blocks shared by the MONET North (MONETN) and MONET
South (MONETS) telescope applications. It extends the BROTLib core and the
HalfBROT hardware layer with the full MONET telescope-control logic: the
complete Alt-Az telescope lifecycle, TwinSAFE safety handling, hydraulics,
pendant control, cabinet I/O, focus, power monitoring and mirror-cover control.

MONETcommon is the **reference implementation** of the MONET control code:
MONETN historically carries vendored copies of these function blocks
(`MONETN/MONETNRuntime/Components/`, `.../POUs/`), while MONETS references the
library directly (placeholder resolution `MONETcommon, * (IAG)`) — see
[BROTLib/MONET_Unification.md](../BROTLib/MONET_Unification.md) for the
file-by-file comparison and the plan to eliminate the duplication (MONETcommon's
versions win by default where the copies have drifted).

The library is versioned in the project file as **0.1** (no release tags yet)
and is built by the Institut für Astrophysik Göttingen (company field `IAG`).

---

## Repository layout

```
MONETcommon/
├── MONETcommon.sln               # TwinCAT solution
├── MONETcommon/
│   ├── MONETcommon.tspproj       # TwinCAT library project
│   └── MONETcommon/
│       ├── MONETcommon.plcproj   # PLC library project
│       └── FB_*.TcPOU            # Function blocks (see below)
└── README.md
```

## Function blocks

| Function block | Description |
|---|---|
| `FB_MonetTelescopeControl` | Full Alt-Az telescope lifecycle — extends `FB_AltAzTelescopeControl` (BROTLib) |
| `FB_MonetSafetyHandling` | TwinSAFE startup/safety handling — E-stop, STO reset for all three axes |
| `FB_MonetHydraulicsControl` | Hydraulic pump/brake system and oil monitoring |
| `FB_MonetPendantControl` | Manual hand pendant (BCD selector + buttons) |
| `FB_MonetCabinetControl` | Cabinet physical I/O — buttons, switches, lamps, temperature |
| `FB_MonetFocusControl` | Focus motor (Faulhaber, 43:1 gear) |
| `FB_MonetPowerMonitoring` | Three-phase power-quality monitoring |
| `FB_MonetCoverControl` | Three mirror covers, sequenced open **1→3→2**, close **2→3→1** (`I_MirrorCovers`) |

### FB_MonetTelescopeControl

The central function block: a complete Alt-Az telescope controller extending
BROTLib's `FB_AltAzTelescopeControl`. It implements the telescope command
interface (`E_TCSCommand`: `PowerOn`, `GoHome`, `GoTo`, `Track`, `Slew`,
`Park`, `Stop`, ...), drives the Az/El/derotator axes through the HalfBROT
`FB_AxisControl`-based blocks, coordinates the mirror covers, focus,
hydraulics and safety, publishes MQTT telemetry and handles error states and
recovery. (In MONETN, the vendored copy is named `FB_MonetTelescopeControl` in
`MONETNRuntime/Components/`; in MONETS the library version is used.)

### Safety, hydraulics and auxiliary systems

- `FB_MonetSafetyHandling` — handles the TwinSAFE startup sequence and the
  safe-torque-off (STO) reset handshake for all three drive axes.
- `FB_MonetHydraulicsControl` — hydraulic pump/brake system with oil
  monitoring and watchdog timers.
- `FB_MonetPendantControl` — BCD-selector manual hand pendant.
- `FB_MonetCabinetControl` — cabinet buttons/switches/lamps and temperature
  monitoring.
- `FB_MonetPowerMonitoring` — monitors the three-phase supply quality.
- `FB_MonetFocusControl` — focus drive (Faulhaber motor, 43:1 gear).

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
  directly by **MONETS**; **MONETN** currently uses vendored copies of the same
  blocks (unification in progress, see
  [MONET_Unification.md](../BROTLib/MONET_Unification.md)).

## Unification status (per `BROTLib/MONET_Unification.md`)

- **Done**: `FB_MonetCoverControl` promoted into MONETcommon from MONETN
  (commit `c277028`, same POU Id); `CoverAutoOpen` hard-locked to TRUE on
  `FB_MonetTelescopeControl` (HEAD `6e4b835`, main/develop).
- **Outstanding**: adopt the MONETcommon library reference in MONETN and delete
  its six vendored copies; unify the `FB_MonetTelescopeControl` divergence
  (pointing-model wiring, `fReadyState`, azimuth wrap, `_PowerOn` staging,
  MQTT topic naming, elevation homing velocity).
- **On `feature/monet-unification` only** (not yet on main/develop):
  `E_ModeLanguage` DUT, restored velocity-aware azimuth wrap, parameterised
  `fElevationHomingVelocity`.

## Dependencies

- **BROTLib**, **AstroBROT**, **HalfBROT** (BROT namespace).
- Beckhoff system libraries: `Tc2_MC2`, `Tc2_Standard`, `Tc2_System`,
  `Tc2_Utilities`.

## Building and deployment

The library is built with TwinCAT 3.1 Build 4024 in TwinCAT XAE as a TwinCAT
library (`.tspproj`, AmsPort 851; solution platforms Debug/Release × TwinCAT
RT (x64/x86), TwinCAT CE7 (ARMV7), TwinCAT OS (ARMT2)) and referenced from the
MONETN / MONETS application projects.
