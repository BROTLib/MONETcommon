# Code review of MONETcommon (develop @ 4073ee6)

**Status: draft. Review finished; the body below describes `develop` at the review commit. Since then fixes for #4 (`_HomeTelescope` guard, `4e9f876`), #5 (Stop outranks power-on, `7b627cc`) and the `Reset()` part of #9 (`a623090`) landed on `develop`, plus a reset input on `FB_MonetSafetyHandling` (`5d959af`, workaround for #26). Nothing has been run on a PLC or a telescope; #29 tracks the check of the #5 fix. Open items: GitHub issues.**

Reviewed at `develop` 4073ee6 (v0.3.2). `origin/main` is 5 commits ahead (visualization profile, two
`Released` flag flips, a manual TcBuild workflow, runner labels); those matter only for the release
section.

Scope: all 8 POUs (`FB_Monet*`, about 3350 lines including the XML wrapper, 1653 of them in
`FB_MonetTelescopeControl`), `E_ModeLanguage`, `Global_Version`, `MONETcommon.plcproj`,
`.github/workflows/release.yml`, README. Consumers (MONETS and MONETN `MAIN.TcPOU`) and the base
classes (BROTLib `FB_BaseTelescopeControl`/`FB_AltAzTelescopeControl`, HalfBROT `FB_FocusControl`,
`FB_HydraulicsControl`, `FB_ElevationControl`) were read only as far as needed to confirm a finding.
The TwinSAFE project is not in the repo and was not reviewed.

## How this was checked (and what that is worth)

- **Static read** of every POU, plus `git log`/`git show`/`diff` against the vendored copies in MONETN
  and the HalfBROT originals.
- There is no numerical library code here, so no Python port and no cross-check against a reference
  (unlike AstroBROT). This is a logic review of state machines and interlocks.
- **Limit:** nothing was run. Findings are marked **verified (diff/git)** (mechanical, reproducible with
  the command), **read from code** (I traced it by hand, no execution), or **unsure** (depends on TwinCAT
  behavior, hardware wiring or the safety project, which I cannot see). Read-from-code findings on
  motion and safety should be reproduced on a simulator or the real controller before anyone acts on them.
- Which blocks are actually used: MONETS `MAIN` instantiates `FB_MonetTelescopeControl`,
  `FB_MonetPendantControl`, `FB_MonetCoverControl`, `FB_MonetSafetyHandling`, `FB_MonetCabinetControl`
  and `FB_MonetPowerMonitoring` from this library, but **HalfBROT's** `FB_HydraulicsControl` and
  `FB_FocusControl`. So `FB_MonetHydraulicsControl` and `FB_MonetFocusControl` are only exercised through
  MONETN's vendored copies (verified by grep).
- Drift against MONETN's vendored copies (verified with `diff`, CR stripped): `FB_MonetPendantControl` and
  `FB_MonetCoverControl` are byte-identical (so their bugs exist at both sites), `FB_MonetTelescopeControl`
  differs by 329 lines, `FB_MonetFocusControl` by 107, `FB_MonetHydraulicsControl` by 28.

## Findings, worst first

### High

**H1. The azimuth and derotator wrap is hard-coded for MONETN's travel range and does not fit MONETS.**
*Read from code, consumer limits from `MAIN.TcPOU`; not run.* Commit 2bdd79b (2026-07-16, "restore
velocity-aware wrap logic") replaced the wrap that was active here (`> 270` / `< -274`, which matches
MONETS) with MONETN's (`> 310 and v > 0`, `> 440`, `< 80 and v < 0`, `< -50`). That maps azimuth into
[-50, 440]. MONETN's axis is limited to [-72, 495], so it fits there. MONETS passes
`fMinPosition := -274`, `fMaxPosition := 270` to `FB_AzimuthControl`, which clamps with `LIMIT`
(`FB_AzimuthControl` line 27).
For MONETS, every target the new wrap places above 270 is silently clamped to 270. Example: a target at
az 50° moving toward north (negative velocity; my inference is that this is the usual case at latitude
-32°) is mapped to 410° and the
axis is sent to 270°. The derotator has the same mismatch (new range [-70, 380], MONETS limit 360).
Whether MONETS already runs this version depends on when it last rebuilt against `MONETcommon, *`; I did
not check. `_TrackTelescope` also hard-codes `fAzimuthCurrent > 440.0`. Fix: derive the wrap window from
the axis limits (or `ST_TelescopeConfig`), not from literals in a shared library.

