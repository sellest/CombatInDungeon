# Enemy System Documentation

**Status:** ✅ Boss AI Complete + Playable  
**Sessions:** 16-20.11.2025  
**Playtest Result:** "Actually feels like a game!" ✅

---

## Overview

The enemy system uses parent-child Blueprint architecture where **BP_EnemyBase** contains all reusable boss logic, and specific boss instances (like Mav) configure the details. This allows rapid creation of new bosses by inheriting from the base class.

**Major Update (Session 20.11):** Complete Boss AI state machine with flinch resistance system. First playable boss combat encounter achieved.

---

## BP_EnemyBase (Parent Class)

**Type:** Character Blueprint (changed from Actor in Session 20.11)  
**Purpose:** Reusable boss foundation containing all core enemy systems  

### Components

**Mesh** (Skeletal Mesh Component - inherited from Character)

- Displays enemy model
- Assigned to specific character mesh per boss
- Drives AnimBP (ABP_Enemy)
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

**AttackHitbox** (Sphere Collision Component - Session 21.11)

- Socket-based hitbox attached to hand bone
- Collision Preset: NoCollision (default)
- Enabled/disabled via ANS_EnableEnemyHitbox during attacks
- Radius: 60 (tune per boss/attack)
- Socket: Attached to `Socket_WeaponHitbox` on hand_r bone

---

## State Machine System (Session 20.11)

### E_BossState Enum

**Values:**

- `Idle` - Waiting for player
- `Approach` - Chasing player
- `Attack` - Executing attack
- `Recover` - Post-attack recovery (placeholder)
- `Flinch` - Hit reaction interrupt
- `Dead` - Terminal state

### State Implementation

#### Pre-State Check: Player Death Detection (Session 21.11)

**Before entering state machine:**

```
TickBossAI:
    ↓
    Branch: Is Valid (PlayerRef)?
    └─ True → Get bIsDead from PlayerRef
              ↓
              Branch: bIsDead?
              ├─ True → Set CurrentBossState = Idle
              │         Print: "He died. Mav's done."
              │         Return (exit AI processing)
              │
              └─ False → Switch on E_BossState
                         [... state machine continues ...]
```

**Why before state machine:**

- Dead player = no AI processing needed
- Prevents edge cases (mid-flinch, mid-attack, etc.)
- Single point of truth for player death
- Boss "freezes" in victory state

#### Idle State

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

#### Approach State

**Purpose:** Chase player until in attack range

**Logic:**

```
TickBossAI → Approach pin:
    Check: IsPlayerInRange(AttackRange = 300)?
    ├─ True → Set CurrentBossState = Attack
    └─ False → AI Move To Location (PlayerRef)
               - Acceptance Radius: 250
               - Use Pathfinding: False
               - Can Strafe: False
```

**Movement:**

- Uses AI Controller (Auto Possess AI: Placed in World)
- CharacterMovement handles rotation (Orient Rotation to Movement)
- Speed: ApproachSpeed variable (default 400)

**Transition:** Approach → Attack (when in range)

---

#### Attack State

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
               │         - Get attack from DT_MavAttacks (index 0 for now)
               │         - Play montage via Anim Instance
               │         - Stop AI movement
               │         - Set bIsExecutingAttack = True
               │         - Set LastAttackTime = Game Time
               │         - Set CurrentAttackCooldown (from table)
               │
               └─ False → COOLDOWN NOT READY
                          Branch: IsPlayerInRange(AttackRange)?
                          ├─ True → Wait in Attack state
                          └─ False → Set CurrentBossState = Approach
```

**Cooldown Pattern:**

- Any state can REQUEST attack (set state to Attack)
- Attack state VALIDATES cooldown
- If not ready: Either waits or returns to Approach

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

#### Recover State

**Purpose:** Post-attack recovery (placeholder)

**Logic:**

```
TickBossAI → Recover pin:
    Set CurrentBossState = Approach
