# Player Damage System Documentation

**Status:** ✅ Complete + Directional Flinch System  
**Sessions:** 21.11.2025 (initial), 27.11.2025 (flinch, hit feedback, mesh detection), 28.11.2025 (directional flinch, combat interruption)  
**Purpose:** Player health, damage mitigation, flinch reactions, and death mechanics

---

## Overview

The Player Damage System handles all incoming damage to the player character, including mitigation through defensive actions (blocking, dodging), flinch level calculation, health management, and death state. The system is designed around player-owned defensive state - the boss reports damage and flinch level, the player determines how much to actually take.

**Design Philosophy:**
- **Player owns mitigation logic** - All defensive state (blocking, i-frames, armor, buffs) checked player-side
- **Boss just reports damage + flinch + attacker** - Enemy doesn't need to know about player's defensive capabilities
- **Expandable foundation** - Easy to add new mitigation sources (armor, potions, guard points, etc.)
- **Interface-based** - Uses BPI_Damageable for consistent damage application
- **Mesh-based hit detection** - Accurate collision against player body, not oversized capsule
- **Flinch as punishment** - Monster Hunter style: getting hit disrupts flow, no input during flinch

---

## Architecture

### Separation of Concerns

**Boss Perspective:**
```
"I'm attacking for 10 damage, flinch level 1, it's me attacking"
→ ApplyDamageToActor(10, 1, Self)
```

**Player Perspective:**
```
"I received 10 damage attack, flinch level 1, from that boss"
→ Check MY state: Blocking? I-frames? Buffed?
→ Apply damage mitigation → 3 final damage
→ Apply flinch mitigation → 0 final flinch (blocked)
→ Calculate hit direction from attacker position
→ Play block impact animation
```

**Why this architecture:**
- ✅ Boss code never changes when adding player abilities
- ✅ Player encapsulates all defensive mechanics
- ✅ Clean interface boundary
- ✅ Scalable to complex mitigation systems
- ✅ Attacker reference enables directional reactions

---

## Variables (BP_ThirdPersonCharacter)

### Health Management

| Variable | Type | Default | Category | Purpose |
|----------|------|---------|----------|---------|
| `MaxHealth` | Float | 100.0 | Combat Stats | Maximum health pool |
| `CurrentHealth` | Float | 100.0 | Combat State | Current health value |

**Instance Editable:** ☑ Yes (tune in editor per level/difficulty)

---

### State Flags

| Variable | Type | Default | Category | Purpose |
|----------|------|---------|----------|---------|
| `bIsDead` | Boolean | False | Combat State | Is player dead? |
| `bIsInvulnerable` | Boolean | False | Combat State | I-frames active? (dodge, etc.) |
| `bShouldFlinch` | Boolean | False | Combat State | Triggers flinch state in ABP |

**Usage:**
- `bIsDead` - Triggers death animation, disables input, stops boss AI
- `bIsInvulnerable` - Set by ANS_IFrames during dodge, negates all damage and flinch
- `bShouldFlinch` - Set by TriggerFlinch, cleared by AN_FlinchEnd

---

### Flinch Variables (Session 28.11)

| Variable | Type | Default | Category | Purpose |
|----------|------|---------|----------|---------|
| `bShouldFlinch` | Boolean | False | Combat State | ABP flinch state trigger |
| `LastHitAngle` | Float | 0.0 | Combat State | Direction of last hit (-180 to 180) |

**Usage:**
- `LastHitAngle` - Feeds blend space for directional flinch animation
- Calculated from attacker position relative to player facing

---

## BPI_Damageable Interface (Updated 28.11.2025)

**Function:** ApplyDamageToActor

**Signature:**
- Input: `Damage` (Float) - Raw damage from attacker
- Input: `FlinchLevel` (Integer) - Raw flinch level from attacker
- Input: `Attacker` (Actor Reference) - Who dealt the damage
- Output: None

**Why Attacker in interface (Session 28.11):**
- Enables directional flinch calculation
- Player calculates hit angle from attacker position
- Useful for future systems (damage numbers position, "attacked by" tracking)

**Note:** Boss (BP_EnemyBase) also implements this interface but ignores FlinchLevel - uses own flinch resistance system.

---

## Core Functions

### CalculateDamageReduction

**Location:** BP_ThirdPersonCharacter → Functions  
**Type:** Function (Pure: No)  
**Purpose:** Central mitigation logic - checks all defensive state and returns final damage

**Signature:**
- Input: `IncomingDamage` (Float) - Raw damage from attacker
- Output: `FinalDamage` (Float) - Damage after all mitigation

**Implementation:**

```
CalculateDamageReduction (IncomingDamage):
    ↓
    Set local variable: WorkingDamage = IncomingDamage
    ↓
    Branch: bIsInvulnerable?
    ├─ True → Set WorkingDamage = 0.0
    │         Print: "Dodged! No damage received"
    │         Skip to Return
    │
    └─ False → Branch: bIsBlocking?
               ├─ True → WorkingDamage = WorkingDamage * 0.3
               │         Print: "Blocked! {IncomingDamage} → {WorkingDamage}"
               │
               └─ False → Print: "Hit! {WorkingDamage} damage"
    ↓
    Return WorkingDamage
```

**Mitigation Priority:**
1. **I-frames (bIsInvulnerable)** - 100% negation, highest priority
2. **Block (bIsBlocking)** - 70% reduction (multiply by 0.3)
3. **No mitigation** - Full damage

