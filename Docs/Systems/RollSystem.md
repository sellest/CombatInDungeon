# Roll System Documentation

**Status:** ✅ Complete (Armed & Unarmed)  
**Sessions:** January 2026  
**Blueprint:** ABP_Unarmed (Top Level), BP_ThirdPersonCharacter  
**Purpose:** Evasion system with i-frames and directional control

---

## Overview

The Roll system provides emergency evasion with invincibility frames. It's a top-level state in ABP that can interrupt most other states. Rolls are directional when locked on, and feature rotate-then-roll behavior when in free camera mode.

**Design Philosophy:**
- **Panic button** - Can cancel attack recovery, block, locomotion
- **Commitment-based** - Once rolling, committed until recovery
- **Cancel window** - Recovery frames (50+) allow early exit with input
- **Cornered rat feel** - Rolls, not quick dodges (desperate, not agile)

---

## Variables (BP_ThirdPersonCharacter)

### Roll State

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bWantsRoll` | Boolean | False | Intent flag - roll requested |
| `bIsDodging` | Boolean | False | Currently in roll animation |
| `bCanDodge` | Boolean | True | Roll allowed (not flinched, etc.) |
| `RollDirection` | Float | 0 | Direction for BlendSpace (-180 to 180) |

### Rotate-Then-Roll (Free Camera)

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bRotatingIntoRoll` | Boolean | False | Currently rotating before roll |
| `TargetRollRotation` | Float | 0 | Target yaw for pre-roll rotation |

### Cancel Window

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bIdleExit` | Boolean | False | Animation reached idle exit point |

### Input Timing

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `LastMoveInputTime` | Float | 0 | Timestamp of last movement input |
| `DodgeDirectionalWindow` | Float | 0.2 | Time window for directional roll |

---

## Roll Flow

### Locked-On Roll (Directional)

```
1. Player presses roll
2. Check guards (bCanDodge, bIsDodging, etc.)
3. Calculate RollDirection from LastMoveRight/Forward
4. Set bWantsRoll = True
5. ABP enters Roll state → BlendSpace picks direction
6. I-frames active (frames 10-45)
7. Locks end (frame 50) - cancel window opens
8. AN_DodgeComplete (frame 49) - flags reset
9. If input: early exit to locomotion
10. If no input: play to frame 65, bIdleExit → Idle
```

### Free Camera Roll (Rotate-Then-Roll)

```
1. Player presses roll
2. Check guards
3. Calculate InputWorldYaw from input + camera
4. Calculate RotationDiff from actor rotation
5. IF RotationDiff > 15:
   - Set TargetRollRotation, bRotatingIntoRoll = True
   - Event Tick interpolates rotation
   - When rotation complete: bWantsRoll = True
6. ELSE:
   - RollDirection = 0, bWantsRoll = True (immediate)
7. Roll always goes forward (character already facing input direction)
```

---

## Input Handling

### CallForDodge() / IA_Dodge

**Guards:**
```
IF bCanDodge == False → RETURN
IF bRotatingIntoRoll == True → RETURN
IF bIsDodging == True → RETURN
```

**Stop Current Actions:**
```
Stop Anim Montage  // Cancel attack if in recovery
Set CurrentAttackName = None
```

**Direction Calculation:**
```
IF (GameTime - LastMoveInputTime) < DodgeDirectionalWindow
  // Directional roll
  IF bIsLockedOn
    RollDirection = Atan2(LastMoveRight, LastMoveForward)
    bWantsRoll = True
  ELSE
    // Free camera - rotate then roll
    InputWorldYaw = Atan2(LastMoveRight, LastMoveForward) + ControlRotationYaw
    // Normalize InputWorldYaw
    CurrentYaw = ActorRotationYaw
    RotationDiff = Abs(InputWorldYaw - CurrentYaw)
    // Normalize RotationDiff
    
    IF RotationDiff > 15
      TargetRollRotation = InputWorldYaw
      bRotatingIntoRoll = True
      RollDirection = 0
    ELSE
      RollDirection = 0
      bWantsRoll = True
ELSE
  // No recent input - default back roll
  RollDirection = 180
  bWantsRoll = True
```

### Event Tick - Pre-Roll Rotation

```
IF bRotatingIntoRoll
  CurrentYaw = Get Actor Rotation Yaw
  NewYaw = FInterpTo(CurrentYaw, TargetRollRotation, DeltaTime, 35.0)
  Set Actor Rotation (0, NewYaw, 0)
  
  IF Abs(NewYaw - TargetRollRotation) < 5
    bRotatingIntoRoll = False
    bWantsRoll = True  // Now trigger roll
