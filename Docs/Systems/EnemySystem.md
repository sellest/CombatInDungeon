# Enemy System Documentation

**Status:** ✅ Prototype Complete  
**Sessions:** 16-18.11.2025  
---

## Overview

The enemy system uses parent-child Blueprint architecture where **BP_EnemyBase** contains all reusable boss logic, and specific boss instances (like Mav) configure the details. This allows rapid creation of new bosses by inheriting from the base class.

---

## BP_EnemyBase (Parent Class)

**Type:** Actor Blueprint  
**Purpose:** Reusable boss foundation containing all core enemy systems  

### Components

**EnemyMesh** (Skeletal Mesh Component)

- Displays enemy model
- Assigned to specific character mesh per boss
- Drives AnimBP (ABP_Enemy)

**Capsule Component** (Capsule Collision)

- Root component
- Handles physics collision and overlap
- Disabled on death

---

### Variables

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `Health` | Float | 100.0 | Current health |
| `bIsDead` | Boolean | False | Death state flag |

---

### Systems Implemented

#### 1. Damage System

**Interface:** BPI_Damageable  
**Function:** ApplyDamageToActor

**Flow:**

```
ApplyDamageToActor (Damage input)
    ↓
Branch: bIsDead? (ignore if already dead)
    ↓
Health - Damage = NewHealth
Set Health
    ↓
PlayRandomHitSound (BPL_AudioHelpers)
├─ HitSoundTable: DT_SwordHitSounds_Flesh
├─ HitLocation: Get Actor Location
└─ WorldContextObject: Get Player Controller (inside function)
    ↓
Branch: Health <= 0?
├─ True → Death sequence
└─ False → Flinch sequence
```

---

#### 2. Death System

**Triggered when:** Health <= 0

**Logic:**

```
Set bIsDead = True
    ↓
Print: "Mav died"
    ↓
Get EnemyMesh → Get Anim Instance → Cast to ABP_Enemy
└─ Set bIsDead = True (triggers death animation)
    ↓
Get Capsule Component
└─ Set Collision Enabled (No Collision)
```

**Result:**

- Death animation plays via state machine
- Collision disabled (can walk through corpse)
- Corpse remains in world (does not despawn)

---

#### 3. Flinch System (Prototype)

**Triggered when:** Damaged but not killed

**Logic:**

```
Branch: Health > 0?
└─ True →
    Get EnemyMesh → Get Anim Instance → Cast to ABP_Enemy
    └─ Set bShouldFlinch = True
```

**Current behavior:** Flinches on every non-lethal hit

**Future:** Will use poise/stagger threshold system (Monster Hunter style)

---

## ABP_Enemy (Animation Blueprint)

**Type:** Animation Blueprint  
**Target Skeleton:** Mav J Largo skeleton  
**Purpose:** Controls enemy animations via state machine

---

### Variables

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bIsDead` | Boolean | False | Triggers death state transition |
| `bShouldFlinch` | Boolean | False | Triggers flinch state transition |

---

### State Machine

```
Entry → Idle
Idle → Flinch (when bShouldFlinch = True)
Flinch → Idle (when animation complete)
Idle → Death (when bIsDead = True)
Death (terminal state)
```

---

### States

#### Idle State

- **Animation:** Idle (looping)
- **Behavior:** Default standing animation

#### Flinch State

- **Animation:** Hit Reaction (no loop)
- **Entry:** bShouldFlinch = True
- **Exit:** Animation time remaining <= 0.1s (automatic)
- **Reset:** bShouldFlinch set to False after 0.1s delay

#### Death State

- **Animation:** Death (no loop)
- **Entry:** bIsDead = True
- **Exit:** Never (terminal state)

---

### Event Graph Logic

**Event Blueprint Update Animation** runs every frame:

```
1. Update bIsDead from owner:
   Try Get Owner Actor
   → Cast to BP_EnemyBase
   → Get bIsDead
   → Set bIsDead (local)

2. Reset flinch trigger (with delay):
   Branch: bShouldFlinch?
   → Delay (0.1s)
   → Set bShouldFlinch = False
