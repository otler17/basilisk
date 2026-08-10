# ASTRAL Payload Boresight / Targeting / Pointing Qualification

Date: 2026-08-10  
Runtime: Basilisk 2.10.2 / CPython 3.13.5  
Execution: non-realtime, Vizard disabled  
Plant step / independent truth scoring: 0.02 s where truth-scored  

## Executive result

The payload geometry and target-pointing architecture can physically drive a configured body-frame payload boresight onto an Earth-fixed target under high-fidelity OFFLINE_TRUTH dynamics. This was independently verified from the actual Basilisk spacecraft state and attitude using a separate WGS-84 ray/ellipsoid intersection scorer rather than relying only on flight-side navigation/geometry messages.

Three complete scheduler authority workflows (PLANNED, OPPORTUNISTIC, and OPERATOR override) all physically intercepted the selected T1 ground target using +X boresight and LOS-rate feedforward. Each achieved a 465.481 m minimum truth footprint-center error and a 10.76 s uninterrupted interval in which the independent truth geometry was valid and the flight guidance command had payload_enable asserted. The qualification campaign's target settings required footprint error <= 2 km, incidence <= 75 deg, smear <= 500 m, and 2 s target dwell; the continuous 10.76 s overlap exceeds the 2 s dwell requirement.

A +Z-mounted payload also physically intercepted the target when translation navigation was made temporally consistent at 50 Hz: 759.449 m minimum truth center error and 22.54 s uninterrupted valid+enabled time. This supports that the boresight authority is not hard-coded to +X.

However, the current flight-like end-to-end chain is NOT qualified as a mandatory production payload-targeting path. The supplied ASTRAL_MEKF path misses the target by tens of kilometres during the aggressive acquisition scenario, and the full algorithm-dispatch/executive path destroys MissionAccess payload_ready at its canonical guidance boundary. There is also a 1 Hz translation sample/hold defect that creates artificial reference-rate spikes unless LOS-rate feedforward masks it.

## Qualification geometry

Initial spacecraft WGS-84 subpoint derived from the Basilisk state:
- latitude: ~31.503298 deg N
- longitude: ~30.144902 deg W
- altitude: ~248.810 km

Direct T0 begins nearly overhead. With the body initially inertially aligned and the payload boresight +X_B, target LOS is ~66.189 deg off the payload axis, so the direct test requires a real large-angle spacecraft slew; it is not initialized on target.

For scheduler tests, T1 was placed on the future ground track (~+150 s subpoint):
- latitude: 32.8106784 deg N
- longitude: -18.9922758 deg

Qualification target settings used by the campaign:
- maximum footprint center error: 2,000 m
- maximum incidence: 75 deg
- maximum smear: 500 m
- payload FOV: 12 x 12 deg
- exposure: 0.005 s
- target required dwell: 2.0 s
- target stability: 0.5 deg
- target rate stability: 0.1 deg/s
- guidance settle: 0.5 deg, 0.15 deg/s, 1.0 s dwell

## Architecture findings

### Boresight authority is correctly unified

`run_bsk_sim.py` resolves `payload_boresight_b` as the authoritative mounting axis and derives the target-access boresight, target-pointing guidance boresight, payload geometry boresight, and payload FOV gate from it. The rectangular 12 x 12 deg detector is bounded by a ~8.485 deg circular half-angle for access gating.

`target_pointing_guidance.desired_reference_dcm()` explicitly constructs the commanded attitude so the configured payload body axis aligns with Earth-target LOS. `PayloadGeometryService` then independently ray-intersects the actual body-mounted boresight with the WGS-84 ellipsoid and checks center error, incidence, smear, and target limits.

### MissionAccess correctly separates acquisition from payload eligibility

`slew_eligible` is intentionally less restrictive than payload readiness, allowing the spacecraft to begin acquiring a target even while the payload axis is outside its detector FOV. `payload_ready` applies the tighter payload constraints and dwell logic.

## Scenario results

### 1. Direct +X, default finite-difference reference rate, 1 Hz GNSS-like translation

Result: FAIL.

- Initial boresight error: ~66.19 deg
- minimum flight-side footprint center error: 34.824 km
- minimum pointing error: 0.450 deg
- payload enabled: never
- finite-difference reference-rate spike: ~89.107 deg/s
- controller torque saturation observed in 180/901 sampled control rows

The target LOS is not physically moving at ~89 deg/s. The spike is an artifact of the 1 Hz held translation estimate used by the target publisher.

