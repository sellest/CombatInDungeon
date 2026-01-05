# Locomotion System Documentation

**Status:** ✅ Complete (Armed & Unarmed, Turns, Block Locomotion)  
**Sessions:** December 2025 - January 2026  
**Blueprint:** ABP_Unarmed, BP_ThirdPersonCharacter  
**Purpose:** Physics-based locomotion with commitment mechanics

---

## Table of Contents
1. [Overview](#overview)
2. [Core Architecture](#core-architecture)
3. [Variables](#variables-bp_thirdpersoncharacter)
4. [Input Processing](#input-processing)
5. [Acceleration System](#acceleration-system)
6. [Turn System](#turn-system)
7. [Block Locomotion](#block-locomotion)
8. [Animation Notify States](#animation-notify-states)
9. [BlendSpaces](#blendspaces)
10. [Sync Groups](#sync-groups)
11. [State Transitions](#state-transitions)
12. [Lock-On Integration](#lock-on-integration)
13. [Testing Checklist](#testing-checklist)

---

## Overview

The Locomotion System provides weighted, commitment-based movement for both armed and unarmed states. It features physics-based acceleration, sync marker transitions, input locking during animations, and seamless integration with the sheathe/unsheathe system.

**Design Philosophy:**
- **Commitment-based** - Once you start moving or stopping, you're committed until animation completes
- **Physics-driven** - Acceleration/deceleration creates weight and inertia
- **Sync marker transitions** - Foot alignment preserved between animations
- **Monster Hunter inspired** - Deliberate, weighty movement

---

## Core Architecture

### State Machine Structure (JogSequence)

```
Entry → Armed? (Conduit)
         │
         ├─ False (Unarmed) ─────────────────────────────────────┐
         │   │                                                    │
         │   └→ Idle → RunWalk? (Conduit)                        │
         │        │    ├→ Walk_Start → WalkRun_Loop              │
         │        │    └→ Run_Start ──→ WalkRun_Loop             │
         │        │                      │                        │
         │        │    Walk_Stop ←───────┤                        │
         │        │    Run_Stop ←────────┘                        │
         │        │    │                                          │
         │        │    └────→ Idle                                │
         │        │                                               │
         │        └→ Turn → Idle / Walk_Start / Run_Start         │
         │                                                        │
         ├─ True (Armed) ────────────────────────────────────────┤
         │   │                                                    │
         │   └→ Armed_Idle → Armed_RunWalk? (Conduit)            │
         │        │          ├→ Armed_Walk_Start → Armed_WalkRun_Loop
         │        │          └→ Armed_Run_Start ──→ Armed_WalkRun_Loop
         │        │                                 │
         │        │          Armed_Walk_Stop ←──────┤
         │        │          Armed_Run_Stop ←───────┘
         │        │          │
         │        │          └────→ Armed_Idle
         │        │
         │        └→ Armed_Turn → Armed_Idle / Armed_Walk_Start
         │
         ├─ Block States ───────────────────────────────────────┤
         │   │                                                   │
         │   └→ Block_Idle → Guard_RunWalk? (Conduit)           │
         │        │          └→ Guard_Walk_Start → Guard_Walk_Loop
         │        │                                │
         │        │          Guard_Walk_Stop ←─────┘
         │        │          │
         │        │          └────→ Block_Idle
         │
         └─ Sheathe/Unsheathe Transitions ──────────────────────┘
             Idle ↔ Sheathe_Idle ↔ Armed_Idle
             WalkRun_Loop ↔ Sheathe_WalkRun ↔ Armed_WalkRun_Loop
```

---

## Variables (BP_ThirdPersonCharacter)

### Speed Control

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `TargetSpeed` | Float | 0 | Intended speed (0, 250, or 600) |
| `CurrentSpeed` | Float | 0 | Actual speed, interpolates toward TargetSpeed |
| `MaxAcceleration` | Float | 2000 | Speed increase rate |
| `MaxDeceleration` | Float | 2000 | Speed decrease rate (full stop) |
| `MomentumDeceleration` | Float | 800 | Speed decrease rate (tier shift) |

### Input State

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `LastMoveForward` | Float | 0 | Last Y input (for direction) |
| `LastMoveRight` | Float | 0 | Last X input (for direction) |
| `bSprintPressed` | Boolean | False | Sprint button held |
| `StopDirection` | Float | 0 | Captured direction when stopping (for lock-on) |

### State Flags

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bWasRunning` | Boolean | False | Was running when stop initiated (selects stop animation) |
| `bLockMovementInput` | Boolean | False | Locks input during Start/Stop animations |
| `bInputReleasedDuringLock` | Boolean | False | Deferred stop - input released while locked |
| `bLocomotionFinished` | Boolean | True | Stop animation completed (outer state transition) |

### Rotation

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bLockRotation` | Boolean | False | Locks rotation during transitions |
| `LockedRotation` | Rotator | 0,0,0 | Captured rotation when lock begins |

### Turn System

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `TurnAngle` | Float | 0 | Raw calculated turn angle |
| `CapturedTurnAngle` | Float | 0 | Snapped angle for BlendSpace (-90, 90, 180) |
| `bWantsTurn` | Boolean | False | Intent flag - turn requested |
| `PreviousMoveRight` | Float | 0 | Last frame input for direction change detection |
| `PreviousMoveForward` | Float | 0 | Last frame input for direction change detection |

### Block Locomotion

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bIsBlocking` | Boolean | False | Currently holding block |
| `bBlockEstablished` | Boolean | False | Block stance achieved (for state routing) |

---

## Input Processing

### AnalogueInputHandler Function

Converts raw gamepad input to clean digital values.

**Input:** RawX (Float), RawY (Float)  
**Output:** SmoothedX (Float), SmoothedY (Float)

**Logic:**
```
RawMagnitude = VectorLength(RawX, RawY)

IF RawMagnitude < 0.2
  Return (0, 0)  // Deadzone
ELSE
  // Normalize direction, output unit vector
  Return (RawX / RawMagnitude, RawY / RawMagnitude)
```

**Result:** Only two states - no input (0,0) or full input (normalized direction).

### CharacterSpeedHandler Function

Sets TargetSpeed based on processed input.

**Called from:** IA_Move Triggered

**Logic:**
```
SmoothedX, SmoothedY = AnalogueInputHandler(RawX, RawY)
SmoothedMagnitude = VectorLength(SmoothedX, SmoothedY)

IF SmoothedMagnitude < 0.1
  RETURN  // No valid input

IF bSprintPressed
  TargetSpeed = 600  // Run
ELSE
  TargetSpeed = 250  // Walk

LastMoveForward = SmoothedY
LastMoveRight = SmoothedX
```

### CharacterStopHandler Function

Handles input release.

**Called from:** IA_Move Completed

**Logic:**
```
IF bLockMovementInput == True
  bInputReleasedDuringLock = True
  RETURN  // Defer stop until lock releases

StopDirection = CalculateStopDirection()  // For lock-on stops
TargetSpeed = 0
```

---

## Acceleration System

### Event Tick - Speed Update

```
SpeedDelta = TargetSpeed - CurrentSpeed

IF SpeedDelta < 0  // Decelerating
  IF TargetSpeed == 0
    Acceleration = MaxDeceleration (2000)  // Full stop
  ELSE
    Acceleration = MomentumDeceleration (800)  // Speed tier shift
ELSE  // Accelerating
  Acceleration = MaxAcceleration (2000)

CurrentSpeed = FInterpTo(CurrentSpeed, TargetSpeed, DeltaTime, Acceleration)
Set MaxWalkSpeed = CurrentSpeed
```

### Event Tick - Locked Movement

```
IF bLockMovementInput == True
  Add Movement Input (LastMoveForward, LastMoveRight)
```

Keeps character moving during Start animations even after button released.

---

## Turn System

### Overview

Idle turns allow the character to rotate in place when input direction differs significantly from current facing. Turns only trigger from standstill (CurrentSpeed near 0) and are disabled during motion - BlendSpace handles direction changes while moving.

**Design Philosophy:**
- Turns from idle only (not during movement)
- Angle snapping to discrete values (-90, 90, 180)
- Works with both keyboard and gamepad
- Intent flag pattern (bWantsTurn) prevents state desync

### CalculateTurnAngle Function

The most guarded function in the project. Called from IA_Move after AnalogueInputHandler.

**Guards (in order):**
```
1. IF SmoothedX == 0 AND SmoothedY == 0 → RETURN
   (No input)

2. IF bWantsTurn == True → RETURN
   (Already committed to turn)

3. IF bLockMovementInput == True → RETURN
   (Locked in animation)

4. IF CurrentSpeed > 50 → bWantsTurn = False, RETURN
   (Moving - turns disabled)

5. IF VectorLength(SmoothedX, SmoothedY) < 0.5 → RETURN
   (Input too weak - gamepad threshold)

6. IF bIsExecutingAttack == True → bWantsTurn = False, RETURN
   (Attacking)

7. IF bIsDodging == True → bWantsTurn = False, RETURN
   (Dodging)
```

**Angle Calculation:**
```
// Input direction relative to camera
InputWorldYaw = Atan2(SmoothedY, SmoothedX) + ControlRotationYaw

// Character facing
CurrentYaw = ActorRotationYaw

// Raw difference
RawAngle = InputWorldYaw - CurrentYaw

// Normalize to -180 to 180
IF RawAngle > 180
  TurnAngle = RawAngle - 360
ELSE IF RawAngle < -180
  TurnAngle = RawAngle + 360
ELSE
  TurnAngle = RawAngle
```

**Angle Snapping:**
```
// Snap to discrete values for BlendSpace
IF TurnAngle > 135 → CapturedTurnAngle = 180
ELSE IF TurnAngle < -135 → CapturedTurnAngle = 180
ELSE IF TurnAngle > 45 → CapturedTurnAngle = 90
ELSE IF TurnAngle < -45 → CapturedTurnAngle = -90
ELSE → bWantsTurn = False, RETURN (angle too small)

bWantsTurn = True
```

### Turn BlendSpace (BS_Turn)

**Type:** 1D BlendSpace
**Axis:** TurnAngle (-180 to 180)

**Samples:**
- -180: Turn_180
- -90: Turn_Left_90
- 90: Turn_Right_90
- 180: Turn_180

**Loop Setting:** True (important for spam prevention)

### Turn State Transitions

**Idle → Turn:**
```
bWantsTurn == True
(Priority above Idle → RunWalk)
```

**Turn → Walk_Start:**
```
Time Remaining < 0.3 AND TargetSpeed > 0 AND NOT bSprintPressed
```

**Turn → Run_Start:**
```
Time Remaining < 0.3 AND TargetSpeed > 0 AND bSprintPressed
```

**Turn → Idle:**
```
Time Remaining < 0.1 AND TargetSpeed == 0
```

### Clearing Turn Intent

**AN_ClearTurnIntent** placed at START of:
- All Walk_Start animations
- All Run_Start animations
- All Walk_Stop animations
- All Run_Stop animations

**Purpose:** Clears stale bWantsTurn when entering locomotion. Prevents delayed turns after sharp direction changes during motion.

### Gamepad Edge Cases Solved

1. **Stick passes through zero:** Magnitude check (< 0.5) filters weak input
2. **Direction changes mid-calculation:** bWantsTurn guard prevents recalculation
3. **Stale intent during stops:** AN_ClearTurnIntent on start/stop animations
4. **Angle normalization wrap:** Proper -180 to 180 normalization

---

## Block Locomotion

### Overview

Block is integrated into JogSequence as a parallel state set, similar to armed/unarmed. Block has its own idle sequence (Block_Start → Block_Idle → Block_End) and movement states (Guard_Walk_Start → Guard_Walk_Loop → Guard_Walk_Stop).

**Design Decisions:**
- Block only available when armed
- Hold input (not toggle)
- No running while blocking (walk only)
- bBlockEstablished flag for smart state routing

### Block States in Idle (Nested State Machine)

```
Entry → Already Blocking? (Conduit)
         │
         ├─ bIsBlocking == True AND bBlockEstablished == True → Block_Idle
         ├─ bIsBlocking == True AND bBlockEstablished == False → Block_Start
         └─ bIsBlocking == False → Regular Idle
         
Block_Start → Block_Idle: Time Remaining < 0.1
Block_Idle → Block_End: bIsBlocking == False
Block_End → Regular Idle: Time Remaining < 0.1
```

### Block Movement States in JogSequence

```
Block_Idle → Guard_RunWalk? (Conduit): TargetSpeed > 0
Guard_RunWalk? → Guard_Walk_Start: Always (no run option)
Guard_Walk_Start → Guard_Walk_Loop: Automatic
Guard_Walk_Loop → Guard_Walk_Stop: TargetSpeed == 0 AND bIsBlocking == True
Guard_Walk_Stop → Block_Idle: Time Remaining < 0.1
Guard_Walk_Loop → Armed_WalkRun_Loop: bIsBlocking == False
```

### bBlockEstablished Logic

**Purpose:** Routes correctly when returning from block movement to idle.

**Problem Solved:** When walking while blocking, then stopping, should go to Block_Idle, not Block_Start (shield is already up).

**Set True:**
- AN_BlockEstablished at end of Block_Start animation
- When pressing block while already moving (skip Block_Start)

**Set False:**
- When bIsBlocking becomes False (block button released)

**Routing Logic:**
```
IF bIsBlocking == True AND bBlockEstablished == True
  → Block_Idle (skip start animation)
  
IF bIsBlocking == True AND bBlockEstablished == False
  → Block_Start (play raise shield)
```

### Block Input Handling

**IA_Block Pressed:**
```
IF bIsExecutingAttack == True → RETURN
IF bIsDodging == True → RETURN

bIsBlocking = True

IF CurrentSpeed > 50
  bBlockEstablished = True  // Already moving, skip Block_Start
```

**IA_Block Released:**
```
bIsBlocking = False
bBlockEstablished = False
```

---

## Animation Notify States

### ANS_LockMovementInput

**Purpose:** Commits player to Start/Stop animations.

**Notify Begin:**
- `bLockMovementInput = True`

**Notify End:**
- `bLockMovementInput = False`
- `IF bInputReleasedDuringLock == True`
  - Call CharacterStopHandler()
  - `bInputReleasedDuringLock = False`

**Placed on:**
- All Walk_Start / Run_Start animations
- All Walk_Stop / Run_Stop animations
- All Armed variants
- All Guard (block) variants
- Turn animations

### ANS_SlowRotation

**Purpose:** Reduces rotation speed during transitions for smoother movement.

**Notify Begin:**
- `Set Rotation Rate Yaw = 100` (via SetSlowRotation())

**Notify End:**
- `Set Rotation Rate Yaw = 500` (via ResetRotation())

**Placed on:**
- All Start animations
- All Stop animations
- Turn animations

### AN_ClearTurnIntent

**Purpose:** Clears stale turn intent when entering locomotion.

**Action:**
- `bWantsTurn = False`

**Placed on (at START):**
- All Walk_Start / Run_Start animations
- All Walk_Stop / Run_Stop animations

### AN_BlockEstablished

**Purpose:** Signals block stance achieved.

**Action:**
- `bBlockEstablished = True`

**Placed on:**
- End of Block_Start animation

### ANS_LockRotation (Optional)

**Purpose:** Prevents rotation during transitions.

**Alternative approach used:** Reduce Rotation Rate during transitions instead of full lock.

---

## BlendSpaces

### Locomotion BlendSpaces

#### BS_Walk_Start (1D)
- **Axis:** Direction (-180 to 180)
- **Samples:** 8 directional walk start animations

#### BS_Run_Start (1D)
- **Axis:** Direction (-180 to 180)
- **Samples:** 8 directional run start animations

#### BS_WalkRun_Loop (2D)
- **Axis X:** Direction (-180 to 180)
- **Axis Y:** Speed (0-600)
- **Samples:** 
  - Speed 250: Walk loop animations (8 directions)
  - Speed 600: Run loop animations (8 directions)

#### BS_Walk_Stop (1D)
- **Axis:** Direction (-180 to 180)
- **Samples:** 8 directional walk stop animations

#### BS_Run_Stop (1D)
- **Axis:** Direction (-180 to 180)
- **Samples:** 8 directional run stop animations

### Armed Variants
Same structure with armed animation sets:
- BS_Walk_Start_Armed
- BS_Run_Start_Armed
- BS_WalkRun_Loop_Armed
- BS_Walk_Stop_Armed
- BS_Run_Stop_Armed

### Turn BlendSpaces

#### BS_Turn (1D)
- **Axis:** CapturedTurnAngle (-180 to 180)
- **Samples:**
  - -180: Turn_180
  - -90: Turn_Left_90
  - 90: Turn_Right_90
  - 180: Turn_180
- **Loop:** True (critical for spam prevention)

#### BS_Turn_Armed (1D)
- Same structure with armed turn animations

### Block BlendSpaces

#### BS_Guard_Walk_Start (1D)
- **Axis:** Direction (-180 to 180)
- **Samples:** 8 directional block walk start animations

#### BS_Guard_Walk_Loop (2D)
- **Axis X:** Direction (-180 to 180)
- **Axis Y:** Speed (only walk speed, no run)
- **Samples:** Block walk animations (8 directions)
- **Note:** Includes Block_Idle at Speed 0

#### BS_Guard_Walk_Stop (1D)
- **Axis:** Direction (-180 to 180)
- **Samples:** 8 directional block walk stop animations

---

## Sync Groups

**Group Name:** "Walk"

### State Roles:
- **Walk_Start / Run_Start:** Always Leader
- **WalkRun_Loop:** Always Follower
- **Walk_Stop / Run_Stop:** No sync group (terminal)

### Markers:
- **"L"** - Left foot plant
- **"R"** - Right foot plant

### Transition Rules:
- Start → Loop: `Has Marker Been Hit ("R")`
- Loop → Stop: `TargetSpeed == 0` (no marker, use blend)

---

## State Transitions

### Entry Conduit (Armed?)
```
bIsArmed == True → Armed_Idle
bIsArmed == False → Idle
```

### Idle → Movement Conduit (RunWalk?)
```
TargetSpeed > 0 AND bSprintPressed == True → Run_Start
TargetSpeed > 0 AND bSprintPressed == False → Walk_Start
```

### Start → Loop
```
Has Marker Been Hit This Frame (Group: "Walk", Marker: "R")
```

### Loop ↔ Loop (Walk/Run transition)
```
WalkRun_Loop uses CurrentSpeed input
BlendSpace handles smooth transition
Sync markers maintain foot alignment
```

### Loop → Stop
```
Walk_Stop: TargetSpeed == 0 AND bWasRunning == False
Run_Stop: TargetSpeed == 0 AND bWasRunning == True
Blend time: 0.2-0.3s (no marker condition)
```

### Stop → Idle
```
Time Remaining (ratio) < 0.1
Also sets bLocomotionFinished = True via AnimNotify
```

### Outer State (Walk/Run → Idle)
```
bLocomotionFinished == True AND CurrentSpeed == 0
```

---

## Lock-On Integration

When `bIsLockedOn == True`:

1. **Character Rotation:** Faces target (separate system)
2. **Movement:** Strafing relative to target
3. **Stop Direction:** Captured at input release for correct stop animation
4. **BlendSpace Input:** Uses last input direction, not camera direction

### StopDirection Calculation (Lock-On)
```
In IA_Move Completed:
  StopDirection = Atan2(LastMoveRight, LastMoveForward) * (180/PI)
```

---

## bWasRunning Logic

Determines which stop animation plays.

**In CharacterSpeedHandler / Event Tick:**
```
IF TargetSpeed >= 400
  bWasRunning = True
ELSE IF TargetSpeed == 0 AND CurrentSpeed < 50
  bWasRunning = False
```

---

## Root Motion Settings

**ABP_Unarmed:**
- Root Motion Mode: Root Motion from Everything

**Per Animation:**
- Walk_Start: Root Motion disabled
- Run_Start: Root Motion disabled
- Walk_Stop: Root Motion disabled
- Run_Stop: Root Motion disabled
- Loop animations: Root Motion disabled

All movement driven by acceleration system, not root motion.

---

## Testing Checklist

**Basic Movement:**
- ✅ Walk from idle
- ✅ Run from idle
- ✅ Walk → Run transition (smooth)
- ✅ Run → Walk transition (smooth)
- ✅ Walk stop (foot aligned)
- ✅ Run stop (foot aligned)

**Input Commitment:**
- ✅ Can't change direction during Start
- ✅ Can't change direction during Stop
- ✅ Quick tap completes full Start → Stop cycle
- ✅ Input spam handled gracefully

**Gamepad:**
- ✅ Analog input converted to digital bands
- ✅ Deadzone prevents drift
- ✅ Same behavior as keyboard

**Lock-On:**
- ✅ Strafing works correctly
- ✅ Stop animation matches movement direction
- ✅ No rotation snap during transitions

**Armed:**
- ✅ All unarmed tests pass with armed variants

**Turns:**
- ✅ Turn from idle (all directions)
- ✅ Angle snapping works (-90, 90, 180)
- ✅ Turn → Walk_Start if input held
- ✅ Turn → Idle if no input
- ✅ No turns during movement (BlendSpace handles)
- ✅ Gamepad direction calculation correct
- ✅ No stale turns after direction change during motion

**Block:**
- ✅ Block from idle (Block_Start plays)
- ✅ Block release (Block_End plays)
- ✅ Block while moving (Guard_Walk_Loop)
- ✅ Stop while blocking (Guard_Walk_Stop → Block_Idle)
- ✅ Release block while moving (→ Armed_WalkRun_Loop)
- ✅ Press block while moving (bBlockEstablished = True)
- ✅ Return from block movement to idle (skips Block_Start)

---

## Known Limitations

1. **Motion turns disabled** - No 180° turn animations while moving. BlendSpace handles direction changes, turns only from idle.
2. **Sharp direction snap when locked on** - Instant BlendSpace transition from L to R strafe (polish item)
3. **No sprint tier** - Only walk (250) and run (600), sprint deferred
4. **Minor transition twitches** - Some blend timing issues between states (polish)
5. **No block turns** - Can't turn while blocking (low priority)

---

## Session Notes

**Key Learnings:**
- BlendSpace with multiple speed tiers caused sync issues → Solved with clean input bands
- Sync groups persist phase → Walk_Idle buffer state breaks chain
- Outer state machine interrupts inner stops → bLocomotionFinished flag
- ABP cannot set variables → All state tracking in BP_ThirdPersonCharacter
- Root motion from state machine fights acceleration → Disabled per-animation
- CalculateTurnAngle needs extensive guards for edge cases
- Gamepad stick passes through zero during direction changes → Magnitude threshold
- BlendSpace Loop=True prevents spam issues
- bBlockEstablished solves "returning from block movement" routing

**Architecture Decisions:**
- Physics acceleration over root motion (consistency)
- Intent flags for deferred actions (bWantsTurn, bWantsRoll, bInputReleasedDuringLock)
- Separate armed/unarmed state machines (clarity over DRY)
- ANS for state communication (clean BP/ABP interface)
- Block as parallel state set in JogSequence (not separate layer)
- Angle snapping for turns (discrete -90, 90, 180 values)

---

*Last Updated: January 2026*  
*Status: Complete - Armed & Unarmed locomotion, Turns, Block locomotion working*