**H2. `FB_MonetPendantControl` leaves outputs stale and has no interlock with automatic mode.**
*Read from code.*
- `bEnable := IbEnableKey; IF NOT bEnable THEN RETURN;` exits before the `CASE`, so whatever the last
  executed branch wrote (`fbElevation.MovePos`, `fbAzimuth.MoveNeg`, `fbDerotator.Enable`, ...) stays
  set. The same happens when the selector moves to another position while a direction button is held: the
  old branch is no longer executed and its `Move*` stays TRUE.
- In manual mode MONETS does not execute `TelescopeControl` at all, so nothing else clears those
  outputs. A key switched off, a selector change, or a pendant cable pull mid-jog can leave a jog running
  until a limit. The comment in the code even says a disconnect "should also trigger emergency stop", but
  `bError := (nSelection = 0)` is never read.
- `eTelescopeMode` is a `VAR_INPUT`, is not passed by MONETS `MAIN`, and the only use in the body is
  `IF eTelescopeMode = E_TelescopeMode.automatic THEN ; END_IF`. The pendant runs every cycle, so with
  the enable key on during automatic operation it writes the same axis properties as `TelescopeControl`.
- **Unsure:** the enable key and buttons may also be wired into TwinSAFE, which would limit the damage.
Fix: on every cycle, first zero all outputs the pendant owns, then set only the selected branch's. Gate
on mode.

**H3. `_HomeTelescope` has no reset/interrupt guard, and it is called to "reset" it.**
*Read from code; unsure whether a one-cycle `MoveAxis` pulse is enough to start the axes (depends on
`FB_AxisControl`).* `_GotoTelescope`, `_SlewTelescope`, `_ParkTelescope` and `_PowerOn` start with
`IF bReset OR bError OR bInterrupted THEN nStage := 0; ...`. `_HomeTelescope` does not. But the main body
calls it in exactly those situations (`IF bReset THEN ... _HomeTelescope()`, and when Park or Stop
supersedes GoHome). The first statement in its body is `IF bReady AND bStopped AND NOT bBusy AND NOT
bTracking THEN` set home positions and `MoveAxis := TRUE` on all three axes at 10 deg/s. So a Reset on a
healthy, stopped telescope (or a Park issued while GoHome is pending) issues a move to home. Fix: give it
the same guard as the other stage methods and make "reset the stage" a separate method that cannot move.

**H4. `bPower` outranks `bStop` in the command ladder, so Stop does not stop a power-on.** *Read from
code.* `IF bPower THEN TCS_command := poweron ELSIF bStop THEN ...`. While `bPower` is set,
`_StopTelescope()` is never called. `_PowerOn` itself returns early on `bStop` (`nStage := 0; RETURN`),
which restarts the sequence when `bStop` clears but does not stop the axes: `MoveAxis` and `HomeAxis`
stay as last written. The base class documents `bStop` as "Stop motion of telescope immediately". If the
precedence is deliberate (do not interrupt homing), it needs to be documented; otherwise Stop must win.

**H5. Safety startup restarts everything unconditionally and never again.** *Read from code; whether this
is acceptable depends on the TwinSAFE project and a safety assessment I cannot do.*
`FB_MonetSafetyHandling` pulses error-acknowledge (1 s), safety restart (1 s) and the three STO resets
(1 s) after a fixed 2 s delay, on every PLC start, with no check of E-stop state, `inError` or the result.
Afterwards `initialStartUpStep` is never set back, so there is no runtime path to acknowledge or restart
after an E-stop or safety error. MONETS `MAIN` has `IF CabinetControl.IsResetPushed() OR
PendantControl.IbResetButton THEN ; END_IF`, an empty block, so a manual reset button does nothing. Also,
during the first ~5 s `state` is forced TRUE and `estop` FALSE regardless of the real state.
An automatic reset of safety functions from a standard PLC output is normally something a machine-safety
review questions (ISO 13849-1 "manual reset function"; I have not looked up the clause, check before
quoting). If TwinSAFE only accepts the restart on a rising edge after the E-stop is released, the risk is
smaller, but the repo does not show that.

