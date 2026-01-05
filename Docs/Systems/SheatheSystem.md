# Sheathe/Unsheathe System Documentation

**Status:** ✅ Complete  
**Sessions:** December 2025 (Session 17)  
**Blueprint:** ABP_Unarmed, BP_ThirdPersonCharacter  
**Purpose:** Weapon draw/holster with proper state management

---

## Overview

The Sheathe/Unsheathe System handles transitions between armed and unarmed states, including weapon attachment to hand/back sockets and animation-driven state changes. Uses intent flags to prevent state desync during locked animations.

**Design Philosophy:**
- **Animation-driven state change** - bIsArmed changes at animation end, not button press
- **Intent flags** - Separate "wants to" from "is"
- **Movement-aware** - Different animations for idle vs walking vs running
- **Attack integration** - Auto-unsheathe on attack input

---

## Variables (BP_ThirdPersonCharacter)

### State

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bIsArmed` | Boolean | False | Actual armed state (weapon in hand) |

### Intent Flags

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bWantsToSheathe` | Boolean | False | Player requested to holster weapon |
| `bWantsToUnsheathe` | Boolean | False | Player requested to draw weapon |

---

## Input Handling

### IA_Sheathe (Toggle Button)

```
// Block during locked animations
IF bLockMovementInput == True
  RETURN

// Set intent based on current state
IF bIsArmed == True
  bWantsToSheathe = True
ELSE
  bWantsToUnsheathe = True
```

**Note:** Does NOT set bIsArmed directly. Intent flags trigger state machine transitions.

### Attack Input (Auto-Unsheathe)

```
IF bIsArmed == False
  IF TargetSpeed == 0
    bWantsToUnsheathe = True  // Draw weapon from idle
  ELSE
    Play AM_AdvancingSlash  // Draw + attack while moving
  RETURN

// Normal attack logic
AttemptCombo()
```

---

## State Machine Integration

### Entry Conduit (Armed?)

Routes based on actual state, not intent:
```
bIsArmed == True → Armed_Idle (or Armed locomotion)
bIsArmed == False → Idle (or Unarmed locomotion)
```

### Sheathe Transitions (Unarmed → Armed)

**From Idle:**
```
Idle → Sheathe_Idle
  Condition: bWantsToUnsheathe == True
  
Sheathe_Idle → Armed_Idle
  Condition: Time Remaining < 0.1
```

**From Movement:**
```
WalkRun_Loop → Sheathe_WalkRun
  Condition: bWantsToUnsheathe == True

Sheathe_WalkRun → Armed_WalkRun_Loop
  Condition: Time Remaining < 0.1
```

### Unsheathe Transitions (Armed → Unarmed)

**From Idle:**
```
Armed_Idle → Unsheathe_Idle
  Condition: bWantsToSheathe == True

Unsheathe_Idle → Idle
  Condition: Time Remaining < 0.1
```

**From Movement:**
```
Armed_WalkRun_Loop → Unsheathe_WalkRun
  Condition: bWantsToSheathe == True

Unsheathe_WalkRun → WalkRun_Loop
  Condition: Time Remaining < 0.1
```

---

## BlendSpaces

### BS_Sheathe_WalkRun (1D)
- **Axis:** Speed (0-600)
- **Speed 250:** Sheathe_Walk animation
- **Speed 600:** Sheathe_Run animation

### BS_Unsheathe_WalkRun (1D)
- **Axis:** Speed (0-600)
- **Speed 250:** Unsheathe_Walk animation
- **Speed 600:** Unsheathe_Run animation

**Input:** CurrentSpeed (picks correct animation based on movement speed)

---

## Animation Notifies

### AN_SheatheComplete

**Placed at:** End of all Sheathe animations (Idle, Walk, Run)

**Actions:**
```
bIsArmed = True
bWantsToUnsheathe = False
AttachWeaponToHand()
```

**Important:** Place a few frames BEFORE ANS_LockMovementInput ends to prevent state desync.

### AN_UnsheatheComplete

**Placed at:** End of all Unsheathe animations (Idle, Walk, Run)

**Actions:**
```
bIsArmed = False
bWantsToSheathe = False
AttachWeaponToBack()
```

### AN_WeaponToHand

**Placed at:** Frame where hand grabs weapon (during draw)

**Actions:**
```
AttachWeaponToHand()
```

### AN_WeaponToBack

**Placed at:** Frame where hand releases weapon (during holster)

**Actions:**
```
AttachWeaponToBack()
```

---

## Weapon Attachment System

### Skeleton Sockets

Create in Skeleton asset:
- `Socket_Weapon_Hand` - on hand_r bone (weapon drawn)
- `Socket_Weapon_Back` - on spine_03 bone (weapon holstered)
- `Socket_Shield_Hand` - on hand_l bone (shield drawn)
- `Socket_Shield_Back` - on spine bone (shield holstered)

**Transform:** Adjust in socket, zero out component transforms.

### AttachWeaponToHand() Function

