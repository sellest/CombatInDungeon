# Block System Documentation

**Status:** ✅ Complete (Prototype)  
**Session:** 18-19.11.2025  
**Purpose:** Defensive stance with directional locomotion while blocking

---

## Overview

The blocking system allows the player to raise their shield and move slowly in all directions while maintaining a defensive posture. Movement animations adapt to both camera state (locked/unlocked) and input direction.

---

## Core Mechanics

**Input:** R2/RT (IA_Block)
- **Hold:** Raise shield, enter blocking stance, can move
- **Release:** Lower shield, return to normal locomotion

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
| `bIsBlocking` | Boolean | False | Currently holding block input |

**Updated by:**
- IA_Block Started → True
- IA_Block Completed → False

**Side effects:**
- Sets Max Walk Speed = 100 when True
- Sets Max Walk Speed = 200 when False

---

### Variables (ABP_Unarmed)

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bIsBlocking` | Boolean | False | Local copy from character |
| `LastMoveForward` | Float | 0.0 | Forward/backward input (-1 to 1) |
| `LastMoveRight` | Float | 0.0 | Left/right input (-1 to 1) |
| `bIsLockedOn` | Boolean | False | Camera lock state |
| `Direction` | Float | 0.0 | Calculated angle for blend spaces |
| `GroundSpeed` | Float | 0.0 | Current movement speed |
| `bShouldMove` | Boolean | False | Is character moving |

**Updated in Event Blueprint Update Animation:**
- Copies from character every frame via Sequence node

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

#### GuardIdle
**Animation:** `UE5_SSH_Guard_Jog_Forward_BlendSpace` (professional blend space)
**Looping:** Yes
**Purpose:** Main blocking locomotion state

**Blend Space Inputs:**
- **Degree (X):** Direction variable (-180° to 180°)
- **Speed (Y):** GroundSpeed (0 to 800)

**Camera-Aware Input Logic:**
```
X Input (Strafe):
Select Float:
├─ Pick A: bIsLockedOn
├─ A (True): LastMoveRight (use actual strafe)
└─ B (False): 0.0 (no strafe when unlocked)

Y Input (Forward/Back):
OUTER Select Float:
├─ Pick A: bIsLockedOn
├─ A (True): LastMoveForward (use actual input)
└─ B (False): INNER Select Float
                ├─ Pick A: bShouldMove
                ├─ A (True): 1.0 (force forward when moving)
                └─ B (False): 0.0 (idle when stationary)
```

**Why This Logic:**
- **Unlocked camera:** Character rotates to face movement, always plays forward animation
- **Locked camera:** Character faces target, uses proper directional animations

**Transition Out:** GuardIdle → GuardEnd
- **Condition:** NOT bIsBlocking (button released)

---

#### GuardEnd
**Animation:** `Guard_End_Seq`
**Duration:** ~0.5-1.0 seconds
**Purpose:** Shield lower animation

**No transitions out** - parent state machine handles exit

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

## Professional Blend Space Details

### UE5_SSH_Guard_Jog_Forward_BlendSpace

**Type:** 2D Blend Space  
**Source:** Essential_SNS

**Axes:**
- **X (Degree):** -180° to 180° (directional angle)
- **Y (Speed):** 0 to 800 (velocity)

**Coverage:**
- Idle at (0°, 0)
- Forward movement: -90° to +90°
- Backward movement: +90° to +180°, -90° to -180°
- Full speed range: Walk → Jog → Run

**Note:** Somehow handles all directions despite being named "Forward" - works perfectly without needing separate Backward blend space.

---

## Blend Space Settings

**Sample Smoothing:**
- **Weight Speed:** 0.5
- **Purpose:** Smooth transitions between directional animations
- **Result:** No snapping when changing direction (e.g., left → right)

---

## Input Handling

### In BP_ThirdPersonCharacter

**IA_Block Started:**
```
Set bIsBlocking = True
    ↓
Get CharacterMovement
    ↓
Set Max Walk Speed = 100
    ↓
Print: "blocking"
```

**IA_Block Completed:**
```
Set bIsBlocking = False
    ↓
Get CharacterMovement
    ↓
Set Max Walk Speed = 200
    ↓