```

---

## Animation Setup

### Roll Animation Timing (66 frames)

| Frames | Phase | Active Systems |
|--------|-------|----------------|
| 0-10 | Windup | ANS_LockDefense, ANS_LockInput |
| 10-45 | Active Roll | ANS_IFrames (invulnerable) |
| 45-50 | Early Recovery | Locks still active |
| 50-65 | Late Recovery | Locks end, cancel window |
| 49 | AN_DodgeComplete | Flags reset |
| 65 | AN_RollIdleExit | bIdleExit = True |

### Animation Notifies

**ANS_LockDefense (0-50):**
- Prevents block during roll
- Prevents roll spam

**ANS_LockInput (0-50):**
- Prevents attack during roll

**ANS_IFrames (10-45):**
- Sets bIsInvulnerable = True/False
- Damage system checks this flag

**AN_DodgeComplete (frame 49):**
- Calls DodgeComplete() function

**AN_RollIdleExit (frame 65):**
- Sets bIdleExit = True
- Only matters if no input during recovery

---

## DodgeComplete() Function

Called by AN_DodgeComplete at frame 49.

```
bIsDodging = False
bWantsRoll = False
bCanCombo = True  // If needed
CurrentSpeed = 0
TargetSpeed = 0
```

---

## BlendSpaces

### BS_Roll (1D) / BS_Roll_Armed (1D)

**Axis:** RollDirection (-180 to 180)

**Samples:**
- -180: Roll_Back
- -90: Roll_Left
- 0: Roll_Forward
- 90: Roll_Right
- 180: Roll_Back

**Loop:** True (critical - prevents spam stuck state)

---

## State Machine (Top Level ABP)

### Roll State Location

Roll is at the TOP level of ABP, alongside:
- Idle
- Walk/Run (JogSequence)
- Flinch
- Death

This allows roll to interrupt locomotion, attacks (via montage stop), and other states.

### Transitions

**Entry → Roll Conduit:**
```
bWantsRoll == True
```

**Roll Conduit → Unarmed_Roll:**
```
bIsArmed == False
```

**Roll Conduit → Armed_Roll:**
```
bIsArmed == True
```

**Roll → Idle:**
```
bWantsRoll == False AND TargetSpeed == 0 AND bIdleExit == True
Blend: 0.0 (prevents momentum carry from root motion)
```

**Roll → Walk/Run:**
```
bWantsRoll == False AND TargetSpeed > 0
Blend: 0.2-0.3
```

---

## Integration with Other Systems

### Attack Canceling

Roll can cancel attack recovery (when ANS_LockDefense is not active on attack).

```
In CallForDodge():
  IF bLockDefense == False
    Stop Anim Montage
    bIsExecutingAttack = False
    // Proceed with roll
```

### Block Canceling

Roll can cancel block (when ANS_LockDefense allows).

Block release → Roll available immediately.

### Turn System

Turns disabled during roll:
```
In CalculateTurnAngle():
  IF bIsDodging == True
    bWantsTurn = False
    RETURN
```

### Locomotion

Roll resets speed on complete:
```
In DodgeComplete():
  CurrentSpeed = 0
  TargetSpeed = 0
```

This ensures clean return to Idle, then fresh Walk_Start if input held.

---

## Edge Cases Solved

### BlendSpace Loop Setting

**Problem:** Armed roll would get stuck when spammed.
**Cause:** BS_Roll_Armed had Loop=False, animation ended but state stayed.
**Fix:** Set Loop=True on all roll BlendSpaces.

### Direction Lag

**Problem:** Roll direction used stale LastMoveRight/Forward values.
**Cause:** Values set after lock check in IA_Move.
**Fix:** Set LastMoveRight/Forward BEFORE lock check.

### Early Exit Twitch

**Problem:** Character flinched forward when exiting roll to Idle.
**Cause:** 0.2s blend interpolating root motion momentum.
**Fix:** Set Roll → Idle blend to 0.0.

### Roll Spam During Roll

**Problem:** Spamming roll during roll caused state machine confusion.
**Fix:** Guard in CallForDodge(): `IF bIsDodging == True → RETURN`

### Rotate-Then-Roll Timing

**Problem:** Roll started before rotation finished.
**Fix:** Only set bWantsRoll = True in Event Tick when rotation complete.

---

## Testing Checklist

**Basic Rolls:**
- ✅ Roll forward (locked on)
- ✅ Roll backward (locked on)
- ✅ Roll left (locked on)
- ✅ Roll right (locked on)
- ✅ Roll forward (free camera)
- ✅ Rotate-then-roll for large angles

**I-Frames:**
- ✅ Invulnerable during frames 10-45
- ✅ Vulnerable during windup (0-10)
- ✅ Vulnerable during recovery (45+)

**Cancel Window:**
- ✅ Can attack after frame 50
- ✅ Can block after frame 50
- ✅ Can roll after frame 50
- ✅ Can move after frame 50

**Transitions:**
- ✅ Roll → Idle (no input, plays to end)
- ✅ Roll → Walk/Run (input during recovery)
- ✅ Roll → Attack (attack input during recovery)
- ✅ Attack recovery → Roll (cancel)

**Edge Cases:**
- ✅ Roll spam handled
- ✅ Direction changes during roll ignored
- ✅ Gamepad directions correct
- ✅ Default back roll when no input

---

*Last Updated: January 2026*  
*Status: Complete - Armed & Unarmed rolls working*