```

**Current:** Immediately returns to Approach  
**Future:** Could add recovery delay, vulnerability window, etc.

**Transition:** Recover → Approach (immediate)

---

#### Flinch State (Session 20.11)

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
└─ True → Get ABP_Enemy.bShouldFlinch
          Branch: bShouldFlinch == False?
          ├─ True → Set CurrentBossState = Approach
          └─ False → Still flinching, wait
```

**Interruption:**

- Can interrupt attacks mid-animation
- Clears `bIsExecutingAttack` flag
- Stops AI movement
- Plays flinch animation in ABP

**Transition:** Flinch → Approach (when animation completes)

---

## Attack Hitbox System (Session 21.11)

### Socket-Based Hitbox Architecture

**Philosophy:**

- Hitbox follows weapon/hand during attack animation
- Socket-based attachment (not floating offset)
- Enabled only during active frames via AnimNotifyState
- Prevents multi-hit during single attack (future: AlreadyHitActors array)

### Socket Setup

**Location:** Mav skeletal mesh → Skeleton

**Socket Name:** `Socket_WeaponHitbox`  
**Parent Bone:** `hand_r` (or `RightHand` for Mixamo)  
**Position:** At fist/weapon striking point (typically 10-20 units forward of hand)

**Why socket-based:**

- Follows animation bone transforms
- Accurate to visual weapon position
- Boss-specific positioning (each boss can adjust socket)
- Scalable to multiple attack types (future: kick sockets, tail sockets, etc.)

---

### AttackHitbox Component

**Type:** Sphere Collision Component  
**Parent:** Mesh (attached to Socket_WeaponHitbox)  
**Default State:** Collision Enabled = NoCollision  

**Settings:**

- Sphere Radius: 60 (tune per boss)
- Collision Preset: Custom
- Object Type: WorldDynamic (or Pawn)
- Collision Response to Pawn: **Overlap** (critical!)
- Generate Overlap Events: ☑ Checked

**Transform (relative to socket):**

- Location: (0, 0, 0) - socket defines position
- Rotation: (0, 0, 0)

---

### Overlap Event Logic

**Event:** On Component Begin Overlap (AttackHitbox)

```
On Component Begin Overlap (AttackHitbox):
    ↓
    Cast to BP_ThirdPersonCharacter (Other Actor)
    ↓
    Branch: Cast Successful?
    └─ True → Get CurrentAttackData variable
              ↓
              Break S_BossAttackData struct
              ↓
              ApplyDamageToActor (BPI_Damageable interface)
              - Target: Player (from cast)
              - Damage: Damage field (from struct)
              ↓
              Print: "Boss delivered {Damage} damage!"
```

**Why check CurrentAttackData:**

- Attack state stores active attack properties before playing montage
- Overlap can read damage from current attack
- No hardcoded damage values
- Scalable to multiple attacks per boss

**Future Enhancement:**

```
Add AlreadyHitActors array (Actor References):
- Check: Array Contains Other Actor?
  ├─ True → EXIT (already hit this attack)
  └─ False → Add to array, apply damage
- Clear array: When new attack starts or ANS begins
```

---

### ANS_EnableEnemyHitbox (Session 21.11)

**Type:** AnimNotifyState  
**Location:** `/Content/Animations/Notifies/`  
**Purpose:** Enable/disable attack hitbox during active frames of attack animation

**Received_NotifyBegin:**

```
Get Mesh Owner
    ↓
Get Owner (Actor)
    ↓
Cast to BP_EnemyBase
    ↓
Get AttackHitbox component
    ↓
Set Collision Enabled: Query Only
    ↓
Print: "Mav hitbox ON" (or boss name)
```

**Received_NotifyEnd:**

```
Get Mesh Owner
    ↓
Get Owner (Actor)
    ↓
Cast to BP_EnemyBase
    ↓
Get AttackHitbox component
    ↓
Set Collision Enabled: No Collision
    ↓
Print: "Mav hitbox OFF"
```

**Placement in Attack Montages:**

- **Start:** When hand begins forward motion (weapon starts swing)
- **End:** When swing completes (weapon stops moving)
- **Typical Duration:** 0.3-0.5 seconds
- **Tune:** Watch animation, place where weapon would visually hit player