**Future Expansion Points:**
```
After blocking check, before return:
    ↓
    WorkingDamage *= (1.0 - ArmorReduction)  // Armor system
    ↓
    Branch: bDefensePotionActive?
    └─ True → WorkingDamage *= 0.5  // Defense buff
    ↓
    Branch: bGuardPointActive?
    └─ True → WorkingDamage *= 0.5  // Guard points in attack animations
    ↓
    Return WorkingDamage
```

**Why separate function:**
- All mitigation logic in one place
- Easy to debug (single point of truth)
- Expandable without modifying damage application
- Reusable from multiple sources (fall damage, DoT, environmental hazards)

---

### CalculateFlinchReduction (Session 27.11.2025)

**Location:** BP_ThirdPersonCharacter → Functions  
**Type:** Function  
**Purpose:** Central flinch mitigation - mirrors damage pattern

**Signature:**
- Input: `IncomingFlinchLevel` (Integer) - Raw flinch level from attacker
- Output: `FinalFlinchLevel` (Integer) - Flinch level after mitigation

**Implementation:**

```
CalculateFlinchReduction (IncomingFlinchLevel):
    ↓
    Branch: bIsInvulnerable?
    ├─ True → Return 0 (dodged, no flinch)
    │
    └─ False → Branch: bIsBlocking?
               ├─ True → FinalFlinchLevel = IncomingFlinchLevel - 1
               │         Clamp (Min: 0, Max: 3)
               │         Return FinalFlinchLevel
               │
               └─ False → Return IncomingFlinchLevel (full flinch)
```

**Flinch Mitigation:**
| Attack Flinch | Blocked | Unblocked |
|---------------|---------|-----------|
| Light (1) | 0 - no reaction | Light flinch |
| Medium (2) | 1 - light flinch | Medium flinch |
| Heavy (3) | 2 - medium flinch | Knockback |

**Same priority as damage:**
1. I-frames → no flinch
2. Block → reduce by 1
3. No mitigation → full flinch

---

### ApplyDamageToActor (Updated 28.11.2025)

**Location:** BP_ThirdPersonCharacter → Functions  
**Interface:** BPI_Damageable  
**Purpose:** Apply final damage/flinch to health and trigger appropriate feedback

**Signature:**
- Input: `Damage` (Float) - Raw damage from attacker
- Input: `FlinchLevel` (Integer) - Raw flinch level from attacker
- Input: `Attacker` (Actor Reference) - Who dealt the damage
- Output: None (interface function)

**Implementation:**

```
ApplyDamageToActor (Damage, FlinchLevel, Attacker):
    ↓
    Branch: bIsDead?
    ├─ True → Return (ignore damage when dead)
    │
    └─ False → 
        FinalDamage = CalculateDamageReduction (Damage)
        FinalFlinchLevel = CalculateFlinchReduction (FlinchLevel)
        ↓
        CurrentHealth - FinalDamage = NewHealth
        Set CurrentHealth = NewHealth
        ↓
        Branch: CurrentHealth <= 0?
        ├─ True → Set CurrentHealth = 0 (clamp)
        │         HandlePlayerDeath (function call)
        │
        └─ False → Branch: FinalDamage > 0?
                   ├─ True → HandlePlayerHit (FinalDamage, FinalFlinchLevel, Attacker)
                   └─ False → Skip (dodged, no feedback needed)
```

**Key Changes (Session 28.11):**
- Added Attacker parameter
- Passes Attacker to HandlePlayerHit for directional flinch calculation

**Why check FinalDamage > 0:**
- Dodge = 0 damage = no hit feedback needed
- Block = reduced damage = still needs feedback
- Clean gate for feedback system

---

### HandlePlayerDeath

**Location:** BP_ThirdPersonCharacter → Functions  
**Type:** Function  
**Purpose:** Execute death sequence and cleanup

**Signature:**
- Input: None
- Output: None

**Implementation:**

```
HandlePlayerDeath:
    ↓
    Set bIsDead = True
    ↓
    Print String: "I'm dead :'("
    ↓
    Get Player Controller (Player Index: 0)
    ↓
    Disable Input
    ├─ Target: Self (BP_ThirdPersonCharacter)
    └─ Player Controller: [from Get Player Controller]
```

**What this does:**
- Sets death flag (triggers animation state, stops boss AI)
- Disables ALL input (movement, attacks, camera)
- Debug print for testing

**Future Enhancements:**
```
HandlePlayerDeath:
    [... existing logic ...]
    ↓
    Play Sound 2D: Death Scream
    ↓
    Camera: Zoom out slightly (cinematic death)
    ↓
    Delay (2.0s)
    ↓
    Show death screen UI / Restart level
```

**Why separate function:**
- Reusable from multiple death sources (damage, fall, instant-kill hazards)
- Easy to expand (VFX, SFX, UI, camera effects)
- Clean separation from damage calculation

---

### HandlePlayerHit (Updated 28.11.2025)

**Location:** BP_ThirdPersonCharacter → Functions  
**Type:** Function  
**Purpose:** Non-lethal damage feedback - sound, flinch animations, combat interruption

**Signature:**
- Input: `FinalDamage` (Float) - Damage after mitigation
- Input: `FinalFlinchLevel` (Integer) - Flinch level after mitigation
- Input: `Attacker` (Actor Reference) - Who dealt the damage

**Implementation:**

