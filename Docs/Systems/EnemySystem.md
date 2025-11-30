# Enemy System Documentation

**Status:** ✅ Boss AI Complete + Combat State + Hitbox System + Hit React + Stun  
**Sessions:** 16-20.11.2025, 24-28.11.2025  
**Latest:** Hit react (layered blend), stun states (Begin→End), death in ABP, ProcessDamage/ProcessFlinch refactor

---

## Overview

The enemy system uses parent-child Blueprint architecture where **BP_EnemyBase** contains all reusable boss logic, and specific boss instances configure the details. This allows rapid creation of new bosses by inheriting from the base class.

**Major Updates:**

- **Session 20.11:** Complete Boss AI state machine with flinch resistance system
- **Session 24-25.11:** Combat State system - threatening AI behavior within danger zone
- **Session 26.11:** Threat system, attack gap-close, hitbox arrays, lock-on socket
- **Session 27.11:** Mesh-based hit detection, AlreadyHitActors array, FlinchLevel in attacks
- **Session 28.11:** Hit react (layered blend), stun system (Begin→End), death ABP state, ProcessDamage/ProcessFlinch refactor, Attacker reference in interface

---

## BP_EnemyBase (Parent Class)

**Type:** Character Blueprint  
**Purpose:** Reusable boss foundation containing all core enemy systems  

### Components

**Mesh** (Skeletal Mesh Component - inherited from Character)

- Displays enemy model
- Assigned to specific character mesh per boss
- Drives AnimBP (per-boss configurable)
- **Note:** Mixamo meshes require Z rotation = -90° in component

**Capsule Component** (inherited from Character)

- Root component
- Handles physics collision and overlap
- Disabled on death

**CharacterMovement** (inherited from Character)

- AI locomotion
- Max Walk Speed: 400 (can be tuned per boss)
- Orient Rotation to Movement: True
- Rotation Rate Z: 540

**AttackHitboxes** (Array of Collision Components)

- Configurable per boss via child collisions
- Socket-based hitboxes attached to weapon/hand bones
- Collision Preset: NoCollision (default)
- Enabled/disabled via ANS_EnableEnemyHitbox during attacks
- Dynamic event binding on BeginPlay

**LockOnTarget** (Scene Component)

- Attached to socket on mesh (e.g., chest height)
- Camera lock-on follows this component
- Variable: `LockOnTargetSocket` (Name) - configurable per boss

### Per-Boss Configuration

BP_EnemyBase is fully configurable. Child blueprints only set:

| Variable | Type | Purpose |
|----------|------|---------|
| Mesh | Skeletal Mesh | Boss model |
| AttackHitboxes | Array of Collision Components | Hitbox collision shapes |
| AnimBP | Anim Blueprint Class | Per-boss animation logic |
| AttackDataTable | DataTable | DT_[BossName]Attacks |
| CombatBehaviorTable | DataTable | DT_[BossName]CombatBehaviors |
| LockOnTargetSocket | Name | Socket for camera lock-on |

---

## State Machine System

### E_BossState Enum

**Values:**

- `Idle` - Waiting for player
- `Approach` - Chasing player (outside danger zone)
- `Combat` - Threatening behavior in danger zone
- `Attack` - Executing attack
- `Recover` - Post-attack recovery
- `Flinch` - Hit reaction interrupt (stun)
- `Dead` - Terminal state

### State Flow

```
Idle → Approach (player detected at AggroRange)
Approach → Combat (player enters AttackRange)
Combat → Attack (ThreatLevel >= ThreatThreshold)
Attack → Recover → Combat (attack complete)
Any → Flinch (resistance breaks)
Any → Dead (health <= 0)
```

---

## Damage Processing (Session 28.11)

### ProcessDamage Function

**Location:** BP_EnemyBase → Functions  
**Purpose:** Handle incoming damage, health reduction, death check

**Signature:**

- Input: `IncomingDamage` (Float), `Attacker` (Actor Reference)
- Output: None

**Implementation:**