### 2. Direct +X with analytic LOS-rate feedforward

Flight-side result: PASS; independent truth result: PASS but marginal/fragmented under the normal 1 Hz navigation timing.

Flight-side:
- minimum center error: 41.037 m
- first <=2 km: 71.0 s
- first <=100 m: 113.4 s
- first payload enable: 78.2 s
- minimum pointing error: 0.00228 deg
- feedforward reference rate max: ~1.723 deg/s

Independent plant truth under OFFLINE_TRUTH + 1 Hz GNSS/5 m noise:
- minimum truth center error: 770.031 m
- first truth <=2 km: 70.44 s
- total payload-enabled + truth-valid time: 2.32 s
- longest uninterrupted payload-enabled + truth-valid segment: 0.80 s
- minimum truth error during valid+enabled samples: 833.496 m

This is a real physical intercept but not a robust 2 s continuous truth-qualified dwell.

### 3. Direct +X, reference shaping only

Flight-side result: PASS.

- minimum flight-side center error: 535.174 m
- first <=2 km: 63.3 s
- first payload enable: 59.6 s
- minimum pointing error: 0.0308 deg

This path was not used as the final truth qualification because the shaping/LOS-rate interaction requires correction (see below).

### 4. Direct +X, reference shaping + LOS-rate feedforward

Result: FAIL.

- minimum flight-side center error: ~10.646 km
- payload never enabled
- final pointing error: ~24.396 deg

Shaping and feedforward are individually useful in the tested cases but their current combined handoff is not safe/qualified.

### 5. Alternate payload mounting axes

Flight-side +Z with feedforward:
- initial geometric boresight offset: ~121.57 deg
- minimum flight-side center error: 139.326 m
- first <=2 km: 81.5 s
- first payload enable: 87.2 s

Flight-side -X with feedforward:
- initial geometric boresight offset: ~113.81 deg
- minimum flight-side center error: 28.014 m
- first <=2 km: 79.92 s
- first <=100 m: 89.88 s
- first payload enable: 78.2 s

Independent high-fidelity +Z truth with 50 Hz translation:
- minimum truth center error: 759.449 m
- first valid + payload-enabled: 81.62 s
- uninterrupted valid + payload-enabled duration: 22.54 s

This verifies physical body-axis boresight authority beyond +X. The +Z case fails under the default held 1 Hz translation, reinforcing that translation timing, not the boresight transform, is the dominant issue.

### 6. Opportunistic scheduler

An initially tested target only ~90 s ahead was selected correctly but did not provide enough acquisition time to satisfy the strict settle/rate gate before the best observation window.

Moving the same priority target to a ~150 s lead produces a complete acquisition.

High-fidelity independent truth, OPPORTUNISTIC:
- selected target: T1 throughout the eligible campaign
- scheduler selection state: OPPORTUNISTIC
- guidance enters PAYLOAD_ACTIVE
- minimum truth center error: 465.481 m
- first truth-valid + payload-enabled: 78.96 s
- uninterrupted valid + enabled interval: 78.96-89.70 s
- uninterrupted duration: 10.76 s

Result: PASS for engineering/ideal-attitude navigation chain.

### 7. Planned scheduler

High-fidelity independent truth, PLANNED:
- selected target: T1
- scheduler selection state: PLANNED
- guidance enters PAYLOAD_ACTIVE
- minimum truth center error: 465.481 m
- first truth-valid + payload-enabled: 78.96 s
- uninterrupted valid + enabled interval: 78.96-89.70 s
- uninterrupted duration: 10.76 s

Result: PASS for engineering/ideal-attitude navigation chain.

### 8. Operator override

High-fidelity independent truth, OPERATOR:
- selected target: T1
- scheduler selection state: OPERATOR
- guidance enters PAYLOAD_ACTIVE
- minimum truth center error: 465.481 m
- first truth-valid + payload-enabled: 78.96 s
- uninterrupted valid + enabled interval: 78.96-89.70 s
- uninterrupted duration: 10.76 s

Result: PASS for engineering/ideal-attitude navigation chain.

### 9. 50 Hz translation experiment, no LOS-rate feedforward

High-fidelity +X, ideal attitude navigation, GNSS-like translation at 50 Hz with zero injected translation noise:
- finite-difference reference-rate max: 1.6969 deg/s
- minimum truth footprint error: 700.833 m
- first truth-valid + payload-enabled: 93.10 s
- uninterrupted valid + enabled duration: 10.98 s
- minimum truth error during valid+enabled interval: 1,238.365 m

