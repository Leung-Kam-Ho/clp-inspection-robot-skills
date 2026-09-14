---
name: positioning
description: "Slot-to-angle mapping, setpoint tolerance, reset."
last_updated: 2026-09-11
---

# Positioning

## Slot-to-Angle Mapping

The launch platform has 30 slots, each 12° apart with a 6° offset:

| Slot | Angle | Slot | Angle | Slot | Angle |
|------|-------|------|-------|------|-------|
| 1 | 6° | 11 | 132° | 21 | 252° |
| 2 | 18° | 12 | 144° | 22 | 264° |
| 3 | 30° | 13 | 156° | 23 | 276° |
| 4 | 42° | 14 | 168° | 24 | 282° |
| 5 | 54° | 15 | 180° | 25 | 300° |
| 6 | 66° | 16 | 192° | 26 | 312° |
| 7 | 78° | 17 | 204° | 27 | 324° |
| 8 | 90° | 18 | 216° | 28 | 336° |
| 9 | 102° | 19 | 228° | 29 | 348° |
| 10 | 114° | 20 | 240° | 30 | 354° |

Formula: `angle = (slot - 1) * 12 + 6`

## Response Format Convention

After completing a slot change, always return the status in this exact format:

```
Done! ✅ **Moved to Slot X (Y°)**

| Check | Value |
|-------|-------|
| Angle | Z.ZZ° (within ±0.2° tolerance) |
| Setpoint | Y.Y° |
| Brake | Engaged (`01100000`) |

Launcher is at slot X with brake locked.
```

Replace X with slot number, Y with target angle, Z.ZZ with actual measured angle. Always include the tolerance check in parentheses.

## Setpoint Tolerance

The launcher setpoint floats ±0.2° from the target angle. When verifying position, allow a tolerance of **±0.2°** around the expected angle (e.g., slot 30 at 354° is acceptable if actual angle is between 353.8° and 354.2°).

## Reset

Saying "reset" sets the launch platform to slot 1 (6°):

```bash
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/angle -H "Content-Type: application/json" -d "{\"value\": {\"angle\": 6}}"'
```

## Move to Specific Slot (Example)

Move to slot 3 (30°):

```bash
# Disengage brake first if needed
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":2}}"'

# Send angle command
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/angle -H "Content-Type: application/json" -d "{\"value\": {\"angle\": 30}}"'

# Wait and verify
sleep 8
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".angle"'  # verify → expect ~30° (within ±0.2°)

# Re-engage brake
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":2}}"'
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'  # verify → bit 2 should be ON
```