**Why Query Only (not Query and Physics):**

- Hitbox is detection only, not physical collision
- Same pattern as player sword hitbox
- Prevents physics interactions

---

## Flinch Resistance System (Session 20.11)

### Variables

| Variable | Type | Default | Category | Purpose |
|----------|------|---------|----------|---------|
| `MaxFlinchResistance` | Float | 100.0 | Combat Stats | Damage needed to trigger flinch |
| `CurrentFlinchResistance` | Float | 100.0 | Combat State | Current resistance value |
| `FlinchRegenRate` | Float | 20.0 | Combat Stats | Resistance recovered per second |
| `FlinchRegenDelay` | Float | 3.0 | Combat Stats | Seconds before regen starts |
| `LastFlinchDamageTime` | Float | 0.0 | Combat State | Timestamp of last hit taken |

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
               │         - Set CurrentFlinchResistance = MaxFlinchResistance (reset)
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

**Example (10 damage per hit, 100 resistance):**

- Hits 1-9: Resistance decreases (90, 80, 70...)
- Hit 10: Resistance breaks (0) → Flinch triggered
- After flinch: Resistance resets to 100
- Wait 3 seconds: Regeneration starts at 20/sec

### Design Rationale

**Why Animation-Driven:**

- Flinch is brief (~0.8s)
- Duration defined by animation (designer control)
- No time tracking needed (simpler)

**Why Reset to Max:**

- Monster Hunter pattern
- Prevents flinch-locking (spam)
- Creates rhythm (10 hits → flinch → repeat)

**Future Expansion:**

- Stagger resistance (longer interrupt, 200 damage)
- Knockdown resistance (ground state, 500 damage)
- Part-break resistance (permanent weakpoint)
- Same pattern, different thresholds

---

## AI Configuration Variables

### Movement & Detection

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `AggroRange` | Float | 1000.0 | How far boss detects player |
| `AttackRange` | Float | 300.0 | Distance to trigger attack |
| `ApproachSpeed` | Float | 400.0 | Movement speed when chasing |

### Combat State

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `CurrentBossState` | E_BossState | Idle | Active state |
| `bIsExecutingAttack` | Boolean | False | Currently attacking? |
| `LastAttackTime` | Float | 0.0 | When last attack occurred |
| `CurrentAttackCooldown` | Float | 0.0 | Cooldown from attack table |

### References

| Variable | Type | Purpose |
|----------|------|---------|
| `PlayerRef` | Actor Reference | Cached player for distance checks |
| `AttackDataTable` | DataTable Reference | DT_MavAttacks (per-boss config) |

---

## Data-Driven Attacks (Session 20.11)

### DT_MavAttacks Structure

**Row Structure:** S_BossAttackData

| Column | Type | Purpose |
|--------|------|---------|
| `AttackName` | Name | Row identifier |
| `AttackMontage` | Anim Montage | Animation to play |
| `Damage` | Float | Health damage (also used for flinch) |
| `Cooldown` | Float | Seconds before can attack again |
| `MinRange` | Float | Too close? Don't use (future) |
| `MaxRange` | Float | Too far? Can't reach (future) |

**Current Data:**

| Row Name | AttackMontage | Damage | Cooldown | MinRange | MaxRange |
|----------|---------------|--------|----------|----------|----------|
| MavSwing | AM_MavSwing | 10.0 | 3.0 | 0.0 | 300.0 |

**Future Expansion:**

- Add 2-3 more attacks (variety)
- Random attack selection (weighted by table)
- Possibly split Damage into HealthDamage/FlinchDamage (weapon type variety)

---

## Health & Damage System

### Variables

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `Health` | Float | 100.0 | Current health |
| `bIsDead` | Boolean | False | Death state flag |

### Damage System

**Interface:** BPI_Damageable  
**Function:** ApplyDamageToActor(Damage)

**Flow (Updated Session 20.11):**