Result: PASS.

This is a critical diagnosis: once the spacecraft position used for LOS is temporally current, the raw finite-difference target reference becomes physically smooth and LOS-rate feedforward is no longer required to avoid ~89 deg/s spikes.

### 10. Full ADCS executive / algorithm dispatcher

Result: FAIL.

The flight-side payload geometry reaches the target (minimum center error ~89.23 m) and MissionAccess asserts payload_ready for 476 guidance samples. Yet guidance reports payload_ready=False for every sample, never settles/enables the payload, and the executive remains in TARGET_SLEW rather than PAYLOAD_ACTIVE.

Root cause at the canonical dispatcher boundary:
- `TargetPointingGuidanceService._canonical_target_input()` exports only `access_valid = access.slew_eligible`.
- `AstralTargetPointingGuidanceAlgorithm.compute()` reconstructs `access = SimpleNamespace(..., payload_ready=False)`.

Therefore authoritative MissionAccess payload readiness is discarded by the canonical algorithm path. This is a deterministic functional blocker independent of boresight accuracy.

### 11. Flight-like ASTRAL_MEKF attitude navigation

Default high-fidelity WMM environment + current ONBOARD_MODEL centered-dipole magnetic reference:
- minimum truth footprint error: ~20.569 km
- no truth-valid observation
- no valid + enabled interval
- attitude knowledge error was already ~12.3 deg at acquisition and remained roughly 12-17 deg during the campaign

Changing the high-fidelity environment to the same centered-dipole model removes the initial magnetic-model bias (~0.1 deg initial attitude error), but the estimator error then grows to roughly 9 deg during the aggressive slew and still misses the target badly (~60.6 km minimum footprint error in that trajectory).

A diagnostic run using oracle-consistent magnetic/sun reference vectors also grows to roughly 8.9 deg attitude error during the maneuver and misses by ~62.3 km. This demonstrates a second limitation beyond magnetic model mismatch: the current estimator/maneuver combination does not maintain the attitude knowledge required for payload pointing during the aggressive slew.

A pre-pointed +X/T0 test demonstrates that the physical geometry is not the root cause: the true payload footprint can transiently reach ~5.6 m from target, while the MEKF/guidance chain still fails the payload-enable rate/settle gate and later drifts badly.

Result: FAIL for mandatory flight-like payload qualification.

## Root causes and severity

### P0 - Translation epoch/sample-hold creates artificial target-reference dynamics

`OnboardNavigationService` publishes at the configured sample period (default 1.0 s) and holds that position until the next sample. Earth targeting consumes the held position directly even though velocity and a time tag are available. The observed LOS therefore stair-steps: small changes between held intervals followed by ~1.7 deg jumps at each one-second update. Finite differencing that command creates ~87-89 deg/s reference-rate impulses.

Evidence: at 50 Hz navigation update rate, the same finite-difference reference is limited to ~1.697 deg/s and the no-feedforward case physically reaches target for >10 s continuously.

Required correction: target LOS generation should propagate the time-tagged translation state to the current guidance epoch (at minimum r(t)=r_sample+v_sample*dt with bounded age/quality handling), or target/reference generation must operate on a temporally coherent navigation stream. Feedforward may remain useful but must not be the only mechanism hiding stale position samples.

### P0 - Canonical guidance boundary destroys payload_ready

MissionAccess payload readiness is not represented in the canonical guidance target input. The canonical target guidance algorithm explicitly reconstructs `payload_ready=False`, so the full executive/dispatcher path cannot preserve mission payload gating.

Required correction: make MissionAccess readiness an authoritative canonical input (preferred), or have the service re-overlay authoritative access/payload gating after reference-algorithm computation. A reference generator should not manufacture/erase mission access authority.

### P0 - Current MEKF chain does not maintain payload-grade attitude knowledge through acquisition

There are at least two independent contributors:
1. High-fidelity environment uses WMM while the current onboard magnetic reference configuration only accepts CENTERED_DIPOLE, producing a large initial model mismatch.
2. Even with matched/oracle reference vectors, estimator error grows to ~9 deg during the aggressive slew.

The navigation arbiter still accepts the solution as valid in these failing cases, so validity/quality gating is also insufficiently tied to payload-grade truth/innovation consistency.

