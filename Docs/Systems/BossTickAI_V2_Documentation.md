# BossTickAI V2 Documentation

## Overview

BossTickAI V2 is a geometry-aware, reactive AI system that makes decisions every frame based on player position. The boss either attacks (when valid geometry exists) or moves to create valid attack geometry.

**Core Principle:** Movement exists to CREATE valid attack geometry. Not "go to player."

---

## Architecture

```
Event Tick
    │
    ├─ bActivateBossAI == FALSE? → RETURN (AI disabled)
    │
    ├─ bIsExecutingAttack == TRUE? → RETURN (attack montage playing)
    │
    ├─ bIsExecutingRotation == TRUE? → RETURN (turn montage playing)
    │
    └─ RepositionForAttack()
            │
            ├─ ShouldAttack == TRUE → ExecuteAttack(AttackData)
            │
            └─ ShouldAttack == FALSE → ExecuteMovement(MovementType)
```

---

## State Flags

| Variable | Type | Purpose |
|----------|------|---------|
| bActivateBossAI | Boolean | Master toggle for AI (debug) |
| bIsExecutingAttack | Boolean | TRUE while attack montage plays |
| bIsExecutingRotation | Boolean | TRUE while turn montage plays |
| bIsMoving | Boolean | TRUE while movement active |
| CurrentMovementType | E_BossMovementType | Current movement being executed |

---

## Core Functions

### RepositionForAttack()

**Purpose:** Main decision function. Called every tick. Decides attack or movement.

**Returns:**
- ShouldAttack (Boolean)
- AttackData (S_BossAttackData_V2)
- MovementType (E_BossMovementType)

**Logic:**
```
1. Try to attack:
   SelectAttackFromIdle() → AttackData
   IF valid:
     ShouldAttack = TRUE
     RETURN

2. Diagnose movement needed (priority order):

   PlayerLocationSector == Out?     → MovementType = Sprint
   PlayerLocationCone == Back?      → MovementType = Turn180
   PlayerLocationCone == Right?     → MovementType = CircleLeft
   PlayerLocationCone == Left?      → MovementType = CircleRight
   PlayerLocationSector == Far?     → MovementType = Approach
   PlayerLocationSector == Close?   → MovementType = Backstep

3. ShouldAttack = FALSE
   RETURN
```

---

### ExecuteAttack(AttackData)

**Purpose:** Starts attack state.

**Input:** AttackData (S_BossAttackData_V2)

**Logic:**
```
Set bIsMoving = FALSE
Set CurrentMovementType = None
Set bIsExecutingAttack = TRUE
Set CurrentAttackData_V2 = AttackData
Play Montage (AttackData.AttackMontage)
```

**Exit Condition:** Attack montage ends → Set bIsExecutingAttack = FALSE

---

### ExecuteMovement(MovementType)

**Purpose:** Sets up and executes movement toward valid attack geometry.

**Input:** MovementType (E_BossMovementType)

**Logic:**
```
Set bIsMoving = TRUE
Set CurrentMovementType = MovementType

SWITCH MovementType:

  Sprint:
    Set MaxWalkSpeed = 600
    Calculate direction: Get Unit Direction (Self → Player)
    
  Approach:
    Set MaxWalkSpeed = 300
    Calculate direction: Get Unit Direction (Self → Player)
    
  CircleLeft:
    Set MaxWalkSpeed = 200
    ToPlayer = Get Unit Direction (Self → Player)
    Direction = Cross Product (ToPlayer, UpVector)
    
  CircleRight:
    Set MaxWalkSpeed = 200
    ToPlayer = Get Unit Direction (Self → Player)
    Direction = Cross Product (UpVector, ToPlayer)
    
  Backstep:
    Set MaxWalkSpeed = 200
    Calculate direction: Get Unit Direction (Player → Self)
    
  Turn180:
    Set bIsMoving = FALSE
    Set bIsExecutingRotation = TRUE
    SelectTurnOnAngle() → Montage
    Play Montage

// For all except Turn180:
RotateToPlayerDirection()  // Smooth rotation facing player
Add Movement Input (Direction, 1.0)
```

