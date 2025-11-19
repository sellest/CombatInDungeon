# Block System Documentation

**Status:** ✅ Complete  
**Session:** 18-20.11.2025  
**Purpose:** Defensive stance with directional locomotion and hold-state input handling

---

## Overview

The blocking system allows the player to raise their shield and move slowly in all directions while maintaining a defensive posture. Movement animations adapt to both camera state (locked/unlocked) and input direction. The system uses hold-state tracking to enable responsive blocking even during attack animations.

---

## Core Mechanics

**Input:** R2/RT (IA_Block) - HOLD input

- **Press & Hold:** Raise shield, enter blocking stance, can move
- **Release:** Lower shield, return to normal locomotion
- **During Attacks:** Holding R2 during attack will activate block as soon as ANS_LockDefense allows

**Movement while blocking:**

- Max Walk Speed reduced from 200 → 100
- Full 8-directional movement (F, B, L, R, FL, FR, BL, BR)
- Camera-aware: Unlocked always shows forward animation

**Defense:** (Not yet implemented)

- Damage reduction: TBD
- Stamina cost: TBD
- Block breaking: TBD

---

## Implementation

### Variables (BP_ThirdPersonCharacter)

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bIsBlocking` | Boolean | False | Currently in blocking state |
| `bCanBlock` | Boolean | True | Can block? (controlled by ANS_LockDefense) |
| `bBlockPressed` | Boolean | False | Is player holding R2 right now? |

**State Tracking:**

- `bIsBlocking`: Internal state (actually blocking)
- `bCanBlock`: External lock (attack animations control this)
- `bBlockPressed`: Input tracking (for hold-state detection)

---

### Input Handling

#### IA_Block Started (R2 Press)

```
IA_Block Started:
    ↓
Set bBlockPressed = True
    ↓
Branch: bCanBlock?
    ├─ True → ExecuteBlock (function)
    └─ False → EXIT (do nothing, but bBlockPressed stays True!)
```

**Key behavior:** When locked by attack, input is "remembered" via bBlockPressed staying True.

---

#### IA_Block Completed (R2 Release)

```
IA_Block Completed:
    ↓
Set bBlockPressed = False
    ↓
Set bIsBlocking = False
    ↓
Get CharacterMovement → Set Max Walk Speed = 200
    ↓
Print: "block released"
```

---

### ExecuteBlock Function

**Location:** BP_ThirdPersonCharacter → Functions

**Purpose:** Centralized block activation logic (clears combat state)

**Implementation:**

```
ExecuteBlock():
    ↓
Set CurrentAttackName = None (clear combo state)
    ↓
Set bCanCombo = False (prevent attack buffering)
    ↓
Set bIsBlocking = True
    ↓
Get Mesh → Get Anim Instance → Montage Stop (0.15 blend)
    ↓
Get CharacterMovement → Set Max Walk Speed = 100
    ↓
Print: "blocking"
```

**Why this exists:** Same pattern as ExecuteDodge - clears all combat state to prevent stuck states.

---

### ANS_LockDefense Integration

**Location:** ANS_LockDefense AnimNotifyState

**Purpose:** Placed in attack montages to prevent defensive actions during commitment windows.

#### Received_NotifyBegin

```
Cast to BP_ThirdPersonCharacter
    ↓
Set bCanDodge = False
    ↓
Set bCanBlock = False
```

#### Received_NotifyEnd

```
Cast to BP_ThirdPersonCharacter
    ↓
Set bCanDodge = True
    ↓
Set bCanBlock = True
    ↓
Branch: bBlockPressed? (is player HOLDING R2 right now?)
    ├─ True → ExecuteBlock (activate buffered block)
    └─ False → Do nothing
```

**Key Innovation:** Checks `bBlockPressed` (hold state) when lock ends, not a one-shot buffer variable.

---

### Hold-State Input Pattern

**How it works:**

**Scenario 1: Hold Block During Attack**

```
Start attack → ANS_LockDefense active (bCanBlock = False)
    ↓
Press & HOLD R2:
    ├─ IA_Block Started fires
    ├─ Set bBlockPressed = True
    ├─ Branch: bCanBlock? → False → EXIT
    ├─ (But bBlockPressed stays True because still holding!)
    ↓