Required correction: align flight/onboard magnetic modeling, qualify/tune the MEKF through the actual slew-rate envelope, inspect gyro/measurement timing and process/measurement covariance, tighten innovation/consistency gates for payload modes, and ensure the arbiter cannot label a multi-degree-error solution acceptable for precision pointing.

### P1 - Reference shaping + LOS-rate feedforward interaction is unsafe

Shaping-only and feedforward-only cases can work; their combination diverged badly in the tested case. The shaper changes attitude-reference evolution while feedforward is derived from the raw target/reference kinematics, producing an inconsistent attitude/rate pair.

Required correction: derive angular rate/acceleration from the same shaped reference trajectory, or apply LOS feedforward inside/upstream of a kinematically consistent reference shaper.

### P1 - Full attitude construction can command unnecessary roll/secondary-axis motion

`desired_reference_dcm()` aligns the boresight to LOS but also resolves the remaining roll degree of freedom from the orbit-normal convention. In some target scenarios the resulting full attitude maneuver is substantially larger than the boresight-only angular separation. A continuous minimum-slew roll strategy would reduce acquisition burden, actuator saturation, and estimator stress.

### P1 - Flight-side geometry is not an independent success oracle

MissionAccess/targeting/payload geometry use the onboard navigation estimate. With default 1 Hz translation, flight-side geometry can report tens-of-metres error while independent plant truth is hundreds of metres away. Mandatory mission qualification must retain a separate truth-side scorer and should not use the same navigation source for command and acceptance.

### P2 - Actuator catalog is still an engineering placeholder

These tests exercise a 0.003 N m torque-limited reaction-wheel control chain, but this is not a final hardware wheel/torque/momentum qualification. The actual selected spacecraft hardware must be checked for slew time, torque saturation, momentum capacity, unloading, jitter, and smear margins.

## Recommended acceptance gate for Project M

Do not declare the production payload-targeting path qualified until all of the following are demonstrated in the same complete scenario:

1. Flight-like attitude navigation (MEKF or the intended production estimator), not ideal attitude navigation.
2. Time-coherent translational navigation at the target/guidance epoch.
3. Full executive/algorithm-dispatch path, including MissionAccess -> scheduler/router -> guidance -> controller -> PAYLOAD_ACTIVE.
4. Independent plant-truth WGS-84 payload-boresight intercept.
5. Continuous truth-valid + payload-enabled interval >= required target dwell (2 s in this campaign), with margin.
6. Actual flight reaction-wheel/actuator catalog and disturbance environment.
7. No use of Vizard or realtime execution for qualification dependency; visualization may remain observation-only.
8. Regression coverage for PLANNED, OPPORTUNISTIC, OPERATOR, direct pointing, and at least two non-collinear payload mounting axes.

## Overall status

- Payload boresight transform/geometry authority: PASS.
- Physical target intercept with ideal attitude navigation under high-fidelity dynamics: PASS.
- Planned targeting workflow: PASS under engineering/ideal-attitude chain (465.5 m, 10.76 s continuous truth-valid+enabled).
- Opportunistic targeting workflow: PASS under engineering/ideal-attitude chain (same physical result).
- Operator targeting workflow: PASS under engineering/ideal-attitude chain (same physical result).
- Alternate +Z payload boresight physical intercept: PASS with temporally coherent 50 Hz translation (759.4 m, 22.54 s continuous truth-valid+enabled).
- Default 1 Hz no-feedforward pointing: FAIL.
- Reference shaping + feedforward combination: FAIL.
- Full canonical executive/dispatcher payload activation: FAIL (payload_ready lost).
- Flight-like ASTRAL_MEKF end-to-end precision pointing: FAIL.
- Mandatory production Project M payload-targeting qualification: NOT YET PASSING.

The project has a working physical boresight/targeting/control foundation and multiple complete non-realtime scenarios that genuinely hit the target. The remaining blockers are concentrated in navigation epoch coherence, canonical payload-ready authority, estimator/reference-model/performance, and reference-shaper/feedforward consistency—not in the basic body-to-payload boresight geometry.

## Regression sanity

Focused repository tests executed after the campaign:
- mission payload
- Earth target service
- guidance/control mode chains
- ASTRAL MEKF navigation
- navigation quality arbiter
- onboard attitude navigation

Result: 35 passed / 35. This demonstrates that the end-to-end defects identified above are not currently covered by the existing unit/contract tests.
