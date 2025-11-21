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

## Dodge System

**Status:** ✅ Complete with I-Frames  
**Sessions:** 9.11 (initial), 21.11 (i-frames)

### Components

**Two modes: free camera and lock-on camera**

**Unlocked camera:**
- Always performs Forward animation
- Character rotates to face movement direction

**Locked camera:**
- Character always faces target
- 4 animations: Forward/Backward/Left/Right
- Direction: Based on WASD (LastMoveForward/Right)

**Input:** Spacebar

### I-Frames System (Session 21.11)

**ANS_IFrames** - AnimNotifyState for invulnerability

**Placement in dodge montages:**
- Start: 0.1s (after initial step)
- End: 0.5s (before recovery)
- Duration: 0.4s (50% of dodge - God of War 2018 style)

**Effect:**
- Sets `bIsInvulnerable = True`
- CalculateDamageReduction returns 0 damage
- Print: "Dodged! No damage received"

**Design:** God of War style (generous, middle-placed) vs Monster Hunter style (tight, start-only)

**Why 50% coverage:**
- Balances accessibility with skill
- Compensates for close camera (limited visibility)
- Matches strafing aesthetic (not roll)
- Rewards reads over reflexes

### Variables

- `bIsDodging` (Boolean)
- `bIsInvulnerable` (Boolean) - I-frames active
- `LastMoveForward` (Float)
- `LastMoveRight` (Float)

### Functions

- `ExecuteDodge(Montage)` - Plays dodge animation
- Direction determined by movement input at press time

### Montages

- AM_Dodge_Forward (with ANS_IFrames)
- AM_Dodge_Backward (with ANS_IFrames)
- AM_Dodge_Left (with ANS_IFrames)
- AM_Dodge_Right (with ANS_IFrames)

**See:** DodgeSystem.md for full documentation

---

## Block System

**Status:** ✅ Complete with Locomotion  
**Session:** 18-20.11.2025

### Components

**Input:** R2/RT (IA_Block) - Hold input

**Movement while blocking:**
- Max Walk Speed: 200 → 100 (50% slower)
- Full 8-directional movement
- Camera-aware animations (locked vs unlocked)

**Defense (Session 21.11):**
- Damage reduction: 70% (multiply by 0.3)
- Integrated with player damage system
- Print: "Blocked! {IncomingDamage} → {ReducedDamage}"

### Variables

- `bIsBlocking` (Boolean) - Currently in blocking state
- `bCanBlock` (Boolean) - Can block? (ANS_LockDefense control)
- `bBlockPressed` (Boolean) - Hold-state tracking

### Animation System

**ABP_Unarmed → BlockSequence (nested state machine):**
- GuardBegin - Shield raise (~0.5s)
- GuardIdle - Main locomotion (UE5_SSH_Guard_Jog_Forward_BlendSpace)
- GuardEnd - Shield lower (~0.5s)

**Blend Space Inputs:**
- Degree: Direction (-180° to 180°)
- Speed: GroundSpeed (0-800)
- Camera-aware: Unlocked = force forward, Locked = use directional

### Integration

**With Combat System:**
- ExecuteBlock function clears CurrentAttackName, bCanCombo
- ANS_LockDefense controls when player can block during attacks
- Hold-state tracking enables blocking during attack animations

**With Damage System (Session 21.11):**
- CalculateDamageReduction checks bIsBlocking
- Applies 0.3 multiplier (70% reduction)
- Boss delivers 10 damage → Player takes 3 damage

**See:** BlockSystem.md for full documentation

---

---

## Lock-On System


**Status:** ✅ Complete (Session 21.11 fixes)  
**Sessions:** 8-9.11 (initial), 21.11 (chest targeting)

### Components

- Toggle: Tab key (IA_LockOn)
- God of War 2018 style (character faces target, strafing)

### Variables

- `bIsLockedOn` (Boolean)
- `LockedTarget` (Scene Component Reference) - **Updated Session 21.11**

**Type Change:** Actor Reference → Scene Component Reference
- Targets LockOnTarget component (chest height)
- Better camera framing than actor root (feet)

### Enemy Setup

**LockOnTarget Component** (Scene Component in BP_EnemyBase)
- Location: X=0, Y=0, Z=80 (chest height)
- Parented to Mesh
- Per-boss positioning

### Behavior

- **Unlocked:** Character rotates with movement (Orient Rotation to Movement = True)
- **Locked:** Character faces target, strafes (Orient Rotation to Movement = False)
- **Camera:** Aims at LockOnTarget component + offset [50, 50, 120]

### Integration

- Enables directional dodges (left/right make spatial sense)
- Enables directional block animations
- Camera macro: LockCamera (called in Event Tick)

**See:** LockOnSystem.md for full documentation

---

## Player Damage System