---

### RotateToPlayerDirection()

**Purpose:** Smooth rotation toward player during movement.

**Logic:**
```
Find Look at Rotation (Self → PlayerRef) → TargetRotation

RInterpTo:
  Current: Get Actor Rotation
  Target: TargetRotation
  DeltaTime: Get World Delta Seconds
  InterpSpeed: MovementRotationSpeed (default 3.0)
  → NewRotation

Set Actor Rotation (NewRotation)
```

**Note:** Character Movement Component must have "Orient Rotation to Movement = FALSE" or engine overrides manual rotation.

---

## Movement Types

| Type | When Used | Speed | Behavior |
|------|-----------|-------|----------|
| Sprint | Player in Out band | 600 | Fast gap close |
| Approach | Player in Far + Front | 300 | Measured advance |
| CircleLeft | Player in Right cone | 200 | Orbit to get player in Front |
| CircleRight | Player in Left cone | 200 | Orbit to get player in Front |
| Backstep | Player in Close band | 200 | Create distance |
| Turn180 | Player in Back cone | N/A | Play turn montage |

---

## Attack Selection (Randomized)

### SelectAttackFromIdle()

**Logic:**
```
1. Query all Idle context transitions
2. For each: Get attack data, check IsAttackValidForGeometry()
3. Collect valid attacks into array
4. Random select from array
5. Return selected attack (or None if empty)
```

### SelectComboAttack()

**Logic:**
```
1. Query all Combo context transitions WHERE FromAttack == CurrentAttack
2. For each: Get attack data, check IsAttackValidForGeometry()
3. Collect valid attacks into array
4. Random select from array
5. Return selected attack (or None if empty)
```

---

## Integration with Existing Systems

### Geometry Detection (runs in Event Tick)
- GetPlayerSector() → PlayerLocationSector, PlayerLocationCone
- GetPlayerOptimalAngle() → PlayerLocationOptimalAngle
- CalculatePlayerAngle() → PlayerLocationAngle

### Head Tracking (runs in Event Tick)
- InterpPlayerAngles() updates LookAtYaw, LookAtPitch
- Disabled when bIsExecutingAttack = TRUE

### Combo System
- ANS_ComboWindow sets bInComboWindow = TRUE
- Triggers SelectComboAttack() for chain attacks

### Hitbox System
- ANS_EnableEnemyHitbox calls PerformAttackTrace()
- DeliverHitboxDamage() applies damage via interface

---

## Debug Features

| Variable | Purpose |
|----------|---------|
| bActivateBossAI | Toggle AI on/off with button |
| bShowDebugGeometry | Visual debug for sectors |

**Debug HUD displays:**
- Boss State
- Movement Type
- Current Attack
- Player Location (Band - Cone - OptAngle)
- Player Angle

---

## Configuration Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| MovementRotationSpeed | 3.0 | How fast boss rotates during movement |
| Sprint MaxWalkSpeed | 600 | Sprint movement speed |
| Approach MaxWalkSpeed | 300 | Approach movement speed |
| Circle/Backstep MaxWalkSpeed | 200 | Strafe movement speed |

---

## Future Additions (Not Yet Implemented)

| Feature | Purpose |
|---------|---------|
| Exhaustion System | Combat rhythm, breathing room for player |
| Idle State | Wait for player aggro before engaging |
| Flinch State | Interrupt AI on damage threshold |
| Dead State | Stop AI on death |
| Attack Cooldowns | Prevent spam of same attack |
| Weighted Selection | Control attack frequency distribution |

---

## State Machine Comparison

| Current Implementation | Planned E_BossState_V2 |
|------------------------|------------------------|
| bIsExecutingAttack | Attack |
| bIsExecutingRotation | (part of Reposition) |
| RepositionForAttack() | Engage |
| ExecuteMovement() | Reposition |
| ❌ Not built | Idle |
| ❌ Not built | Flinch |
| ❌ Not built | Dead |

---

*Document Version: 1.0*
*Last Updated: December 11, 2025*