```
ProcessDamage (IncomingDamage, Attacker):
    ↓
    Branch: bIsDead?
    ├─ True → Return (ignore damage when dead)
    │
    └─ False → 
        CurrentHealth - IncomingDamage = NewHealth
        Set CurrentHealth = NewHealth
        ↓
        Branch: CurrentHealth <= 0?
        ├─ True → Set CurrentHealth = 0
        │         Set bIsDead = True
        │         Set CurrentBossState = Dead
        │
        └─ False → Continue
```

---

### ProcessFlinch Function

**Location:** BP_EnemyBase → Functions  
**Purpose:** Handle hit reactions and flinch resistance

**Signature:**

- Input: `IncomingDamage` (Float), `Attacker` (Actor Reference)
- Output: None

**Implementation:**

```
ProcessFlinch (IncomingDamage, Attacker):
    ↓
    Branch: bShouldFlinch? (already in stun)
    ├─ True → Return (skip hit react during stun)
    │
    └─ False → 
        === HIT REACT (micro-flinch) ===
        Calculate hit angle:
            Find Look At Rotation (Start: Self, Target: Attacker)
            Delta Rotator (A: LookAtRotation, B: Self Rotation)
            Yaw → LastHitAngle
        ↓
        Set bJustGotHit = True
        ↓
        Set Timer: ClearHitReact (0.25s)
        ↓
        === FLINCH RESISTANCE ===
        CurrentFlinchResistance - IncomingDamage
        ↓
        Branch: CurrentFlinchResistance <= 0?
        ├─ True → Set bShouldFlinch = True
        │         Set CurrentBossState = Flinch
        │         Reset CurrentFlinchResistance = MaxFlinchResistance
        │
        └─ False → Continue (just hit react, no stun)
```

### ClearHitReact Function

**Location:** BP_EnemyBase → Functions  
**Purpose:** Clear hit react flag after timer

```
ClearHitReact:
    Set bJustGotHit = False
```

---

### ApplyDamageToActor (Updated 28.11)

**Location:** BP_EnemyBase → Functions  
**Interface:** BPI_Damageable

**Signature:**

- Input: `Damage` (Float), `FlinchLevel` (Integer), `Attacker` (Actor Reference)
- Output: None

**Implementation:**

```
Event ApplyDamageToActor (Damage, FlinchLevel, Attacker):
    ↓
    ProcessDamage (Damage, Attacker)
    ↓
    ProcessFlinch (Damage, Attacker)
```

**Note:** Boss ignores FlinchLevel parameter - uses own flinch resistance system. Parameter exists for interface consistency with player.

---

## Hit React System (Session 28.11)

**Purpose:** Visual feedback on every hit without interrupting actions

### Variables

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bJustGotHit` | Boolean | False | Triggers hit react blend |
| `LastHitAngle` | Float | 0.0 | Direction of hit (-180 to 180) |
| `HitReactWeight` | Float | 0.0 | Blend weight (ABP) |

### ABP Implementation (Layered Blend Per Bone)

**AnimGraph Structure:**

```
[Locomotion State Machine] → Layered Blend Per Bone → Output
                                    ↑
                         [Hit React Blend Space]
                         (weight driven by HitReactWeight)
```

**Layered Blend Per Bone Settings:**

- **Base Pose:** Locomotion output
- **Blend Pose:** Hit react blend space (BS_HitReact)
- **Blend Weights:** HitReactWeight variable
- **Bone Name:** spine_01
- **Blend Depth:** 0 (all children)

**Event Graph (ABP) - Smooth Blend:**

```
Event Blueprint Update Animation:
    ↓
    Branch: bJustGotHit?
    ├─ True → Set HitReactWeight = 1.0
    └─ False → FInterpTo (Current: HitReactWeight, Target: 0.0, DeltaTime, Speed: 5.0)
               → Set HitReactWeight