**Status:** ✅ Complete  
**Session:** 21.11.2025

### Components

**Health Management:**
- MaxHealth: 100.0 (configurable)
- CurrentHealth: 100.0 (runtime)

**State Flags:**
- bIsDead: Triggers death animation, stops boss AI
- bIsInvulnerable: I-frames from dodge (100% damage negation)

### Core Functions

**CalculateDamageReduction(IncomingDamage) → FinalDamage**
- Central mitigation logic
- Priority: I-frames (100%) → Block (70%) → Full damage
- Expandable: armor, buffs, guard points, etc.

**ApplyDamageToActor(Damage)**
- Interface: BPI_Damageable
- Calls CalculateDamageReduction
- Applies final damage to health
- Triggers HandlePlayerDeath or HandlePlayerHit

**HandlePlayerDeath()**
- Sets bIsDead = True
- Disables input (movement, camera, attacks)
- Triggers death animation in ABP_Unarmed

### Death Animation

**ABP_Unarmed → Death State:**
- Terminal state (no exit)
- Single death animation (currently)
- Future: Directional deaths, ragdoll option

### Integration

**With Block System:**
- bIsBlocking checked in CalculateDamageReduction
- 70% damage reduction (10 damage → 3 damage)

**With Dodge System:**
- bIsInvulnerable checked first (highest priority)
- 100% damage negation during i-frames
- Print: "Dodged! No damage received"

**With Boss AI:**
- Boss checks player.bIsDead before processing AI
- Boss stops attacking when player dies
- Boss just reports damage, player handles mitigation

**See:** PlayerDamageSystem.md for full documentation

---

## Boss AI System

**Status:** ✅ Complete + Player Damage Integration  
**Sessions:** 16-20.11 (AI), 21.11 (hitboxes + damage)

### State Machine

**E_BossState enum:**
- Idle - Waiting for player
- Approach - Chasing player
- Attack - Executing attack
- Recover - Post-attack recovery
- Flinch - Hit reaction interrupt
- Dead - Terminal state

**Pre-State Check (Session 21.11):**
- Checks player.bIsDead before entering state machine
- Boss stops AI if player dead
- Print: "He died. Mav's done."

### Attack System (Session 21.11)

**Socket-Based Hitboxes:**
- AttackHitbox (Sphere Collision Component)
- Attached to Socket_WeaponHitbox on hand_r bone
- Follows attack animation naturally
- Radius: 60 (tune per boss)

**ANS_EnableEnemyHitbox:**
- Enables hitbox during active attack frames
- Default: NoCollision
- Active: Query Only (overlaps with player)
- Placement: 0.3-0.5s during swing

**Damage Application:**
- Overlap with player → Cast to BP_ThirdPersonCharacter
- Get CurrentAttackData.Damage (from DT_MavAttacks)
- Player.ApplyDamageToActor(Damage)
- Print: "Boss delivered {Damage} damage!"

### Flinch Resistance System

**Variables:**
- MaxFlinchResistance: 100.0
- CurrentFlinchResistance: 100.0
- FlinchRegenRate: 20.0/second
- FlinchRegenDelay: 3.0 seconds

**Flow:**
- Each hit reduces resistance by damage amount
- Resistance <= 0 → Flinch state
- Flinch resets resistance to max
- Regenerates after 3s delay

### Data-Driven Attacks

**DT_MavAttacks (S_BossAttackData):**
- AttackName
- AttackMontage
- Damage (10.0 currently)
- Cooldown (3.0 seconds)
- MinRange / MaxRange (future)

**CurrentAttackData variable:**
- Stores active attack properties
- Hitbox overlap reads damage from this

**See:** EnemySystem.md for full documentation

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
- Boss attacks in DT_BossAttacks
- Easy to add new content without code changes

### Scalability

- Enum-based (type-safe)
- Normalized database structure
- Reusable systems (perfect timing, i-frames, hitboxes)
- Parent-child architecture (BP_EnemyBase → BP_Boss_Mav)

### Separation of Concerns

**Player owns:**
- Defensive state (blocking, i-frames, armor)
- Damage mitigation calculation
- Health management

**Boss owns:**
- Attack patterns
- AI state machine
- Raw damage reporting

**Clean interface:** Boss.Attack(Damage) → Player.ApplyDamage(Damage)

### Placeholder Assets

- Using Manny + Mixamo for prototyping
- $10k budget for production assets when ready
- Material/animation agnostic systems
- Focus: Systems over content

---

## Next Priorities

1. **Additional Inputs** (RMB, holds, Y+B simultaneous)
2. **Enemy AI** (basic chase/attack)
3. **Material/Bounce System** (stone vs wood surfaces)
4. **Damage System** (integrate perfect timing multiplier)
5. **Boss Part-Break** (foundation for MH-style gameplay)
