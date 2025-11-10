# Dodge System

## Overview

**Status:** ✅ Complete  
**Session:** 7 (Initial), 15 (Defense Lock + Directional Fix)  
**Purpose:** Directional evasion mechanic with i-frames and defense locking

---

## Design

Four-directional dodge roll system with behavior dependent on camera lock-on state and recent movement input.

**Core Mechanics:**

- Spacebar/designated button triggers dodge
- Direction determined by recent movement input (0.15s window)
- Default direction varies by lock-on state
- Cannot be spammed (locked during execution)
- Cannot be used during attacks (defense lock)

---

## Input Flow

```
Player presses Dodge button
    ↓
Check: bCanDodge? (not locked by attacks)
    ↓
Check: bIsDodging? (not already dodging)
    ↓
Check: Recent movement input? (within 0.15s)
    ├─ NO → Default direction
    │   ├─ Unlocked: Forward
    │   └─ Locked: Backward
    │
    └─ YES → Directional
        ├─ Forward (LastMoveForward > 0.5)
        ├─ Backward (LastMoveForward < -0.5)
        ├─ Right (LastMoveRight > 0.5)
        └─ Left (LastMoveRight < -0.5)
```

---

## Implementation

### PerformDodge Macro

**Location:** BP_ThirdPersonCharacter Macros  
**Input:** None (reads state variables)  
**Purpose:** Main dodge logic controller

**Logic Flow:**

```
PerformDodge:

1. Guard Check: Defense Lock
   Branch: bCanDodge?
   ├─ False → EXIT (locked by ANS_LockDefense during attacks)
   └─ True → Continue

2. Guard Check: Already Dodging
   Branch: bIsDodging?
   ├─ True → EXIT (prevent dodge spam)
   └─ True → Continue

3. Input Timing Check
   Get Game Time - LastMoveInputTime = TimeSinceInput
   Branch: TimeSinceInput < DirectionalWindow (0.15s)?
   
   ├─ FALSE (No recent input) → Default Direction
   │   └─ Branch: IsLockedOn?
   │       ├─ True → ExecuteDodge(BackwardDodge)
   │       └─ False → ExecuteDodge(ForwardDodge)
   │
   └─ TRUE (Recent input) → Directional Input
       └─ Branch: IsLockedOn?
           ├─ False → ExecuteDodge(ForwardDodge)
           │   (Unlocked always dodges forward)
           │
           └─ True → Check Direction
               ├─ Compare LastMoveForward:
               │   ├─ > 0.5 → ExecuteDodge(ForwardDodge)
               │   └─ < -0.5 → ExecuteDodge(BackwardDodge)
               │
               └─ If neither, Compare LastMoveRight:
                   ├─ > 0.5 → ExecuteDodge(RightDodge)
                   └─ < -0.5 → ExecuteDodge(LeftDodge)
```

---

### ExecuteDodge Function

**Location:** BP_ThirdPersonCharacter Functions  
**Inputs:**

- `Target` (Actor Reference, default Self)
- `DodgeMontage` (Anim Montage Reference)

**Purpose:** Execute dodge animation and set state flags

**Logic:**

```
ExecuteDodge(Target, DodgeMontage):
├─ Set bIsDodging = True
├─ Set bCanCombo = False
├─ Set CurrentAttackName = None
└─ Play Anim Montage: DodgeMontage
```

**State Changes:**

