# Enemy System Documentation

**Status:** ✅ Boss AI Complete + Combat State + Hitbox System  
**Sessions:** 16-20.11.2025, 24-27.11.2025  
**Latest:** Mesh-based hit detection, threat system, flinch level delivery

---

## Overview

The enemy system uses parent-child Blueprint architecture where **BP_EnemyBase** contains all reusable boss logic, and specific boss instances configure the details. This allows rapid creation of new bosses by inheriting from the base class.

**Major Updates:**
- **Session 20.11:** Complete Boss AI state machine with flinch resistance system
- **Session 24-25.11:** Combat State system - threatening AI behavior within danger zone
- **Session 26.11:** Threat system, attack gap-close, hitbox arrays, lock-on socket
- **Session 27.11:** Mesh-based hit detection, AlreadyHitActors array, FlinchLevel in attacks

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
- `Flinch` - Hit reaction interrupt
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

**Correct Loop Order (Fixed Session 26.11):**
1. Filter behaviors (repeat check, distance check)
2. Calculate TotalWeight from filtered rows only
3. Weighted selection from filtered rows

---

### CombatStateBranch Macro (TickBossAI)

```
Combat pin → CombatStateBranch Macro:
    ↓
Branch: (GameTime > CombatBehaviorEndTime) OR (CombatBehaviorEndTime == 0)?
    │
    ├─ True → SelectNewCombatBehavior()
    │              ↓
    │         Set Max Walk Speed (Target: CharacterMovement)
    │              └─ Speed from CurrentCombatBehavior.MovementSpeed
    │              ↓
    │         [continues to Set Actor Rotation below]
    │
    └─ False → [skips directly to Set Actor Rotation]
            ↓
Set Actor Rotation
    └─ NewRotation = FindLookAtRotation(Start: Self Location, Target: Player Location)
            ↓
Switch on CurrentMovementType:
    │
    ├─ None → Stop Movement Immediately (Target: CharacterMovement)
    │
    ├─ TowardPlayer → Add Movement Input
    │                  - WorldDirection: Normalize(Player Location - Self Location)
    │                  - Scale: 1.0
    │
    ├─ AwayFromPlayer → Add Movement Input
    │                    - WorldDirection: Normalize(Self Location - Player Location)
    │                    - Scale: 1.0
    │
    ├─ StrafeLeft → Add Movement Input
    │                - WorldDirection: Normalize(RotateVectorAroundAxis(
    │                    InVect: Player Location - Self Location,
    │                    Axis: [0, 0, 1],
    │                    AngleDeg: 90))
    │                - Scale: 1.0
    │
    └─ StrafeRight → Add Movement Input
                     - WorldDirection: Normalize(RotateVectorAroundAxis(
                         InVect: Player Location - Self Location,
                         Axis: [0, 0, 1],
                         AngleDeg: -90))
                     - Scale: 1.0
            ↓
ProcessThreatAccumulation()
```

**Key Implementation Notes:**

- **True branch** calls SelectNewCombatBehavior AND sets walk speed
- **False branch** skips behavior selection, goes straight to rotation
- **Both branches** converge at Set Actor Rotation
- **All movement types** use Add Movement Input (direct control, no pathfinding)
- **Strafe** uses RotateVectorAroundAxis with 90° / -90° for perpendicular movement
- **ProcessThreatAccumulation** called every tick in Combat state

---

### Example: DT_GothicKnightCombatBehaviors

| Row Name | MinDur | MaxDur | Speed | Weight | CanRepeat | MovementType | MinDist | MaxDist |
|----------|--------|--------|-------|--------|-----------|--------------|---------|---------|
| MoveRight | 1.5 | 3.0 | 200 | 1.0 | False | StrafeLeft | 0 | 9999 |
| MoveLeft | 1.5 | 3.0 | 200 | 1.0 | False | StrafeRight | 0 | 9999 |
| MoveForward | 0.5 | 1.5 | 350 | 0.8 | False | TowardPlayer | 300 | 9999 |
| MoveBack | 0.5 | 1.0 | 200 | 0.3 | False | AwayFromPlayer | 0 | 600 |
| HoldPosition | 1.0 | 2.0 | 0 | 0.5 | False | None | 0 | 9999 |

