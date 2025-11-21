# Player Damage System Documentation

**Status:** ✅ Complete  
**Session:** 21.11.2025  
**Purpose:** Player health, damage mitigation, and death mechanics

---

## Overview

The Player Damage System handles all incoming damage to the player character, including mitigation through defensive actions (blocking, dodging), health management, and death state. The system is designed around player-owned defensive state - the boss reports damage, the player determines how much to actually take.

**Design Philosophy:**
- **Player owns mitigation logic** - All defensive state (blocking, i-frames, armor, buffs) checked player-side
- **Boss just reports damage** - Enemy doesn't need to know about player's defensive capabilities
- **Expandable foundation** - Easy to add new mitigation sources (armor, potions, guard points, etc.)
- **Interface-based** - Uses BPI_Damageable for consistent damage application

---

## Architecture

### Separation of Concerns

**Boss Perspective:**
```
"I'm attacking for 10 damage"
→ ApplyDamageToActor(10)
```

**Player Perspective:**
```
"I received 10 damage attack"
→ Check MY state: Blocking? I-frames? Buffed?
→ Apply mitigation
→ Take 3 final damage
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
- `bIsInvulnerable` - Set by ANS_IFrames during dodge, negates all damage

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

### ApplyDamageToActor

**Location:** BP_ThirdPersonCharacter → Functions  
**Interface:** BPI_Damageable  
**Purpose:** Apply final damage to health and trigger appropriate feedback

**Signature:**
- Input: `Damage` (Float) - Raw damage from attacker
- Output: None (interface function)

**Implementation:**

```
ApplyDamageToActor (Damage):
    ↓
    Branch: bIsDead?
    ├─ True → Return (ignore damage when dead)
    │
    └─ False → CalculateDamageReduction (Damage)
               → Returns FinalDamage
               ↓
               CurrentHealth - FinalDamage = NewHealth
               Set CurrentHealth = NewHealth
               ↓
               Branch: CurrentHealth <= 0?
               ├─ True → Set CurrentHealth = 0 (clamp)
               │         HandlePlayerDeath (function call)
               │
               └─ False → HandlePlayerHit (function call)
```

**Why check bIsDead first:**
- Prevents damage processing on corpse
- Boss might still be attacking when player dies
- Clean exit for all damage sources

**Why clamp to 0:**
- Prevents negative health values
- Clean for UI display
- Consistent death threshold

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

### HandlePlayerHit

**Location:** BP_ThirdPersonCharacter → Functions  
**Type:** Function (placeholder)  
**Purpose:** Non-lethal damage feedback

**Current Implementation:**
```
HandlePlayerHit:
    ↓
    Print String: "Player Hit! Health: {CurrentHealth}"
```

**Future Enhancements:**
```
HandlePlayerHit:
    ↓
    Branch: FinalDamage > 0?
    └─ True → Play hit sound
              Trigger camera shake (small)
              Flash screen red briefly
              Play hit reaction animation (if not blocking/dodging)
```

**Why placeholder:**
- Core feedback works (health decreases)
- Polish layer can be added incrementally
- Doesn't block core system functionality

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
Boss attacks (10 damage)
    ↓
Player.ApplyDamageToActor(10)
    ↓
CalculateDamageReduction checks bIsBlocking
    ├─ True → 10 * 0.3 = 3 damage
    └─ False → 10 damage
    ↓
CurrentHealth - FinalDamage
```

**Block system provides:**
- `bIsBlocking` flag (set by R2/RT hold)
- Already implemented in BlockSystem.md
- No changes needed to block logic

**Damage reduction:**
- **70% reduction** (multiply by 0.3)
- Can be tuned: Change multiplier in CalculateDamageReduction
- Future: Different reduction per block type (shield vs parry vs deflect)

---

### I-Frames Integration (Dodge)

**How it works:**
```
Boss attacks (10 damage)
    ↓
Player.ApplyDamageToActor(10)
    ↓
CalculateDamageReduction checks bIsInvulnerable
    ├─ True → 0 damage (100% negation)
    └─ False → Continue to blocking check
```

**I-frame system provides:**
- `bIsInvulnerable` flag (set by ANS_IFrames during dodge)
- Controlled via AnimNotifyState in dodge montages
- Duration: 0.4s (God of War style - 50% of dodge)

**Why i-frames check first:**
- I-frames = complete immunity
- Overrides all other mitigation
- Dodge while blocking = i-frames take priority

**See DodgeSystem.md for i-frame implementation details.**

---

### Priority Order (Critical)

**Mitigation checks in CalculateDamageReduction:**

1. **bIsInvulnerable** (I-frames) - 100% negation
2. **bIsBlocking** (Block) - 70% reduction
3. **Future systems** (armor, buffs, guard points)

**Why this order matters:**
- Player dodging while blocking = i-frames win (full immunity)
- Prevents weird interactions (blocked dodge? dodged block?)
- Clear hierarchy: Invulnerable > Reduced > Normal

