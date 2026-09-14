---
name: robot_commands
description: "Servo, relay, pressure, LED, EL CID controls; digital valve mapping."
last_updated: 2026-09-11
---

# Robot Commands

## Robot API Endpoints (POST, JSON body `{"value": ...}`)

| Endpoint | Description |
|----------|-------------|
| `/robot/servo` | Set servos: `{"value": {"servo": [s1, s2, s3, s4]}}` |
| `/robot/relay` | Toggle robot relay bit: `{"value": {"relay": [idx]}}` (idx 0-7) |
| `/robot/pressure` | Set pressure: `{"value": {"channel": 0-3, "pressure": val}}` (0 = physical ch1, 1 = ch2, 2 = ch3, 3 = ch4) |
| `/robot/led` | Set LED brightness: `{"value": {"brightness": float}}` |

## Digital Valve Channel Mapping

The API uses 0-indexed channels, but they correspond to physical channels 1–4:

| API Channel | Physical Channel |
|-------------|-----------------|
| 0 | ch1 |
| 1 | ch2 |
| 2 | ch3 |
| 3 | ch4 |

## Pressure Initialization

- `desired_pressure` is lazily initialized from `digital_valve.pressure` on first `setPressure` call
- The first pressure set may behave differently if `desired_pressure` hasn't been initialized yet

## Usage Examples

```bash
# Set servo angles (top view: servo1=TL, servo2=TR, servo3=BL, servo4=BR)
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/robot/servo -H "Content-Type: application/json" -d "{\"value\": {\"servo\": [45, 90, 45, 90]}}"'

# Set pressure on physical channel 1 (API channel 0)
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/robot/pressure -H "Content-Type: application/json" -d "{\"value\": {\"channel\": 0, \"pressure\": 5.0}}"'

# Toggle robot relay #0 (flips the state of relay bit 0)
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/robot/relay -H "Content-Type: application/json" -d "{\"value\":{\"relay\":[0]}}"'

# Toggle robot relay #3
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/robot/relay -H "Content-Type: application/json" -d "{\"value\":{\"relay\":[3]}}"'

# Set LED brightness (0.0–1.0)
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/robot/led -H "Content-Type: application/json" -d "{\"value\": {\"brightness\": 0.5}}"'

# Trigger EL CID
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/EL_CID -H "Content-Type: application/json" -d "{\"value\": {\"state\": true}}"'

# Check robot status
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} "curl -s http://localhost:5001/robot_status | jq ."
```
