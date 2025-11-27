# Player Damage System Documentation

**Status:** ✅ Complete + Flinch System  
**Sessions:** 21.11.2025 (initial), 27.11.2025 (flinch, hit feedback, mesh detection)  
**Purpose:** Player health, damage mitigation, flinch reactions, and death mechanics

---

## Overview

The Player Damage System handles all incoming damage to the player character, including mitigation through defensive actions (blocking, dodging), flinch level calculation, health management, and death state. The system is designed around player-owned defensive state - the boss reports damage and flinch level, the player determines how much to actually take.

**Design Philosophy:**
- **Player owns mitigation logic** - All defensive state (blocking, i-frames, armor, buffs) checked player-side
- **Boss just reports damage + flinch** - Enemy doesn't need to know about player's defensive capabilities
- **Expandable foundation** - Easy to add new mitigation sources (armor, potions, guard points, etc.)
- **Interface-based** - Uses BPI_Damageable for consistent damage application
- **Mesh-based hit detection** - Accurate collision against player body, not oversized capsule

---

## Architecture

### Separation of Concerns

**Boss Perspective:**
```
"I'm attacking for 10 damage, flinch level 1"
→ ApplyDamageToActor(10, 1)
```

**Player Perspective:**
```
"I received 10 damage attack, flinch level 1"
→ Check MY state: Blocking? I-frames? Buffed?
→ Apply damage mitigation → 3 final damage
→ Apply flinch mitigation → 0 final flinch (blocked)
→ Play block impact animation
```

**Why this architecture:**
- ✅ Boss code never changes when adding player abilities
- ✅ Player encapsulates all defensive mechanics
- ✅ Clean interface boundary
- ✅ Scalable to complex mitigation systems

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

**Usage:**
- `bIsDead` - Triggers death animation, disables input, stops boss AI
- `bIsInvulnerable` - Set by ANS_IFrames during dodge, negates all damage and flinch

---

## BPI_Damageable Interface (Updated 27.11.2025)

**Function:** ApplyDamageToActor

**Signature:**
- Input: `Damage` (Float) - Raw damage from attacker
- Input: `FlinchLevel` (Integer) - Raw flinch level from attacker
- Output: None

**Why FlinchLevel in interface:**
- Boss reports both values, player handles both internally
- Same pattern as damage - boss doesn't know player's mitigation
- Cleaner than separate calls or casting

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

### ApplyDamageToActor (Updated 27.11.2025)

**Location:** BP_ThirdPersonCharacter → Functions  
**Interface:** BPI_Damageable  
**Purpose:** Apply final damage/flinch to health and trigger appropriate feedback

**Signature:**
- Input: `Damage` (Float) - Raw damage from attacker
- Input: `FlinchLevel` (Integer) - Raw flinch level from attacker
- Output: None (interface function)

**Implementation:**

```
ApplyDamageToActor (Damage, FlinchLevel):
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
                   ├─ True → HandlePlayerHit (FinalDamage, FinalFlinchLevel)
                   └─ False → Skip (dodged, no feedback needed)
```

**Key Changes (Session 27.11):**
- Added FlinchLevel parameter
- Calls both CalculateDamageReduction and CalculateFlinchReduction
- Only calls HandlePlayerHit if FinalDamage > 0 (no feedback on dodge)
- Passes both FinalDamage and FinalFlinchLevel to HandlePlayerHit

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

### HandlePlayerHit (Updated 27.11.2025)

**Location:** BP_ThirdPersonCharacter → Functions  
**Type:** Function  
**Purpose:** Non-lethal damage feedback - sound and flinch animations

**Signature:**
- Input: `FinalDamage` (Float) - Damage after mitigation
- Input: `FinalFlinchLevel` (Integer) - Flinch level after mitigation

**Implementation:**

```
HandlePlayerHit (FinalDamage, FinalFlinchLevel):
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
               ├─ True → Play Montage: AM_Flinch_Light
               └─ False → No flinch animation
```

**Hit Feedback (Session 27.11):**
- **Blocked:** Block impact sound + block flinch animation (always, regardless of flinch level)
- **Unblocked:** Pain grunt + directional flinch animation (if flinch level > 0)

**Montages:**
- `AM_BlockFlinch` - Small shield impact animation
- `AM_Flinch_Light` - Light hit reaction (can use blend space for directional later)

