# Step 4c - client check before read-only command inputs (issue #24 plan, D3)

Date: 2026-11-02. Read-only investigation for step 4c of
`2026-11-02-reference-implementation-issue-24.md`. No code changed.

Question: before step 5 (M stops writing `bPower`, `bGoHome`, `bPark`, `bTrack`, `bGoto`, `bSlew`, `bStop`, `bReset`;
inputs become edge-triggered, methods set request latches), who writes these inputs directly and who reads them back
as status?

## Sources read

| Source | Version |
|---|---|
| pybrotlib = `BROTLib/pyBROT` (`src/pybrotlib`) | `ef233b0` v1.7.1 (shallow clone via `gh`, no local checkout) |
| pyobs-brot = `pyobs/pyobs-brot` | `ead7024` v2.4.4 (same) |
| brotgui = `BROTLib/BROTgui` | `9dbb448` v0.6.1 (same) |
| `BROTLib/BROTweb` (TwinCAT HMI, private, not in the D3 list) | `d07f64e` 2026-09-20 (same) |
| `BROTLib/dummyBROT` | `a58a9d8` (same; no hits) |
| BROTLib | `origin/develop` `e92e48c` 0.17.0 |
| MONETcommon | `origin/develop` `644dce5` |
| MONETS, MONETN, HalfBROT, MONETRoof, IAG, IAG50cm, TcTool | local checkouts under `Documents/brotlib` |

Search: the eight symbol names (bare and dotted) across all checkouts and clones, including `.TcVIS`, TcHMI `.content`
and `TcHmiSrv.Config.default.json`, PowerShell ADS tools, Python sources; `pyads`/ADS imports in the Python repos;
`Publish*` calls carrying the flags.

## Findings

### (1) How pybrotlib sends commands: MQTT methods only

- Every command is one MQTT publish (QoS 1, waits for PUBACK) to `<site>/Telescope/SET`:
  `components/telescope.py:99-107` (track: RA, Dec, `track=1`), `:109-116` (move: elevation, azimuth, `slew=1`),
  `:210-214` (`power=1`/`power=2`), `:216-217` (`stop=true`), `:219-220` (`park=true`), `:222-223` (`reset=1`);
  transport `transport/mqtttransport.py:55-68`. No `gohome` method exists in pybrotlib.
- No ADS path at all: no `pyads` or other ADS dependency or import in pybrotlib, pyobs-brot or brotgui; none of the
  eight symbol names occurs in any of the three repos.
- PLC side: `FB_Comm_MQTT_Influx._ApplyField` (BROTLib `POUs/Comm/FB_Comm_MQTT_Influx.TcPOU:124-162`) maps every verb
  to a method of `I_Telescope`: `power` -> `Init()` (`:124-134`), `park` -> `Park()` (`:135-136`), `gohome` ->
  `Home()` (`:137-138`), `track` -> `Track(ra, dec)` (`:139-149`), `slew` -> `Slew(az, alt)` (`:150-158`), `stop` ->
  `Stop()` (`:159-160`), `reset` -> `Reset()` (`:161-162`). The flags are written only inside M's methods (MONETcommon
  `FB_MonetTelescopeControl.TcPOU:2387, 2405, 2419, 2552, 2569, 2587, 2606, 2619`, `Reset` sets `bReset`).
- pyobs-brot and brotgui call only these pybrotlib methods: pyobs-brot `pyobs_brot/brottelescope.py:240, 246, 315,
  348, 364`; brotgui `BROTgui/qtelescopecontrol.py:233, 237, 243, 254, 361-363, 370-372, 403`,
  `BROTgui/stellarium.py:97`. brotgui imports `aiomqtt` only to receive log messages (`BROTgui/main.py:26-30`).

### (2) Holding a flag / re-triggering by writing again

- No Python client holds a flag; they cannot, they never see one. Each user action is one MQTT verb, so each becomes
  one method call.
- No timer re-send of a command verb. pyobs-brot re-sends only absolute setpoints (offsets, focus) via
  `wait_until_settled(resend=...)` (`brottelescope.py:286-291, 415-429, 472-487`); `_settle.py:36-38` states this is
  "only safe for idempotent absolute-setpoint commands, not one-shot triggers". `init()` deliberately does not send
  `power` again while READY_STATE is 0.7/0.8 (`brottelescope.py:309-315`).