Print: "block released"
```

---

### Movement Input During Block

**IA_Move event still processes:**
- Updates LastMoveForward, LastMoveRight, LastMoveInputTime
- Applies movement (scaled by Max Walk Speed = 100)
- Blend space receives direction/speed, plays appropriate animation

**Stale Input Handling:**
- Movement input cleared if older than 0.15s
- Prevents old directional values from affecting new block stance
- Handled in Event Tick (BP_ThirdPersonCharacter)

---

## Animation Assets Used

### Currently In Use:
- ✅ `Guard_Begin_Seq` - Shield raise
- ✅ `Guard_Idle_Seq` - Static idle (not used, blend space covers it)
- ✅ `UE5_SSH_Guard_Jog_Forward_BlendSpace` - Main locomotion
- ✅ `Guard_End_Seq` - Shield lower

### Available But Deferred:
- `UE5_SSH_Guard_Jog_Start_Forward_Blendspace` - Start transitions
- `UE5_SSH_Guard_Jog_Start_Backward_Blendspace`
- `UE5_SSH_Guard_Jog_Stop_Forward_BlendSpace` - Stop transitions
- `UE5_SSH_Guard_Jog_Stop_Backward_BlendSpace`

**Decision:** Start/Stop blend spaces add complexity without major benefit for prototype. Defer to polish phase.

---

## Known Issues

### Minor
- ✅ ~~Slight foot sliding at speed 100~~ - Acceptable for prototype, can fine-tune to 90-110 if needed
- ✅ ~~Brief leg freeze when raising shield while moving~~ - Fixed by early transition (< 0.3s)
- ✅ ~~Direction snapping when changing direction~~ - Fixed with Weight Speed = 0.5

### Resolved
- ✅ ~~Button mashing breaks state machine~~ - Fixed by removing bBlockExitComplete, using simple NOT bIsBlocking
- ✅ ~~Stale input causes wrong animations~~ - Fixed by clearing old LastMove values
- ✅ ~~Wrong animations when camera unlocked~~ - Fixed with Select Float logic

---

## Integration Points

### With Combat System
**Current:** Independent - can block during neutral, returns to Idle
**Future:** 
- Block during combo? (cancel attack into block?)
- Block after dodge? (defensive flow)
- Perfect block timing? (parry mechanic)

### With Damage System (Not Yet Implemented)
**Planned:**
```
When taking damage:
├─ Branch: bIsBlocking?
│   ├─ True → Apply damage * 0.2-0.5 (reduced)
│   │         Play shield impact VFX/SFX
│   │         Small knockback/stagger
│   └─ False → Apply full damage
```

### With Stamina System (Not Yet Implemented)
**Planned:**
- Holding block: Drain stamina slowly (1-2/sec)
- Taking hit while blocking: Drain stamina chunk (10-20)
- Stamina depleted: Block breaks, forced out of stance

---

## Design Decisions

### Why Speed = 100?
- Tested: 200 (too fast, sliding), 100 (feels good), 50 (too slow)
- **100 matches guard walk animation speed** with minimal sliding
- Clear visual/gameplay distinction from normal movement (200)

### Why NOT Use Start/Stop Blend Spaces?
- Added complexity to state machine transitions
- Variable animation lengths break simple time-based transitions
- Current system works reliably
- **80/20 rule:** 20% visual improvement for 80% complexity
- Can revisit during polish phase

### Why Camera-Aware Animation Logic?
**Unlocked camera:**
- Character rotates to face movement direction
- Always moving "forward" relative to self
- Must play forward animation or looks like moonwalking

**Locked camera:**
- Character faces target, strafes
- Directional animations match actual movement
- Proper tactical positioning feedback

**Solution:** Conditional input to blend space based on lock-on state

### Why Allow Movement During Block?
- Monster Hunter inspiration: Can reposition while defending
- Tactical depth: Position vs damage mitigation tradeoff
- Speed penalty (50% slower) makes it balanced
- Future: Stamina cost while moving adds additional cost

---

## Performance Notes

**Cost:** Negligible

**Blend Space:** 2D lookup, interpolates 4 animations max  
**State Machine:** Simple boolean checks, no complex logic  
**Select Nodes:** 3-4 float comparisons per frame  

**Scales well:** System works identically for player or 100 AI-controlled characters

---

## Future Enhancements

### Short Term (After Damage System)
- [ ] Damage reduction implementation (0.2-0.5x multiplier)
- [ ] Shield impact VFX (sparks, flash)
- [ ] Shield impact SFX (metal clang)
- [ ] Block pushback/stagger (small)

### Medium Term (After Stamina System)
- [ ] Stamina drain while holding block
- [ ] Stamina drain on blocked hit
- [ ] Block break when stamina depleted
- [ ] Visual feedback (stamina bar, shield crack VFX)

### Long Term (Polish Phase)
- [ ] Integrate Start/Stop blend spaces
- [ ] Perfect block timing (parry window)
- [ ] Directional block effectiveness (front = full, side = partial, back = none)
- [ ] Guard Break state (heavy attack breaks block)
- [ ] Different block animations per weapon type

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
- ✅ Unlocked: Moving any direction plays forward animation
- ✅ Locked: Moving shows correct directional animations (F/B/L/R/BL/BR)
- ✅ Switching lock-on mid-block transitions animations correctly

**Edge Cases:**
- ✅ Button mashing doesn't break state machine
- ✅ Blocking while moving shows appropriate animation
- ✅ Blocking while stationary shows idle
- ✅ No animation snapping when changing direction

---

## Design Philosophy

**This system embodies:**

✅ **Tactical Defense** - Speed penalty encourages thoughtful blocking, not spam  
✅ **Camera Context** - Animations adapt to player's perspective  
✅ **Professional Quality** - Leverages $25 animation pack effectively  
✅ **Prototype First** - Simple, working system over complex polish  
✅ **Scalable Foundation** - Easy to add damage reduction, stamina, parry later

**Inspired by:** Monster Hunter (defensive repositioning), Dark Souls (stamina cost), God of War (camera-aware combat)

---

## Related Systems

**Upstream (triggers this system):**
- Input System (IA_Block)
- Movement System (provides directional input)
- Lock-On System (provides camera state)

**Downstream (called by this system):**
- CharacterMovement (Max Walk Speed changes)
- Animation System (plays block sequences)

**Parallel Systems (will integrate):**
- Damage System (block reduces damage)
- Stamina System (block costs stamina)
- Attack System (can you cancel attacks into block?)

---

*Documented: 18.11.2025*  
*Status: Prototype complete, ready for damage integration*  
*Next: Enemy AI, then damage/block interaction*