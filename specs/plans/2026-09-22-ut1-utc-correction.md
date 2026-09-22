# Wire UT1-UTC (dut1) correction into MONET pointing

**Status: planned, not started.** Follow-up to [MONETcommon#18](https://github.com/BROTLib/MONETcommon/issues/18)
(M12: AstroBROT refraction/UT1 findings reach MONET pointing).

## Context

AstroBROT's two underlying findings referenced by MONETcommon#18 are both fixed and released
(`v0.4.0`): the Kelvin/Celsius refraction default bug ([AstroBROT#4](https://github.com/BROTLib/AstroBROT/issues/4))
and the missing UT1-UTC input ([AstroBROT#12](https://github.com/BROTLib/AstroBROT/issues/12)).

Checked what each fix actually requires from callers:

- **Refraction (pressure/temperature): no MONETcommon change needed.** `FB_CO_REFRACT`
  (`AstroBROT/AstroBROT/POUs/FUNCTION_BLOCKS/FB_CO_REFRACT.TcPOU`) auto-derives both from
  `altitude` via a standard-atmosphere formula when `pressure`/`temperature` are left at their
  `0.0` sentinel defaults. `FB_EQ2HOR` already passes `altitude := telescopeConfig.altitude`
  through to it. This now works correctly automatically once MONETcommon/consumers run against
  AstroBROT ≥ v0.4.0 — no code change required, just confirm the referenced library version.

- **UT1-UTC (`dut1`): does need a MONETcommon change.** `FB_EQ2HOR.dut1` defaults to `0.0`
  ("0 treats UTC as UT1" — i.e. no correction), and MONETcommon's `eq2hor`/`hor2eq` calls in
  `FB_MonetTelescopeControl` never pass it. The AstroBROT fix only added the *capability*; a
  caller has to actively supply a real value to benefit.

**There's already a working precedent for exactly this kind of external push.**
`BROTLib/BROTLib/BROTLib/POUs/Comm/FB_Comm_MQTT_Influx.TcPOU` already receives `Temperature`,
`Humidity`, and `Pressure` (lines 14-16) via MQTT `SET` messages — its own comment says "set via
MQTT by an external weather-station bridge" — parsed in `_handleMQTTMessage`
(`parameter = 'temperature'/'humidity'/'pressure'`, lines 74-79). **None of these three are
actually read anywhere in MONETcommon, MONETN, or MONETS today** — confirmed via search, they're
dead data once received. `dut1` should follow this exact same mechanism, and while touching this,
it's worth also wiring the already-arriving `Temperature`/`Pressure` into the refraction call for
a marginal accuracy improvement over the altitude-only estimate (optional, secondary to `dut1`).

UT1-UTC drifts slowly (bounded to ±0.9s by design — leap seconds keep it there), so even an
infrequently-updated value (daily, or even weekly) is far better than the current permanent
`0.0`. A companion issue is filed against `pyBROT` (the Python MQTT client/bridge repo) to add a
push for this, mirroring however the weather bridge works — see below.

## Staleness and reasonable defaults

`Temperature`/`Humidity`/`Pressure` currently default to `GVL_Math.LREAL_MIN` (an error sentinel,
not a physical default) and, once a value has been received once, hold it **forever** with no
staleness check -- if the weather bridge dies, a reading from days ago would silently keep being
used indefinitely.

Rather than invent a separate "reasonable" weather constant, reuse `FB_CO_REFRACT`'s own
convention: it already treats `pressure := 0.0` (and `temperature_set := FALSE`) as "no data,
estimate from altitude" via a standard-atmosphere formula -- which *is* the reasonable default
for refraction, since it's altitude/site-aware rather than a generic guess. So:

- Change `Temperature`'s and `Pressure`'s default -- both the initial value and what staleness
  reverts to -- from `GVL_Math.LREAL_MIN` to `0.0`, matching `FB_CO_REFRACT`'s own sentinel. Any
  consumer can then pass `fbComm.Pressure`/`fbComm.Temperature` straight through to `co_refract`
  with no guard needed: "no data yet" and "reasonable fallback" become the same value. (Drop the
  `> 0.0` guard mentioned for the optional pressure passthrough in step 2 below -- it's no longer
  needed once the default itself is `0.0`.)
- `Humidity` isn't consumed by `FB_CO_REFRACT` or anywhere else in this codebase today -- no
  refraction-style "reasonable default" exists to reuse for it. Leave its default as `GVL_Math.LREAL_MIN`
  unless/until something actually reads it, at which point pick a default appropriate to that use.
- `dut1`'s natural default (`0.0`, "treat UTC as UT1") is already a genuinely sensible fallback,
  not just a sentinel, and UT1-UTC itself drifts slowly (bounded to +/-0.9s by design) --
  staleness matters far less for it, but it's cheap to handle the same way for consistency.

Add a staleness timeout in `FB_Comm_MQTT_Influx` itself, alongside receiving the values (step 1):
a `TON` per value (or one shared timer if the bridge publishes all of them together in practice)
that resets on each received message; if the timer elapses before a fresh value arrives, revert
that value to its default above. Pick the timeout comfortably longer than the bridge's expected
publish interval (exact cadence depends on how pyBROT/the weather bridge ends up scheduled --
see the companion pyBROT issue).

## Steps

### 1. BROTLib: receive `dut1` over MQTT, same as weather, with staleness revert for all four

In `FB_Comm_MQTT_Influx.TcPOU`:
- Add a member var next to `Temperature`/`Humidity`/`Pressure` (line ~14-16):
  ```
  Dut1		: LREAL := 0.0;	// last received UT1-UTC (seconds, IERS), set via MQTT by an external time bridge
  ```
- Add a case to `_handleMQTTMessage`'s parameter dispatch (alongside `'temperature'`/`'pressure'`,
  around line 74-79), setting a rising-edge marker alongside the value so the staleness timer
  below can be reset on receipt:
  ```
  ELSIF parameter = 'dut1' THEN
  	Dut1 := STRING_TO_LREAL(value);
  	bWeatherOrTimeMessageReceived := TRUE;	// or a dedicated flag, if dut1's cadence differs from weather's
  ```
- Add a staleness `TON` (e.g. `tonWeatherStale : TON := (PT := T#10M);`, tune `PT` once the
  bridge's real publish cadence is known) that resets on `bWeatherOrTimeMessageReceived` and, on
  elapsing, reverts `Temperature`/`Humidity`/`Pressure` to `GVL_Math.LREAL_MIN` and `Dut1` to
  `0.0`. This is new behavior for the three existing weather fields too, not just `dut1` -- see
  "Staleness" above.

### 2. MONETcommon: wire `fbComm.Dut1` into the `eq2hor`/`hor2eq` calls

In `FB_MonetTelescopeControl.TcPOU`, pass `dut1 := fbComm.Dut1` (or a per-call read, matching
how the file already reads `fbComm` elsewhere) into all three coordinate-transform calls:
`eq2hor` (line ~116) and both `hor2eq` calls (lines ~179, ~189). Leaving `fbComm.Dut1` at its
`0.0` default (before the bridge ever pushes anything, or if it's offline) reproduces today's
exact behavior — safe fallback, no regression if the Python side isn't running.

Optionally, same call sites: pass `pressure := fbComm.Pressure` (and `temperature :=
fbComm.Temperature, temperature_set := fbComm.Temperature <> 0.0` if going further) for the
marginal refraction accuracy improvement — secondary, can be dropped from scope if it
complicates the change. No guard needed against `fbComm.Pressure` being unset: per the
"Staleness and reasonable defaults" section above, its default is now `0.0`, `FB_CO_REFRACT`'s
own "estimate from altitude" sentinel, so passing it straight through is always safe.

### 3. Confirm AstroBROT library version

Check MONETcommon's (and MONETN's, once migrated per the separate MONETN migration plan)
referenced/cached AstroBROT library is `v0.4.0` or later, so the refraction fix and the `dut1`
input actually exist to be called.

### 4. pyBROT: push `dut1` (tracked separately)

See the companion issue filed against `BROTLib/pyBROT`. Whatever mechanism currently pushes
weather (if `pyBROT` is that bridge, or if weather comes from a separate script — not confirmed,
`pyBROT`'s own source wasn't checked as part of this plan) should also periodically fetch
UT1-UTC (e.g. via `astropy.utils.iers`) and publish it to `<telescope>/Telescope/SET` with
`parameter=dut1`.

## Verification

No unit tests exist for this library (MONETcommon#20). Verification: confirm on a running PLC
that `fbComm.Dut1` updates when the bridge publishes to the `SET` topic, and that a nonzero
`dut1` measurably shifts the computed RA by the expected sub-arcsecond-to-few-arcsecond amount
(the review's own estimate: up to ~13.5" in RA) relative to the current always-`0.0` behavior.
Not urgent/safety-relevant — this is model-accuracy, not defect-fixing — can be verified whenever
convenient rather than gating other work.
