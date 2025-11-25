# Enemy System Documentation

**Status:** ✅ Boss AI Complete + Combat State System  
**Sessions:** 16-20.11.2025, 24-25.11.2025  
**Latest:** Combat State with weighted behavior selection

---

## Overview

The enemy system uses parent-child Blueprint architecture where **BP_EnemyBase** contains all reusable boss logic, and specific boss instances configure the details. This allows rapid creation of new bosses by inheriting from the base class.

**Major Updates:**
- **Session 20.11:** Complete Boss AI state machine with flinch resistance system
- **Session 24-25.11:** Combat State system - threatening AI behavior within danger zone

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

**AttackHitbox** (Abstract Scene Component)

- Configurable per boss via child collisions
- Socket-based hitbox attached to hand bone
- Collision Preset: NoCollision (default)
- Enabled/disabled via ANS_EnableEnemyHitbox during attacks
- Socket: Attached to `Socket_WeaponHitbox` on hand_r bone

### Per-Boss Configuration (Session 24-25.11)

BP_EnemyBase is now fully configurable. Child blueprints only set:

| Variable | Type | Purpose |
|----------|------|---------|
| Mesh | Skeletal Mesh | Boss model |
| Hitbox | Scene Component | Hitbox collision shape |
| AnimBP | Anim Blueprint Class | Per-boss animation logic |
| AttackDataTable | DataTable | DT_[BossName]Attacks |
| CombatBehaviorTable | DataTable | DT_[BossName]CombatBehaviors |

---

## State Machine System

### E_BossState Enum

**Values:**

- `Idle` - Waiting for player
- `Approach` - Chasing player (outside danger zone)
- `Combat` - Threatening behavior in danger zone **(NEW Session 24-25.11)**
- `Attack` - Executing attack
- `Recover` - Post-attack recovery (placeholder)
- `Flinch` - Hit reaction interrupt
- `Dead` - Terminal state

### State Flow

```
Idle → Approach (player detected at AggroRange)
Approach → Combat (player enters AttackRange) [TODO: refactor to DangerZoneRange]
Combat → Attack (TODO: not implemented yet)
Attack → Recover → Combat (attack complete)
Any → Flinch (resistance breaks)
Any → Dead (health <= 0)
```

---

## Combat State (Session 24-25.11)

**Purpose:** Threatening AI behavior within combat range - circling, feinting, positioning

**Entry:** Approach state when `IsPlayerInRange(AttackRange)` (TODO: should be DangerZoneRange)

### Combat State Variables

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| CombatBehaviorTable | DataTable | - | Per-boss behavior config |
| CurrentCombatBehavior | S_CombatBehavior | - | Active behavior data |
| CurrentMovementType | E_CombatMovementType | - | Current movement pattern |
| CombatBehaviorEndTime | Float | 0 | When to select new behavior |
| LastCombatBehavior | Name | - | For repeat prevention |
| TotalWeight | Float | 0 | Sum of all SelectionWeights (weighted random) |
| AccumulatedWeight | Float | 0 | Running sum during selection loop |
| RandomValue | Float | 0 | Target value for weighted selection |
| ValidRowNames | Name Array | - | Local: filtered behavior candidates |

**Note:** DangerZoneRange and CombatMovementSpeed variables exist but are not currently used. AttackRange currently triggers Combat state entry.

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
═══ LOOP 1: Calculate Total Weight (all rows) ═══
ForEach Loop (Array: OutRowNames)
    └─ Loop Body:
       Get Data Table Row (Table: CombatBehaviorTable, RowName: ArrayElement)
           └─ OutRow → Break S_CombatBehavior → SelectionWeight
                   ↓
               Set TotalWeight = TotalWeight + SelectionWeight
    └─ On Completed → continue
    ↓
═══ LOOP 2: Filter Valid Behaviors ═══
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

**Known Issues:**

1. **TotalWeight Bug:** TotalWeight is calculated from ALL rows, but selection iterates only ValidRowNames. If filtered rows had high weights, AccumulatedWeight from ValidRowNames may never reach RandomValue → loop completes without selection.

2. **Distance Filtering Not Implemented:** MinDistanceToUse/MaxDistanceToUse fields exist in struct but aren't checked in filter loop yet.

3. **No Fallback:** If ValidRowNames is empty (all filtered out), no behavior is selected.

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
    ├─ TowardPlayer → AI Move To Location
    │                  - Target: AI Controller
    │                  - Dest: Player Location
    │                  - Acceptance Radius: AttackRange - 100
    │
    ├─ AwayFromPlayer → AI Move To Location
    │                    - Target: AI Controller
    │                    - Dest: Self + Normalize(Self - Player) * 300
    │
    ├─ StrafeLeft → Add Movement Input
    │                - Target: Self
    │                - WorldDirection: Normalize(RotateVectorAroundAxis(
    │                    InVect: Self - Player,
    │                    Axis: [0, 0, 1]))
    │                - Scale: 1.0
    │
    └─ StrafeRight → Add Movement Input
                     - Target: Self
                     - WorldDirection: Normalize(RotateVectorAroundAxis(
                         InVect: Self - Player,
                         Axis: [0, 0, -1]))
                     - Scale: 1.0