**Design Notes:**
- Higher weight on strafing (1.0) = more circling behavior
- Lower weight on retreat (0.3) = occasional backing off
- Distance filters prevent nonsensical behaviors (can't retreat when far away)

---

## Idle State

**Purpose:** Passive state, waiting for player

**Logic:**

```
TickBossAI → Idle pin:
    Check: IsPlayerInRange(AggroRange = 1000)?
    ├─ True → Set CurrentBossState = Approach
    └─ False → Do nothing
```

**Transition:** Idle → Approach (when player detected)

---

## Approach State

**Purpose:** Chase player until in combat range

**Logic:**

```
TickBossAI → Approach pin:
    Check: IsPlayerInRange(AttackRange)?
    ├─ True → Set CurrentBossState = Combat
    └─ False → AI Move To Location (PlayerRef)
               - Acceptance Radius: 250
               - Use Pathfinding: False
               - Can Strafe: False
```

**Movement:**

- Uses AI Controller (Auto Possess AI: Placed in World)
- CharacterMovement handles rotation (Orient Rotation to Movement)
- Speed: ApproachSpeed variable (default 400)

**Transition:** Approach → Combat (when in AttackRange)

---

## Attack State (Updated Session 26-27.11)

**Purpose:** Execute attack from data table with gap-close and hitbox delivery

### AttackStateBranch Macro

```
Attack pin → AttackStateBranch Macro:
    ↓
Branch: bIsExecutingAttack?
│
├─ True → GAP-CLOSE DURING ATTACK
│         Add Movement Input
│             - WorldDirection: Normalize(Player Location - Self Location)
│             - Scale: 0.3 (reduced speed)
│
└─ False → EXECUTE NEW ATTACK
           Clear (Target: AlreadyHitActors)  ← Reset multi-hit prevention
           ↓
           Get Random Attack from AttackDataTable
           ↓
           Set CurrentAttackData = attack row
           ↓
           Play Montage (CurrentAttackData.AttackMontage)
           ↓
           Set bIsExecutingAttack = True
           Set LastAttackTime = Game Time
           Set CurrentAttackCooldown = CurrentAttackData.Cooldown
           ↓
           Print: "[AttackName]"
```

**Gap-Close Logic (Session 26.11):**
- Uses Add Movement Input (not AI Move To)
- Scale 0.3 = 30% speed toward player during attack
- Prevents boss rooting in place during long wind-ups
- Player can still dodge away

### OnAttackAnimationComplete Function

**Called by:** ANS_EnableEnemyHitbox NotifyEnd OR AnimNotify at end of attack

```
OnAttackAnimationComplete:
    ↓
    Set bIsExecutingAttack = False
    ↓
    Set CurrentBossState = Recover
    ↓
    Print: "Recovering, back to approach!"
```

**Transition:** Attack → Recover (when animation completes)

---

## Recover State

**Purpose:** Post-attack recovery (placeholder)

**Logic:**

```
TickBossAI → Recover pin:
    Set CurrentBossState = Combat
```

**Current:** Immediately returns to Combat  
**Future:** Could add recovery delay, vulnerability window, etc.

**Transition:** Recover → Combat (immediate)

---

## Flinch State

**Purpose:** Animation-driven interrupt when resistance breaks

**Trigger:** CurrentFlinchResistance <= 0 (from damage system)

**Logic:**

```
TickBossAI → Flinch pin:
    (Do nothing - animation handles duration)
```

**Animation-Driven Exit (Event Tick):**

```
Branch: CurrentBossState == Flinch?
└─ True → Get AnimBP.bShouldFlinch
          Branch: bShouldFlinch == False?
          ├─ True → Set CurrentBossState = Combat
          └─ False → Still flinching, wait
```

**Interruption:**

- Can interrupt attacks mid-animation
- Clears `bIsExecutingAttack` flag
- Stops AI movement
- Plays flinch animation in AnimBP

**Transition:** Flinch → Combat (when animation completes)

---

## Pre-State Check: Player Death Detection

**Before entering state machine:**

```
TickBossAI:
    ↓
    Branch: Is Valid (PlayerRef)?
    └─ True → Get bIsDead from PlayerRef
              ↓
              Branch: bIsDead?
              ├─ True → Set CurrentBossState = Idle
              │         Return (exit AI processing)
              │
              └─ False → Switch on E_BossState
                         [... state machine continues ...]
```

---

## Attack Hitbox System (Updated Session 27.11)

### Socket-Based Hitbox Architecture

**Philosophy:**

- Hitbox follows weapon/hand during attack animation
- Socket-based attachment (not floating offset)
- Enabled only during active frames via AnimNotifyState
- Multi-hit prevention via AlreadyHitActors array
- **Mesh-based hit detection** - only damage on mesh overlap, not capsule

### Socket Setup

**Socket Name:** `Socket_WeaponHitbox`  
**Parent Bone:** `hand_r` (or weapon bone)  
**Position:** At weapon striking point

### AttackHitboxes Array

**Type:** Array of Collision Components  
**Setup:** Child blueprint adds collision components, adds to array

**Dynamic Event Binding (BeginPlay):**

```
Event BeginPlay:
    ↓
    ForEach Loop (Array: AttackHitboxes)
        └─ Bind Event to OnComponentBeginOverlap
           └─ Delegate: OnHitboxOverlap (custom event)
```

### OnHitboxOverlap Event

**Signature:** Same as OnComponentBeginOverlap

```
OnHitboxOverlap (OverlappedComponent, OtherActor, OtherComp, ...):
    ↓
    DeliverHitboxDamage (OtherActor, OtherComp)
```

### AlreadyHitActors Array (Session 27.11)

**Purpose:** Prevent multiple damage applications per swing

**Variable in BP_EnemyBase:**

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| AlreadyHitActors | Array of Actor References | Empty | Actors hit this swing |

**Cleared:** At start of each attack (in AttackStateBranch, before Play Montage)

**Checked:** In DeliverHitboxDamage before applying damage

---

### DeliverHitboxDamage Function (Session 27.11)

**Location:** BP_EnemyBase → Functions  
**Purpose:** Validate hit and deliver damage with flinch level

**Signature:**
- Input: `OtherActor` (Actor) - What was overlapped
- Input: `HitComponent` (Primitive Component) - Which component was hit

**Implementation:**

```
DeliverHitboxDamage (OtherActor, HitComponent):
    ↓
    Cast to BP_ThirdPersonCharacter (OtherActor)
    ├─ Failed → Return (not player)
    │
    └─ Success → Branch: Is Valid? (cast result)
                 ├─ False → Return
                 │
                 └─ True → Get Mesh (from cast result)
                           ↓
                           Branch: HitComponent == Mesh?
                           │
                           ├─ False → Return (hit capsule, ignore)
                           │          Print: "Hit capsule, filtered out"
                           │
                           └─ True → Branch: AlreadyHitActors Contains (cast result)?
                                     │
                                     ├─ True → Return (already hit this swing)
                                     │
                                     └─ False → Add (cast result) to AlreadyHitActors
                                                ↓
                                                Get CurrentAttackData.Damage → Damage
                                                Get CurrentAttackData.FlinchLevel → FlinchLevel
                                                ↓
                                                ApplyDamageToActor (Target: cast result)
                                                    - Damage: Damage
                                                    - FlinchLevel: FlinchLevel
                                                ↓
                                                Print: "Boss created Hit with [Damage] damage to [HitComponent name]"
```

**Critical Order of Checks:**
1. Cast to player (is this the player?)
2. Mesh check (did we hit the body mesh, not capsule?)
3. AlreadyHitActors check (already hit this swing?)
4. Add to array and apply damage

**Why Mesh Check Before Array Check:**
- Capsule overlap fires first (larger collision)
- If we check array first, capsule hit adds actor to array
- Then mesh hit is rejected as "already hit"
- Correct order: filter capsule hits BEFORE they touch the array

---

### Player Mesh Collision Settings

**Required for mesh-based hit detection:**

**BP_ThirdPersonCharacter → Mesh component → Collision:**
- Collision Enabled: Query Only
- Object Type: Pawn
- Generate Overlap Events: ☑ Checked
- Collision Response to WorldDynamic: Overlap

**Physics Asset (assigned to mesh):**
- All bodies: Physics Type = Kinematic
- All bodies: Collision Response = Enabled

---

### ANS_EnableEnemyHitbox

**Type:** AnimNotifyState  
**Purpose:** Enable/disable attack hitbox during active frames

**Received_NotifyBegin:**
```
Get Owning Actor → Cast to BP_EnemyBase
    ↓
    ForEach (AttackHitboxes array)
        └─ Set Collision Enabled: Query Only
```

**Received_NotifyEnd:**
```
Get Owning Actor → Cast to BP_EnemyBase
    ↓
    ForEach (AttackHitboxes array)
        └─ Set Collision Enabled: No Collision
    ↓
    OnAttackAnimationComplete()
```

---

## Attack Audio (Session 26.11)

### Telegraph Pattern

**Wind-up:** Grunt/vocal effort sound at animation start  
**Swing:** Whoosh sound during active frames

### Implementation

**AN_PlayFastSwing:** AnimNotify placed at swing frame in attack montage

**Grunt Sound:** Triggered at montage start or via separate AnimNotify

**Recommended Asset Packs:**
- Human Vocalizations (Gamemaster Audio) - 1034 vocal sounds
- Video Game Voices (WOW Sound) - 1554 sounds across 10 voice actors
- Effort Vocalizations Male (VoiceBosch) - focused combat efforts

---

## Flinch Resistance System

### Variables

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `MaxFlinchResistance` | Float | 100.0 | Damage needed to trigger flinch |
| `CurrentFlinchResistance` | Float | 100.0 | Current resistance value |
| `FlinchRegenRate` | Float | 20.0 | Resistance recovered per second |
| `FlinchRegenDelay` | Float | 3.0 | Seconds before regen starts |
| `LastFlinchDamageTime` | Float | 0.0 | Timestamp of last hit taken |

### System Flow

**Damage Application:**

```
ApplyDamageToActor (Damage, FlinchLevel):
    Health - Damage → Set Health
    Play hit sound
    ↓
    Branch: Health <= 0?
    ├─ True → Death sequence
    │
    └─ False → FLINCH RESISTANCE
               CurrentFlinchResistance - Damage
               Set LastFlinchDamageTime = Game Time
               ↓
               Branch: CurrentFlinchResistance <= 0?
               ├─ True → FLINCH TRIGGERED
               │         - Set CurrentBossState = Flinch
               │         - Set CurrentFlinchResistance = MaxFlinchResistance
               │         - Set bIsExecutingAttack = False (interrupt)
               │         - Stop AI movement
               │
               └─ False → Hit absorbed, continue
```

**Note:** Boss ignores FlinchLevel parameter - uses own flinch resistance system. FlinchLevel is for player's flinch calculation.

**Resistance Regeneration (Event Tick):**

```
Branch: CurrentBossState != Flinch AND != Dead?
└─ True → Branch: (GameTime - LastFlinchDamageTime) > FlinchRegenDelay?
          └─ True → Regenerate:
                    CurrentFlinchResistance + (FlinchRegenRate * DeltaSeconds)
                    Clamp (0, MaxFlinchResistance)
```

---

## Data-Driven Attacks (Updated Session 27.11)

### S_BossAttackData Structure

| Column | Type | Purpose |
|--------|------|---------|
| `AttackName` | Name | Row identifier |
| `AttackMontage` | Anim Montage | Animation to play |
| `Damage` | Float | Health damage |
| `FlinchLevel` | Integer | Player flinch severity (0-3) |
| `Cooldown` | Float | Seconds before can attack again |
| `MinRange` | Float | Too close? Don't use (future) |
| `MaxRange` | Float | Too far? Can't reach (future) |

### FlinchLevel Values

| Level | Name | Effect on Player (unblocked) |
|-------|------|------------------------------|
| 0 | None | No flinch animation |
| 1 | Light | Light flinch |
| 2 | Medium | Medium flinch |
| 3 | Heavy | Knockback |

**Note:** Player's CalculateFlinchReduction handles mitigation (block reduces by 1)

### Example: DT_GothicKnightAttacks

| AttackName | Montage | Damage | FlinchLevel | Cooldown |
|------------|---------|--------|-------------|----------|
| OpeningSwing | AM_GK_OpeningSwing | 10.0 | 1 | 2.0 |
| HeavyOverhead | AM_GK_HeavyOverhead | 25.0 | 3 | 3.5 |
| QuickSlash | AM_GK_QuickSlash | 8.0 | 1 | 1.0 |

---

## Health & Death System

### Variables

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `Health` | Float | 100.0 | Current health |
| `bIsDead` | Boolean | False | Death state flag |

### Death Sequence

```
Set bIsDead = True
    ↓
Set AnimBP.bIsDead = True (triggers death animation)
    ↓
Disable capsule collision
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
- bShouldFlinch (from BP_EnemyBase.CurrentBossState == Flinch)

**State Machine:**

- Idle (combat idle with DefaultSlot)
- JogStart → JogLoop → JogStop (locomotion with start/stop transitions)
- Flinch (TODO)
- Death (TODO)

**Speed Thresholds:**

- Idle ↔ JogStart: Speed > 300 (running)
- Idle ↔ JogLoop: Speed > 10 AND <= 300 (walking)

### Pattern for New Bosses

Each boss gets its own AnimBP that:
1. Pulls state from BP_EnemyBase via owner cast
2. Has boss-specific state machine transitions
3. Uses DefaultSlot in Idle for montage playback

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
| `bShouldFlinch` | Boolean | False | For AnimBP communication |

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

## Known Issues

### Fixed (Session 26-27.11)

- ✅ TotalWeight calculated from all rows (now filtered first)
- ✅ Camera lock targets root (now uses LockOnTarget socket)
- ✅ Multi-hit per swing (AlreadyHitActors array)
- ✅ Capsule-based hit detection (now mesh-based)

### Remaining Issues

- **Distance filtering not implemented:** MinDistanceToUse/MaxDistanceToUse not checked in filter loop
- **No fallback for empty ValidRowNames:** If all behaviors filtered out, no selection
- **Attack variety:** Gothic Knight only has 1 attack configured

### Combat Tuning Needed

- Attack telegraphs (visual wind-up cues)
- More attack variety (different timings, ranges)
- Threat accumulation rate tuning
- Flinch threshold tuning

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

### Future Systems

8. **Stagger/Knockdown states** - Extended interrupts
9. **Part-Break System** - MH-style targeting
10. **Boss #2** - Validate reusability of BP_EnemyBase

---

## Architecture Validation

### What's Proven

**Hitbox System (Session 27.11):**
- ✅ Mesh-based hit detection working
- ✅ AlreadyHitActors prevents multi-hit
- ✅ FlinchLevel passed through interface
- ✅ Check order correct (mesh → array → damage)

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

---

*Last Updated: 27.11.2025*  
*Status: Hitbox system complete, threat system working, flinch level delivery working*  
*Next: Attack variety → Distance filtering → Attack telegraphs*
