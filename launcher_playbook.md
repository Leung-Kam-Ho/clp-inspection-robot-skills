---
name: launcher_playbook
description: "Launcher control — relay, brake, locking, positioning, start/stop."
last_updated: 2026-09-11
---

# Launcher Playbook

## Launcher API Endpoints (POST, JSON body `{"value": ...}`)

| Endpoint | Description |
|----------|-------------|
| `/launch_platform/angle` | Set angle: `{"value": {"angle": float}}` |
| `/launch_platform/relay` | Toggle launch platform relay bit: `{"value": {"idx": int}}` (idx 0-7) |
| `/launch_platform/movement` | Set direction: `{"value": {"direction": int}}` |
| `/launch_platform/laser` | Select laser sensor: `{"value": {"idx": int}}` (1 = laser 1, 2 = laser 2; only one active at a time) |

## Usage Examples

```bash
# Move to slot 3 (30°)
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/angle -H "Content-Type: application/json" -d "{\"value\": {\"angle\": 30}}"'

# Toggle launch platform relay #1
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":1}}"'

# Toggle launch platform relay #5
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":5}}"'

# Check inspection progress
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} "curl -s http://localhost:5001/progress_status | jq ."
```

## Related Docs

- **[Launcher Relay Control](launcher-relay-control.md)** — Relay string format, bit functions, lock/unlock, brake operations
- **[Launcher Positioning](launcher-positioning.md)** — Slot-to-angle mapping, setpoint tolerance, reset
- **[Launcher Start/Stop](launcher-start-stop.md)** — Start and stop launcher (hardware + Docker)