### Medium

**M1. Errors raised by the stage machines are overwritten within one cycle.** *Read from code.* Line 86
assigns `bError` from the axes and focus on every cycle. `_GotoTelescope` (`16#10`, `16#15`),
`_HomeTelescope` (`16#30`) and `_SlewTelescope` (`16#40`) set `bError := TRUE` later in the same cycle, and
the next cycle recomputes it from the axes, then `nErrorID := 0` (line 99). The error lives for the rest
of one 10 ms cycle, while telemetry runs every 500 to 1000 ms. The command is aborted and the TCS is never
told. `nErrorID` for these is never logged either. Fix: latch software errors in their own flag that
`bReset` clears, and OR it into `bError`.

**M2. Nothing parks or closes the covers on error, and the "emergency park" branch is unreachable.**
*Read from code.* `_ParkTelescope` returns at the top when `bError` is set, so `IF bError THEN // emergency
park` in stage 0 (brake close, covers close) can never run. `TCS_command := park` needs `bReady`, which
needs `NOT bError`. Consequence at MONETS: the MQTT watchdog sets `bPark := TRUE` and calls
`RoofControl.Close()` in the same cycle; with any axis in error the park never runs, and the roof does not
wait for `bIsParked`. **Not verified:** whether the roof has its own interlock (MONETRoof was not read).

**M3. `FB_MonetCoverControl` cannot be reset through its own method, has no drive timeout, and has dead
outputs.** *Read from code (the code is identical in MONETN).*
- `METHOD Reset : BOOL ... Reset := TRUE;` writes the method's return value, not `bReset` (compare the
  hydraulics `Reset`, which writes `bReset`). Nothing else sets `bReset` in MONETS, so `srErrorTrigger`
  latches until the PLC restarts.
- `OabOpenCover`/`OabCloseCover` stay on until the end switch reports. There is no run-time limit;
  `afbTimeoutEvent[1..3]` are stubs with `Trigger := FALSE` ("handle RoboTel differently"). A failed end
  switch or a jammed cover keeps the output driven.
- `bWarning` is never assigned (its event block is dead), `fTelemetryInterval` is an input but `tonComm`
  takes `PT` from it only at initialization, and `bOpen`/`bClose` are inputs that the block writes.
- The "drive error" text is raised when both end switches are active, which is a switch fault, not a
  drive fault.

**M4. `FB_MonetPendantControl` cover control cannot open or close the covers.** *Read from code; the code
is identical in MONETN.* `fbCovers.bOpen := IbDirectionUp AND fbCovers.Opened` and
`bClose := IbDirectionDown AND fbCovers.Closed` only fire when the covers are already open (closed). The
elevation/azimuth branches use the same pattern with "limit switch not active", so this looks like a
copy with the wrong sense (`NOT Opened` was probably meant). The `ELSE` branch that calls
`fbCovers.Open()/Close()` is unreachable, because `IbEnableKey` is TRUE at that point (see H2). Selector
positions 1 to 3 all drive the same three-cover sequence, not covers 1 to 3 individually.
Smaller items in the same block: `ObLampError := ObLampError;` is a no-op, `fbBrake`/`fbHydraulics` and
the outputs of three `R_TRIG`s and `bHorn` are unused, the derotator branch uses `NOT inDigitalInputs.n` while elevation
and azimuth use the raw input (**unsure**, may be hardware), `__ISVALIDREF` is checked for the focus only.

**M5. `bEstopTriggered` and `bMainReady` are not wired at either site.** *Verified by grep on both
`MAIN.TcPOU`.* The base declares `bEstopTriggered := FALSE; // TODO: CHANGE` and `bMainReady := TRUE`.
Neither `MAIN` passes `SafetyHandling.estop` or `PowerMonitoring.phaseOK` (`ready` in MONETS is
only published). So `STATUS.GLOBAL = 1` (E-stop) is never sent, the "disable axes on E-stop" block never
runs, and power quality never gates `_PowerOn`. `FB_MonetPowerMonitoring.phaseOK` also ignores the
`*GuardError`, `QualityGuard*` and `UnbalanceGuard*` flags, which may be intentional (**unsure**).