```

**Key Implementation Notes:**

- **True branch** calls SelectNewCombatBehavior AND sets walk speed
- **False branch** skips behavior selection, goes straight to rotation
- **Both branches** converge at Set Actor Rotation
- **None** uses Stop Movement Immediately (not just "do nothing")
- **TowardPlayer/AwayFromPlayer** use AI Move To Location
- **Strafe** uses Add Movement Input with RotateVectorAroundAxis for orbit direction

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

**TODO:** Refactor to use separate DangerZoneRange variable so AttackRange can be used for Attack state transition

---

## Attack State

**Purpose:** Execute attack from data table with cooldown validation

**Logic:**

```
TickBossAI → Attack pin:
    Branch: bIsExecutingAttack?
    │
    ├─ True → Wait for animation (do nothing)
    │
    └─ False → Branch: (GameTime - LastAttackTime) > CurrentAttackCooldown?
               │
               ├─ True → EXECUTE ATTACK
               │         - Get attack from AttackDataTable
               │         - Play montage via Anim Instance
               │         - Stop AI movement
               │         - Set bIsExecutingAttack = True
               │         - Set LastAttackTime = Game Time
               │         - Set CurrentAttackCooldown (from table)
               │
               └─ False → COOLDOWN NOT READY
                          Branch: IsPlayerInRange(AttackRange)?
                          ├─ True → Wait in Attack state
                          └─ False → Set CurrentBossState = Combat
```

**Attack Execution Detection (Event Tick):**

```
Branch: bIsExecutingAttack?
└─ True → Check: Is any montage playing?
          Branch: NOT playing?
          ├─ True → Set bIsExecutingAttack = False
          │         Set CurrentBossState = Recover
          └─ False → Still playing, wait
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

## Attack Hitbox System

### Socket-Based Hitbox Architecture

**Philosophy:**

- Hitbox follows weapon/hand during attack animation
- Socket-based attachment (not floating offset)
- Enabled only during active frames via AnimNotifyState
- Prevents multi-hit during single attack (future: AlreadyHitActors array)

### Socket Setup

**Socket Name:** `Socket_WeaponHitbox`  
**Parent Bone:** `hand_r` (or `RightHand` for Mixamo)  
**Position:** At fist/weapon striking point

### AttackHitbox Component

**Type:** Sphere Collision Component  
**Parent:** Mesh (attached to Socket_WeaponHitbox)  
**Default State:** Collision Enabled = NoCollision  

**Settings:**

- Sphere Radius: 60 (tune per boss)
- Collision Preset: Custom
- Object Type: WorldDynamic
- Collision Response to Pawn: **Overlap**
- Generate Overlap Events: ☑ Checked

### ANS_EnableEnemyHitbox

**Type:** AnimNotifyState  
**Purpose:** Enable/disable attack hitbox during active frames

**Received_NotifyBegin:** Set Collision Enabled: Query Only  
**Received_NotifyEnd:** Set Collision Enabled: No Collision

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
ApplyDamageToActor (Damage parameter):
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

**Resistance Regeneration (Event Tick):**

```
Branch: CurrentBossState != Flinch AND != Dead?
└─ True → Branch: (GameTime - LastFlinchDamageTime) > FlinchRegenDelay?
          └─ True → Regenerate:
                    CurrentFlinchResistance + (FlinchRegenRate * DeltaSeconds)
                    Clamp (0, MaxFlinchResistance)
```

---

## Animation Blueprints

### ABP_Boss_GothicKnight (Session 24-25.11)

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
| `bShouldFlinch` | Boolean | False | For AnimBP communication |

### References

| Variable | Type | Purpose |
|----------|------|---------|
| `PlayerRef` | Actor Reference | Cached player for distance checks |
| `AttackDataTable` | DataTable Reference | Per-boss attack config |
| `CombatBehaviorTable` | DataTable Reference | Per-boss combat behaviors |

---

## Data-Driven Attacks

### S_BossAttackData Structure

| Column | Type | Purpose |
|--------|------|---------|
| `AttackName` | Name | Row identifier |
| `AttackMontage` | Anim Montage | Animation to play |
| `Damage` | Float | Health damage |
| `Cooldown` | Float | Seconds before can attack again |
| `MinRange` | Float | Too close? Don't use (future) |
| `MaxRange` | Float | Too far? Can't reach (future) |

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

