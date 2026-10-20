# Azimuth/derotator wrap from the axis limits, not from literals (#2)

Date: 2026-10-20. Status: planned, not started.
Repo: BROTLib/MONETcommon. Base: `origin/develop` @ `badcbf0`.

Main file: `MONETcommon/MONETcommon/FB_MonetTelescopeControl.TcPOU`.
Source: code review H1 ([2026-09-20-code-review.md](2026-09-20-code-review.md)). Everything below is a static
read; nothing reproduced on a PLC or a telescope.

## 1. Problem
The body wraps the computed targets with literals tuned for MONETN (lines 236-246):

| Axis | Wrap in code | Result window | MONETN limits | MONETS limits |
|---|---|---|---|---|
| Azimuth | `> 310 AND v > 0` or `> 440` -> -360; `< 80 AND v < 0` or `< -50` -> +360 | [-50, 440] | [-72, 495] | [-274, 270] |
| Derotator | `> 290 AND v > 0` or `> 380` -> -360; `< 20 AND v < 0` or `< -70` -> +360 | [-70, 380] | [-170, 300] | [-90, 360] |

Limits from `MAIN.TcPOU` of MONETN (lines 135-141) and MONETS (lines 137-143), both `origin/main`.
`FB_AzimuthControl` (line 26) and `FB_DerotatorControl` (line 28) in HalfBROT clamp with
`LIMIT(fMinPosition, fPosition, fMaxPosition)`, so a target outside the limits is moved to the endpoint
without any error.

- MONETS azimuth: az 50 deg with negative velocity -> 410 -> clamped to 270, about 220 deg off.
- MONETS derotator: targets in (360, 380] pass the wrap and are clamped to 360, a wrong field angle.
- **New, not in the review:** MONETN's derotator limits are [-170, 300], so targets in (300, 380] are clamped
  on MONETN too. *Unsure* whether they occur in practice (depends on `fDerotatorSign` and `fDerotatorOffset`).
- `_TrackTelescope` (lines 1391-1396) stops tracking at `fAzimuthCurrent > 440.0` and at derotator
  -69 / 379 (with the velocity sign). MONETS never reaches 440; the derotator values are beyond the limits
  of both sites.
- *Read from code, unsure:* the wrap runs every scan, also while tracking. If the velocity sign or the
  thresholds flip during a track, the target jumps by 360 deg (MONETN: v > 0 crossing az 310 -> -50).
- `_SlewTelescope` sends the user's azimuth (plus offsets) without any wrap, so it is clamped too.

The derotator has real hard limits: beyond them the cables are ripped off. The limits are measured on
site by moving the axis to its safe endpoints.

Open: whether MONETS already runs a build with the 2bdd79b wrap (it references `MONETcommon, * (IAG)`).

## 2. Requirements
- The per-site axis limits (`fMinPosition`/`fMaxPosition` of the axis FBs) are the single source of truth.
  No travel-range literals in the shared library.
- A target is never silently clamped to a different position than requested. No valid position -> refuse.
- No safety margin below the measured endpoints, unless it is a named config value.

## 3. Proposed design (proposal, not decided)
**Pure function** like `F_HomeBelowHorizon`, testable with TcUnit:

```
FUNCTION F_AxisTurn : BOOL          // TRUE = a valid position exists
VAR_INPUT
    fTarget    : LREAL;   // target, degrees, any turn
    fVelocity  : LREAL;   // sign of the expected motion
    fMin, fMax : LREAL;   // axis limits (endpoints inclusive)
END_VAR
VAR_OUTPUT
    fPosition  : LREAL;   // chosen turn, only valid if TRUE
END_VAR
```
- Candidates `fTarget`, `fTarget + 360`, `fTarget - 360` (normalise `fTarget` into [0, 360) first so three
  candidates are always enough).
- Keep the candidates inside `[fMin, fMax]`.
- None -> FALSE. One -> that one. Two -> prefer by velocity direction, so a track does not run into an
  end soon: `v > 0` the lower one (more room up), `v < 0` the higher one. `v = 0`: see question 3.
- Name and signature are a proposal; maybe one function per call with a `bValid` output instead.

Note: with zero margin both sites span more than 360 deg on both axes (MONETS az 544, MONETN az 567,
derotators 450 and 470), so "no valid turn" cannot happen today for the target itself. It becomes
reachable with a margin, with a narrower future config, and for tracking near an end.

**Use in `FB_MonetTelescopeControl`:**
1. Replace lines 236-246 with two calls, limits taken from the axis (question 4).
2. Goto (`_GotoTelescope`, next to the 0x15 / 0x16 checks): no valid turn -> error, `bGoto := FALSE`,
   no move. Proposed codes 0x17 (azimuth) and 0x18 (derotator), unused in MONETcommon, MONETN, MONETS today.
3. Slew: check `fAzimuth + offsets` against the limits (and choose a turn, or not: question 3); reject
   instead of clamping.
4. Tracking: keep the turn chosen at goto time (pick the candidate nearest the current axis position), do
   not re-choose by velocity every scan. Stop tracking with a warning when that candidate leaves
   `[fMin + fTrackMargin, fMax - fTrackMargin]` in the direction of motion. This replaces 440 and -69 / 379.
   `fTrackMargin` is a named value (default 0, or set per site), see question 2.
5. The axis FBs keep their `LIMIT` as a last line of defence; it should never act after this change.
   Optionally an event if it would have changed the position (HalfBROT, separate issue).

## 4. Out of scope
- #77 (stale target raising 0x15): fixed by #91, not touched here.
- Slew derotator target `fElevation + fDerotatorOffset` (line 1285) looks wrong, but is a separate finding.
- HalfBROT `LIMIT` behaviour and IAG50cm's `FB_TelescopeControl`: separate issues if needed.