**M6. `fElevationHomingVelocity` is applied to slew, not homing.** *Verified (git).* The input comment and
commit d946e5a say it replaces a hard-coded 5.0 in `_HomeTelescope`. The diff shows the 5.0 was in
`_SlewTelescope` (line 1020 now); `_HomeTelescope`, `_PowerOn`, Goto and Park still use literal `10.0`.
MONETN's "10.0 homing velocity" therefore already matches what the code does for homing, and passing
`10.0` to this input would change its slew speed. Fix the comment or move the input to the method it was
meant for. There are 17 literal `Velocity := 10` assignments in the file.

**M7. Wrong elevation in the azimuth-offset term, and a division by `COS(elevation)`.** *Read from
code.* `fAzimuthOffset / COS(fElevationCalc * d2r)` is used in three places where the elevation should be
the one belonging to the value being converted: `_SlewTelescope` stage 25 uses `fElevationCalc` (the
elevation of the last RA/Dec target, unrelated to an alt/az slew target `fElevation`); the two `hor2eq`
calls and the telemetry use `fElevationCalc`/`fElevation` where the current elevation is meant. The error
is `offset * (1/cos(a) - 1/cos(b))`, zero when the offset is zero. At elevation 90° `COS` is about 6e-17,
so a non-zero offset gives a huge azimuth; only the lower elevation bound is checked in Goto (`< 2.0`).
Also `eq2hor` uses `fJd + fTimeOffset` but `hor2eq` uses `fJd`, so the reported RA/Dec differ from the
commanded ones by the time offset when it is not zero (**unsure** whether that is intended).

**M8. Null-reference handling is inconsistent.** *Read from code.* `fbFocus <> 0 AND_THEN ...` guards
appear at four places, but `bHomed` (`fbFocus.Calibrated`), `bBusy` (`fbFocus.Busy`) and `_PowerOn` stage
10 dereference it unguarded, so "optional focus" does not work: every cycle hits `fbFocus.Calibrated`.
`fbCovers`, `fbBrake`, `fbHydraulics` (interfaces) and the `REFERENCE TO` axes are never checked. Every
block stores `comm` in `FB_Init` and calls `fbComm.Publish(...)` without a check, so instantiating without
`comm :=` faults at the first publish. Decide which inputs are mandatory, check them once, and fail with a
message.

**M9. `FB_MonetHydraulicsControl` is an older fork of HalfBROT's `FB_HydraulicsControl`.** *Verified
(diff); it shares POU Id `{0d9c5732-...}` with HalfBROT's.* Only used through MONETN's copy.
- HalfBROT has a dominant `bCloseBrake` that survives a later `OpenBrake()` in the same cycle; this copy
  clears `bOpenBrake`, which any later `OpenBrake()` call re-sets (`FB_ElevationControl` calls it every
  cycle while enabled). Order dependent.
- *Read from code:* `bError`, `bOilLow` or loss of pressure does not reset `rsBrakeState` directly; the
  brake only closes when the pump-running feedback drops. If the pressure input fails but the pump keeps
  running, `bBrakeOpen` stays TRUE until the 30 s pressure watchdog trips.
- `bOilHot := IbOilHot` is normally-open sense, while `bOilLow`, `bOilCold` and `bOilFilterDirty` are
  inverted (NC). An open circuit on the temperature sensor is not detected (**unsure**, wiring).
- `tonHydraulicsWatchdog` raises `bHydraulicsFailure` when the main pump runs for 140 s with the suction
  pump idle. The suction pump is started only when the pan is full, so a normal run could trip this
  (**unsure**, depends on whether the suction pump cycles on its own).