**Future: Flinch Level Variations:**
```
Branch: FinalFlinchLevel?
├─ 0 → No animation
├─ 1 → AM_Flinch_Light
├─ 2 → AM_Flinch_Medium
└─ 3 → AM_Knockback
```

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
                                     ApplyDamageToActor (Damage, FlinchLevel)
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
DeliverHitboxDamage → ApplyDamageToActor(10, 1)
    ↓
Player: CalculateDamageReduction(10) → 3 (blocked)
Player: CalculateFlinchReduction(1) → 0 (blocked, reduced by 1)
    ↓
HandlePlayerHit(3, 0)
    ↓
bIsBlocking = True → Play block sound + AM_BlockFlinch
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

### Multiple Death Animations (Future)

**Current:** Single death animation for prototype

**Future Enhancement:**
```
Replace single animation with:
- Blend Space (based on hit direction)
- Random Selection (variety in deaths)
- Specific deaths per damage type (fire, fall, grab, etc.)
```

**Example: Directional Deaths:**
```
Death State:
    ↓
    Blend Space: BS_DeathDirectional
    ├─ Input: LastHitDirection (-180° to 180°)
    └─ Animations:
        - Death_Forward (hit from front)
        - Death_Backward (hit from back)
        - Death_Left (hit from left)
        - Death_Right (hit from right)
```

---

## Integration with Defensive Systems

### Block Integration

**How it works:**
```
Boss attacks (10 damage, flinch 1)
    ↓
Player.ApplyDamageToActor(10, 1)
    ↓
CalculateDamageReduction checks bIsBlocking
    ├─ True → 10 * 0.3 = 3 damage
    └─ False → 10 damage
    ↓
CalculateFlinchReduction checks bIsBlocking
    ├─ True → 1 - 1 = 0 flinch
    └─ False → 1 flinch
    ↓
HandlePlayerHit(3, 0) → Block sound + block animation
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
Player.ApplyDamageToActor(10, 1)
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

## Boss Integration (Updated 27.11.2025)

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
                                   ApplyDamageToActor (Damage, FlinchLevel)
```

**Boss doesn't know:**
- ❌ If player is blocking
- ❌ If player has i-frames
- ❌ If player has armor/buffs
- ❌ Player's final damage/flinch taken

**Boss only knows:**
- ✅ "I hit with my attack"
- ✅ "My attack deals 10 damage, flinch level 1"

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
   - Light flinch animation plays

**Test 2: Blocked Damage**
1. Boss attacks (10 damage, flinch 1)
2. Player holds R2 (blocking)
3. Expected:
   - Print: "Blocked! 10 → 3"
   - Health: 97 / 100
   - Block impact sound plays
   - Block flinch animation plays (flinch reduced to 0)

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

---

### Edge Cases

**Test 6: Damage While Dead**
1. Player at 0 HP (dead)
2. Boss attacks again
3. Expected:
   - No damage processing (early exit in ApplyDamageToActor)
   - No prints, no health change

**Test 7: Overkill Damage**
1. Player at 5 HP remaining
2. Boss deals 50 damage
3. Expected:
   - Health clamped to 0 (not -45)
   - Death triggers normally

**Test 8: Dodge + Block Simultaneously**
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

**Single Flinch Animation:**
- Only light flinch implemented
- Medium flinch and knockback deferred
- Directional flinch deferred

**Sound Variety:**
- Single sound per state (block hit, unblocked hit)
- Future: Randomized sound selection from data table

---

### Fixed (Session 27.11.2025)

- ✅ No hit feedback (added sounds and animations)
- ✅ Capsule hit detection (now mesh-based)
- ✅ No flinch system (now implemented)

---

## Future Enhancements

### High Priority

1. **Block Hitbox** (2 hours)
   - Shield should be raised towards attack
   - Damage outside of block-hitbox not affected by blocking state
   - Directional blocking

2. **Medium/Heavy Flinch** (1 hour)
   - AM_Flinch_Medium animation
   - AM_Knockback animation
   - Expanded HandlePlayerHit branching

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

10. **Directional Flinch** (1 hour)
    - Use UE5_SSH_Hurt_Light_BlendSpace for directional reactions
    - Calculate hit direction from attacker position
    - Different animations based on hit angle

11. **Death Screen / Respawn** (2 hours)
    - Game over UI
    - Respawn at checkpoint
    - "You Died" screen (Dark Souls style)

