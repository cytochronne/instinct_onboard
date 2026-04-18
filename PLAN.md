# Add Optional `walkrun` Mode to `g1_parkour`

## Summary
Add a fourth runtime mode, `walkrun`, to `g1_parkour` alongside `cold_start`, `stand`, and `parkour`.

`walkrun` will:
- load from a new `--walkrundir` model directory
- use only `exported/actor.onnx`
- consume proprioceptive observations only
- not require or invoke any depth encoder
- be entered with `R2` inside `g1_parkour`
- keep `L2` as the only emergency-stop button for this script

Cold start remains bound to the existing `parkour` model defaults and gains.

## Key Changes
### 1. Add a dedicated proprio-only parkour locomotion agent
In `instinct_onboard/agents/parkour_agent.py`:
- add a new agent class for `walkrun`, `ParkourWalkRunAgent`
- reuse the existing parkour-style command handling and action reconstruction logic:
  - same joystick-to-velocity mapping style as `ParkourAgent`
  - same zero-action-joint masking / full-action reconstruction
- override model loading to load only `exported/actor.onnx`
- parse observation config without requiring any `depth_*` observation term
- assert that the `walkrun` config is proprio-only, so a mismatched model fails fast
- build actor input directly from the concatenated proprio observation terms
- implement `reset()` by calling the base reset path so history buffers are cleared on mode switches

### 2. Extend `g1_parkour` state machine and CLI
In `scripts/g1_parkour.py`:
- add `--walkrundir` as an optional CLI argument
- register `walkrun` only when `--walkrundir` is provided
- keep `cold_start` target pose/gains sourced from the existing `parkour` agent
- add a new `walkrun` branch in `main_loop_callback`
- make transitions decision-complete as follows:
  - startup: `None -> cold_start`
  - `cold_start` done + `R1` -> `stand`
  - `cold_start` done + `R2` -> `walkrun`
  - `stand` + `L1` -> `parkour`
  - `stand` + `R2` -> `walkrun`
  - `parkour` + `R1` -> `stand`
  - `parkour` + `R2` -> `walkrun`
  - `walkrun` + `R1` -> `stand`
- do not add direct `walkrun -> parkour`; the path is `walkrun -> stand -> parkour`
- update the top-of-file usage docs and button descriptions to match actual behavior

### 3. Restrict the `R2` safety change to `g1_parkour`
In `instinct_onboard/ros_nodes/unitree.py` and the `g1_parkour` node construction path:
- make emergency-stop handling configurable at the node level, e.g. `estop_on_r2: bool = True`
- preserve current default behavior for all existing nodes/scripts: `R2` and `L2` both emergency-stop unless explicitly overridden
- instantiate `G1ParkourNode` with `estop_on_r2=False`
- resulting behavior:
  - in `g1_parkour`, `L2` is the only emergency stop
  - in other scripts, `R2/L2` emergency stop behavior remains unchanged

## Interfaces / Inputs
Add one new external interface:
- `--walkrundir PATH`
  - optional
  - when present, enables `walkrun`
  - expected contents:
    - `params/env.yaml`
    - `exported/actor.onnx`
  - no `depth_encoder.onnx` is required or loaded

Add one internal node interface:
- configurable `R2` emergency-stop policy on the Unitree node layer, defaulting to current behavior

## Test Plan
### Static / startup checks
- start `g1_parkour` in dry-run without `--walkrundir`; confirm behavior stays compatible and no `walkrun` agent is registered
- start `g1_parkour` in dry-run with `--walkrundir`; confirm `walkrun` loads successfully with only `actor.onnx`
- verify `walkrun` startup does not attempt to open `0-depth_encoder.onnx`

### Mode-switch scenarios
- `cold_start` done + `R1` enters `stand`
- `cold_start` done + `R2` enters `walkrun`
- `stand` + `L1` enters `parkour`
- `stand` + `R2` enters `walkrun`
- `parkour` + `R1` returns `stand`
- `parkour` + `R2` enters `walkrun`
- `walkrun` + `R1` returns `stand`

### Safety / regression checks
- in `g1_parkour`, pressing `R2` no longer shuts the process down
- in `g1_parkour`, pressing `L2` still turns off motors and exits
- in another Unitree-based script, `R2` still behaves as emergency stop
- verify mode switches clear the new `walkrun` history buffers correctly

## Assumptions
- `walkrun` uses a parkour-family proprio-only policy layout, so reusing parkour-style command generation and action masking is correct
- `walkrun` is optional; omitting `--walkrundir` must not break existing `g1_parkour` usage
- `stand` remains in the script and remains the hub state for returning from `walkrun` before entering `parkour`
- `g1_parkour` now reserves `R2` for mode switching and treats `L2` as the sole emergency stop, regardless of whether `walkrun` is enabled