---

## Boss Integration

**Boss-side implementation:**

In BP_EnemyBase → AttackHitbox overlap event:

```
On Component Begin Overlap:
    ↓
    Cast to BP_ThirdPersonCharacter
    ↓
    Get CurrentAttackData.Damage (10.0)
    ↓
    ApplyDamageToActor (Target: Player, Damage: 10.0)
```

**Boss doesn't know:**
- ❌ If player is blocking
- ❌ If player has i-frames
- ❌ If player has armor/buffs
- ❌ Player's final damage taken

**Boss only knows:**
- ✅ "I hit with my attack"
- ✅ "My attack deals 10 damage"

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
1. Boss attacks (10 damage)
2. Player stands still (no block, no dodge)
3. Expected:
   - Print: "Hit! 10 damage"
   - Health: 90 / 100

**Test 2: Blocked Damage**
1. Boss attacks (10 damage)
2. Player holds R2 (blocking)
3. Expected:
   - Print: "Blocked! 10 → 3"
   - Health: 97 / 100

**Test 3: I-Frame Dodge**
1. Boss attacks (10 damage)
2. Player dodges during i-frame window (0.1s - 0.5s)
3. Expected:
   - Print: "Dodged! No damage received"
   - Health: unchanged (100 / 100)

**Test 4: Death**
1. Take 10 hits without blocking (100 HP / 10 damage)
2. Expected:
   - Print: "I'm dead :'("
   - Death animation plays
   - Input disabled
   - Boss stops attacking

---

### Edge Cases

**Test 5: Damage While Dead**
1. Player at 0 HP (dead)
2. Boss attacks again
3. Expected:
   - No damage processing (early exit in ApplyDamageToActor)
   - No prints, no health change

**Test 6: Overkill Damage**
1. Player at 5 HP remaining
2. Boss deals 50 damage
3. Expected:
   - Health clamped to 0 (not -45)
   - Death triggers normally

**Test 7: Dodge + Block Simultaneously**
1. Player dodging while holding R2
2. Boss attacks during i-frame window
3. Expected:
   - I-frames take priority
   - 0 damage taken (not reduced damage)

---

## Performance Considerations

**Cost:** Negligible (< 0.01ms per damage event)

**CalculateDamageReduction:**
- 2-3 boolean checks (extremely fast)
- 1-2 float multiplications
- No loops, no arrays
- Runs only when hit (not every frame)

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

**No Hit Stun:**
- Player can attack while taking damage
- No flinch/stagger animation yet
- Deferred to polish phase

**No Visual Feedback:**
- No screen flash, camera shake, or hit VFX
- Debug prints only
- Placeholder for polish

---

## Future Enhancements

### High Priority

0. **Block Hit box**
    - Shield should be raised towards attack
    - Damage outside of the block-hitbox should not be affected by the blocking state

1. **Player Flinch Animation** (20 min)
   - Add Flinch state to ABP_Unarmed
   - Trigger on heavy hits (> 20 damage?)
   - Brief interrupt (0.5s)

2. **Hit Feedback VFX/SFX** (30 min)
   - Screen flash (red overlay, 0.1s fade)
   - Camera shake (small on hit, large on death)
   - Hit sound (grunt, pain sound)
   - Blood spray particles (optional)

3. **Health Bar UI** (1 hour)
   - Widget Blueprint: Health bar
   - Update on damage
   - Flash/pulse on low health

---

### Medium Priority

4. **Armor System** (1-2 hours)
   - Add ArmorValue variable (0.0 to 1.0)
   - Integrate in CalculateDamageReduction
   - Equipment that modifies armor
   - Visual feedback (sparks on armored hits)

5. **Perfect Block** (1 hour)
   - Timing window at block start (0.2s)
   - 100% damage negation on perfect block
   - Unique feedback (parry flash, sound ding)
   - Possible: Counter-attack window

6. **Guard Points** (2 hours)
   - Specific frames in attack animations have defensive properties
   - ANS_GuardPoint sets bGuardPointActive
   - Reduces damage during specific attack frames
   - Monster Hunter-style active defense

7. **Stamina System Integration** (2-3 hours)
   - Block costs stamina per hit blocked
   - Dodge costs stamina to execute
   - Out of stamina = can't block/dodge
   - Stamina regeneration system

---

### Polish Phase

8. **Damage Numbers** (30 min)
   - Floating damage text on hit
   - Color-coded: Red (full), Yellow (reduced), Blue (blocked)

9. **Directional Hit Reactions** (1 hour)
   - Different flinch animations based on hit direction
   - Blend space for smooth directional flinches

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
- ✅ Boss just reports raw damage
- ✅ Easy to expand player abilities without touching boss
- ✅ Clean separation of concerns

---

### Why Separate CalculateDamageReduction?

**Alternative considered:** All mitigation logic inline in ApplyDamageToActor

