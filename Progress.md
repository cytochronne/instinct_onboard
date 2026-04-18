# Progress

## Completed
- Added `ParkourWalkRunAgent` in `instinct_onboard/agents/parkour_agent.py`
- Kept `walkrun` proprio-only and actor-only
- Reused parkour velocity-command mapping and action-mask reconstruction
- Hooked `ParkourAgent.reset()` back to the base reset path so observation history buffers are cleared on mode switches
- Added `--walkrundir` to `scripts/g1_parkour.py`
- Registered `walkrun` only when `--walkrundir` is provided
- Extended `g1_parkour` state handling to include:
  - `cold_start(done)+R1 -> stand`
  - `cold_start(done)+R2 -> walkrun`
  - `stand+L1 -> parkour`
  - `stand+R2 -> walkrun`
  - `parkour+R1 -> stand`
  - `parkour+R2 -> walkrun`
  - `walkrun+R1 -> stand`
- Updated the top-of-file `g1_parkour.py` docs and button descriptions to match the implemented behavior
- Added configurable `estop_on_r2` to `instinct_onboard/ros_nodes/unitree.py`
- Kept default behavior unchanged for other Unitree nodes and scripts
- Instantiated `G1ParkourNode` with `estop_on_r2=False` so `g1_parkour` treats `L2` as the only emergency stop

## Implementation Notes
- `ParkourWalkRunAgent` fails fast if the provided model config contains any `depth_*` observation terms
- `walkrun` loads only `exported/actor.onnx`
- `cold_start` remains bound to the existing `parkour` model defaults and gains
- No direct `walkrun -> parkour` transition was added; the path remains `walkrun -> stand -> parkour`

## Files Changed
- `instinct_onboard/agents/parkour_agent.py`
- `scripts/g1_parkour.py`
- `instinct_onboard/ros_nodes/unitree.py`

## Validation
Ran:

```bash
python -m py_compile instinct_onboard/agents/parkour_agent.py scripts/g1_parkour.py instinct_onboard/ros_nodes/unitree.py
```

Result:
- static compilation passed

## Not Yet Verified Runtime
- loading a real `walkrun` model directory through `--walkrundir`
- confirming `walkrun` never attempts to open `0-depth_encoder.onnx` at runtime
- button-driven transitions on the real controller
- `R2` no longer emergency-stopping only inside `g1_parkour`
- `L2` still emergency-stopping inside `g1_parkour`
- `R2` still emergency-stopping in other Unitree-based scripts