```
ApplyDamageToActor (Damage input):
    Branch: bIsDead? → EXIT if already dead
    ↓
    Health - Damage → Set Health
    Play hit sound (DT_SwordHitSounds_Flesh)
    ↓
    Branch: Health <= 0?
    ├─ True → DEATH SEQUENCE
    │         - Set bIsDead = True
    │         - Set ABP_Enemy.bIsDead = True
    │         - Disable capsule collision
    │         - Print: "Boss died"
    │
    └─ False → FLINCH RESISTANCE LOGIC
               (See Flinch Resistance System above)
```

---

## Death System

**Triggered when:** Health <= 0

**Logic:**

```
Set bIsDead = True
    ↓
Get Mesh → Get Anim Instance → Cast to ABP_Enemy
└─ Set bIsDead = True (triggers death animation)
    ↓
Get Capsule Component
└─ Set Collision Enabled (No Collision)
```

**Result:**

- Death animation plays via state machine
- Collision disabled (can walk through corpse)
- Corpse remains in world (does not despawn)
- AI state machine stops (bIsDead check in Event Tick)

---

## ABP_Enemy (Animation Blueprint)

**Type:** Animation Blueprint  
**Target Skeleton:** Mav J Largo skeleton  
**Purpose:** Controls enemy animations via state machine

### Variables

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bIsDead` | Boolean | False | Triggers death state transition |
| `bShouldFlinch` | Boolean | False | Triggers flinch state transition |

### State Machine

```
Entry → Idle
Idle → Flinch (when bShouldFlinch = True)
Flinch → Idle (when animation complete)
Idle → Death (when bIsDead = True)
Death (terminal state)
```

**Critical:** Requires **DefaultSlot** node in Idle state for montages to play through AnimBP.

### States

#### Idle State

- **Animation:** Idle (looping)
- **Behavior:** Default standing animation
- **Slot:** DefaultSlot node (for attack montages)

#### Flinch State

- **Animation:** Hit Reaction (no loop)
- **Entry:** bShouldFlinch = True
- **Exit:** Animation time remaining <= 0.1s (automatic)
- **Reset:** bShouldFlinch set to False after 0.1s delay

#### Death State

- **Animation:** Death (no loop)
- **Entry:** bIsDead = True
- **Exit:** Never (terminal state)

### Event Graph Logic

**Event Blueprint Update Animation:**

```
1. Update bIsDead from owner:
   Get Owner → Cast to BP_EnemyBase
   Get bIsDead → Set bIsDead (local)

2. Update flinch trigger (Session 20.11):
   Get Owner → Cast to BP_EnemyBase
   Get CurrentBossState
   Branch: == Flinch?
   ├─ True → Set bShouldFlinch = True
   └─ False → Set bShouldFlinch = False

3. Reset flinch trigger (with delay):
   Branch: bShouldFlinch?
   → Delay (0.1s)
   → Set bShouldFlinch = False
