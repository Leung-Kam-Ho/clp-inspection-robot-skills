---
name: relay_control
description: "Relay string format, bit functions, lock/unlock, brake operations."
last_updated: 2026-09-11
---

# Relay Control

## ⚠️ SSH Timeout

**Every SSH command MUST include `timeout=2`** — these target a local network device.

```bash
ssh -o ConnectTimeout=2 ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl ...'
```

Omitting this can cause commands to hang indefinitely if the dev machine is unresponsive.

## Relay String Format

8-character binary string (e.g., `10000000` means bit 0 is ON, bits 1–7 are OFF).

- Bit 0 = leftmost character
- Bit 7 = rightmost character
- `0` = OFF, `1` = ON

## Relay Bit Functions

| Bit | Name | Function |
|-----|------|----------|
| 0 | Lock/Unlock | Part of the lock/unlock sequence (see below). Used for software state tracking. |
| 1 | Locker | Mechanical brake that holds the launcher at its current position to prevent slipping. Does NOT prevent movement — the launcher can still rotate when the locker is engaged. The launcher can move freely whether the locker is ON or OFF. |
| 2 | Brake | Physically locks the launcher in place and **prevents all movement** when engaged. When bit 2 is ON (`001*****`), the launcher cannot rotate regardless of any angle or movement commands. **Always check that bit 2 is OFF before sending angle or movement commands.** |

Example relay states:
- `01000000` = Locker ON (bit 1), brake OFF — launcher holds position but can still rotate
- `01100000` = Locker ON (bit 1) + Brake ON (bit 2) — launcher is physically clamped, cannot move
- `00000000` = All OFF — launcher free to move

## ⚠️ Critical: Relay Toggle Is NOT a Set Operation

The relay API **toggles** bits — it flips them ON↔OFF. It does NOT set them to a specific state.

**This means toggling the same bit twice returns it to its original state.**

### Always Read Current State Before Toggling

Before toggling any relay bit, **always read the current relay state first**:

```bash
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'
```

Then decide whether to toggle based on the current bit value:

| Current bit 2 (brake) | Action needed | Toggle idx=2? |
|----------------------|---------------|---------------|
| `1` (ON, e.g. `01100000`) | Disengage brake | ✅ Yes |
| `0` (OFF, e.g. `01000000`) | Already disengaged | ❌ No |

| Current bit 2 (brake) | Action needed | Toggle idx=2? |
|----------------------|---------------|---------------|
| `0` (OFF, e.g. `01000000`) | Engage brake | ✅ Yes |
| `1` (ON, e.g. `01100000`) | Already engaged | ❌ No |

**Common mistake:** Assuming the brake is ON when it's already OFF, then toggling idx=2 — which engages the brake instead of disengaging it, leaving the launcher clamped and unable to move.

### Safe Disengage Pattern

```bash
# Step 1: Read current state
RELAY=$(ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq -r ".relay"')
echo "Current relay: $RELAY"

# Step 2: Check bit 2 (3rd character from left)
BIT2=${RELAY:2:1}
if [ "$BIT2" = "1" ]; then
  # Brake is ON, toggle it OFF
  ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":2}}"'
  echo "Brake toggled OFF"
else
  echo "Brake already OFF — no toggle needed"
fi

# Step 3: Verify
sleep 3
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'  # verify → bit 2 should be OFF
```

## Standard Relay Toggle Workflow

When toggling launcher relays, always follow this pattern:

1. **Send** the toggle command
2. **Verify** the new relay state before proceeding
3. **Wait** if the sequence requires delays between steps

### Single relay toggle

```bash
# Toggle relay #0 on the launch platform
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":0}}"'

# Immediately verify the new relay state
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'
```

### Important

- **Always verify after every toggle** — never assume the previous step succeeded before sending the next one
- If verification fails at any step, stop and diagnose before proceeding

## Lock the Launcher

Sequence: toggle bit 0 → check status → toggle bit 1 → check status → toggle bit 0 OFF

```bash
# Step 1: Toggle bit 0 → 10000000
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":0}}"'
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'  # verify → expect 10000000

# Step 2: toggle bit 1 → 11000000
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":1}}"'
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'  # verify → expect 11000000

# Step 3: toggle bit 0 OFF → 01000000
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":0}}"'
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'  # verify → expect 01000000
```

## Unlock the Launcher

Reverse of lock. Start from `01000000` (bit 1 ON):

```bash
# Step 0: Read current state (verify starting point)
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'  # verify → expect 01000000

# Step 1: Toggle bit 0 → 11000000
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":0}}"'
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'  # verify → expect 11000000

# Step 2: toggle bit 1 → 10000000
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":1}}"'
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'  # verify → expect 10000000

# Step 3: toggle bit 0 → 00000000
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":0}}"'
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'  # verify → expect 00000000
```

## Brake Standard Action

**Always disengage the brake before moving the launcher, and re-engage it after reaching the desired position.**

### Disengage Brake

```bash
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":2}}"'
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'  # verify → bit 2 should be OFF (e.g., 01000000)
```

### Engage Brake

```bash
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s -X POST http://localhost:5001/launch_platform/relay -H "Content-Type: application/json" -d "{\"value\":{\"idx\":2}}"'
ssh ${CLP_DEV_HOST:-dev@development-mac-m1.local} 'curl -s http://localhost:5001/launch_platform_status | jq ".relay"'  # verify → bit 2 should be ON (e.g., 01100000)
```

### Standard Workflow for Position Change

1. **Disengage brake** (toggle bit 2 OFF)
2. **Send angle command** to desired slot
3. **Wait and verify** the angle is within tolerance
4. **Re-engage brake** (toggle bit 2 ON)
5. **Verify** final position with brake engaged