## 5. Tests (TcUnit, `MONETcommonTests`)
New `F_AxisTurn_Tests.TcPOU`, pattern of `F_HomeBelowHorizon_Tests.TcPOU`:
- Site limits: MONETS az [-274, 270], derotator [-90, 360]; MONETN az [-72, 495], derotator [-170, 300].
  MONETS az 50 deg, v < 0 -> 50 (not 410). MONETS derotator 370 -> 10.
- Endpoints: target exactly at `fMin` / `fMax` -> valid, unchanged; 1 deg beyond (with a window that has no
  other turn) -> invalid.
- No valid turn: narrow window, e.g. [0, 300], target 330 -> FALSE.
- Velocity tie-break: target with two valid turns, v > 0 -> lower, v < 0 -> higher, v = 0 -> agreed rule.
- MONETN regression: today's azimuth outputs for the [-50, 440] window. **Caveat:** the pure velocity rule
  does not reproduce them everywhere. Today az 100 with v < 0 stays 100 (threshold 80), the rule gives 460;
  az 300 with v > 0 stays 300 (threshold 310), the rule gives -60. Either the regression test covers only
  the agreeing zones and the difference is accepted, or the function gets a named "room ahead" threshold
  so MONETN keeps its behaviour (question 3). Derotator: today's outputs in (300, 380] are not reproducible
  on purpose, they are the bug.
- Register the suite in tests `MAIN.TcPOU`, `MONETcommonTests.plcproj`, the coverage table in
  `MONETcommonTests/README.md`; `<Compile>` entry in `MONETcommon.plcproj`.

`FB_MonetTelescopeControl` itself cannot be unit tested (REFERENCE TO HalfBROT axis FBs).

## 6. Verification on site (required: this changes where the mount points)
Checklist, MONETS first, then MONETN as regression. Derotator with someone watching the cables and a hand on
the emergency stop.
1. XAE: build MONETcommon (no errors, no new warnings), install the library, rebuild MONETS / MONETN.
2. Before the change: note the measured endpoints of both axes on both sites and compare with `MAIN.TcPOU`.
3. Goto to targets on either side of north (az 50 / 310 deg, both velocity signs): azimuth axis goes to the
   requested position, within the limits, no clamp.
4. Goto with a derotator target that today maps into (360, 380] on MONETS: the turn inside [-90, 360] is used.
5. Goto that cannot be reached (temporary narrow test limits, not the real ones): command rejected with the
   new code, no motion.
6. Track towards an end: tracking stops at `fMax - fTrackMargin` with a warning, axis stands still before the
   hard limit.
7. MONETN: a normal night's targets give the same azimuth positions as before (or the documented difference).
8. Park / home / slew still work on both sites.

## 7. Risks
- Safety: a wrong turn or a missed stop on the derotator damages cables. Hence the hardware checklist, and
  refusal instead of clamping.
- Behaviour change on MONETN (turn choice near the thresholds, derotator above 300) and on MONETS (if it
  already runs the 2bdd79b wrap, the pointing changes back to correct).
- Limits as `REAL` on the axis vs `LREAL` targets: compare with the same type, endpoint inclusive, no float
  equality traps at exactly `fMax`.
- Open PR #95 (0.8.6) touches the same file: whichever merges second rebases.
- On develop `iBuild := 4` does not match `sVersion := '0.8.5'`; #95 fixes it to 0.8.6.

## 8. Rollout and versioning
- Version: `release.yml` (workflow_dispatch) bumps `<ProjectVersion>` in `MONETcommon.plcproj` and
  `stLibVersion_MONETcommon` in `Global_Version.TcGVL` on develop and opens develop -> main;
  `tag-release.yml` tags the merge. Behaviour change -> next minor (0.9.0) is suggested, after #95.
- MONETS and MONETN do not vendor the library; both reference `MONETcommon, * (IAG)`. They pick it up after
  the library is installed on the engineering PC and the project is rebuilt. No site repo change needed,
  unless question 4 adds config fields (then BROTLib `ST_TelescopeConfig` and both `MAIN.TcPOU` change).

## 9. Open questions (not decided)
1. No valid turn: reject the command with an error code (like 0x15 for elevation) and stop tracking with a
   warning, or other behaviour? Leaning: reject at command time, stop during tracking. Not decided.
2. Do the same hard-limit rules apply to MONETS azimuth, or can it be softer? Value of `fTrackMargin` per axis
   and site (0 = up to the measured endpoint).
3. MONETS azimuth convention (north at 0? cable winding direction?) and the preferred turn when two fit,
   including `v = 0` and whether MONETN keeps its 310 / 80 thresholds (see the regression caveat in 5).
4. How do the limits reach `FB_MonetTelescopeControl`: new fields in `ST_TelescopeConfig` (BROTLib, both sites
   duplicate the numbers) or read from `fbAzimuth.fMinPosition` / `fMaxPosition` (single source, already
   there)? And does MONETS already run the affected version?

## 10. Tasks (in order)
1. Answer questions 1-4 (on site: measured endpoints, azimuth convention, which build MONETS runs).
2. Branch `fix/wrap-from-limits-2` from `origin/develop` (after #95 merged).
3. Add `F_AxisTurn` + `F_AxisTurn_Tests`, run the tests (all cases in 5).
4. Replace the wrap (lines 236-246) with the function, limits per question 4.
5. Goto/slew: reject when no valid turn (new codes, events, README error table if any).
6. Tracking: hold the turn, limit-derived stop with `fTrackMargin`; remove 440 and -69 / 379.
7. Build, PR to develop, review; release as the next minor.
8. On-site checklist (6) on MONETS, then MONETN. Close #2 after both pass.

Branch: `fix/wrap-from-limits-2` from `origin/develop`.
