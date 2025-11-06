# Systems Overview

Quick reference for all implemented game systems.

---

## Combat System

**Status:** ✅ Core Complete  
**Blueprint:** BP_ThirdPersonCharacter  

### Components

- 4-attack combo (LMB+RMB)
- Data-driven: DT_Attacks + DT_Combos
- Enum-based state machine
- Loop support (Attack 2a → Attack 1)

### Key Functions

- `AttemptCombo()` - Handles combo logic
- Input: LMB press
- Queries data tables for valid transitions

### Data Structure
**DT_Attacks:**

- AttackName (Enum)
- AttackMontage
- BaseDamage
- AttackType
- HitboxDelay/Duration
- CameraShakeScale

**DT_Combos:**

- FromAttack (Enum)
- ToAttack (Enum)
- InputAction (Enum)

---

## Perfect Timing System

**Status:** ✅ Complete  
**Session:** 8

### Components

- **ANS_PerfectWindow** - AnimNotifyState for window detection
- **Sound feedback** - Ding on perfect hit
- **Visual feedback** - Weapon glow with smooth fade
- **Timeline** - TL_WeaponGlow (0.3s fade curve)

### Variables

- `bPerfectWindowActive` (Boolean)
- `bLastAttackWasPerfect` (Boolean) - for damage calc
- `SwordMaterialDynamic` (Material Instance)

### Functions

- `CheckPerfectTiming()` → Boolean
- `ExecutePerfectStrikeFX()` - Triggers feedback
- `SetWeaponGlow(Intensity)` - Material helper

### Flow

1. ANS Begin → bPerfectWindowActive = True
2. Input during window → CheckPerfectTiming
3. If perfect → ExecutePerfectStrikeFX (sound + glow)
4. ANS End → bPerfectWindowActive = False

### Configuration

- Window: ~0.2s (adjust ANS placement in montages)
- Glow: EmissiveIntensity 0→8→0
- Material parameter: "EmissiveIntensity"

---

## Dodge System

**Status:** ✅ Basic Complete  
**Session:** 7

### Components

- 4 animations: Forward/Backward/Left/Right
- Input: Spacebar
- Direction: Based on WASD (LastMoveForward/Right)

### Variables

- `bIsDodging` (Boolean)
- `LastMoveForward` (Float)
- `LastMoveRight` (Float)

### Functions

- `ExecuteDodge(Montage)` - Plays dodge animation
- Direction determined by movement input at press time

### Montages

- AM_Dodge_Forward
- AM_Dodge_Backward
- AM_Dodge_Left
- AM_Dodge_Right

### Future

- [ ] I-frames (via AnimNotify)
- [ ] Stamina cost
- [ ] Cancel rules (dodge from attacks?)

---

## Lock-On System

**Status:** ✅ Complete  
**Session:** [unknown]

### Components

- Toggle: Tab key
- God of War style (character faces target, strafing)

### Variables

- `bIsLockedOn` (Boolean)
- `LockedTarget` (Actor Reference)

### Behavior

- **Unlocked:** Character rotates with movement
- **Locked:** Character faces target, movement strafes
- Enables directional dodges (left/right make sense)

---

## VFX System

**Status:** ✅ Working  
**Session:** 6-8

### Sword Trails

- **NS_SwordTrail** - Niagara system
- **ANS_SwordTrail** - AnimNotifyState control
- Persistent component pattern (spawn once, toggle visibility)

### Hit Effects

- **NS_HitSpark** - Sparks on impact
- **NS_SlashMark** - Slash marks (currently using material glow, not particles)
- Camera shake + hitstop

### Weapon Glow

- Dynamic material instance
- Material parameter: EmissiveIntensity
- Timeline-driven smooth fade

---

## Architecture Notes

### Data-Driven Design

- Attack properties in DT_Attacks
- Combo transitions in DT_Combos
- Easy to add new attacks without code changes

### Scalability

- Enum-based (type-safe)
- Normalized database structure
- Reusable systems (perfect timing works on all attacks)

### Placeholder Assets

- Using Manny + cubes until boss phase
- $10k budget for production assets when ready
- Material/animation agnostic systems

---

## Next Priorities

1. **Additional Inputs** (RMB, holds, Y+B simultaneous)
2. **Enemy AI** (basic chase/attack)
3. **Material/Bounce System** (stone vs wood surfaces)
4. **Damage System** (integrate perfect timing multiplier)
5. **Boss Part-Break** (foundation for MH-style gameplay)