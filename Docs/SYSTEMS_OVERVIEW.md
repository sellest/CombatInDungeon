# Systems Overview

Quick reference for all implemented game systems.

**Last Updated:** January 2026

---

## Locomotion System

**Status:** ✅ Complete (Armed/Unarmed/Block/Turns)  
**Sessions:** December 2025 - January 2026  
**Blueprint:** ABP_Unarmed, BP_ThirdPersonCharacter

### Core Features

- Physics-based acceleration (not root motion)
- Armed and Unarmed states with full locomotion
- Sheathe/Unsheathe system with weapon attachment
- Block locomotion (walk only, no run)
- Idle turns with angle snapping
- Input commitment (locked during Start/Stop animations)
- Sync marker foot alignment

### Key Variables

| Variable | Purpose |
|----------|---------|
| `TargetSpeed` | Intent (0, 250, 600) |
| `CurrentSpeed` | Actual speed, interpolates toward TargetSpeed |
| `bLockMovementInput` | Prevents input during committed animations |
| `bIsArmed` | Armed/Unarmed state |
| `bWantsTurn` | Turn intent flag |
| `bIsBlocking` | Currently blocking |
| `bBlockEstablished` | Block stance achieved (for routing) |

### State Machine (JogSequence)

```
Entry → Armed? (Conduit)
  ├─ Unarmed: Idle ↔ Turn ↔ Walk/Run states
  ├─ Armed: Armed_Idle ↔ Armed_Turn ↔ Armed_Walk/Run states
  ├─ Block: Block_Idle ↔ Guard_Walk states
  └─ Sheathe transitions between Armed/Unarmed
```

### Animation Notify States

- **ANS_LockMovementInput** - Commits to Start/Stop animations
- **ANS_SlowRotation** - Reduces rotation speed during transitions
- **AN_ClearTurnIntent** - Clears stale turn flags

**See:** LocomotionSystem.md for full documentation

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

## Debug Visualization System - Boss AI Geometry

### Overview

The debug visualization system provides real-time visual feedback of the boss AI's spatial awareness and geometry-based decision making. It renders geometric zones, attack ranges, and player position data to aid in development, testing, and tuning.

**Purpose:**
- Visualize the 12 geometric sectors (3 distance bands × 4 directional cones)
- Show which sector the player currently occupies
- Display optimal angle precision zones
- Debug attack selection geometry
- Validate hitbox traces
- Tune distance bands and angle requirements

---

### Enabling/Disabling Debug Mode

**Master Toggle:**
- **Variable:** `bShowDebugGeometry` (Boolean) in BP_EnemyBase
- **Default:** False (disabled)
- **Set at runtime:** Can be toggled via Blueprint logic or debug console

**What Gets Drawn When Enabled:**
1. Distance band circles (3 rings)
2. Cone boundary lines (4 radial lines)
3. Dynamic optimal angle wedge (changes with player position)
4. Weapon trace visualization (red/green spheres during attacks)
5. Socket location spheres (during attacks)

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

## Roll System (Evasion)

**Status:** ✅ Complete (Armed/Unarmed)  
**Sessions:** January 2026  
**Blueprint:** ABP_Unarmed (Top Level), BP_ThirdPersonCharacter

### Core Features

- Top-level ABP state (interrupts most actions)
- Directional rolls when locked on (4 directions)
- Rotate-then-roll when free camera
- I-frames (frames 10-45 of 66)
- Cancel window (frames 50+)
- Can cancel attack recovery

### Key Variables

| Variable | Purpose |
|----------|---------|
| `bWantsRoll` | Intent flag |
| `bIsDodging` | Currently rolling |
| `RollDirection` | BlendSpace input (-180 to 180) |
| `bRotatingIntoRoll` | Pre-roll rotation in progress |
| `TargetRollRotation` | Target yaw for rotation |
| `bIdleExit` | Animation reached idle exit point |

### Animation Timing (66 frames)

| Frames | Phase |
|--------|-------|
| 0-10 | Windup (vulnerable) |
| 10-45 | Active roll (I-frames) |
| 45-50 | Early recovery (locked) |
| 50-65 | Late recovery (cancel window) |

### Animation Notifies

- **ANS_IFrames (10-45)** - Invulnerability
- **ANS_LockDefense (0-50)** - Prevents block/roll spam
- **ANS_LockInput (0-50)** - Prevents attack during roll
- **AN_DodgeComplete (49)** - Resets flags
- **AN_RollIdleExit (65)** - Enables idle exit

### Behavior

**Locked On:**
- Roll direction from input (Atan2)
- Character stays facing target
- 4-directional rolls

**Free Camera:**
- Calculate input world direction
- If angle > 15°: rotate first, then roll forward
- Fast interpolation (35.0 speed)

**See:** RollSystem.md for full documentation

---

## Block System

**Status:** ✅ Complete with Locomotion  
**Sessions:** November 2025, January 2026 (locomotion integration)

### Core Features

- Hold input (R2/RT)
- Armed only (must have weapon drawn)
- Full locomotion while blocking (walk only, no run)
- Block_Start → Block_Idle → Block_End sequence
- 70% damage reduction

### Key Variables

| Variable | Purpose |
|----------|---------|
| `bIsBlocking` | Currently holding block |
| `bBlockEstablished` | Block stance achieved |
| `bCanBlock` | Can block (ANS_LockDefense control) |

### State Machine Integration

Block states live inside JogSequence:
```
Armed_Idle ↔ Block_Idle
  │
  ├─ Block_Start → Block_Idle (fresh block)
  ├─ Block_Idle → Block_End → Armed_Idle (release)
  │
Guard_Walk states:
  ├─ Guard_Walk_Start → Guard_Walk_Loop
  ├─ Guard_Walk_Loop → Guard_Walk_Stop
  └─ Guard_Walk_Stop → Block_Idle
```

### bBlockEstablished Logic

**Purpose:** Routes correctly when returning from movement.

- **Set True:** End of Block_Start, or when pressing block while moving
- **Set False:** When block released

**Routing:**
- `bIsBlocking AND bBlockEstablished` → Block_Idle (skip start)
- `bIsBlocking AND NOT bBlockEstablished` → Block_Start

### Defense Integration

- CalculateDamageReduction checks bIsBlocking
- 70% reduction (multiplier 0.3)
- Block reactions: Light hit, Block break

**See:** LocomotionSystem.md (Block Locomotion section)

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

1. **Flinch System** - Player hit reactions (Light, Heavy, Knockdown)
2. **Stamina System** - Resource for rolls, attacks, blocking
3. **Combat Polish** - Attack tuning, boss AI adjustments
4. **Boss Exhaustion** - Combat rhythm system
5. **Part-Break System** - Core pillar, not yet implemented

---

## January 2026 Playtest Checklist

**Player Character:**
- ✅ Locomotion (Armed/Unarmed/Block/Turns)
- ✅ Sheathe/Unsheathe
- ✅ Combat (Attacks, Combos)
- ✅ Block with locomotion
- ✅ Roll with I-frames
- ⬜ Flinch reactions
- ⬜ Stamina

**Boss:**
- ✅ AI State Machine
- ✅ Attack system
- ⬜ Exhaustion system
- ⬜ Multiple attack patterns

**Polish:**
- ⬜ Minor transition twitches
- ⬜ Sharp direction snap (lock-on strafe)