```
HandlePlayerHit (FinalDamage, FinalFlinchLevel, Attacker):
    ↓
    Branch: bIsBlocking?
    │
    ├─ True → BLOCKED HIT
    │         Play Sound: Block impact sound
    │         Play Montage: AM_BlockFlinch
    │
    └─ False → UNBLOCKED HIT
               Play Sound: Pain grunt
               ↓
               Branch: FinalFlinchLevel > 0?
               ├─ True → TriggerFlinch (Attacker)
               └─ False → No flinch animation
```

**Key Changes (Session 28.11):**
- Added Attacker parameter
- Calls TriggerFlinch with Attacker for directional calculation
- Separated flinch triggering into dedicated function

---

### TriggerFlinch (Session 28.11.2025)

**Location:** BP_ThirdPersonCharacter → Functions  
**Type:** Function  
**Purpose:** Initiate flinch state - interrupt combat, calculate direction, disable input

**Signature:**
- Input: `Attacker` (Actor Reference) - Who dealt the damage
- Output: None

**Implementation:**

```
TriggerFlinch (Attacker):
    ↓
    === INTERRUPT CURRENT ACTION ===
    Montage Stop (Target: Mesh → Get Anim Instance)
    Print: "Montage Stopped!"
    ↓
    ResetCombatState ()
    ↓
    === CALCULATE HIT DIRECTION ===
    Find Look At Rotation (Start: Self Location, Target: Attacker Location)
        → LookAtRotation
    ↓
    Delta Rotator (A: LookAtRotation, B: Self Rotation)
        → Delta
    ↓
    Set LastHitAngle = Delta.Yaw
    ↓
    === TRIGGER FLINCH STATE ===
    Set bShouldFlinch = True
    ↓
    === DISABLE INPUT (Monster Hunter style) ===
    Get Player Controller (0)
    ↓
    Disable Input (Target: Self)
```

**Why this order:**
1. Stop montage FIRST - interrupts attacks/dodges immediately
2. Reset combat state - clears flags that might block future actions
3. Calculate direction - needs attacker position
4. Trigger ABP state - starts flinch animation
5. Disable input - player can't act during flinch (punishment)

**Design Decision - No input during flinch:**
- Monster Hunter style: getting hit disrupts your flow
- Punishment for missing dodge/block
- Creates tension and stakes
- Flinch duration = animation length (not arbitrary)

---

### ResetCombatState (Session 28.11.2025)

**Location:** BP_ThirdPersonCharacter → Functions  
**Type:** Function  
**Purpose:** Clear all combat state flags - prevents state leakage when interrupted

**Signature:**
- Input: None
- Output: None

**Implementation:**

```
ResetCombatState:
    ↓
    Set CurrentAttackName = None
    Set bCanCombo = False
    Set bIsCharging = False
    Set bIsDodging = False
    Set bCanDodge = True
    Set bCanBlock = True
```

**Why this function exists:**
- ANS_LockDefense/ANS_ComboWindow set flags during animations
- If animation is interrupted (flinch), ANS End never fires
- Flags stay in bad state (can't dodge, can't block, etc.)
- This function force-resets everything to safe defaults

**Called from:**
- TriggerFlinch (before flinch animation)
- HandlePlayerDeath (cleanup on death)
- Any future interrupt (stagger, knockback, grab)

**State Leakage Prevention:**
```
Without ResetCombatState:
    Player attacking → ANS_LockDefense sets bCanDodge = False
    Boss hits player → Attack interrupted
    ANS_LockDefense End never fires
    bCanDodge stays False forever
    Player stuck unable to dodge

With ResetCombatState:
    Player attacking → ANS_LockDefense sets bCanDodge = False
    Boss hits player → TriggerFlinch calls ResetCombatState
    bCanDodge = True (reset)
    Player can dodge after flinch ends
```

---

## Flinch Animation System (Session 28.11.2025)

### ABP_Unarmed Flinch State

**Location:** ABP_Unarmed → Locomotion State Machine

**State:** Flinch
- **Animation:** Blend Space (BS_Flinch_Directional)
- **Input:** LastHitAngle (-180 to 180)
- **Looping:** No (plays once)

**Transitions:**
| From | To | Condition |
|------|-----|-----------|
| Idle | Flinch | bShouldFlinch == True |
| Walk/Run | Flinch | bShouldFlinch == True |
| Flinch | Idle | bShouldFlinch == False |

**Why always exit to Idle:**
- Input disabled during flinch → player not moving
- Speed = 0 at flinch end
- Idle then transitions to Walk if player holds movement

### AN_FlinchEnd (AnimNotify)

**Location:** Animation Notifies  
**Placed on:** All flinch blend space animations (5 directions)  
**Purpose:** Clear flinch state and re-enable input

**Implementation:**

```
Received_Notify:
    Get Owning Actor → Cast to BP_ThirdPersonCharacter
    ↓
    Set bShouldFlinch = False
    ↓
    Get Player Controller (0)
    ↓
    Enable Input (Target: Character from cast)
```

**Why notify instead of timer:**
- Animation length is self-documenting
- Change animation → timing updates automatically
- Same pattern as dodge/attack end notifies