**Rejected because:**
- ❌ Function becomes bloated (50+ nodes)
- ❌ Hard to debug (where's the bug?)
- ❌ Not reusable (fall damage needs different application logic)

**Chosen approach:**
- ✅ Mitigation = separate concern from application
- ✅ Reusable from multiple damage sources
- ✅ Easy to unit test (pass in damage, check output)
- ✅ Expandable without modifying application logic

---

### Why HandlePlayerDeath Function?

**Alternative considered:** Death logic inline in ApplyDamageToActor

**Rejected because:**
- ❌ Not reusable (instant-kill hazards need same death sequence)
- ❌ Mixed responsibility (damage calc + death effects)
- ❌ Hard to expand (where do I add death camera?)

**Chosen approach:**
- ✅ Death sequence is separate concern
- ✅ Reusable from any death source
- ✅ Easy to expand (VFX, SFX, UI, camera)
- ✅ Single Responsibility Principle

---

### Why I-Frames Over Invincibility Timer?

**Alternative considered:** Dodge sets invincible for X seconds

**Rejected because:**
- ❌ Doesn't sync with animation
- ❌ Hard to tune per dodge type
- ❌ Animation speed changes break timing

**Chosen approach (ANS_IFrames):**
- ✅ Synced to animation frames (designer-friendly)
- ✅ Visual in timeline (easy to tune)
- ✅ Per-montage control (forward dodge ≠ backward dodge)
- ✅ Same pattern as other systems (perfect timing, hitboxes)

---

## Related Systems

**Upstream (triggers damage):**
- Enemy Attack System (boss hitbox overlaps)
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
- Audio System (future: hit sounds, death scream)

---

## Code References

### Key Files

**Blueprints:**
- `BP_ThirdPersonCharacter` - All damage system logic
- `ABP_Unarmed` - Death animation state
- `BP_EnemyBase` - Calls ApplyDamageToActor on player

**Interfaces:**
- `BPI_Damageable` - Damage application contract

**Data Tables:**
- (None currently - future: damage type tables)

---

### Key Functions

**BP_ThirdPersonCharacter:**
- `CalculateDamageReduction(IncomingDamage)` → Returns FinalDamage
- `ApplyDamageToActor(Damage)` → Interface implementation
- `HandlePlayerDeath()` → Death sequence
- `HandlePlayerHit()` → Hit feedback (placeholder)

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

**ABP_Unarmed:**
- `bIsDead` (Boolean) - Triggers death animation

---

## Architectural Principles Demonstrated

**Single Responsibility:**
- CalculateDamageReduction = mitigation only
- ApplyDamageToActor = health modification only
- HandlePlayerDeath = death sequence only

**Encapsulation:**
- Player owns all defensive state
- Boss doesn't know about mitigation
- Clean interface boundary

**Open/Closed Principle:**
- Easy to add mitigation sources (armor, buffs)
- No modification to core damage flow
- Extends via CalculateDamageReduction

**Separation of Concerns:**
- Damage calculation ≠ Damage application
- Damage application ≠ Death sequence
- Clear functional boundaries

**Data-Driven (Future):**
- Ready for damage type tables
- Ready for resistance tables
- Ready for armor stat systems

---

## Session Notes

**Session 21.11.2025:**
- Duration: ~3 hours
- Major Systems: Complete player damage system
- Functions Created: 3 (CalculateDamageReduction, HandlePlayerDeath, HandlePlayerHit)
- Variables Added: 4 (MaxHealth, CurrentHealth, bIsDead, bIsInvulnerable)
- AnimBP States: 1 (Death state in ABP_Unarmed)
- Integration Points: 2 (Block system, I-frames system)
- Lines of Blueprint: ~200-300 nodes
- Bugs Fixed: 1 (input disable target pin)

**Validation:**
- ✅ Player takes damage from boss
- ✅ Block reduces damage (70%)
- ✅ Dodge i-frames negate damage (100%)
- ✅ Death at 0 HP works
- ✅ Death animation plays
- ✅ Input disabled on death
- ✅ Boss stops attacking dead player

---

## Design Philosophy

**This system embodies:**

✅ **Player Agency** - Mitigation is player choice (block vs dodge vs take hit)  
✅ **Clear Feedback** - Prints show exactly what happened  
✅ **Expandable Foundation** - Easy to add armor, buffs, perfect blocks  
✅ **Clean Architecture** - Separation of concerns, single responsibility  
✅ **Defensive Options** - Multiple viable strategies (not just "dodge everything")

**Inspired by:**
- **Dark Souls** - Health as resource, death as consequence
- **Monster Hunter** - Block vs dodge as meaningful choice
- **God of War 2018** - Intimate combat with stakes, death matters

---

*Last Updated: 21.11.2025*  
*Status: Complete and tested*  
*Next: Player flinch animation → Hit feedback VFX/SFX → Health UI*