```
Sword → Attach Component To Component
  Parent: Mesh
  Socket Name: Socket_Weapon_Hand
  Location Rule: Snap to Target
  Rotation Rule: Snap to Target
  Scale Rule: Keep World

Shield → Attach Component To Component
  Parent: Mesh
  Socket Name: Socket_Shield_Hand
  Location Rule: Snap to Target
  Rotation Rule: Snap to Target
  Scale Rule: Keep World
```

### AttachWeaponToBack() Function

```
Sword → Attach Component To Component
  Parent: Mesh
  Socket Name: Socket_Weapon_Back
  Location Rule: Snap to Target
  Rotation Rule: Snap to Target
  Scale Rule: Keep World

Shield → Attach Component To Component
  Parent: Mesh
  Socket Name: Socket_Shield_Back
  Location Rule: Snap to Target
  Rotation Rule: Snap to Target
  Scale Rule: Keep World
```

### BeginPlay Initialization

```
IF bIsArmed == False
  AttachWeaponToBack()
ELSE
  AttachWeaponToHand()
```

---

## Input Locking

Sheathe/Unsheathe animations use ANS_LockMovementInput:
- Prevents movement input during transition
- Prevents sheathe spam
- Ensures animation completes

**ANS Placement:**
- Start frame to near-end frame
- AN_SheatheComplete/AN_UnsheatheComplete placed BEFORE ANS ends

---

## Edge Cases Handled

### Sheathe During Attack

**Problem:** Player presses sheathe during attack, bWantsToSheathe set but attack still playing.

**Solution:** IA_Sheathe checks `bLockMovementInput`. Attacks should also set this or use `bIsExecutingAttack` check.

### State Desync Prevention

**Problem:** bIsArmed changes before state machine can transition.

**Solution:** 
- bIsArmed only changes via AnimNotify at animation end
- Intent flags (bWantsToSheathe/bWantsToUnsheathe) trigger transitions
- AN_SheatheComplete placed before ANS_LockMovementInput ends

### Interrupt Handling

**Future consideration:** If sheathe interrupted (flinch, stun), reset intent flags:
```
bWantsToSheathe = False
bWantsToUnsheathe = False
```

---

## State Flow Diagrams

### Draw Weapon (Idle)
```
Player presses Attack (unarmed, idle)
  ↓
bWantsToUnsheathe = True
  ↓
State: Idle → Sheathe_Idle
  ↓
Animation plays, ANS_LockMovementInput active
  ↓
AN_WeaponToHand fires (weapon attaches)
  ↓
AN_SheatheComplete fires (bIsArmed = True, bWantsToUnsheathe = False)
  ↓
State: Sheathe_Idle → Armed_Idle
```

### Holster Weapon (Moving)
```
Player presses Sheathe (armed, running)
  ↓
bWantsToSheathe = True
  ↓
State: Armed_WalkRun_Loop → Unsheathe_WalkRun
  ↓
BS_Unsheathe_WalkRun selects Run animation (CurrentSpeed = 600)
  ↓
Animation plays, ANS_LockMovementInput active
  ↓
AN_WeaponToBack fires (weapon attaches to back)
  ↓
AN_UnsheatheComplete fires (bIsArmed = False, bWantsToSheathe = False)
  ↓
State: Unsheathe_WalkRun → WalkRun_Loop
```

---

## Testing Checklist

**Basic Functionality:**
- ✅ Sheathe from idle
- ✅ Unsheathe from idle
- ✅ Sheathe while walking
- ✅ Sheathe while running
- ✅ Unsheathe while walking
- ✅ Unsheathe while running

**Weapon Attachment:**
- ✅ Sword moves to hand on draw
- ✅ Sword moves to back on holster
- ✅ Shield moves to hand on draw
- ✅ Shield moves to back on holster

**Edge Cases:**
- ✅ Button spam handled
- ✅ Sheathe → immediate move works
- ✅ Attack while unarmed auto-unsheathes
- ✅ Can't sheathe during attack (blocked by lock)

**Integration:**
- ✅ Locomotion continues after sheathe
- ✅ Correct armed/unarmed locomotion plays after transition
- ✅ Entry conduit routes correctly

---

## Component Hierarchy

```
BP_ThirdPersonCharacter
├─ Capsule Component (root)
├─ Mesh (Skeletal Mesh)
│   ├─ Sword (Static Mesh) - attached to hand or back socket
│   │   └─ SwordHitbox
│   └─ Shield (Static Mesh) - attached to hand or back socket
│       └─ ShieldHitbox
```

**Important:** Use "Attach Component To Component", NOT "Attach Actor To Component".

---

## Future Enhancements

1. **Advancing Slash** - Unsheathe + attack while moving (AM_AdvancingSlash)
2. **Sprint to Sheathe** - Monster Hunter style sprint auto-sheathes
3. **Combat Stance Timeout** - Auto-sheathe after idle for X seconds
4. **Interrupt Recovery** - Reset intent flags on flinch/stun

---

*Last Updated: December 2025*  
*Status: Complete - Sheathe/Unsheathe working for all states*