**Why on all 5 animations:**
- Blend space plays different animations based on angle
- Each animation needs the notify at its end
- Small manual work, but stable (directions won't change often)

### Directional Flinch Calculation

**Hit Angle Math:**

```
Find Look At Rotation (Start: Player, Target: Attacker)
    → "Rotation to face attacker"
    
Delta Rotator (A: LookAtRotation, B: PlayerRotation)
    → Difference between facing and hit direction
    
Delta.Yaw = Hit angle relative to player facing
    → Positive = hit from right
    → Negative = hit from left
    → ~0 = hit from front
    → ~180/-180 = hit from back
```

**Blend Space Input:**
- Center (0°): Front flinch
- Right (90°): Right flinch
- Left (-90°): Left flinch
- Back (180°/-180°): Back flinch

---

## Mesh-Based Hit Detection (Session 27.11.2025)

### Problem

Boss hitbox overlaps player Capsule (huge for movement) before Mesh (actual body). Capsule triggers damage even when sword visually misses.

### Solution

Check which component was hit. Only process damage if hit component is the Mesh.

### Player Mesh Collision Settings

**BP_ThirdPersonCharacter → Mesh component → Collision:**
- **Collision Enabled:** Query Only
- **Object Type:** Pawn
- **Generate Overlap Events:** ☑ Checked
- **Collision Response to WorldDynamic:** Overlap

**Physics Asset (on mesh):**
- Select all bodies
- **Physics Type:** Kinematic (not Simulated)
- **Collision Response:** Enabled

### Boss-Side Check (DeliverHitboxDamage)

```
DeliverHitboxDamage (OtherActor, HitComponent):
    ↓
    Cast to BP_ThirdPersonCharacter (OtherActor)
    ├─ Failed → Return
    │
    └─ Success → Branch: Is Valid?
                 ├─ False → Return
                 │
                 └─ True → Get Mesh (from cast)
                           ↓
                           Branch: HitComponent == Mesh?
                           ├─ False → Return (hit capsule, ignore)
                           │
                           └─ True → [AlreadyHitActors check]
                                     ↓
                                     ApplyDamageToActor (Damage, FlinchLevel, Self)
```

**Result:**
- Capsule overlaps → filtered out
- Mesh overlaps → damage processed
- Visually accurate hit detection

---

## Flinch Level System (Session 27.11.2025)

### Design Overview

Flinch level is separate from damage. A weak attack can have high flinch (stagger), a strong attack can have low flinch (clean hit).

### Flinch Levels

| Level | Name | Unblocked Effect | Blocked Effect |
|-------|------|------------------|----------------|
| 0 | None | No reaction | Block impact only |
| 1 | Light | Light flinch | No flinch (reduced to 0) |
| 2 | Medium | Medium flinch | Light flinch (reduced to 1) |
| 3 | Heavy | Knockback | Medium flinch (reduced to 2) |

### Data Structure

**S_BossAttackData (updated):**
| Column | Type | Purpose |
|--------|------|---------|
| AttackName | Name | Row identifier |
| AttackMontage | Anim Montage | Animation to play |
| Damage | Float | Health damage |
| FlinchLevel | Integer | Flinch severity (0-3) |
| Cooldown | Float | Seconds before can attack again |

### Flow

```
Boss Attack (10 damage, flinch 1)
    ↓
DeliverHitboxDamage → ApplyDamageToActor(10, 1, BossRef)
    ↓
Player: CalculateDamageReduction(10) → 3 (blocked)
Player: CalculateFlinchReduction(1) → 0 (blocked, reduced by 1)
    ↓
HandlePlayerHit(3, 0, BossRef)
    ↓
bIsBlocking = True → Play block sound + AM_BlockFlinch
(No TriggerFlinch because FlinchLevel = 0)
```

---

## Death Animation System (ABP_Unarmed)

### Variable

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bIsDead` | Boolean | False | Triggers death animation state |

**Updated in Event Blueprint Update Animation:**
```
Sequence → Then 4 (or next available):
    Get Owner (Pawn)
    → Cast to BP_ThirdPersonCharacter
    → Get bIsDead
    → Set bIsDead (local AnimBP variable)
```

---

### Death State

**Location:** ABP_Unarmed → AnimGraph → Locomotion State Machine

**State Configuration:**
- **Animation:** Death animation sequence (e.g., Death_Forward)
- **Looping:** ☐ Unchecked (play once and hold final frame)
- **Play Rate:** 1.0

**Transition In:**
- From: Idle (or multiple states: Idle, Walk, Blocking)
- Condition: `bIsDead == True`
- Blend Time: 0.0 (instant)

**Transition Out:**
- None (terminal state)
- Player stays in death pose until level restarts

**Why terminal state:**
- Death is irreversible (no respawn in-place)
- Clear feedback (combat ended)
- Level restart or game over screen handles respawn

---

## Integration with Defensive Systems

### Block Integration

**How it works:**
```
Boss attacks (10 damage, flinch 1)
    ↓
Player.ApplyDamageToActor(10, 1, BossRef)
    ↓
CalculateDamageReduction checks bIsBlocking
    ├─ True → 10 * 0.3 = 3 damage
    └─ False → 10 damage
    ↓
CalculateFlinchReduction checks bIsBlocking
    ├─ True → 1 - 1 = 0 flinch
    └─ False → 1 flinch
    ↓
HandlePlayerHit(3, 0, BossRef) → Block sound + block animation
(FlinchLevel 0, so no TriggerFlinch)
```

**Block system provides:**
- `bIsBlocking` flag (set by R2/RT hold)
- Already implemented in BlockSystem.md
- No changes needed to block logic

**Damage reduction:**
- **70% reduction** (multiply by 0.3)
- Can be tuned: Change multiplier in CalculateDamageReduction
- Future: Different reduction per block type (shield vs parry vs deflect)

**Flinch reduction:**
- **-1 level** (light negated, medium→light, heavy→medium)
- Tunable: Change subtraction value in CalculateFlinchReduction

---

### I-Frames Integration (Dodge)

**How it works:**
```
Boss attacks (10 damage, flinch 1)
    ↓
Player.ApplyDamageToActor(10, 1, BossRef)
    ↓
CalculateDamageReduction checks bIsInvulnerable
    ├─ True → 0 damage (100% negation)
    └─ False → Continue to blocking check
    ↓
CalculateFlinchReduction checks bIsInvulnerable
    ├─ True → 0 flinch (100% negation)
    └─ False → Continue to blocking check
    ↓
FinalDamage = 0 → HandlePlayerHit NOT called
```

**I-frame system provides:**
- `bIsInvulnerable` flag (set by ANS_IFrames during dodge)
- Controlled via AnimNotifyState in dodge montages
- Duration: 0.4s (God of War style - 50% of dodge)

**Why i-frames check first:**
- I-frames = complete immunity (damage AND flinch)
- Overrides all other mitigation
- Dodge while blocking = i-frames win

**See DodgeSystem.md for i-frame implementation details.**

---

### Priority Order (Critical)

**Mitigation checks in both functions:**

1. **bIsInvulnerable** (I-frames) - 100% negation, highest priority
2. **bIsBlocking** (Block) - 70% damage reduction, -1 flinch level
3. **Future systems** (armor, buffs, guard points)

**Why this order matters:**
- Player dodging while blocking = i-frames win (full immunity)
- Prevents weird interactions (blocked dodge? dodged block?)
- Clear hierarchy: Invulnerable > Reduced > Normal

---

## Boss Integration (Updated 28.11.2025)

**Boss-side implementation:**

In BP_EnemyBase → DeliverHitboxDamage:

```
DeliverHitboxDamage (OtherActor, HitComponent):
    ↓
    Cast to BP_ThirdPersonCharacter
    ↓
    Branch: Is Valid?
    └─ True → Get Mesh
              ↓
              Branch: HitComponent == Mesh?
              ├─ False → Return (capsule hit, ignore)
              │
              └─ True → Branch: AlreadyHitActors Contains?
                        ├─ True → Return (already hit this swing)
                        │
                        └─ False → Add to AlreadyHitActors
                                   ↓
                                   Get CurrentAttackData.Damage
                                   Get CurrentAttackData.FlinchLevel
                                   ↓
                                   ApplyDamageToActor (Damage, FlinchLevel, Self)
                                                                            ↑
                                                              Attacker = Boss
```

**Boss doesn't know:**
- ❌ If player is blocking
- ❌ If player has i-frames
- ❌ If player has armor/buffs
- ❌ Player's final damage/flinch taken
- ❌ Which direction player will flinch

**Boss only knows:**
- ✅ "I hit with my attack"
- ✅ "My attack deals 10 damage, flinch level 1"
- ✅ "I'm the attacker"

**Player handles everything else.**

---

## Initialization

**Event BeginPlay:**

```
Event BeginPlay:
    ↓
    [... existing logic ...]
    ↓
    Set CurrentHealth = MaxHealth
    ↓
    Print: "Player Health: {CurrentHealth}"
```

**Why initialize:**
- Ensures health starts at max
- Useful if MaxHealth changed in editor
- Debug confirmation of starting state

---

## Testing & Validation

### Basic Damage Flow

**Test 1: Unmitigated Damage**
1. Boss attacks (10 damage, flinch 1)
2. Player stands still (no block, no dodge)
3. Expected:
   - Print: "Hit! 10 damage"
   - Health: 90 / 100
   - Pain grunt plays
   - Directional flinch animation plays
   - Input disabled during flinch

**Test 2: Blocked Damage**
1. Boss attacks (10 damage, flinch 1)
2. Player holds R2 (blocking)
3. Expected:
   - Print: "Blocked! 10 → 3"
   - Health: 97 / 100
   - Block impact sound plays
   - Block flinch animation plays (flinch reduced to 0, no TriggerFlinch)

**Test 3: I-Frame Dodge**
1. Boss attacks (10 damage, flinch 1)
2. Player dodges during i-frame window
3. Expected:
   - Print: "Dodged! No damage received"
   - Health: unchanged (100 / 100)
   - No sound, no animation (HandlePlayerHit not called)

**Test 4: Death**
1. Take 10 hits without blocking (100 HP / 10 damage)
2. Expected:
   - Print: "I'm dead :'("
   - Death animation plays
   - Input disabled
   - Boss stops attacking

**Test 5: Mesh vs Capsule Hit**
1. Boss swings near player but misses body
2. Sword passes through capsule but not mesh
3. Expected:
   - No damage (capsule hit filtered out)

**Test 6: Flinch Interrupts Attack**
1. Player mid-attack animation
2. Boss hits player (outside i-frames)
3. Expected:
   - Attack montage stops immediately
   - Flinch animation plays
   - Combat state reset (can attack again after flinch)

**Test 7: Flinch Interrupts Dodge**
1. Player mid-dodge animation (outside i-frame window)
2. Boss hits player
3. Expected:
   - Dodge montage stops immediately
   - Flinch animation plays
   - Dodge state reset (can dodge again after flinch)

**Test 8: Directional Flinch**
1. Boss attacks from player's right side
2. Expected:
   - LastHitAngle ≈ 90°
   - Right-side flinch animation plays

---

### Edge Cases

**Test 9: Damage While Dead**
1. Player at 0 HP (dead)
2. Boss attacks again
3. Expected:
   - No damage processing (early exit in ApplyDamageToActor)
   - No prints, no health change

**Test 10: Overkill Damage**
1. Player at 5 HP remaining
2. Boss deals 50 damage
3. Expected:
   - Health clamped to 0 (not -45)
   - Death triggers normally

**Test 11: Dodge + Block Simultaneously**
1. Player dodging while holding R2
2. Boss attacks during i-frame window
3. Expected:
   - I-frames take priority
   - 0 damage, 0 flinch (not reduced values)

---

## Performance Considerations

**Cost:** Negligible (< 0.01ms per damage event)

**CalculateDamageReduction + CalculateFlinchReduction:**
- 4-6 boolean checks (extremely fast)
- 2-3 arithmetic operations
- No loops, no arrays
- Runs only when hit (not every frame)

**Mesh Hit Detection:**
- Simple component comparison
- No performance impact

**Directional Flinch Calculation:**
- 2 rotation operations
- Only runs on hit

**Scalability:**
- Works identically for 1 player or 4 (co-op ready)
- Easy to add more mitigation sources
- No optimization needed

---

## Known Issues

### Current (Accepted for Prototype)

**Input Lock on Death:**
- Disables ALL input including camera
- Expected behavior: Full input lock
- Future: Allow camera movement during death

**Single Flinch Animation Set:**
- Only light flinch implemented (directional)
- Medium flinch and knockback deferred
- All flinch levels use same animation for now

**Sound Variety:**
- Single sound per state (block hit, unblocked hit)
- Future: Randomized sound selection from data table

**Occasional Queued Flinch:**
- Sometimes flinch appears delayed
- Unclear reproduction steps
- Low priority until reproducible

---

### Fixed (Session 27-28.11.2025)

- ✅ No hit feedback (added sounds and animations)
- ✅ Capsule hit detection (now mesh-based)
- ✅ No flinch system (now implemented)
- ✅ Flinch not directional (now uses blend space)
- ✅ Flinch doesn't interrupt attacks (now stops montage)
- ✅ Flinch doesn't interrupt dodges (now stops montage)
- ✅ State leakage after interrupt (ResetCombatState fixes)
- ✅ bIsDodging cleared immediately (now uses AN_DodgeEnded)

---

## Future Enhancements

### High Priority

1. **Medium/Heavy Flinch** (1 hour)
   - AM_Flinch_Medium animation
   - AM_Knockback animation
   - Expanded TriggerFlinch branching by level

2. **I-Frames During Flinch** (30 min)
   - Playtest decision: does player need protection during flinch?
   - Monster Hunter style: brief immunity prevents stunlock
   - Set bIsInvulnerable = True in TriggerFlinch, clear in AN_FlinchEnd

3. **Randomized Hit Sounds** (30 min)
   - Data table with sound variants
   - Random selection per hit
   - Same pattern as weapon hit sounds

4. **Health Bar UI** (1 hour)
   - Widget Blueprint: Health bar
   - Update on damage
   - Flash/pulse on low health

---

### Medium Priority

5. **Armor System** (1-2 hours)
   - Add ArmorValue variable (0.0 to 1.0)
   - Integrate in CalculateDamageReduction
   - Equipment that modifies armor
   - Visual feedback (sparks on armored hits)

6. **Perfect Block** (1 hour)
   - Timing window at block start (0.2s)
   - 100% damage negation on perfect block
   - 100% flinch negation
   - Unique feedback (parry flash, sound ding)
   - Possible: Counter-attack window

7. **Guard Points** (2 hours)
   - Specific frames in attack animations have defensive properties
   - ANS_GuardPoint sets bGuardPointActive
   - Reduces damage/flinch during specific attack frames
   - Monster Hunter-style active defense

8. **Stamina System Integration** (2-3 hours)
   - Block costs stamina per hit blocked
   - Dodge costs stamina to execute
   - Out of stamina = can't block/dodge
   - Stamina regeneration system

---

### Polish Phase

9. **Damage Numbers** (30 min)
   - Floating damage text on hit
   - Color-coded: Red (full), Yellow (reduced), Blue (blocked)

10. **Death Screen / Respawn** (2 hours)
    - Game over UI
    - Respawn at checkpoint
    - "You Died" screen (Dark Souls style)

11. **Invincibility Frames on Spawn** (30 min)
    - Brief invulnerability after respawn
    - Prevent spawn camping by boss

---

## Design Rationale

### Why Player-Owned Mitigation?

**Alternative considered:** Boss calculates reduced damage before applying

**Rejected because:**
- ❌ Boss needs to know all player defensive state
- ❌ Adding new mitigation = modify boss code
- ❌ Doesn't scale to complex systems (armor, buffs, guard points)
- ❌ Violates encapsulation

**Chosen approach:**
- ✅ Player owns all defensive state
- ✅ Boss just reports raw damage + flinch + attacker
- ✅ Easy to expand player abilities without touching boss
- ✅ Clean separation of concerns

---

### Why Attacker in Interface? (Session 28.11)

**Alternatives considered:**
1. Player finds nearest enemy
2. Store "last attacker" globally
3. Don't track attacker

**Rejected because:**
- ❌ Nearest enemy might not be the attacker
- ❌ Global state is fragile
- ❌ Directional flinch needs attacker position

**Chosen approach:**
- ✅ Attacker passes Self in interface call
- ✅ Explicit, no guessing
- ✅ Works for multiple enemies
- ✅ Useful for future systems (damage numbers, aggro)

---

### Why FlinchLevel in Interface?

**Alternatives considered:**
1. Separate interface call for flinch
2. Cast and set pending variable
3. Player-specific function bypassing interface

**Rejected because:**
- ❌ Multiple calls for one hit event
- ❌ Implicit state (pending variable)
- ❌ Different patterns for different damage sources

**Chosen approach:**
- ✅ Single interface call with all values
- ✅ Boss (attacker) ignores FlinchLevel in own implementation
- ✅ Clean, explicit contract
- ✅ Feels slightly impure but most pragmatic

---

### Why Separate TriggerFlinch Function? (Session 28.11)

**Alternative considered:** Inline flinch logic in HandlePlayerHit

**Rejected because:**
- ❌ HandlePlayerHit becomes bloated
- ❌ Flinch needs montage stop, state reset, direction calc
- ❌ Hard to debug and expand

**Chosen approach:**
- ✅ Single responsibility (TriggerFlinch = start flinch)
- ✅ Easy to debug (one function to check)
- ✅ Expandable (add i-frames, screen shake, etc.)

---

### Why ResetCombatState Function? (Session 28.11)

**Alternative considered:** Reset flags inline in TriggerFlinch

**Rejected because:**
- ❌ Same reset needed in HandlePlayerDeath
- ❌ Same reset needed in future interrupts
- ❌ Easy to miss a flag

**Chosen approach:**
- ✅ Single source of truth for "clean slate"
- ✅ Reusable from any interrupt
- ✅ Prevents state leakage
- ✅ Easy to add new flags

---

### Why No Input During Flinch? (Session 28.11)

**Alternative considered:** Allow movement/dodge during flinch

**Rejected because:**
- ❌ Reduces punishment for getting hit
- ❌ Not Monster Hunter style
- ❌ Makes i-frames less meaningful

**Chosen approach:**
- ✅ Flinch = commitment (you got hit, you pay)
- ✅ Creates tension and stakes
- ✅ Matches design pillar: "every action matters"
- ✅ Flinch duration = animation length (fair, predictable)

---

### Why Mesh-Based Hit Detection?

**Alternative considered:** Capsule-based (default UE behavior)

**Rejected because:**
- ❌ Capsule is huge (for movement/navigation)
- ❌ Hits register when visually missing
- ❌ I-frames feel unfair ("I dodged that!")

**Chosen approach:**
- ✅ Mesh collision follows body
- ✅ Visually accurate hits
- ✅ Fair i-frame timing
- ✅ Physics Asset follows skeleton animation

---

## Related Systems

**Upstream (triggers damage):**
- Enemy Attack System (boss hitbox overlaps player mesh)
- Environmental Hazards (future: spikes, fire, fall damage)
- DoT Effects (future: poison, bleed)

**Parallel (provides mitigation):**
- Block System (sets bIsBlocking)
- Dodge System (ANS_IFrames sets bIsInvulnerable)
- Armor System (future: ArmorValue)
- Buff System (future: defense potions, skills)

**Downstream (called by damage):**
- Death Animation (ABP_Unarmed death state)
- Flinch Animation (ABP_Unarmed flinch state + blend space)
- Boss AI (checks player.bIsDead to stop attacking)
- UI System (future: health bar, damage numbers)
- Audio System (hit sounds integrated)

---

## Code References

### Key Files

**Blueprints:**
- `BP_ThirdPersonCharacter` - All damage/flinch system logic
- `ABP_Unarmed` - Death and flinch animation states
- `BP_EnemyBase` - DeliverHitboxDamage, calls ApplyDamageToActor

**Interfaces:**
- `BPI_Damageable` - Damage + FlinchLevel + Attacker application contract

**AnimNotifies:**
- `AN_FlinchEnd` - Clears bShouldFlinch, enables input

**Montages:**
- `AM_BlockFlinch` - Block impact animation

**Blend Spaces:**
- `BS_Flinch_Directional` - 5-direction flinch animations

**Data Tables:**
- `DT_GothicKnightAttacks` - Includes FlinchLevel per attack

---

### Key Functions

**BP_ThirdPersonCharacter:**
- `CalculateDamageReduction(IncomingDamage)` → Returns FinalDamage
- `CalculateFlinchReduction(IncomingFlinchLevel)` → Returns FinalFlinchLevel
- `ApplyDamageToActor(Damage, FlinchLevel, Attacker)` → Interface implementation
- `HandlePlayerDeath()` → Death sequence
- `HandlePlayerHit(FinalDamage, FinalFlinchLevel, Attacker)` → Hit feedback routing
- `TriggerFlinch(Attacker)` → Flinch initiation (stop montage, calc direction, disable input)
- `ResetCombatState()` → Clear all combat flags

**BP_EnemyBase:**
- `DeliverHitboxDamage(OtherActor, HitComponent)` → Mesh check + damage call

**ABP_Unarmed:**
- Event Blueprint Update Animation → Copies bIsDead, bShouldFlinch, LastHitAngle from character

---

### Key Variables

**BP_ThirdPersonCharacter:**
- `MaxHealth` (Float) - Health pool
- `CurrentHealth` (Float) - Current HP
- `bIsDead` (Boolean) - Death state
- `bIsInvulnerable` (Boolean) - I-frame state
- `bIsBlocking` (Boolean) - Block state (from BlockSystem)
- `bShouldFlinch` (Boolean) - Flinch state trigger
- `LastHitAngle` (Float) - Flinch direction

**BP_EnemyBase:**
- `AlreadyHitActors` (Array of Actor References) - Prevents multi-hit per swing

**ABP_Unarmed:**
- `bIsDead` (Boolean) - Triggers death animation
- `bShouldFlinch` (Boolean) - Triggers flinch animation
- `LastHitAngle` (Float) - Blend space input

---

## Architectural Principles Demonstrated

**Single Responsibility:**
- CalculateDamageReduction = damage mitigation only
- CalculateFlinchReduction = flinch mitigation only
- ApplyDamageToActor = health modification + routing
- HandlePlayerHit = feedback routing
- TriggerFlinch = flinch initiation only
- ResetCombatState = state cleanup only
- HandlePlayerDeath = death sequence only

**Encapsulation:**
- Player owns all defensive state
- Boss doesn't know about mitigation
- Clean interface boundary

**Open/Closed Principle:**
- Easy to add mitigation sources (armor, buffs)
- No modification to core damage flow
- Extends via Calculate functions

**Separation of Concerns:**
- Damage calculation ≠ Flinch calculation
- Damage application ≠ Death sequence ≠ Hit feedback ≠ Flinch trigger
- Clear functional boundaries

**Data-Driven:**
- FlinchLevel in attack data table
- Per-attack tuning without code changes
- Ready for expansion

**State Leakage Prevention:**
- ResetCombatState clears all flags
- Called on any interrupt
- Prevents stuck states

---

## Session Notes

**Session 21.11.2025:**
- Duration: ~3 hours
- Major Systems: Complete player damage system
- Functions Created: 3 (CalculateDamageReduction, HandlePlayerDeath, HandlePlayerHit)
- Variables Added: 4 (MaxHealth, CurrentHealth, bIsDead, bIsInvulnerable)
- AnimBP States: 1 (Death state in ABP_Unarmed)
- Integration Points: 2 (Block system, I-frames system)

**Session 27.11.2025:**
- Duration: ~3 hours
- Major Systems: Flinch system, mesh-based hit detection, hit feedback
- Functions Created: 2 (CalculateFlinchReduction, TriggerFlinch)
- Functions Updated: 2 (ApplyDamageToActor, HandlePlayerHit)
- Interface Updated: BPI_Damageable (added FlinchLevel parameter)
- Data Structure Updated: S_BossAttackData (added FlinchLevel)
- Mesh collision configured for accurate hit detection
- Hit sounds implemented (blocked vs unblocked)
- Flinch animations implemented (AM_BlockFlinch, AM_Flinch_Light)

**Session 28.11.2025:**
- Duration: ~2.5 hours
- Major Systems: Directional flinch, combat interruption, state leakage prevention
- Functions Created: 2 (TriggerFlinch refactored, ResetCombatState)
- Functions Updated: 2 (ApplyDamageToActor, HandlePlayerHit - added Attacker)
- Interface Updated: BPI_Damageable (added Attacker parameter)
- AnimNotify Created: AN_FlinchEnd (clears state, enables input)
- ABP States: Flinch state with blend space
- Directional flinch via hit angle calculation
- Flinch interrupts attacks and dodges (Montage Stop)
- Input disabled during flinch (Monster Hunter style)
- Fixed bIsDodging flag timing (was clearing immediately)

**Validation:**
- ✅ Player takes damage from boss
- ✅ Block reduces damage (70%) and flinch (-1 level)
- ✅ Dodge i-frames negate damage and flinch (100%)
- ✅ Death at 0 HP works
- ✅ Death animation plays
- ✅ Input disabled on death
- ✅ Boss stops attacking dead player
- ✅ Mesh-based hit detection working
- ✅ Hit sounds play (different for blocked/unblocked)
- ✅ Flinch animations play (directional)
- ✅ Flinch interrupts attacks
- ✅ Flinch interrupts dodges
- ✅ Input disabled during flinch
- ✅ State resets properly after flinch

---

## Design Philosophy

**This system embodies:**

✅ **Player Agency** - Mitigation is player choice (block vs dodge vs take hit)  
✅ **Clear Feedback** - Sounds and animations show exactly what happened  
✅ **Expandable Foundation** - Easy to add armor, buffs, perfect blocks, more flinch levels  
✅ **Clean Architecture** - Separation of concerns, single responsibility  
✅ **Defensive Options** - Multiple viable strategies (not just "dodge everything")  
✅ **Fair Hit Detection** - Mesh-based = visually accurate  
✅ **Punishment for Mistakes** - Flinch = commitment, no input, disrupts flow  
✅ **State Safety** - ResetCombatState prevents stuck states

**Inspired by:**
- **Dark Souls** - Health as resource, death as consequence
- **Monster Hunter** - Block vs dodge as meaningful choice, flinch levels, punishment for getting hit
- **God of War 2018** - Intimate combat with stakes, death matters

---

*Last Updated: 28.11.2025*  
*Status: Complete with directional flinch and combat interruption*  
*Next: Medium/Heavy flinch → I-frames during flinch → Health UI*