## Current Boss: Gothic Knight (Session 24-25.11)

**Source:** Professional animation pack  
**Type:** Humanoid knight with 2H sword

### Configuration in BP_Boss_GothicKnight

- Mesh: SKM_GK_Full_With_Sword
- AnimBP: ABP_Boss_GothicKnight
- AttackDataTable: DT_GothicKnightAttacks
- CombatBehaviorTable: DT_GothicKnightCombatBehaviors

Everything else inherited from BP_EnemyBase.

---

## Known Issues

### Combat State Bugs (High Priority)

1. **TotalWeight calculated from ALL rows:** Weighted selection uses TotalWeight from all behaviors, but iterates only ValidRowNames. If filtered-out rows had significant weights, RandomValue may land in "dead zone" and no behavior gets selected.

2. **Distance filtering not implemented:** S_CombatBehavior has MinDistanceToUse/MaxDistanceToUse fields, but filter loop doesn't check them yet.

3. **No fallback for empty ValidRowNames:** If all behaviors filtered out, function completes without selecting anything.

4. **Combat → Attack transition not implemented:** bCanAttackFrom field exists in struct but transition logic not built yet.

5. **AttackRange dual-use:** Currently triggers both Approach→Combat transition AND will be used for attack range. Need separate DangerZoneRange variable.

### Other Issues

- **Camera lock:** Targets root (strange angles)
- **Stick drift:** No deadzone on movement input

### Combat Tuning Needed

- Attack telegraphs (can't predict attacks)
- Boss only has 1 attack (repetitive)
- Attack cooldown tuning
- Flinch threshold tuning

---

## Future Enhancements

### High Priority (Combat State Fixes)

1. **Fix TotalWeight bug** - Calculate TotalWeight from ValidRowNames only, OR recalculate after filtering
2. **Add fallback behavior** - Default when ValidRowNames empty (e.g., HoldPosition)
3. **Implement distance filtering** - Check MinDistanceToUse/MaxDistanceToUse in filter loop
4. **Implement Combat → Attack transition** - Use bCanAttackFrom + AttackRange check
5. **Separate DangerZoneRange** - Split from AttackRange for proper state boundaries

### Medium Priority

6. **Multiple boss attacks** - Add variety to combat
7. **Attack telegraphs** - Visual/audio wind-ups
8. **Fix camera lock** - Add LockOnTarget at chest height
9. **Fix stick drift** - Add deadzone

### Future Systems

10. **Stagger/Knockdown states** - Extended interrupts
11. **Part-Break System** - MH-style targeting
12. **Boss #2** - Validate reusability

---

## Architecture Validation

### What's Proven

**Combat State foundation works:**
- ✅ Weighted random selection structure works
- ✅ Data-driven behavior tables work
- ✅ Movement types (strafe, retreat, approach) work
- ✅ Boss exhibits threatening presence
- ⚠️ Weighted selection has bug (TotalWeight from all rows)
- ⚠️ Distance-based filtering not implemented yet

**Parent-child architecture validated:**
- ✅ BP_EnemyBase contains all systems
- ✅ Child blueprints are pure configuration
- ✅ New boss = set 4 variables (mesh, AnimBP, 2 tables)

**Per-boss AnimBP pattern established:**
- ✅ Each boss can have unique animation logic
- ✅ Pulls state from parent cleanly
- ✅ Supports different locomotion styles

---

## Key Learnings (Session 24-25.11)

### Technical

1. **Add Movement Input for direct control**
   - AI Move To uses pathfinding (not wanted for strafing)
   - Add Movement Input = direct, responsive

2. **RotateVectorAroundAxis for orbit movement**
   - Axis [0,0,1] = rotate around Z (left strafe)
   - Axis [0,0,-1] = opposite direction (right strafe)

3. **Weighted random selection pitfall**
   - TotalWeight must be calculated from the SAME set used for selection
   - If filtering happens AFTER weight calculation, selection breaks

4. **Per-boss AnimBP is cleaner**
   - Different skeletons need different setups
   - Owner cast pattern works well

### Design

1. **Combat State transforms boss feel**
   - Not just "walk at player and attack"
   - Circling, backing off, holding position = threatening

2. **Data tables make tuning easy**
   - Adjust weights without code changes
   - Distance filters prevent silly behaviors (once implemented)

---

## Session Stats

**Session 24-25.11.2025:**
- Combat State system with weighted behavior selection
- E_CombatMovementType enum
- S_CombatBehavior struct
- SelectNewCombatBehavior function
- BP_EnemyBase refactored for reusability
- ABP_Boss_GothicKnight created
- Gothic Knight boss configuration

---

*Last Updated: 25.11.2025*  
*Status: Combat State complete, needs fallback + attack transition logic*
