# Systems Overview

Quick reference for all implemented game systems.

---

## Combat System

**Status:** ✅ Core Complete  
**Blueprint:** BP_ThirdPersonCharacter
**MainFunction:** AttemptCombo()

### Key Functions

- `AttemptCombo()` - Handles combo logic
- Input: LMB press
- Queries data tables for valid transitions

### Data Structure

**DT_Attacks:**

- Row Name (str)
- AttackName (Enum)
- AttackMontage
- BaseDamage
- AttackType
- HitboxDelay
- HitboxDuration
- CameraShakeScale

**DT_Combos:**

- Row Name
- FromAttack (Enum)
- ToAttack (Enum)
- InputAction (Enum)

---

## Perfect Timing System

**Status:** ✅ Core Complete  

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
3. If perfect → ExecutePerfectStrikeFX
4. ANS End → bPerfectWindowActive = False

### Configuration

- Window: ~0.2s (adjust ANS placement in montages)
- Glow: EmissiveIntensity 0→0.25→0
- Material parameter: "EmissiveIntensity"

---

## Dodge System

**Status:** ✅ Basic Complete  

### Components

- Two modes: free camera and lock-on camera.

If camera is unlocked:

- always performs Forward animation to the faced direction

If camera is locked:

- Character always faced towards the target
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

- [ ] Cancel rules (dodge from attacks?)
- [ ] I-frames (via AnimNotify) - after basic enemy AI
- [ ] Stamina cost

---

## Lock-On System

**Status:** ✅ Complete  

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
## Boss AI System

**Status:** ✅ Prototype Complete
**Session:** 20.11.2025

See: [Session_20.11.2025_BossAI_FlinchSystem.md](Sessions/Session_20.11.2025_BossAI_FlinchSystem.md)

- State machine (Idle, Approach, Attack, Recover, Flinch)
- Flinch resistance system (accumulation + regeneration)
- Attack cooldown management
- Data-driven attacks (DT_MavAttacks)
---

## Surface Material System

**Status:** ✅ Complete  
**Session:** 9

### Components

- Surface type enum (E_SurfaceType: Stone, Wood)
- BPI_Damageable interface extended with GetSurfaceType()
- Bounce logic in hit detection (BP_ThirdPersonCharacter)

### Behavior

- **Stone surfaces:** Deflect attacks, play bounce animation, no damage
- **Wood surfaces:** Normal damage with full combat feedback
- **Multi-target:** Handles sequential hits correctly

### Variables

- `SurfaceType` (E_SurfaceType) - Per-enemy variable

### Implementation

- Added to hit detection after "Already Hit Check"
- Interface check → GetSurfaceType → Switch on enum
- Stone path: Bounce animation, reset CurrentAttackName
- Wood path: Existing damage → camera shake → hitstop flow

### Test Assets

- BP_TestDummy (Stone variant) - Stone texture, bounces attacks
- BP_TestDummy_Wood - Wood texture, takes damage normally

### Future Foundation

- Boss part-break system (armored parts = stone, vulnerable = wood)
- Dynamic material transitions (stone breaks → becomes wood)
- Surface-specific VFX/SFX

---

## Audio System

**Status:** ✅ Complete

### Hit Sounds
- Function library: BPL_AudioHelpers
- Data-driven: DT_[Weapon]HitSounds_[Surface]
- Randomized playback (3-4 variants per table)
- Sound concurrency (max 3 instances, prevents clipping)

### Integration
- Enemies reference sound tables
- Surface type determines which table to use
- Scales with part-break system (armored vs vulnerable)

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