- `bIsDodging = True` - Prevents dodge spam
- `bCanCombo = False` - Prevents attacking during dodge
- `CurrentAttackName = None` - Clears combat state (can't combo from dodge)

---

## Animation Montages

### All Dodge Montages

**Files:**

- AM_Dodge_Forward
- AM_Dodge_Backward
- AM_Dodge_Left
- AM_Dodge_Right

**AnimNotifies (all montages):**

**1. ANS_LockDefense** (spans most/all of animation)

- Prevents dodge spam
- Prevents attacking during dodge
- Locks both `bCanDodge` and `bCanBlock`

**2. ANS_IFrames** (future - planned)

- Invincibility frames during active dodge
- Player cannot take damage
- Typically middle ~60% of animation

**3. AN_DodgeEnded** (at end)

- Sets `bIsDodging = False`
- Re-enables dodge input

---

## State Variables

**Used by Dodge System:**

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bIsDodging` | Boolean | False | Currently executing dodge animation |
| `bCanDodge` | Boolean | True | External lock (from ANS_LockDefense) |
| `bCanCombo` | Boolean | True | Combo input permission |
| `LastMoveForward` | Float | 0.0 | Forward/Backward input value (-1 to 1) |
| `LastMoveRight` | Float | 0.0 | Left/Right input value (-1 to 1) |
| `LastMoveInputTime` | Float | 0.0 | Game time of last movement input |
| `DirectionalWindow` | Float | 0.15 | Time window for "recent" input (seconds) |
| `bIsLockedOn` | Boolean | False | Camera lock-on state |

---

## Direction Determination

### Priority System

**1. Defense Lock Check** (highest priority)

- If `bCanDodge = False` → No dodge at all

**2. Already Dodging Check**

- If `bIsDodging = True` → Ignore input

**3. Input Timing Check**

- If movement input is stale (> 0.15s old) → Use default direction
- If movement input is recent (< 0.15s) → Use directional input

**4. Lock-On State**

- Affects default direction
- Affects which directional inputs are valid

---

### Direction Logic Tables

**When Unlocked (bIsLockedOn = False):**

| Recent Input? | Movement Direction | Result |
|--------------|-------------------|---------|
| No | (any) | Forward Dodge |
| Yes | (any) | Forward Dodge |

*Note: Unlocked camera always dodges forward regardless of input. Other directions unused until lock-on enables them.*

---

**When Locked On (bIsLockedOn = True):**

| Recent Input? | LastMoveForward | LastMoveRight | Result |
|--------------|----------------|---------------|---------|
| No | (any) | (any) | **Backward Dodge** |
| Yes | > 0.5 | (any) | Forward Dodge |
| Yes | < -0.5 | (any) | Backward Dodge |
| Yes | -0.5 to 0.5 | > 0.5 | Right Dodge |
| Yes | -0.5 to 0.5 | < -0.5 | Left Dodge |

**Priority:** Forward/Backward checked first, then Left/Right if forward/backward near neutral.

---

## Design Decisions

### Why Recent Input Check? (Session 15 Fix)

**Problem:** Stale directional input caused confusing dodge directions.

**Example:**

```
Player presses W (forward) at T=5.0s
Player waits...
Player presses Dodge at T=8.0s
→ Would dodge forward (3-second-old input!)
```

**Solution:** Only use directional input if it occurred within `DirectionalWindow` (0.15s).

**Why 0.15s?** Matches simultaneous input detection window used throughout combat system. Feels natural - if you're pressing direction "with" dodge button, it counts.

---

### Why Default Direction Varies by Lock-On?

**Unlocked (Forward Default):**

- Player controls facing with camera/movement
- Forward = "away from danger" (usually)
- Aggressive positioning

**Locked (Backward Default):**

- Player faces boss automatically
- Backward = "away from target" (safe)
- Defensive positioning

**Matches God of War 2018 and Monster Hunter behavior.**

---

### Why Lock Dodge During Attacks?

**Without ANS_LockDefense:**

```
Player attacks → Immediately dodge
No commitment, no risk
Can animation-cancel any mistake
Trivializes combat
```

**With ANS_LockDefense:**

```
Player attacks → Locked until animation ends
Must commit to attack choice
Dodge is escape, not cancel
Tactical decision-making
```

**Matches design pillar:** "Every action matters, punishes button mashing"

---

## Integration Points

### Input System

**Dodge triggered by:**

- IA_Dodge input action (Spacebar/Button)
- Processed in Event Graph or input event
- Calls PerformDodge macro

**Movement tracking:**

- IA_Move updates `LastMoveForward`, `LastMoveRight`, `LastMoveInputTime`
- Shared infrastructure with directional attacks
- Single source of truth for movement state

---

### Lock-On System

**Provides `bIsLockedOn` state:**

- Affects default dodge direction
- Enables directional dodges (Left/Right only useful when locked)
- Independent system, just reads the flag

---

### Combat System

**State coordination:**

- `CurrentAttackName = None` when dodging (clears combo state)
- `bCanCombo = False` during dodge (prevents attack buffering)
- ANS_LockDefense from attacks prevents dodge (commitment)

**Future i-frames:**

- Will check `bHasIFrames` flag (set by ANS_IFrames)
- Damage system will skip damage application during i-frames

---

## Future Enhancements

### I-Frames (Planned)

**ANS_IFrames AnimNotifyState:**

```
Received_NotifyBegin:
└─ Set bHasIFrames = True

Received_NotifyEnd:
└─ Set bHasIFrames = False
```

**Placement:** Middle ~60% of dodge animation (active roll frames)

**Damage System Check:**

```
When taking damage:
├─ If bHasIFrames = True → Ignore damage
└─ Else → Apply damage
```

---

### Stamina Cost (Planned)

**Dodge costs stamina:**

```
ExecuteDodge:
├─ Check: Stamina >= DodgeCost?
│   ├─ True → Continue, drain stamina
│   └─ False → EXIT (not enough stamina)
```

**Prevents infinite dodge spam even without ANS_LockDefense.**

---

### Perfect Dodge (Planned)

**Dodge just before attack hits = bonus:**

```
If dodged within 0.2s of incoming damage:
├─ Slow-mo effect
├─ Extended i-frames
└─ Counter-attack window
```

**Rewards skillful timing, similar to perfect attack timing system.**

---

### Directional Dodge Unlocked (Future Design Decision)

**Current:** Unlocked camera only dodges forward

**Alternative:** Enable all 4 directions even when unlocked

- Forward = camera forward
- Backward = camera backward  
- Left/Right = camera relative

**Trade-off:** More control vs. more complex (might confuse players)

**Decision deferred** until combat testing with enemies validates current approach.

---

## Testing & Debugging

### Test Cases

**Test 1: Spam Prevention**

```
Input: Spacebar spam
Expected: Only one dodge, subsequent inputs ignored
Validates: bIsDodging flag, ANS_LockDefense
```

**Test 2: Stale Input Rejection**

```
1. Press W, release
2. Wait 1 second
3. Press Dodge
Expected: 
- Unlocked → Forward dodge
- Locked → Backward dodge (default, not forward)
Validates: Input timing check
```

**Test 3: Recent Input Acceptance**

```
1. Press W
2. Immediately press Dodge (within 0.15s)
Expected: Forward dodge
Validates: Directional input when recent
```

**Test 4: Lock During Attack**

```
1. Start attack (Tri)
2. Press Dodge during attack
Expected: No dodge until attack ends
Validates: ANS_LockDefense in attack montages
```

---

### Debug Outputs

**For monitoring dodge system:**

```
PerformDodge macro:
├─ "Dodge blocked: defense locked" (bCanDodge = False)
├─ "Dodge blocked: already dodging" (bIsDodging = True)
├─ "Dodge: Recent input, direction = {direction}"
└─ "Dodge: Default direction (locked = {bIsLockedOn})"
```

**For tracking input timing:**

```
Print: "TimeSinceInput: {CurrentTime - LastMoveInputTime}"
```

---

### Common Issues & Solutions

**Issue:** Dodge goes wrong direction despite correct input

**Check:**

- Is `LastMoveInputTime` being updated in IA_Move?
- Is `DirectionalWindow` too small? (try 0.2s for testing)
- Print TimeSinceInput to verify timing logic

---

**Issue:** Can dodge during attacks

**Check:**

- Is ANS_LockDefense placed in attack montages?
- Does ANS_LockDefense span the full attack duration?
- Is `bCanDodge` check at the start of PerformDodge?

---

**Issue:** Stuck unable to dodge after taking damage

**Check:**

- Does damage system reset `bCanDodge = True`?
- Does damage interruption leave ANS_LockDefense active?
- Add safety reset in damage/hitstun logic (see State Leakage section)

---

**Issue:** Can spam dodge infinitely

**Check:**

- Is ANS_LockDefense in dodge montages?
- Is AN_DodgeEnded resetting `bIsDodging = False`?
- Does ANS_LockDefense span enough of the animation?

---

## State Leakage Prevention

### Problem

When animations are interrupted (damage, hitstun), AnimNotifyStates may not reach `Received_NotifyEnd`, leaving flags in bad states.

**Example:**

```
Player dodging → ANS_LockDefense active → bCanDodge = False
Boss hits player → Dodge interrupted
ANS_LockDefense End never fires
bCanDodge stays False forever
Player stuck unable to dodge
```

---

### Solutions (Future Implementation)

**Option 1: Reset on Damage**

```
When taking damage:
└─ Force reset all defensive flags
    ├─ bCanDodge = True
    ├─ bCanBlock = True
    ├─ bIsDodging = False
```

**Option 2: Safety Check in PerformDodge**

```
At start of PerformDodge:
├─ If CurrentAttackName = None AND bCanDodge = False
└─ Set bCanDodge = True (fix leaked state)
```

**Recommendation:** Implement both when damage system is built.

---

## Performance Notes

**Cost:** Negligible

**PerformDodge execution:**

- 3 boolean checks (guard clauses)
- 1 float subtraction (time check)
- 2-4 float comparisons (direction check)
- Cost: ~0.001ms

**Only executes on dodge button press**, not every frame.

---

## Design Philosophy

**This system embodies core combat principles:**

✅ **Commitment** - Cannot spam, locked during attacks  
✅ **Clarity** - Direction determined by recent input only  
✅ **Context-Aware** - Behaves differently based on lock-on  
✅ **Skill Expression** - Future i-frames and perfect dodge timing  
✅ **Fail-Safe** - Default direction prevents accidental neutral dodges

**Inspired by:** Monster Hunter (directional rolls), Dark Souls (i-frames), God of War 2018 (lock-on behavior)

---

## Related Systems

**Upstream (triggers this system):**

- Input System (IA_Dodge button press)
- Movement System (provides LastMoveForward/Right/Time)
- Lock-On System (provides bIsLockedOn state)

**Downstream (affected by this system):**

- Animation System (plays dodge montages)
- Combo System (CurrentAttackName cleared)
- Future damage system (i-frames will prevent damage)

**Parallel Systems (coordinate state):**

- Attack System (ANS_LockDefense prevents dodge during attacks)
- Block System (future - shares bCanBlock flag)

---

*Documented: Session 15 - Dodge System Complete with Defense Lock & Directional Fix*  
*Input timing pattern now consistent across directional attacks and dodge*