ANS_LockDefense ends:
    ├─ Set bCanBlock = True
    ├─ Check: bBlockPressed? → True (still holding!)
    └─ ExecuteBlock → Block activates automatically!
```

**Scenario 2: Quick Tap During Attack**

```
Attack → ANS_LockDefense active
    ↓
Press R2 briefly:
    ├─ Started → bBlockPressed = True
    ├─ Can't block (locked)
    ├─ Completed → bBlockPressed = False (released)
    ↓
ANS ends:
    ├─ Check: bBlockPressed? → False
    └─ Do nothing (correct - they released button)
```

**Why not a buffer variable?** Block is a HOLD action, not a press-and-go action like dodge. We check if player is CURRENTLY holding, not if they pressed recently.

---

### Variables (ABP_Unarmed)

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bIsBlocking` | Boolean | False | Local copy from character |
| `LastMoveForward` | Float | 0.0 | Forward/backward input (-1 to 1) |
| `LastMoveRight` | Float | 0.0 | Left/right input (-1 to 1) |
| `bIsLockedOn` | Boolean | False | Camera lock state |
| `Direction` | Float | 0.0 | Calculated angle for blend spaces (-180° to 180°) |
| `GroundSpeed` | Float | 0.0 | Current movement speed |
| `bShouldMove` | Boolean | False | Is character moving |

**Updated in Event Blueprint Update Animation:**

- Copies from character every frame via Sequence node (Then 3)
- Clears stale input (LastMoveForward/Right set to 0.0 if older than 0.15s)

---

## Animation State Machine

### Structure

```
Locomotion (State Machine)
├─ Idle
├─ Walk/Run
└─ Blocking (State) ← Contains nested BlockSequence
    └─ BlockSequence (Nested State Machine)
        ├─ Entry → GuardBegin
        ├─ GuardBegin (raise shield)
        ├─ GuardIdle (blocking locomotion)
        └─ GuardEnd (lower shield)
```

---

### BlockSequence States

#### GuardBegin

**Animation:** `Guard_Begin_Seq`
**Duration:** ~0.5-1.0 seconds
**Purpose:** Shield raise animation

**Transition Out:** GuardBegin → GuardIdle

- **Condition:** Time Remaining < 0.3
- **Rationale:** Transition early to prevent leg freeze during movement

---

#### GuardIdle (Main Locomotion State)

**Animation:** `UE5_SSH_Guard_Jog_Forward_BlendSpace` (professional blend space)
**Looping:** Yes (blend space handles idle + all movement speeds)
**Purpose:** Main blocking locomotion state

**Blend Space Details:**

- **Type:** 2D Blend Space
- **Source:** Animation pack ($25)
- **X Axis (Degree):** -180° to 180° (directional angle)
- **Y Axis (Speed):** 0 to 800 (velocity: idle → walk → jog → run)
- **Coverage:** Full 360° movement, all speeds

**Blend Space Inputs:**

**X Input (Strafe/Direction):**

```
Select Float:
├─ Pick A (bool): bIsLockedOn
├─ A (True): LastMoveRight (use actual strafe when locked)
└─ B (False): 0.0 (no strafe when unlocked)
    ↓
Direction variable (calculated angle)
    ↓
BS_GuardWalk X input (Degree)
```

**Y Input (Forward/Back Speed):**

```
OUTER Select Float:
├─ Pick A: bIsLockedOn
├─ A (True): LastMoveForward (use actual input when locked)
└─ B (False): INNER Select Float
                ├─ Pick A: bShouldMove
                ├─ A (True): 1.0 (force forward when moving)
                └─ B (False): 0.0 (idle when stationary)
    ↓
GroundSpeed variable
    ↓
BS_GuardWalk Y input (Speed)
```

**Why Nested Selects for Y Input:**

When camera unlocked, character rotates to face movement direction (always moving "forward" relative to self). To prevent moonwalking:

- If moving: Force forward animation (Y = 1.0)
- If idle: Show idle animation (Y = 0.0)

When camera locked, use actual directional input for proper strafing.

**Camera Context Logic:**

- **Unlocked camera:** Character rotates to face movement, always plays forward animation (or idle if stationary)
- **Locked camera:** Character faces target, uses proper directional animations for tactical positioning