- Suction stops only when `fOilLevel < fMinPanPercent`; default `0.0` with `F_YREAL(..., cut := FALSE)`
  means "reading below the calibrated empty point", which a normal level never reaches (**unsure**, MONETS
  passes `0.0` too and uses HalfBROT's copy).
- Header comment says the suction runs 60 s and the pressure watchdog is 10 s; the code has no 60 s timer
  and uses 30 s. `OpenBrake`/`CloseBrake` use `RETURN(bBrakeOpen)`, which is not standard ST; it builds, but
  **unsure** what it returns, check online.
- `IbOilHigh` has no address comment ("?").

**M10. `FB_MonetFocusControl` dropped the brake delays and the auto-lock of HalfBROT's `FB_FocusControl`,
and its stored position can go stale.** *Read from code plus diff; only used through MONETN.* The Monet
body has no `tonFocusDelay`/`tonBrakeDelay` and never disables the focus after standstill, so the brake
stays released after power-on. `fLastPposition` (PERSISTENT in the base) is refreshed only while the
focus is locked (`NOT ObFocusUnlock`) and at `HomeDone`, so it can hold the homing position while the focus
sits elsewhere. After a warm restart `MC_ForceCalibration` then writes that stale value. On error it is
set to -1, which forces a real homing (fine).

**M11. Method-local timers and the timeout conditions.** *Unsure.* `_HomeTelescope` uses
`commandTimeout(IN := NOT _HomeTelescope, ...)`. At the start of a call the return variable is FALSE, so
`IN` is always TRUE and completion cannot reset the timer. `_GotoTelescope` and `_SlewTelescope` use
`IN := nStage < 100`, which is also TRUE while idle at stage 0. If the `TON` instances in method `VAR`
persist between calls, they expire 600 s after the first call and every later call errors immediately;
if they are re-created on each call, the timeouts can never fire. Either way it is not what the code
intends. I do not know which applies in TwinCAT 3.1.4024 (`VAR_INST` exists for this reason); check online.
Same question for the `FB_Eventlog` instances declared in methods.

**M12. Two AstroBROT findings reach the pointing here.** *Cross-reference, see
`../AstroBROT/specs/plans/2026-09-20-code-review.md`.* `eq2hor` is called without pressure or
temperature, so AstroBROT H1 (Kelvin default fed as Celsius, refraction about 10 % too small: 56" at 5°,
10" at 30° of altitude) applies to every MONET pointing. UT1-UTC (AstroBROT M5, up to about 13.5" of RA)
is also not corrected. The pointing model may absorb the constant part but not the altitude dependence.
AstroBROT H3 (default `refract_to_observed` direction) does **not** apply: `eq2hor` passes `TRUE` and
`hor2eq` passes `FALSE` explicitly, which is the correct direction for computed and observed altitudes.

**M13. Telemetry has small errors and unchecked sends.**
- *Verified:* `fHourAngleCurrent` is published as `POSITION.EQUATORIAL.HA_ICRS` but never assigned (the
  `hor2eq` calls do not map `ha =>`), so it is a constant 0.0. The topic `OBJECT.EQUATORIAL.DEC_IRCS`
  (line 869) is a typo of `DEC_ICRS` (the only occurrence in all sibling repos).
- *Read from code:* `_SendTelemetry` makes 67 `Publish` calls (plus the base class's) per 500 ms while
  moving, none checks the return value, and
  `FB_Comm_MQTT_Influx.Publish` sends with `bQueue := FALSE`, QoS 0. **Unsure** whether the client drops
  messages in a burst; log `published` for a while to find out.
- The cadence comments contradict the values (`T#500MS // every second`, `T#1000MS // every five seconds`;
  base class uses 1000/5000). `_SendTelemetry` is a new method next to the base `_PublishTelemetry`; the
  body does not call `SUPER^()` and reimplements its timer.

**M14. Release, branches, docs and tests.**
- *Verified (git):* `origin/main` has 5 commits that `develop` lacks (0 the other way), same situation as
  AstroBROT M8. `release.yml` pushes the bump to `develop`, then runs `git merge --ff-only develop` on
  `main`; that fails and leaves the bump on `develop`, no tag. Also `<Released>` differs (`true` on
  develop, `false` on main after a TcBuild flip), so plan the merge.
- README says the library is "versioned as 0.1 (no release tags yet)"; the repo has v0.2.0 to v0.3.2. The
  layout section misses `Global_Version` and `E_ModeLanguage`. README says MONETS consumes the library
  directly and lists all eight blocks, but MONETS does not use `FB_MonetHydraulicsControl` or
  `FB_MonetFocusControl` (see above).
- There are no tests. TcUnit is not referenced. `E_ModeLanguage` is unused in this library (only MONETN
  copies it).

### Low

- Typos in identifiers and messages: `AreBreakesCleared`, `fLastPposition` (base class),
  `'Safety Error! Emergency stop enganged!'`, `'occured'`. Renaming a public method needs a deprecation
  step.
- `CoverAutoOpen` setter ignores its input "as a hard safety invariant", but `bCoverAutoOpen` is a plain
  `VAR_INPUT` that a caller can set, and `_PowerOn` does not read it (it always calls `fbCovers.Open()`).
  The invariant holds because of the second fact, not the first.
- `TCStargetHorizonEvent` triggers on `bPower AND fElevationCalc < fHorizon`, so it only fires during
  power-on. `TCSreadyEvent` says `'STELLA1 startup finished'`.
- `IF fbElevation.Calibrated AND fElevationCurrent < fHorizon + 0.2` starts Stop/GoHome; a park or home
  elevation at or below `fHorizon + 0.2` would loop (**unsure**, config dependent, MONET values are 40°/60°).
- `_SlewTelescope` continues after its timeout (no `RETURN`, unlike Goto) and after reset falls through to
  stage 0, which can advance to stage 10 in the same call.
- `FB_MonetCabinetControl`: `tempCritical` (> 60 °C) only logs, nothing acts on it. `KeyLocal` and
  `IsKeyOnManual` duplicate each other.
- `FB_MonetFocusControl`: `bCalibrated := FALSE;` on error is overwritten by the next line;
  `MC_SetAcceptBlockedDriveSignal` result is ignored; a stored position of exactly 0.0 counts as "no
  stored value" (`fLastPposition > 0.0`).
- Hydraulics: `bOilPanMaximum` is logged at `ERROR`, but the oil drains back to the pan after every
  shutdown (**unsure**, may be noise); a pump running while not commanded (welded contactor) is not
  detected.
- `FB_MonetSafetyHandling` and other blocks declare `AT%I*`/`AT%Q*` inside the FB, so each instance
  needs its own hardware mapping and a second instance collides. Fine for one per PLC, worth a comment.

## Security

Nothing here is exploitable remotely by itself; the library does no I/O beyond MQTT via a consumer's
`fbComm`. Items are hygiene.

- **S1 (Low). Script injection in `release.yml`.** The "Validate version format" step puts
  `${{ inputs.version }}` inside a shell command, so `$(...)` runs before the regex check. Needs dispatch
  rights (write access, and the job has `contents: write`). Fix: `env: VERSION: ${{ inputs.version }}` and
  use `"$VERSION"`. The later steps interpolate too but run after validation.
- **S2 (Low-Medium). The release job pushes to `develop` and `main` and tags with no build or test
  gate**, and `actions/checkout@v4` is pinned by tag, not SHA. Branch protection on `main` would block the
  fast-forward push, so check whether it is set; the workflow should not need to bypass it.
- **S3 (Low). Public repo.** *Verified with `gh repo view`: visibility PUBLIC.* Text grep for
  password/secret/token/API key and IPv4 addresses found nothing. No binaries besides `.tmc` (a one-line
  XML) and `_Boot/TargetDescription.xml` are tracked, unlike AstroBROT. Hardware wiring notes (DIN
  numbers, motor and brake part numbers) are in comments; harmless but public.
- **S4 (out of scope, unverified). MQTT command channel.** MONETS `MAIN` subscribes to
  `MONETS/Telescope/SET` on a plain broker (`192.168.127.10:1883`, no credentials visible in the call) and
  that topic reaches park, goto, slew, reset. I did not inspect the broker configuration. If the broker
  has no ACL, anyone on that network segment can command the telescope. Also note that address is in
  MONETS's `MAIN.TcPOU` (that repo is public too, checked with `gh`).

## Design assessment

1. **The library is not yet a reference implementation.** Two of eight blocks are unused by MONETS, the
   telescope block differs from MONETN's by 329 lines, and the July unification moved MONETN's literals
   into a library that MONETS also consumes (H1). Until both sites build against the same version and the
   site differences are inputs, "MONETcommon wins" moves bugs between sites.
2. **`FB_MonetTelescopeControl` is a 1653-line block with hand-written stage machines** (`VAR_STAT
   nStage`, shared flags, precedence encoded in an `IF/ELSIF` ladder). The guard set differs per method
   (H3), error latching is per method and undone by the caller (M1), and stop precedence is implicit (H4).
   One stage-machine pattern (a base method with the reset/interrupt guard, a latched error, an
   enumerated stage) would remove that class of bug. Splitting into command, motion, and telemetry parts
   would make it testable.
3. **Inputs used as mutable state** (`bOpen`, `bClose`, `bReset`, `bEnable`, `bOpenBrake`, `bPark`, ...).
   A caller passing them by parameter each cycle and the block clearing them internally fight each other
   (same class as AstroBROT M1). Use methods or edge-triggered commands and keep inputs read-only.
4. **Site constants in code.** Wrap limits, `10.0` velocities, `12H`/`600S` timeouts, `fHorizon + 0.2`,
   `5.0`/`89.5` tracking limits, and `440.0` belong in `ST_TelescopeConfig` or inputs.
5. **Coupling is uneven.** The telescope block takes interfaces for focus, covers, brake, hydraulics, but
   `REFERENCE TO` concrete FBs for the axes; the pendant takes concrete FBs (and `fbHydraulics`/`fbBrake`
   typed as HalfBROT's hydraulics, not `I_Hydraulics`/`I_Brake`). Because the two Monet hydraulics/focus
   blocks are forks of HalfBROT ones with the same POU Id, changes need to be made twice.
6. **Failure behavior is mostly "stop responding".** Error latches exist (hydraulics, covers) but few
   safe-state actions follow from them (M2, M3). For a telescope with an imbalanced elevation axis and a
   roof, the safe state after each error class should be written down and tested.
7. **Checked and consistent:** `eq2hor`/`hor2eq` refraction direction (M12), the cover sequencing logic
   1 to 3 to 2 open and 2 to 3 to 1 close against its comment, the hydraulics latch structure (set-dominant
   error `SR`, reset-dominant pump and brake `RS`), the version bump in `release.yml` against
   `Global_Version.TcGVL` and the `.plcproj` (all three at 0.3.2).

## CI and tests

`origin/main` has a manual `tcbuild-test.yml` on the shared self-hosted runner (`[self-hosted, twincat,
windows]`); it is not on `develop`. Same tooling constraints as in the AstroBROT review's CI section
(TcBuild does not run tests, TcUnit-Runner is archived, unattended test licenses are unverified), so I do
not repeat them. For this library the useful test targets are the pure-logic parts:
- the command ladder (which command wins for each flag combination, incl. H4);
- each stage method under `bReset`/`bError`/`bInterrupted` (H3);
- cover sequencing and its error latch, hydraulics brake/pump interlocks;
- the wrap function against each site's axis limits (H1).
These need a runtime (TcUnit on a test target) or, as a cheap regression net, a Python port of the
ladder and wrap that is written from the ST. A port tests the port, not the ST, so it should not count as
coverage.

## Suggested order of work

1. H1 (check what MONETS runs today, then derive the wrap from the axis limits), H3, H4. Small changes,
   real motion risk.
2. H2 (pendant: zero outputs each cycle, mode interlock, fix the cover sense in M4).
3. H5 and M5: decide with whoever owns the TwinSAFE project what the restart and reset flow should be;
   wire `estop` and `phaseOK` into the telescope block at both sites.
4. M1, M2, M3 (latched software errors, an error path that reaches a safe state, cover reset and timeout).
5. M14: merge `main` into `develop`, fix `release.yml` (S1), add a build gate (S2), correct the README.
6. M11 (check the method-local `TON` behavior online), then M6 to M10 and M12 to M13.
7. Decide whether MONETS moves to `FB_MonetHydraulicsControl`/`FB_MonetFocusControl` or those two are
   dropped from this library.
