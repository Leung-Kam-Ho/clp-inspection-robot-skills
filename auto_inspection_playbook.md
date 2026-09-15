---
name: auto_inspection_playbook
description: "Start/stop auto inspection, monitor progress, troubleshoot."
last_updated: 2026-09-15
---

# Auto Inspection Control

## Start Inspection

Send POST to `/auto` with the **correct payload format** — the handler wraps `value` around `mode`:

```bash
ssh -o ConnectTimeout=2 dev@development-mac-m1.local \
  'curl -s -X POST http://localhost:5001/auto -H "Content-Type: application/json" \
  -d "{\"value\": {\"mode\": \"Enter\"}}"'
```

**⚠️ Critical:** The payload MUST use `"value": {"mode": "..."}` — NOT `{"mode": "..."}` directly. The handler is:
```python
'/auto': lambda v: self.auto_set_mode(v["mode"]),
```
So `v` is the `value` dict. Using `{"mode": "Enter"}` without the `value` wrapper silently fails (returns success but mode stays Manual).

### Available Modes

| Mode | Description |
|------|-------------|
| `Enter` | Enter inspection position |
| `Enter_Stairs` | Enter stairs inspection |
| `Enter_Generator` | Enter generator inspection |
| `Drop` | Drop phase |
| `Exit_Generator` | Exit generator |
| `Exit_Stairs` | Exit stairs |
| `Elevate` | Elevate phase |
| `Exit` | Exit inspection position |
| `Manual` | Manual mode |

The auto system transitions automatically: `Enter` → `Drop` → `Enter_Stairs` → `Enter_Generator` → `Exit_Generator` → `Exit_Stairs` → `Elevate` → `Exit` → `Manual`

## Monitor Progress

```bash
# Full auto status
ssh -o ConnectTimeout=2 dev@development-mac-m1.local \
  'curl -s http://localhost:5001/auto_status | jq .'

# Key fields
ssh -o ConnectTimeout=2 dev@development-mac-m1.local \
  'curl -s http://localhost:5001/auto_status | jq "{mode, action_name, action_update, dominant_machine, kneading_flag}"'

# Inspection progress (ELCID)
ssh -o ConnectTimeout=2 dev@development-mac-m1.local \
  'curl -s http://localhost:5001/progress_status | jq .'
```

### Auto Status Fields

| Field | Description |
|-------|-------------|
| `mode` | Current mode (Manual, Enter, Exit, etc.) |
| `action_name` | Current behavior tree action |
| `action_update` | Latest action result/details |
| `connected` | Auto node connection status |
| `kneading_flag` | Kneading active |
| `dominant_machine` | "robot" or "launch_platform" |
| `last_mode` | Previous mode |
| `last_enter_angle` | Last set angle |
| `sequence_name` | Current sequence |

## Stop Inspection

Switch back to Manual mode:

```bash
ssh -o ConnectTimeout=2 dev@development-mac-m1.local \
  'curl -s -X POST http://localhost:5001/auto -H "Content-Type: application/json" \
  -d "{\"value\": {\"mode\": \"Manual\"}}"'
```

Verify:
```bash
ssh -o ConnectTimeout=2 dev@development-mac-m1.local \
  'curl -s http://localhost:5001/auto_status | jq ".mode"'
```

## Troubleshooting

### Auto Won't Start (Mode Stays Manual)

**Cause:** Wrong payload format. Using `{"mode": "Enter"}` instead of `{"value": {"mode": "Enter"}}`.

**Fix:** Always wrap mode in `value`:
```bash
curl -d '{"value": {"mode": "Enter"}}'
```

### ToF Sensor Errors

```
CheckToF 5+1 is larger than 15 [x] -- ToF 6 -> [6, 6] (Target: 15)
```

The robot's time-of-flight sensors are reading closer than the target threshold (15mm). This can happen if:
- Robot is too close to the ground/surface
- ToF sensors need recalibration
- The target threshold in the behavior tree config is too high for the current setup

**Action:** Stop auto (switch to Manual), reposition the robot, then retry.

### Behavior Tree Looping

If the auto system loops endlessly between actions, check:
1. Physical conditions (ToF readings, relay states, servo positions)
2. Container logs for error patterns
3. Whether the robot can reach the required position

### EL CID Trigger

```bash
ssh -o ConnectTimeout=2 dev@development-mac-m1.local \
  'curl -s -X POST http://localhost:5001/EL_CID -H "Content-Type: application/json" \
  -d "{\"value\": {\"state\": true}}"'
```

## Full Inspection Workflow

1. **Position launcher** at starting slot (see `launcher-positioning.md`)
2. **Disengage brake** on launcher
3. **Set auto mode** to `Enter` (or appropriate starting mode)
4. **Monitor** progress via `/auto_status` and `/progress_status`
5. **Stop** when complete or if errors occur (set mode to `Manual`)
6. **Re-engage brake** on launcher
7. **Move launcher** to next slot if needed for multi-slot inspection

## Related Skills

- `clp-inspection-robot/launcher-positioning` — Slot-to-angle mapping and movement
- `clp-inspection-robot/launcher-relay-control` — Brake and locker relay operations
- `clp-inspection-robot` — General system control and troubleshooting