**Transition Out:** GuardIdle → GuardEnd

- **Condition:** NOT bIsBlocking (button released)

---

#### GuardEnd

**Animation:** `Guard_End_Seq`
**Duration:** ~0.5-1.0 seconds
**Purpose:** Shield lower animation

**No transitions out** - parent state machine handles exit to Idle/Walk/Run

---

### Parent State Machine Transitions

#### Idle/Walk → Blocking

**Condition:** bIsBlocking = True
**Result:** Enters BlockSequence at Entry → GuardBegin

#### Blocking → Idle

**Condition:** NOT bIsBlocking
**Result:** Exits after GuardEnd completes, returns to Idle

#### Blocking → Walk/Run

**Condition:** NOT bIsBlocking AND bShouldMove
**Result:** Exits after GuardEnd, transitions to Walk/Run if moving

---

## Blend Space Configuration

### UE5_SSH_Guard_Jog_Forward_BlendSpace

**Sample Smoothing:**

- **Weight Speed:** 0.5
- **Purpose:** Smooth transitions between directional animations
- **Result:** No snapping when changing direction (e.g., left → right)

**Note:** Despite being named "Forward," this blend space handles ALL directions (forward, backward, strafe, diagonals) across full speed range (idle → walk → jog → run). The animation pack's naming is misleading - it's actually omnidirectional.

---

## Animation Assets Used

### Currently In Use

- ✅ `Guard_Begin_Seq` - Shield raise
- ✅ `UE5_SSH_Guard_Jog_Forward_BlendSpace` - Main locomotion (handles all directions/speeds)
- ✅ `Guard_End_Seq` - Shield lower

### Available But Deferred

- `UE5_SSH_Guard_Jog_Start_Forward_Blendspace` - Start transitions
- `UE5_SSH_Guard_Jog_Start_Backward_Blendspace`
- `UE5_SSH_Guard_Jog_Stop_Forward_BlendSpace` - Stop transitions
- `UE5_SSH_Guard_Jog_Stop_Backward_BlendSpace`

**Decision:** Start/Stop blend spaces add complexity (variable animation lengths break time-based transitions). Current system with static begin/end animations works reliably. Defer to polish phase.

---

## Design Decisions

### Why Speed = 100?

- Tested: 200 (too fast, sliding), 100 (feels good), 50 (too slow)
- **100 matches guard walk animation speed** with minimal sliding
- Clear visual/gameplay distinction from normal movement (200)

### Why Hold-State Tracking Instead of Input Buffering?

**Block is fundamentally different from dodge:**

**Dodge:** Press-and-go action

- Single button press → Plays animation → Done
- Input buffering: "Remember they pressed, execute later"

**Block:** Continuous hold state

- Hold button → Maintain state → Release to exit
- Hold-state tracking: "Are they CURRENTLY holding?"

**Implementation difference:**

- Buffer: Stores "did press" flag, clears after execution or timeout
- Hold-state: Tracks "is pressing" continuously, reflects actual button state

**Why it matters:**

- Player holds R2 during attack → Intent is clear (wants to block when possible)
- Player taps R2 during attack then releases → No intent to block anymore
- Hold-state naturally handles both cases without timeout logic

### Why ExecuteBlock Function?

**Same pattern as ExecuteDodge:**

- Centralizes state clearing logic
- Prevents "stuck" states when canceling actions
- Reusable if block needs to be triggered from other sources (e.g., auto-block on perfect timing?)

**What it clears:**