```

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
- Attack animations (swing, etc.)

### Configuration

**In BP_EnemyBase (currently configured for Mav):**

- Mesh: Mav J Largo skeletal mesh (Z rotation: -90°)
- AnimBP: ABP_Enemy
- Health: 100
- MaxFlinchResistance: 100 (10 hits to flinch)
- Attack table: DT_MavAttacks
- Hit sounds: DT_SwordHitSounds_Flesh

**AI Settings:**

- AggroRange: 1000 (detection distance)
- AttackRange: 300 (attack trigger distance)
- ApproachSpeed: 400 (chase speed)
- Attack Cooldown: 3.0 seconds (from table)

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

## Technical Patterns

### Animation-Driven vs Time-Driven States

**Animation-Driven (Flinch):**

- State duration = animation length
- Exit detected by checking ABP flag
- Clean, designer-friendly
- No time tracking needed

**Time-Driven (Future: Stun, Knockdown):**

- State duration = hardcoded value
- Exit detected by time comparison
- More control, independent of animation
- Requires EndTime tracking

### State Machine Event Flow

**TickBossAI (every frame):**

- Switch on CurrentBossState
- Each state handles logic
- State transitions via CurrentBossState

**Event Tick (parallel checks):**

- Attack execution detection
- Flinch animation completion
- Resistance regeneration
- All independent using Sequence node

### Cooldown Pattern

**Centralized in Attack state:**

- Any state can REQUEST attack
- Attack state VALIDATES cooldown
- If fails: Return to Approach or wait
- Single point of truth

---

## Known Issues

### Fixed (Session 20.11)

- ✅ ~~Rotation on movement start~~ - Mesh Z rotation = -90°
- ✅ ~~AI pathfinding failure~~ - Changed parent to Character class
- ✅ ~~Montage not playing~~ - Added DefaultSlot to ABP_Enemy
- ✅ ~~Flinch spam~~ - Implemented resistance system

### Current (Deferred to Next Session)

**Camera/Input:**

- Camera lock targets root (strange angles)
- Stick drift (no deadzone on movement input)

**Combat:**

- No attack telegraphs (can't predict attacks)
- Boss only has 1 attack (repetitive)
- Attack cooldown tuning needed
- Flinch threshold tuning needed (10 hits right?)

---

## Testing Checklist

**Basic Functionality:**

- ✅ Mav appears in level with idle animation
- ✅ Detects player at 1000 units (Idle → Approach)
- ✅ Chases player smoothly (no rotation issues)
- ✅ Attacks when within 300 units (plays montage)
- ✅ Respects attack cooldown (3 seconds)
- ✅ Takes damage when hit
- ✅ Plays hit sounds on damage
- ✅ Flinch resistance decreases per hit
- ✅ Flinches after 10 hits (plays animation, interrupts attacks)
- ✅ Resistance resets after flinch
- ✅ Resistance regenerates after 3 second delay
- ✅ Dies after sufficient damage (100 HP)
- ✅ Plays death animation when killed
- ✅ Collision disabled on death
- ✅ Corpse remains visible

**Combat Loop:**

- ✅ Chase → Attack → Cooldown → Chase rhythm works
- ✅ Player can build flinch through hits
- ✅ Flinch interrupts boss attacks (mid-swing)
- ✅ Combat "feels like a game" (playtest validated)

---

## Future Enhancements

### High Priority (Next Session - 30 min)

1. **Fix Camera Lock** (20 min)
   - Add LockOnTarget Scene Component
   - Position at chest height
   - Add pitch clamp

2. **Fix Stick Drift** (10 min)
   - Add deadzone to movement input

### Medium Priority (Combat Variety - 1 hour)

3. **Add 2nd Attack** (30 min)
   - Import/create animation
   - Add to DT_MavAttacks
   - Implement random selection

4. **Add 3rd Attack** (30 min)
   - Same process
   - Test combat variety

### Tuning (Varies)

5. **Tune Combat Feel**
   - Flinch threshold
   - Attack cooldown
   - Chase speed
   - Based on playtest

### Future Systems

6. **Player Takes Damage** (basic)
7. **Attack Telegraphs** (wind-ups)
8. **Stagger/Knockdown States** (extended interrupts)
9. **Part-Break System** (MH-style targeting)
10. **Boss #2** (validate reusability)

---

## Architecture Validation

### What's Proven (Session 20.11)

**State machine scales:**

- ✅ Easy to add states (Stun, Knockdown, etc.)
- ✅ Clean separation of concerns
- ✅ Animation-driven vs time-driven patterns work

**Data-driven attacks work:**

- ✅ Add rows = new attacks
- ✅ Centralized properties (damage, cooldown)
- ✅ Ready for random selection

**Parent-child architecture ready:**

- ✅ BP_EnemyBase = complete system
- ✅ Configuration variables identified
- ✅ BP_Boss_Mav would be pure config

**Resistance pattern reusable:**

- ✅ Flinch system established
- ✅ Same pattern for stagger/knockdown
- ✅ Scalable to part-breaks

**Combat loop is FUN:**

- ✅ Validated by playtest
- ✅ Rhythm-based gameplay works
- ✅ Boss feels like opponent

### Boss #2 Estimate

**Time:** 30-60 minutes

**Process:**

1. Create BP_Boss_[Name] child (5 min)
2. Import mesh/animations (10 min)
3. Set configuration variables (5 min)
4. Create DT_[Name]Attacks (10 min)
5. Test and tune (10-30 min)

**Inherits from BP_EnemyBase:**

- All state machine logic
- Flinch resistance system
- Attack execution framework
- AI movement and targeting

---

## Refactoring Plan (When Adding Boss #2)

### Step 1: Clean BP_EnemyBase

- Remove Mav-specific mesh reference → Make configurable
- Move DT_MavAttacks reference → Per-boss variable
- Keep all systems (state machine, flinch, attacks)

### Step 2: Create BP_Boss_Mav

- Parent: BP_EnemyBase
- Set: Mav mesh (Z rotation -90°)
- Set: ABP_Enemy
- Set: DT_MavAttacks
- Configure: Health, flinch resistance, speeds

### Step 3: Create BP_Boss_[NewEnemy]

- Parent: BP_EnemyBase  
- Set: New mesh, animations, sounds
- Set: DT_[Name]Attacks
- Configure: Tuned values per boss

**Result:** Each new boss is 95% inherited, 5% configuration.

---

## Design Philosophy

**Parent-Child Architecture:**

- BP_EnemyBase = Systems (reusable)
- BP_Boss_[Name] = Configuration (per-boss)
- Minimize duplication, maximize reusability

**Monster Hunter Inspiration:**

- Flinch/stagger/knockdown resistance tiers
- Rhythm-based combat (build → break → punish)
- Boss-scale enemies (not crowds)
- Part-break potential (future)

**Prototype → Polish Workflow:**

- Build systems properly first time
- Validate with playtest ("feels like a game")
- Tune based on feel (not theory)
- Polish annoyances after core works

**Data-Driven Design:**

- Attack properties in tables
- State behavior in blueprints
- Easy to tune without code changes
- Designer-friendly iteration

---

## Key Learnings (Session 20.11)

### Technical

1. **Character class required for AI**
   - Actor doesn't have AI Controller or CharacterMovement
   - Always use Character for AI enemies

2. **Mixamo mesh rotation**
   - Import at 0°, must set to -90° in component
   - Standard for all Mixamo characters

3. **AnimBP needs Slot node**
   - Without DefaultSlot, montages won't play
   - Easy to forget, hard to debug

4. **Print node wire carefully**
   - Print math output ≠ Print variable after Set
   - Always print actual variable value

### Design

1. **Animation-driven cleaner for brief states**
   - Flinch duration = animation length
   - No time tracking needed
   - Designer controls feel

2. **Centralize validation logic**
   - Attack cooldown in Attack state
   - All states can request, one validates

3. **Playtest reveals truth**
   - Numbers in tables are guesses
   - Must PLAY to tune correctly
   - "Feels like a game" is the milestone

### Process

1. **Build systems properly first**
   - Flinch resistance reusable (proven)
   - Parent-child validated
   - No throwaway code

2. **Prototype feel before polish**
   - Combat works with janky camera
   - Fix annoyances AFTER validating fun

3. **Debug systematically**
   - Methodical prints and checks
   - Found all bugs quickly

---

## Session Stats (20.11.2025)

**Duration:** ~8 hours  
**Major Systems:** Boss AI state machine, Flinch resistance  
**States Implemented:** 5 (Idle, Approach, Attack, Recover, Flinch)  
**New Variables:** ~10 (AI config + flinch tracking)  
**Bugs Fixed:** 4 (rotation, pathfinding, montage, print)  
**Lines of Blueprint:** ~800-1000 nodes  
**Playtest Result:** "Actually feels like a game!" ✅  

---

## Milestone Achievement

**This system crossed a psychological threshold:**

**Before:** "Am I building systems or a game?"  
**After:** "I have a game, now I polish it."

**The hardest work is done. Now it's tuning and content.**

---

*Last Updated: 20.11.2025*  
*Status: Boss AI complete, playable combat encounter*  
*Next: Polish (camera/input) → Variety (more attacks) → Tuning*