```

**Result:**

- Hit lands → upper body reacts (blend space based on direction)
- Smooth fade out over ~0.2s
- Legs/locomotion unaffected
- No action interruption

---

## Stun System (Session 28.11)

**Purpose:** Full interruption when flinch resistance breaks

### ABP States

**Flinch_Begin:**

- Plays stun entry animation
- Transition in: `bShouldFlinch == True` (from any state)
- Transition out: Automatic (when animation finishes)

**Flinch_End:**

- Plays stun recovery animation
- Transition in: Automatic from Flinch_Begin
- Transition out: `bShouldFlinch == False`
- **AN_BossFlinchEnd** notify at end

### AN_BossFlinchEnd (AnimNotify)

**Location:** Animation Notifies  
**Purpose:** Clear stun state when animation completes

**Implementation:**

```
Received_Notify:
    Get Owning Actor → Cast to BP_EnemyBase
    ↓
    Set bShouldFlinch = False
```

**Note:** Boss state transition (Flinch → Combat) handled in TickAI Flinch branch, not in notify.

### Flinch State in TickAI

```
Flinch State:
    ↓
    Branch: bShouldFlinch?
    ├─ True → Do nothing (animation playing)
    └─ False → Set CurrentBossState = Combat
               Print: "Stun ended, back to combat"
```

---

## Death System (Session 28.11)

### ABP Death State

**Location:** ABP_Boss_GothicKnight → Locomotion State Machine

**State Configuration:**

- **Animation:** Death animation sequence
- **Looping:** Unchecked (play once, hold final frame)
- **Transition In:** `bIsDead == True` (from any state)
- **Transition Out:** None (terminal state)

### Dead State in TickAI

```
Dead State:
    ↓
    [Do nothing - terminal state]