- `CurrentAttackName = None` (prevents combo system thinking we're mid-attack)
- `bCanCombo = False` (prevents attack input buffering during block)
- Stops all montages (cancels active attack)

**Without this:** Blocking mid-attack leaves combat state dirty → Can't attack after blocking → Player feels stuck

### Why Camera-Aware Animation Logic?

**Unlocked camera:**

- Character rotates to face movement direction
- Always moving "forward" relative to self
- Must play forward animation or looks like moonwalking

**Locked camera:**

- Character faces target, strafes
- Directional animations match actual movement
- Proper tactical positioning feedback

**Solution:** Conditional input to blend space based on lock-on state using nested Select Float nodes

### Why Allow Movement During Block?

- Monster Hunter inspiration: Can reposition while defending
- Tactical depth: Position vs damage mitigation tradeoff
- Speed penalty (50% slower) makes it balanced
- Future: Stamina cost while moving adds additional cost

### Why NOT Use Start/Stop Blend Spaces?

- Added complexity to state machine transitions
- Variable animation lengths break simple time-based transitions
- Current system works reliably
- **80/20 rule:** 20% visual improvement for 80% complexity
- Can revisit during polish phase

---

## Known Issues

### None Currently

All issues from development resolved:

- ✅ ~~Slight foot sliding at speed 100~~ - Acceptable for prototype
- ✅ ~~Brief leg freeze when raising shield while moving~~ - Fixed with early transition (< 0.3s)
- ✅ ~~Direction snapping when changing direction~~ - Fixed with Weight Speed = 0.5
- ✅ ~~Button mashing breaks state machine~~ - Fixed by removing bBlockExitComplete complexity
- ✅ ~~Stale input causes wrong animations~~ - Fixed by clearing old LastMove values
- ✅ ~~Wrong animations when camera unlocked~~ - Fixed with Select Float logic
- ✅ ~~Can't block during attacks~~ - Fixed with bCanBlock + ANS_LockDefense
- ✅ ~~Stuck state after blocking during attack~~ - Fixed with ExecuteBlock clearing combat state
- ✅ ~~Must press R2 twice (button mashing)~~ - Fixed with hold-state tracking (bBlockPressed)

---

## Integration Points

### With Combat System

**Current:**

- Block can interrupt attacks (when ANS_LockDefense allows)
- ExecuteBlock clears CurrentAttackName and bCanCombo
- Returns to neutral state cleanly

**Future:**

- Block during combo? (cancel attack into block?) ← Already works
- Block after dodge? (defensive flow) ← Already works
- Perfect block timing? (parry mechanic)

### With Damage System (Not Yet Implemented)

**Planned:**

```
When taking damage:
├─ Branch: bIsBlocking?
│   ├─ True → Apply damage * 0.2-0.5 (reduced)
│   │         Play shield impact VFX/SFX
│   │         Small knockback/stagger
│   │         Cost stamina?
│   └─ False → Apply full damage
```

### With Stamina System (Not Yet Implemented)

**Planned:**

- Holding block: Drain stamina slowly (1-2/sec)
- Moving while blocking: Drain stamina faster (3-4/sec)
- Taking hit while blocking: Drain stamina chunk (10-20)
- Stamina depleted: Block breaks, forced out of stance

### With ANS_LockDefense System

**Tight integration:**

- ANS_LockDefense sets bCanBlock = False during attack commitment windows
- ANS_LockDefense checks bBlockPressed when ending (enables hold-during-attack)
- Per-attack control: Long ANS = committed, short ANS = cancelable

**Attack designers can tune commitment per attack:**

- Heavy attacks: Long ANS_LockDefense (can't block until late)
- Light attacks: Short ANS_LockDefense (can block early)
- Special moves: No ANS_LockDefense (freely cancelable)

---

## Performance Notes

**Cost:** Negligible

**Blend Space:** 2D lookup, interpolates 4 animations max  
**State Machine:** Simple boolean checks, no complex logic  
**Select Nodes:** 3-4 float comparisons per frame  
**Hold-State Tracking:** Single boolean update on input events

**Scales well:** System works identically for player or 100 AI-controlled characters

---

## Future Enhancements

### Short Term (After Damage System)

- [ ] Damage reduction implementation (0.2-0.5x multiplier)
- [ ] Shield impact VFX (sparks, flash)
- [ ] Shield impact SFX (metal clang, varying pitch/volume)
- [ ] Block pushback/stagger (small knockback on blocked hit)

### Medium Term (After Stamina System)

- [ ] Stamina drain while holding block
- [ ] Increased drain when moving while blocking
- [ ] Stamina drain on blocked hit
- [ ] Block break when stamina depleted
- [ ] Visual feedback (stamina bar, shield crack VFX)

### Long Term (Polish Phase)

- [ ] Integrate Start/Stop blend spaces (smooth speed transitions)
- [ ] Perfect block timing (parry window - tight timing, full negate + counter opportunity)
- [ ] Directional block effectiveness (front = full, side = partial, back = none)
- [ ] Guard Break state (heavy attacks break through block)
- [ ] Different block animations per weapon type (if/when adding new weapons)
- [ ] Block durability system (shield takes "poise damage", breaks after threshold)

---

## Testing Checklist

**Basic Functionality:**

- ✅ R2 raises shield (GuardBegin plays)
- ✅ Holding R2 maintains blocking stance
- ✅ Can move in all 8 directions while blocking
- ✅ Movement speed reduced to 100
- ✅ Releasing R2 lowers shield (GuardEnd plays)
- ✅ Returns to normal locomotion after block

**Camera State Tests:**

- ✅ Unlocked: Moving any direction plays forward animation (or idle if stationary)
- ✅ Locked: Moving shows correct directional animations (F/B/L/R/BL/BR)
- ✅ Switching lock-on mid-block transitions animations smoothly

**Attack Integration Tests:**

- ✅ Can block during neutral (not attacking)
- ✅ Holding R2 during attack (before ANS_LockDefense ends) = no immediate block
- ✅ Still holding R2 when ANS_LockDefense ends = block activates automatically
- ✅ Quick tap R2 during attack then release = no block (correct)
- ✅ After blocking mid-attack, can attack again (not stuck)

**Edge Cases:**

- ✅ Button mashing doesn't break state machine
- ✅ Blocking while moving shows appropriate animation
- ✅ Blocking while stationary shows idle
- ✅ No animation snapping when changing direction
- ✅ Stale movement input doesn't cause wrong block animations

---

## Design Philosophy

**This system embodies:**

✅ **Tactical Defense** - Speed penalty encourages thoughtful blocking, not spam  
✅ **Responsive Input** - Hold-state tracking feels natural, no button mashing  
✅ **Camera Context** - Animations adapt to player's perspective  
✅ **Professional Quality** - Leverages $25 animation pack effectively  
✅ **Clean State Management** - ExecuteBlock prevents stuck states  
✅ **Prototype First** - Simple, working system over complex polish  
✅ **Scalable Foundation** - Easy to add damage reduction, stamina, parry later

**Inspired by:**

- Monster Hunter (defensive repositioning, hold-based blocking)
- Dark Souls (stamina cost, commitment)
- God of War (camera-aware combat)

---

## Related Systems

**Upstream (triggers this system):**

- Input System (IA_Block hold/release)
- Movement System (provides directional input)
- Lock-On System (provides camera state)
- ANS_LockDefense (controls bCanBlock)

**Downstream (called by this system):**

- CharacterMovement (Max Walk Speed changes)
- Animation System (plays block sequences, blend space)
- Combat System (ExecuteBlock clears CurrentAttackName, bCanCombo)

**Parallel Systems (will integrate):**

- Damage System (block reduces damage)
- Stamina System (block costs stamina)
- Attack System (ANS_LockDefense integration complete)
- Dodge System (both use defensive lock, similar input patterns)

---

## Code References

### Key Files

- `BP_ThirdPersonCharacter` - Main character blueprint (input, ExecuteBlock function)
- `ABP_Unarmed` - Animation blueprint (state machine, blend space logic)
- `ANS_LockDefense` - AnimNotifyState (controls bCanBlock, checks bBlockPressed)
- `UE5_SSH_Guard_Jog_Forward_BlendSpace` - Professional 2D blend space
- Attack Montages - Have ANS_LockDefense placed for commitment windows

### Key Functions

- `ExecuteBlock()` - Centralized block activation (clears combat state)
- Event Blueprint Update Animation - Copies state from character, clears stale input

### Key Variables

**BP_ThirdPersonCharacter:**

- `bIsBlocking` - Internal blocking state
- `bCanBlock` - External lock (from ANS)
- `bBlockPressed` - Hold-state tracking

**ABP_Unarmed:**

- `Direction` - Calculated angle for blend spaces
- `GroundSpeed` - Movement speed
- `bIsLockedOn` - Camera lock state
- `LastMoveForward`, `LastMoveRight` - Directional input

---

*Documented: 20.11.2025*  
*Status: Complete and tested*  
*Next: Enemy AI, then damage/block interaction*