```

**Why delay:** Gives state machine 1 frame to detect True before resetting

---

## Current Enemy: Mav J Largo

**Source:** Mixamo (free)  
**Type:** Humanoid thug character  
**Scale:** 1.0 (default)

### Imported Assets

**Location:** `/Content/Enemies/MavJLargo/`

**Files:**

- Skeletal Mesh (T-Pose with skin)
- Skeleton
- Idle animation
- Death animation  
- Hit Reaction animation

### Configuration

**In BP_EnemyBase (currently hardcoded):**

- Mesh: Mav J Largo skeletal mesh
- AnimBP: ABP_Enemy
- Health: 100
- Hit sounds: DT_SwordHitSounds_Flesh

---

## Audio Integration

### Hit Sounds

**Data Table:** DT_SwordHitSounds_Flesh

**Structure:**

| Row Name | Sound (Sound Base) |
|----------|-------------------|
| FleshHit_01 | Flesh impact sound |
| FleshHit_02 | Flesh impact sound |
| FleshHit_03 | Flesh impact sound |

**Playback:** BPL_AudioHelpers → PlayRandomHitSound

- Randomly selects from table rows
- Plays at actor location
- Uses Player Controller as world context

---

## Known Issues

### 1. Flinch Spam

**Problem:** Flinches on every hit, can interrupt itself  
**Cause:** No poise/threshold system yet  
**Impact:** Low (acceptable for prototype)  
**Fix planned:** Add poise damage accumulation system

### 2. Hardcoded Configuration

**Problem:** Mav-specific details in BP_EnemyBase  
**Cause:** No child blueprint yet  
**Impact:** None (only one enemy)  
**Fix planned:** When adding Boss #2, refactor to parent/child structure

---

## Future Enhancements

### Short Term (Next Session)

- [ ] Death sound (scream on death)
- [ ] Camera shake on death (bigger impact)
- [ ] Basic AI (idle → chase → attack)

### Medium Term

- [ ] Poise/stagger system (threshold-based flinch)
- [ ] Attack patterns (boss fights back)
- [ ] Health bar UI

### Long Term

- [ ] Part-break system (hitzones with individual health)
- [ ] Boss-specific mechanics
- [ ] Phase transitions

---

## Refactoring Plan (When Adding Boss #2)

### Step 1: Clean BP_EnemyBase

- Remove Mav-specific mesh reference
- Make variables configurable (HitSoundTable, MaxHealth, etc.)
- Keep only generic systems

### Step 2: Create BP_Boss_Mav

- Parent: BP_EnemyBase
- Set: Mav mesh, ABP_Enemy, DT_SwordHitSounds_Flesh
- Configure: Health values, scale, etc.

### Step 3: Create BP_Boss_[NewEnemy]

- Parent: BP_EnemyBase  
- Set: New mesh, animations, sounds
- Inherits: Health, damage, death, flinch systems

**Estimated time per new boss:** 30 minutes (just configuration)

---

## Testing Checklist

**Basic Functionality:**

- ✅ Mav appears in level with idle animation
- ✅ Collision prevents player walking through
- ✅ Takes damage when hit by sword attacks
- ✅ Takes damage when hit by shield attacks
- ✅ Plays flesh hit sounds on damage
- ✅ Flinches on non-lethal hits
- ✅ Dies after 10 hits (10 damage × 10 attacks = 100 health)
- ✅ Plays death animation when killed
- ✅ Collision disabled on death (can walk through corpse)
- ✅ Corpse remains visible (does not despawn)

---

## Design Philosophy

**Parent-Child Architecture:**

- BP_EnemyBase = Systems (reusable)
- BP_Boss_[Name] = Configuration (per-boss)
- Minimize duplication, maximize reusability

**Prototype-First Approach:**

- Hardcoded values acceptable initially
- Refactor to variables when adding Boss #2
- Don't over-engineer before validating design

**Monster Hunter Inspiration:**

- Flinch as reward for damage threshold
- Corpses remain (harvest materials in future)
- Boss-scale enemies (not crowds)

---

*Documented: 16.11.2025*  
*Status: Prototype complete, ready for AI implementation*