```

**Pre-State Check:**

- TickAI checks `bIsDead` before entering state machine
- Also checks `PlayerRef.bIsDead` - stops AI if player dead

---

## Threat System (Session 26.11)

**Purpose:** Boss builds up aggression over time in Combat state, attacks when threshold reached

### Variables

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `ThreatLevel` | Float | 0.0 | Current accumulated threat |
| `ThreatThreshold` | Float | 100.0 | Threat needed to trigger attack |
| `ThreatAccumulationRate` | Float | 50.0 | Threat gained per second in Combat |

### ProcessThreatAccumulation Function

**Location:** BP_EnemyBase → Functions  
**Called from:** CombatStateBranch (every tick while in Combat)

```
ProcessThreatAccumulation:
    ↓
    Branch: bIsDead? (Session 28.11 addition)
    ├─ True → Return (dead boss doesn't accumulate threat)
    │
    └─ False →
        ThreatLevel + (ThreatAccumulationRate * DeltaSeconds)
        ↓
        Set ThreatLevel
        ↓
        Branch: ThreatLevel >= ThreatThreshold?
        ├─ True → Set ThreatLevel = 0 (reset)
        │         Set CurrentBossState = Attack
        │         Print: "Threat Reached, attacking"
        │
        └─ False → Continue in Combat state
```

**Design Notes:**

- Threat resets on attack (not after attack completes)
- Rate tunable per boss via variable
- Higher threshold = more circling before attack
- Lower threshold = more aggressive boss
- Dead boss check prevents threat accumulation after death

---

## Combat State (Session 24-25.11)

**Purpose:** Threatening AI behavior within combat range - circling, feinting, positioning

**Entry:** Approach state when `IsPlayerInRange(AttackRange)`

### Combat State Variables

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| CombatBehaviorTable | DataTable | - | Per-boss behavior config |
| CurrentCombatBehavior | S_CombatBehavior | - | Active behavior data |
| CurrentMovementType | E_CombatMovementType | - | Current movement pattern |
| CombatBehaviorEndTime | Float | 0 | When to select new behavior |
| LastCombatBehavior | Name | - | For repeat prevention |
| TotalWeight | Float | 0 | Sum of all SelectionWeights |
| AccumulatedWeight | Float | 0 | Running sum during selection |
| RandomValue | Float | 0 | Target value for weighted selection |
| ValidRowNames | Name Array | - | Local: filtered behavior candidates |

---

### E_CombatMovementType Enum

| Value | Behavior |
|-------|----------|
| None | Stand still, face player |
| TowardPlayer | Close distance |
| AwayFromPlayer | Create space |
| StrafeLeft | Orbit clockwise |
| StrafeRight | Orbit counter-clockwise |

---

### S_CombatBehavior Struct

| Field | Type | Purpose |
|-------|------|---------|
| MinDuration | Float | Minimum behavior duration (seconds) |
| MaxDuration | Float | Maximum behavior duration (seconds) |
| MovementSpeed | Float | Speed during this behavior |
| SelectionWeight | Float | Higher = more likely to be chosen |
| bCanRepeat | Boolean | Can this behavior follow itself? |
| MovementType | E_CombatMovementType | How to move |
| MinDistanceToUse | Float | Don't use if closer than this |
| MaxDistanceToUse | Float | Don't use if farther than this |
| bCanAttackFrom | Boolean | Can transition to Attack from this behavior |

---

### SelectNewCombatBehavior Function

**Input/Output:** None

**Implementation:**

```
Function Start
    ↓
Set TotalWeight = 0
    ↓
Get Data Table Row Names (Table: CombatBehaviorTable) → OutRowNames
    ↓
═══ LOOP 1: Filter Valid Behaviors ═══
ForEach Loop (Array: OutRowNames)
    └─ Loop Body:
       Get Data Table Row (Table: CombatBehaviorTable, RowName: ArrayElement)
           └─ OutRow → Break S_CombatBehavior → bCanRepeat
                   ↓
               Branch: (LastCombatBehavior != ArrayElement) OR (bCanRepeat == True)?
               ├─ True → Add ArrayElement to ValidRowNames (local array)
               └─ False → skip
    └─ On Completed → continue
    ↓
═══ LOOP 2: Calculate Total Weight (filtered rows only) ═══
ForEach Loop (Array: ValidRowNames)
    └─ Loop Body:
       Get Data Table Row (Table: CombatBehaviorTable, RowName: ArrayElement)
           └─ OutRow → Break S_CombatBehavior → SelectionWeight
                   ↓
               Set TotalWeight = TotalWeight + SelectionWeight
    └─ On Completed → continue
    ↓
Set RandomValue = Random Float In Range [0, TotalWeight]
Set AccumulatedWeight = 0
    ↓
═══ LOOP 3: Weighted Selection (with Break) ═══
ForEach Loop with Break (Array: ValidRowNames)
    └─ Loop Body:
       Get Data Table Row (Table: CombatBehaviorTable, RowName: ArrayElement)
           └─ OutRow → Break S_CombatBehavior
                   ↓
               Set AccumulatedWeight = AccumulatedWeight + SelectionWeight
                   ↓
               Branch: AccumulatedWeight >= RandomValue?
               ├─ True → WINNER FOUND:
               │         - Set CurrentCombatBehavior = OutRow
               │         - Set LastCombatBehavior = ArrayElement
               │         - Set CurrentMovementType = OutRow.MovementType
               │         - Set CombatBehaviorEndTime = 
               │             GameTimeInSeconds + RandomFloatInRange(MinDuration, MaxDuration)
               │         - BREAK
               └─ False → continue loop
    └─ On Completed → Print "Winner: [behavior name]"
```

---

## Flinch Resistance System

### Variables

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `MaxFlinchResistance` | Float | 100.0 | Full resistance pool |
| `CurrentFlinchResistance` | Float | 100.0 | Current resistance value |
| `FlinchRegenRate` | Float | 20.0 | Resistance regained per second |
| `FlinchRegenDelay` | Float | 3.0 | Seconds before regen starts |
| `LastDamageTime` | Float | 0.0 | When last hit received |

### Flinch Resistance Flow

```
Player hits boss
    ↓
ProcessFlinch reduces CurrentFlinchResistance by damage
    ↓
Branch: CurrentFlinchResistance <= 0?
├─ True → STUN
│         Set bShouldFlinch = True
│         Set CurrentBossState = Flinch
│         Reset resistance to max
│
└─ False → Hit react only (micro-flinch)
           Start regen delay timer
```

### Resistance Regeneration

```
Every Tick (in Combat/Approach states):
    ↓
    Branch: GameTime - LastDamageTime > FlinchRegenDelay?
    ├─ True → Regenerate:
    │         CurrentFlinchResistance + (FlinchRegenRate * DeltaTime)
    │         Clamp to MaxFlinchResistance
    │
    └─ False → Wait (recently hit)
```

---

## Animation Blueprints

### ABP_Boss_GothicKnight

**Type:** Per-boss Animation Blueprint  
**Target Skeleton:** Gothic Knight skeleton

**Event Graph - pulls from owner every frame:**

- Speed (from velocity magnitude)
- Direction (from velocity + rotation)
- bIsDead (from BP_Boss_GothicKnight)
- bShouldFlinch (from BP_EnemyBase)
- bJustGotHit (from BP_EnemyBase)
- LastHitAngle (from BP_EnemyBase)
- HitReactWeight (interpolated locally)

**AnimGraph Structure:**

```
[Locomotion State Machine] → [Layered Blend Per Bone] → Output Pose
                                      ↑
                            [Hit React Blend Space]
```

**Locomotion State Machine:**

- Idle (combat idle with DefaultSlot)
- JogStart → JogLoop → JogStop (locomotion)
- Flinch_Begin → Flinch_End (stun sequence)
- Death (terminal state)

**State Transitions:**

| From | To | Condition |
|------|-----|-----------|
| Any | Flinch_Begin | bShouldFlinch == True |
| Flinch_Begin | Flinch_End | Automatic (animation complete) |
| Flinch_End | Idle | bShouldFlinch == False |
| Any | Death | bIsDead == True |
| Idle | JogStart | Speed > 10 |
| JogLoop | Idle | Speed < 10 |

**Layered Blend Per Bone (Hit React):**

- Bone: spine_01
- Blend Depth: 0
- Weight: HitReactWeight (0.0 to 1.0)

### Pattern for New Bosses

Each boss gets its own AnimBP that:

1. Pulls state from BP_EnemyBase via owner cast
2. Has boss-specific state machine transitions
3. Uses DefaultSlot in Idle for montage playback
4. Implements hit react layered blend
5. Has Flinch_Begin → Flinch_End states
6. Has Death terminal state

---

## Current Boss: Gothic Knight

**Source:** Professional animation pack  
**Type:** Humanoid knight with 2H sword

### Configuration in BP_Boss_GothicKnight

- Mesh: SKM_GK_Full_With_Sword
- AnimBP: ABP_Boss_GothicKnight
- AttackDataTable: DT_GothicKnightAttacks
- CombatBehaviorTable: DT_GothicKnightCombatBehaviors
- LockOnTargetSocket: Socket at chest height

Everything else inherited from BP_EnemyBase.

---

## AI Configuration Variables

### Movement & Detection

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `AggroRange` | Float | 1000.0 | How far boss detects player |
| `DangerZoneRange` | Float | 600.0 | Distance to enter Combat state |
| `AttackRange` | Float | 300.0 | Distance to trigger attack |
| `ApproachSpeed` | Float | 400.0 | Movement speed when chasing |

### Combat State

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `CurrentBossState` | E_BossState | Idle | Active state |
| `bIsExecutingAttack` | Boolean | False | Currently attacking? |
| `LastAttackTime` | Float | 0.0 | When last attack occurred |
| `CurrentAttackCooldown` | Float | 0.0 | Cooldown from attack table |
| `CurrentAttackData` | S_BossAttackData | - | Current attack being executed |
| `bShouldFlinch` | Boolean | False | For ABP - triggers stun state |
| `bJustGotHit` | Boolean | False | For ABP - triggers hit react |
| `LastHitAngle` | Float | 0.0 | Direction of last hit |

### Threat System

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `ThreatLevel` | Float | 0.0 | Current accumulated threat |
| `ThreatThreshold` | Float | 100.0 | Threat needed to attack |
| `ThreatAccumulationRate` | Float | 50.0 | Threat per second |

### Hitbox System

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `AttackHitboxes` | Array of Collision Components | - | All attack hitboxes |
| `AlreadyHitActors` | Array of Actor References | Empty | Multi-hit prevention |

### References

| Variable | Type | Purpose |
|----------|------|---------|
| `PlayerRef` | Actor Reference | Cached player for distance checks |
| `AttackDataTable` | DataTable Reference | Per-boss attack config |
| `CombatBehaviorTable` | DataTable Reference | Per-boss combat behaviors |

---

## BPI_Damageable Interface (Updated 28.11)

**Function:** ApplyDamageToActor

**Signature:**

- Input: `Damage` (Float) - Raw damage from attacker
- Input: `FlinchLevel` (Integer) - Flinch severity (ignored by boss)
- Input: `Attacker` (Actor Reference) - Who dealt the damage
- Output: None

**Boss Implementation:**

- Uses Damage for health reduction and flinch resistance
- Ignores FlinchLevel (uses own resistance system)
- Uses Attacker for hit angle calculation

---

## Known Issues

### Fixed (Session 26-28.11)

- ✅ TotalWeight calculated from all rows (now filtered first)
- ✅ Camera lock targets root (now uses LockOnTarget socket)
- ✅ Multi-hit per swing (AlreadyHitActors array)
- ✅ Capsule-based hit detection (now mesh-based)
- ✅ No hit feedback (hit react system added)
- ✅ No stun animation (Flinch_Begin → Flinch_End states)
- ✅ Death not in ABP (Death state added)
- ✅ Threat accumulates after death (dead check added)

### Remaining Issues

- **Distance filtering not implemented:** MinDistanceToUse/MaxDistanceToUse not checked in filter loop
- **No fallback for empty ValidRowNames:** If all behaviors filtered out, no selection
- **Attack variety:** Gothic Knight only has 1 attack configured
- **Boss rotation during attack:** Unclear repro, low priority

### Combat Tuning Needed

- Attack telegraphs (visual wind-up cues)
- More attack variety (different timings, ranges)
- Threat accumulation rate tuning
- Flinch threshold tuning
- Hit react animation quality (current is placeholder)

---

## Future Enhancements

### High Priority

1. **Implement distance filtering** - Check MinDistanceToUse/MaxDistanceToUse
2. **Add fallback behavior** - Default when ValidRowNames empty
3. **Multiple boss attacks** - Add variety to combat
4. **Attack telegraphs** - Visual/audio wind-ups beyond grunt

### Medium Priority

5. **Heavy attacks** - FlinchLevel 2-3 attacks with longer wind-ups
6. **Combo attacks** - Multi-hit sequences
7. **Recovery vulnerability** - Window after attack where boss takes extra damage
8. **Better hit react animations** - Additive animations purpose-made for layering

### Future Systems

9. **Extended stun (loop)** - Begin → Loop → End for longer stagger
10. **Part-Break System** - MH-style targeting
11. **Boss #2** - Validate reusability of BP_EnemyBase

---

## Architecture Validation

### What's Proven

**Hit React System (Session 28.11):**

- ✅ Layered blend per bone working
- ✅ Directional hit react via blend space
- ✅ Smooth fade out with FInterpTo
- ✅ Doesn't interrupt actions
- ✅ Separate from stun system

**Stun System (Session 28.11):**

- ✅ Flinch_Begin → Flinch_End flow working
- ✅ AN_BossFlinchEnd clears state
- ✅ TickAI Flinch branch transitions to Combat
- ✅ Hit react disabled during stun

**Death System (Session 28.11):**

- ✅ Death state in ABP working
- ✅ Terminal state (no exit)
- ✅ bIsDead check prevents AI processing

**Damage Processing (Session 28.11):**

- ✅ ProcessDamage / ProcessFlinch separation clean
- ✅ Attacker reference enables hit direction
- ✅ Dead check at start of both functions

**Combat State foundation works:**

- ✅ Weighted random selection works (after fix)
- ✅ Data-driven behavior tables work
- ✅ Movement types (strafe, retreat, approach) work
- ✅ Threat accumulation triggers attacks
- ✅ Boss exhibits threatening presence

**Parent-child architecture validated:**

- ✅ BP_EnemyBase contains all systems
- ✅ Child blueprints are pure configuration
- ✅ New boss = set variables (mesh, AnimBP, tables)

**Per-boss AnimBP pattern established:**

- ✅ Each boss can have unique animation logic
- ✅ Pulls state from parent cleanly
- ✅ Supports different locomotion styles
- ✅ Hit react + stun + death pattern reusable

---

## Key Learnings

### Session 26.11

1. **Threat System creates pacing**
   - Boss doesn't attack immediately on sight
   - Builds tension, gives player time to prepare
   - Tunable per boss

2. **Add Movement Input for direct control**
   - AI Move To uses pathfinding (not wanted for gap-close)
   - Add Movement Input = direct, responsive

3. **Socket-based lock-on**
   - LockOnTarget component follows animations
   - Camera looks at chest, not root

### Session 27.11

4. **Mesh check before array check**
   - Order matters! Capsule fires first
   - Filter invalid hits before touching the array

5. **FlinchLevel in interface**
   - Slightly impure but most pragmatic
   - Boss reports, player calculates
   - Same pattern as damage

6. **Query Only for hit detection**
   - Mesh needs overlap events but not physics
   - Physics Asset set to Kinematic

### Session 28.11

7. **Layered Blend Per Bone for hit react**
   - Blend space always "plays" - weight controls visibility
   - FInterpTo for smooth fade out
   - spine_01 with Blend Depth 0 = upper body only

8. **Separate hit react from stun**
   - Hit react = juice (every hit, no interruption)
   - Stun = mechanic (resistance breaks, full interruption)
   - bShouldFlinch blocks hit react during stun

9. **ProcessDamage / ProcessFlinch separation**
   - Clean function boundaries
   - Easy to debug (print at function start)
   - Expandable without bloating ApplyDamageToActor

10. **Attacker reference enables direction**
    - Same pattern as player flinch
    - Interface carries the data
    - Receiver calculates angle

---

## Session Stats

**Session 24-25.11.2025:**

- Combat State system with weighted behavior selection
- E_CombatMovementType enum
- S_CombatBehavior struct
- SelectNewCombatBehavior function
- BP_EnemyBase refactored for reusability
- ABP_Boss_GothicKnight created

**Session 26.11.2025:**

- Threat System (ThreatLevel, ThreatThreshold, ProcessThreatAccumulation)
- Fixed SelectNewCombatBehavior loop order
- Attack State gap-close with Add Movement Input
- OnAttackAnimationComplete function
- AttackHitboxes array with dynamic binding
- DeliverHitboxDamage function
- LockOnTarget socket attachment

**Session 27.11.2025:**

- Mesh-based hit detection (HitComponent == Mesh check)
- AlreadyHitActors array for multi-hit prevention
- Critical fix: mesh check before array check
- FlinchLevel added to S_BossAttackData
- BPI_Damageable updated with FlinchLevel parameter
- Attack audio (grunt + swing)

**Session 28.11.2025:**

- Hit react system (layered blend per bone)
- Directional hit react (blend space + LastHitAngle)
- Smooth blend fade out (FInterpTo in ABP Event Graph)
- Stun system (Flinch_Begin → Flinch_End ABP states)
- AN_BossFlinchEnd notify
- Death state in ABP
- ProcessDamage / ProcessFlinch function refactor
- Attacker reference added to BPI_Damageable
- Dead check in threat accumulation
- bShouldFlinch blocks hit react during stun

---

*Last Updated: 28.11.2025*  
*Status: Hit react working, stun working, death working, damage processing refactored*  
*Next: Attack variety → Distance filtering → Attack telegraphs*