12. **Invincibility Frames on Spawn** (30 min)
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
- ✅ Boss just reports raw damage + flinch
- ✅ Easy to expand player abilities without touching boss
- ✅ Clean separation of concerns

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
- ✅ Single interface call with both values
- ✅ Boss (attacker) ignores FlinchLevel in own implementation
- ✅ Clean, explicit contract
- ✅ Feels slightly impure but most pragmatic

---

### Why Separate CalculateFlinchReduction?

**Alternative considered:** Inline flinch logic in ApplyDamageToActor

**Rejected because:**
- ❌ Mirrors damage pattern = should mirror architecture
- ❌ Function becomes bloated
- ❌ Hard to expand (different flinch modifiers)

**Chosen approach:**
- ✅ Same pattern as CalculateDamageReduction
- ✅ Single responsibility
- ✅ Expandable (armor could affect flinch too)
- ✅ Easy to debug

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
- Boss AI (checks player.bIsDead to stop attacking)
- UI System (future: health bar, damage numbers)
- Audio System (hit sounds integrated)

---

## Code References

### Key Files

**Blueprints:**
- `BP_ThirdPersonCharacter` - All damage/flinch system logic
- `ABP_Unarmed` - Death animation state
- `BP_EnemyBase` - DeliverHitboxDamage, calls ApplyDamageToActor

**Interfaces:**
- `BPI_Damageable` - Damage + FlinchLevel application contract

**Montages:**
- `AM_BlockFlinch` - Block impact animation
- `AM_Flinch_Light` - Light hit reaction

**Data Tables:**
- `DT_GothicKnightAttacks` - Includes FlinchLevel per attack

---

### Key Functions

**BP_ThirdPersonCharacter:**
- `CalculateDamageReduction(IncomingDamage)` → Returns FinalDamage
- `CalculateFlinchReduction(IncomingFlinchLevel)` → Returns FinalFlinchLevel
- `ApplyDamageToActor(Damage, FlinchLevel)` → Interface implementation
- `HandlePlayerDeath()` → Death sequence
- `HandlePlayerHit(FinalDamage, FinalFlinchLevel)` → Hit feedback

**BP_EnemyBase:**
- `DeliverHitboxDamage(OtherActor, HitComponent)` → Mesh check + damage call

**ABP_Unarmed:**
- Event Blueprint Update Animation → Copies bIsDead from character

---

### Key Variables

**BP_ThirdPersonCharacter:**
- `MaxHealth` (Float) - Health pool
- `CurrentHealth` (Float) - Current HP
- `bIsDead` (Boolean) - Death state
- `bIsInvulnerable` (Boolean) - I-frame state
- `bIsBlocking` (Boolean) - Block state (from BlockSystem)

**BP_EnemyBase:**
- `AlreadyHitActors` (Array of Actor References) - Prevents multi-hit per swing

**ABP_Unarmed:**
- `bIsDead` (Boolean) - Triggers death animation

---

## Architectural Principles Demonstrated

**Single Responsibility:**
- CalculateDamageReduction = damage mitigation only
- CalculateFlinchReduction = flinch mitigation only
- ApplyDamageToActor = health modification + routing
- HandlePlayerHit = feedback only
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
- Damage application ≠ Death sequence ≠ Hit feedback
- Clear functional boundaries

**Data-Driven:**
- FlinchLevel in attack data table
- Per-attack tuning without code changes
- Ready for expansion

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
- ✅ Flinch animations play

---

## Design Philosophy

**This system embodies:**

✅ **Player Agency** - Mitigation is player choice (block vs dodge vs take hit)  
✅ **Clear Feedback** - Sounds and animations show exactly what happened  
✅ **Expandable Foundation** - Easy to add armor, buffs, perfect blocks, more flinch levels  
✅ **Clean Architecture** - Separation of concerns, single responsibility  
✅ **Defensive Options** - Multiple viable strategies (not just "dodge everything")  
✅ **Fair Hit Detection** - Mesh-based = visually accurate

**Inspired by:**
- **Dark Souls** - Health as resource, death as consequence
- **Monster Hunter** - Block vs dodge as meaningful choice, flinch levels
- **God of War 2018** - Intimate combat with stakes, death matters

---

*Last Updated: 27.11.2025*  
*Status: Complete with flinch system and mesh-based hit detection*  
*Next: Medium/Heavy flinch → Directional flinch → Health UI*