- Repeated verbs do occur and must stay commands: a new `track` while already tracking (next pyobs target; every
  Stellarium goto in brotgui, `stellarium.py:97`), a second `stop` (M's `Stop()` handles it, #96). With the methods
  setting the step-5 request latch directly this is unaffected, as long as the latch accepts a method call while the
  same command is active (today `Track()` writes `bGoto := TRUE` over an already TRUE `bGoto`, `:2587`).

### (3) Reading the flags back as status

- None of the Python clients. pybrotlib's telemetry model has no flag fields (`telemetry.py:22-35`); status comes from
  `TELESCOPE.READY_STATE` (`telescope.py:158-174`), `TELESCOPE.MOTION_STATE` (`:176-186`), `TELESCOPE.STATUS.GLOBAL`
  (`:188-204`), plus `POSITION.INSTRUMENTAL.*.TARGETDISTANCE` and `Telemetry/status` (last will). pyobs-brot polls
  READY_STATE/MOTION_STATE (`brottelescope.py:153-176, 222-236, 248-258, 316-333, 349-360, 365-376`); brotgui uses
  `telescope.status` (`qtelescopecontrol.py:247, 320, 355`).
- The PLC does not publish any of the eight flags (no `Publish*` call carries them); `FB_MonetDiagnostics` reads only
  outputs (`bInterrupted`, `bReady`, `bBusy`, `bStopped`, `bWakeShakeWindow`, `bParkedInPlace`).
- Readers that do exist: the MONET TwinCAT Visu switches below (they display the flag they toggle), and the IAG ADS
  debug tools `IAG/tools/Ads-Watch.ps1:34`, `Ads-Capture.ps1:37`, which read `MAIN.telescopeControl.b*` on IAG50cm only
  (`$t`, `Ads-Watch.ps1:32`), not on MONET.

### (4) Other direct writers of the eight symbols

| Where | What | Effect of step 5 |
|---|---|---|
| MONETS and MONETN `VISUs/Telescope.TcVIS:666` (identical in both) | `VisuFbElemPowerSwitch`, mode `TOGGLE`, variable `MAIN.TelescopeControl.bPower` | **Breaks.** Today a click sets TRUE and M clears it, so the switch acts as a button and shows the request. With M no longer writing, the switch stays TRUE: every second click only writes FALSE (no command), and it shows a stale request instead of a status. |
| same files `:768`, `:1092` (`bPark`), `:2079` (`bSlew`), `:6577` (`bGoto`) | `VisuFbElemPushSwitch`, mode `TOGGLE` | **Breaks**, same as above. The Visu also writes the targets `fAzimuth`/`fElevation`/`fRightAscension`/`fDeclination` directly (`:1826, 2554, 6324, 6871`), so "methods only" would need the Visu changed too. |
| BROTweb (TcHMI) `Pages/Telescope.content:126-141`, `Widgets/MoveHorizontal.content:269-277`, `Widgets/MoveEquatorial.content:270-278`; symbols mapped in `Server/TcHmiSrv/TcHmiSrv.Config.default.json:714-752` | `onPressed` -> `WriteToSymbol` TRUE to `ADS.PLC1.MAIN.TelescopeControl.bPark` / `.bSlew` / `.bTrack` (after writing the targets); never writes FALSE | **Breaks if deployed.** It relies on M clearing the flag; after step 5 only the first press is an edge, later presses do nothing until someone writes FALSE. Deployment status unknown (not in the D3 list, last commit is a `.gitignore` change). Also writes `bTrack`, not `bGoto`, so it uses the level meaning of `bTrack`. |
| MONETS/MONETN `VISUs/Visualization.TcVIS` (e.g. `:19, 199, 755, 1033, 1677, 1857, 2135`) | legacy STELLA1 toggles on `TelescopeControl.power`, `.gohome`, `.goto`, `.park`, `.track`, `.slew`, `.stop`, `.reset` | Not affected: these names match no declared symbol (no global `TelescopeControl`, M has no `power`/`goto` members). Dead screen; candidate for deletion. Not checked whether the Visu compiler reports them. |
| HalfBROT `POUs/FB_PendantControl.TcPOU:239-246, 251-254` | level writes from pendant buttons: `bGoHome := direction_down`, `bPark := direction_up`, `bGoto := direction_right`, `bStop := direction_left`, `bReset := reset_switch`; `bTrack`, `bPower` via `FB_ButtonEnable` latches | Not affected by step 5: it references `FB_AltAzTelescopeControl` (`:13`), and MONETS/MONETN use MONETcommon's `FB_MonetPendantControl` (MONETS `MAIN.TcPOU:30, 196-204`), which does not touch the telescope. Relevant for the BROTLib doc-comment change and SM 5.4: released buttons would no longer stop/untrack. |
| MONETS / MONETN `MAIN` | only `TelescopeControl.Reset()` (MONETS `MAIN.TcPOU:102`, MONETN `:94`) and `SuspendAutomatic()`; watchdog and cabinet use `iTelescope.Park()` | Not affected. |
| MONETRoof, TcTool, dummyBROT, IAG50cm production code | no access to M's flags | Not affected. |
| Runbooks in MONETcommon plans | manual ADS/XAE writes: `MAIN.TelescopeControl.bStop := FALSE` as recovery (`2026-10-20-stop-during-poweron-hang-96.md:29, 269`), `TelescopeControl.bSlew := TRUE` once as a test (`2026-10-20-command-acceptance-83-87.md:567`) | Procedures change: an ADS write of TRUE is a one-shot edge and needs FALSE written before the next one; the `bStop := FALSE` recovery no longer applies. |

## Verdict

**Safe with changes.** pybrotlib, pyobs-brot and brotgui are not affected: they send MQTT verbs that the PLC turns into
method calls, never write or read the flags, do not hold or re-send command verbs, and take status from READY_STATE,
MOTION_STATE and STATUS.GLOBAL. No client release is needed for step 5.

Step 5 needs these changes in the same release:

1. MONETS and MONETN `Telescope.TcVIS`: the five `TOGGLE` switches on `bPower`, `bPark` (2x), `bSlew`, `bGoto` must
   change, either to TAP mode (TRUE while pressed, FALSE on release: one clean edge per click) or, better, to MAIN
   helper bits that call the methods (the `bVisuHome*` pattern of MONETS#72, MONETS `MAIN.TcPOU:32-37, 106-122`).
   Lamps showing state should use the outputs (`bBusy`, `bSlewing`, `bTracking`, `bIsParked`, already on the screen,
   or the new `b*Active` outputs).
2. BROTweb: either confirm it is not deployed (and leave it), or change its three buttons to write TRUE then FALSE,
   or to go through MQTT. Ask Tim.
3. Update the two runbook passages above.

Because the Visu (and BROTweb, if alive) write flags today, "methods only" is not possible without changing them;
step 5 as written (edge-triggered inputs plus methods) keeps them working once item 1 makes each click one edge.
Optional cleanups, not blocking: delete the dead `Visualization.TcVIS`; note for SM 5.4 that HalfBROT's
`FB_PendantControl` writes the inputs as levels.
