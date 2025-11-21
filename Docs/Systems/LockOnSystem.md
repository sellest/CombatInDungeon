# Lock-On System Documentation

**Status:** ✅ Complete (Session 21.11 fixes applied)  
**Sessions:** 8-9.11.2025 (initial), 21.11.2025 (chest targeting fix)  
**Purpose:** God of War 2018-style target lock for intimate boss combat

---

## Overview

The Lock-On System provides a camera-lock mechanism that keeps the player character facing a targeted enemy, enabling tactical strafing movement and directional dodges. The system toggles between free exploration camera and locked combat camera.

**Design Philosophy:**
- **God of War 2018 inspiration** - Over-shoulder, intimate camera
- **Tactical positioning** - Character always faces target, strafing movement
- **Directional dodges enabled** - Left/right dodges make spatial sense
- **Close combat focus** - For 1v1 boss encounters, not crowd combat

---

## Core Mechanics

**Input:** Tab key (IA_LockOn action)

**Behavior:**
- **Unlocked → Lock:** Find nearest enemy, lock onto chest height target
- **Locked → Unlock:** Release lock, return to free camera

**Movement Changes:**
- **Unlocked:** Character rotates with movement direction (Orient Rotation to Movement)
- **Locked:** Character faces target, movement strafes (Orient Rotation to Movement OFF)

**Camera Changes:**
- **Unlocked:** Player controls camera freely
- **Locked:** Camera automatically aims at target's chest (LockCamera macro)

---

## Variables (BP_ThirdPersonCharacter)

| Variable | Type | Default | Category | Purpose |
|----------|------|---------|----------|---------|
| `bIsLockedOn` | Boolean | False | Camera | Currently locked to target? |
| `LockedTarget` | Scene Component Reference | None | Camera | Reference to enemy's LockOnTarget component |
| `LockOnRange` | Float | 500 | Camera | Max distance to lock onto enemy |

**Type Change (Session 21.11):**
- **Old:** `LockedTarget` was Actor Reference (targeted root, bad angles)
- **New:** `LockedTarget` is Scene Component Reference (targets chest component)

**Why Scene Component:**
- Allows targeting specific point on enemy (chest height) controlled by LockOnTarget (under EnemyMesh)
- Better camera framing than actor root (feet level)
- Per-enemy positioning control via component offset

---

## Enemy Setup (BP_EnemyBase)

### LockOnTarget Component (Session 21.11)

**Type:** Scene Component  
**Location:** BP_EnemyBase → Components  
**Purpose:** Defines the lock-on aim point (chest height for humanoids)

**Hierarchy:**
```
BP_EnemyBase
├─ Capsule Component (root)
├─ Mesh
│   └─ LockOnTarget (Scene Component) ← Parented to Mesh
```

**Transform (relative to Mesh):**
- **X:** 0 (centered front-back)
- **Y:** 0 (centered left-right)
- **Z:** 80 (chest height for humanoid - tune per boss)

**Why parent to Mesh:**
- Moves with boss animations
- Rotates with boss facing
- Scales with boss size

---

## Input Handling (IA_LockOn)

### Toggle Logic

**IA_LockOn Started event:**

```
IA_LockOn Started:
    ↓
    Branch: bIsLockedOn?
    │
    ├─ True → UNLOCK
    │   ├─ Set bIsLockedOn = False
    │   ├─ Set LockedTarget = None
    │   ├─ Set Orient Rotation to Movement = True
    │   └─ Print: "Lock released"
    │
    └─ False → TRY TO LOCK
        ↓
        FindNearestEnemy (macro)
        → Returns: ClosestActor
        ↓
        Branch: Is Valid (ClosestActor)?
        ├─ True → LOCK ON
        │   ├─ Cast to BP_EnemyBase
        │   ├─ Get LockOnTarget (component)
        │   ├─ Set LockedTarget = LockOnTarget component
        │   ├─ Set bIsLockedOn = True
        │   ├─ Set Orient Rotation to Movement = False
        │   └─ Print: "Locked onto {Actor name}"
        │
        └─ False → LOCK FAILED
            └─ Print: "No target in range"
```

**Key Points:**
- Tab toggles between states (not hold)
- Lock requires valid enemy in range
- Get component from actor (not actor itself) - Session 21.11 fix
- Orient Rotation to Movement controls character facing

---

## FindNearestEnemy Macro

**Purpose:** Locate closest enemy within lock-on range

**Implementation (typical):**
```
FindNearestEnemy:
    ↓
    Get All Actors of Class (BP_EnemyBase)
    → Returns: Array of enemies
    ↓
    For Each Loop:
        Get Distance to Actor
        Compare to CurrentClosest
        Store if closer
    ↓
    Return: ClosestActor
```

**Line of Sight Check (future):**
- Trace from player to enemy
- Ignore if blocked by walls
- Prevents locking through geometry

---

## Camera System (LockCamera Macro)

### LockCamera Macro

**When:** Called in Event Tick (or camera update event)  
**Condition:** Only runs if `bIsLockedOn = True AND LockedTarget is valid`

**Implementation:**

```
LockCamera:
    ↓
    Branch: bIsLockedOn AND Is Valid (LockedTarget)?
    └─ True → Calculate camera rotation
               ↓
               Get Player Camera Manager
               → Get Camera Location (start point)
               ↓
               Get World Location (LockedTarget component)
               → Add offset [50, 50, 120]
               ↓
               Find Look at Rotation
               ├─ Start: Camera Location
               └─ Target: Enemy chest + offset
               ↓
               Set Control Rotation
               ├─ Target: Player Controller
               └─ New Rotation: Calculated rotation
```

**Mysterious Offset [50, 50, 120]:**
- **X: 50** - Slightly ahead of target (anticipation)
- **Y: 50** - Slightly right of target (framing)
- **Z: 120** - Above chest height (upward bias)

**Why offset exists:**
- Tuned by feel during development
- Exact framing preference
- May vary per boss (future: per-boss offset)

**Note:** "Don't ask why these values exist, I don't remember why" - Ilia, Session 21.11

---

## Character Movement Integration

### Orient Rotation to Movement

**Character Movement Component setting**

**Unlocked (Orient = True):**
```
Player presses W (forward):
    ↓
    Character body rotates to face forward
    ↓
    Moves in that direction
    
Result: Traditional third-person movement
```

**Locked (Orient = False):**
```
Player presses W (forward):
    ↓
    Character STAYS facing target
    ↓
    Moves forward relative to camera (strafe forward)
    
Result: Strafing movement, always facing enemy
```

**Why this works:**
- Locked: Movement input is relative to camera, not character
- Character rotation locked to target via camera aim
- Creates tactical circle-strafing combat

---

## Animation System Integration

### Dodge System

**Lock-on enables directional dodges:**

**Unlocked camera:**
- Dodge always plays forward animation (character rotates to face movement)
- Left/Right dodges don't make spatial sense (no reference point)

**Locked camera:**
- Character faces enemy (fixed reference)
- Left dodge = physically dodge left relative to enemy
- Right dodge = physically dodge right relative to enemy
- Forward = dodge toward enemy
- Backward = dodge away from enemy

**Implementation in ABP_Unarmed:**
```
ExecuteDodge function:
    Branch: bIsLockedOn?
    ├─ True → Use directional dodges (F/B/L/R based on input)
    └─ False → Always use forward dodge (character rotates first)
```

**See DodgeSystem.md for full implementation.**

---

### Blocking System

**Lock-on affects block animations:**

**Unlocked:**
- Moving during block = always forward guard animation
- Character rotates to face movement

**Locked:**
- Moving during block = directional guard animations
- Guard left/right while facing enemy
- Enables tactical defensive positioning

**Implementation in ABP_Unarmed:**
```
Guard locomotion blend space inputs:
    X (Strafe): 
        If bIsLockedOn: LastMoveRight
        Else: 0.0
    Y (Forward):
        If bIsLockedOn: LastMoveForward
        Else: (bShouldMove ? 1.0 : 0.0)
```

**See BlockSystem.md for full implementation.**

---

## Technical Details

### Camera Control

**Set Control Rotation:**
- Controls player's view direction (camera)
- In locked mode: Overridden every frame by LockCamera
- In unlocked mode: Player mouse/stick controls it

**Control Rotation vs Actor Rotation:**
- Control Rotation = where camera points
- Actor Rotation = where character faces
- Locked: Both point at enemy (aligned)
- Unlocked: Independent (camera free, character follows movement)

---

### Component Reference Pattern

**Why store Scene Component, not Actor:**

**Problem with Actor Reference:**
```
Lock onto BP_EnemyBase actor
    ↓
Get Actor Location → Returns root (capsule at feet)
    ↓
Camera aims at feet (bad framing)
```

**Solution with Component Reference:**
```
Lock onto LockOnTarget component
    ↓
Get World Location → Returns component position (chest)
    ↓
Camera aims at chest (good framing)
```

**Benefit:**
- Per-enemy control (adjust Z offset per boss)
- Animated targets (component moves with animations)
- Multiple lock points (future: head, weak spot, etc.)

---

## Known Issues

### Fixed (Session 21.11)

- ✅ ~~Camera locks to feet (strange angles)~~ - Added LockOnTarget component at chest height
- ✅ ~~LockedTarget type wrong~~ - Changed to Scene Component Reference

### Current (Deferred to Real Boss)

**Camera pitch clamp:**
- Camera can rotate to extreme angles (straight up/down)
- Deferred: Needs real boss animations to tune properly
- Future: Clamp pitch to -30° / +30° range

**Camera collision:**
- No obstruction handling (camera clips through walls)
- Deferred: Add spring arm collision or camera smoothing

**Lock-on break on distance:**
- Doesn't auto-unlock if enemy too far
- Should break lock at ~3000 units
- Prevents locking onto fleeing enemies

**No lock-on UI:**
- No visual indicator (reticle, highlight) on locked enemy
- Future: Widget showing lock status

---

## Future Enhancements

### High Priority (Polish)

1. **Lock-On Reticle** (1 hour)
   - Widget at LockedTarget position
   - Shows lock status (circle, crosshair, etc.)
   - Visual confirmation of lock

2. **Camera Pitch Clamp** (15 min)
   - Limit to -30° / +30° range
   - Prevents extreme angles
   - Tune with real boss

3. **Auto-Unlock on Distance** (30 min)
   - Check distance every frame
   - Break lock if > 3000 units
   - Smooth transition to free camera

---

### Medium Priority

4. **Multiple Target Switching** (1 hour)
   - Right stick flicks switch between nearby enemies
   - Useful for multi-boss arenas
   - Cycle through valid targets

5. **Lock-On Sticky Aim** (30 min)
   - Small magnetism toward target
   - Helps tracking fast-moving bosses
   - Tune strength by feel

6. **Camera Smoothing** (1 hour)
   - Interpolate rotation instead of snapping
   - Smooth lock-on engagement
   - Smooth target switches

---

### Low Priority (Future)

7. **Lock-On Zones** (2 hours)
   - Lock onto specific boss parts (head, tail, wings)
   - Cycle through parts with R3 click
   - Part-break system integration

8. **Boss-Specific Camera** (varies)
   - Different offsets per boss
   - Different FOV per boss
   - Different pitch clamps for flying vs ground bosses

9. **Lock-On Audio** (30 min)
   - Sound on lock engage
   - Sound on lock break
   - Audio feedback for clarity

---

## Testing Checklist

**Basic Functionality:**
- ✅ Tab toggles lock-on (on/off/on)
- ✅ Finds nearest enemy in range
- ✅ Camera aims at enemy chest (not feet)
- ✅ Character faces enemy when locked
- ✅ Character rotates freely when unlocked
- ✅ Lock persists during movement
- ✅ Lock persists during attacks
- ✅ Lock persists during blocking
- ✅ Lock persists during dodging

**Movement Integration:**
- ✅ Unlocked: Character rotates with movement
- ✅ Locked: Character strafes while facing enemy
- ✅ Dodge uses correct animations (directional when locked)
- ✅ Block uses correct animations (directional when locked)

**Edge Cases:**
- ✅ No enemies in range → Lock fails gracefully
- ✅ Locked enemy dies → Lock remains (on corpse)
- ✅ Player dies → Lock persists (camera still works)

**Future Tests:**
- ⏸️ Lock breaks at distance (not implemented)
- ⏸️ Camera pitch clamp (deferred)
- ⏸️ Multiple enemy switching (not implemented)

---

## Design Rationale

### Why Toggle, Not Hold?

**Alternative considered:** Hold Tab to maintain lock

**Rejected because:**
- ❌ Requires constant input (finger fatigue)
- ❌ Conflicts with other hold actions (block, dodge)
- ❌ Accidental unlock when shifting hand position

**Chosen approach (toggle):**
- ✅ Set and forget (lock persists)
- ✅ No input conflict
- ✅ Single button press = clear action
- ✅ Standard in genre (God of War, Dark Souls, etc.)

---

### Why Scene Component vs Socket?

**Alternative considered:** Socket on Mesh bone

**Considered but not needed:**
- Scene Component simpler for static offset
- Socket better for animated target (head tracking, weak spot pulsing)
- Current: Static chest point sufficient
- Future: Socket-based if needed for part-breaks

**Chosen approach (Scene Component):**
- ✅ Simple setup (no bone selection)
- ✅ Easy to position (XYZ offset)
- ✅ Moves with mesh (parented)
- ✅ Sufficient for prototype

---

### Why God of War Style vs Dark Souls Style?

**Dark Souls lock-on:**
- Hard lock (camera snaps, limited movement)
- Circle-strafing focus
- Requires frequent lock/unlock

**God of War 2018 lock-on:**
- Softer lock (camera aims but player has some control)
- Fluid combat flow
- Set and forget

**Chosen approach (God of War):**
- ✅ Matches intimate, close camera
- ✅ Matches deliberate, tactical combat
- ✅ Matches 1v1 boss focus (not crowd combat)
- ✅ Less disorienting for new players

---

## Related Systems

**Upstream (uses lock-on state):**
- Dodge System (directional vs forward-only)
- Block System (directional locomotion)
- Camera System (lock vs free)

**Downstream (affects lock-on):**
- Enemy System (provides lock targets)
- Input System (Tab key, movement input relative to camera)

**Parallel (independent):**
- Combat System (attacks work locked or unlocked)
- Perfect Timing (timing windows work regardless)
- Damage System (taking damage doesn't affect lock)

---

## Code References

### Key Files

**Blueprints:**
- `BP_ThirdPersonCharacter` - Lock-on logic, LockCamera macro
- `BP_EnemyBase` - LockOnTarget component
- `ABP_Unarmed` - Conditional animation logic based on bIsLockedOn

**Macros:**
- `FindNearestEnemy` - Enemy detection
- `LockCamera` - Camera rotation calculation

---

### Key Variables

**BP_ThirdPersonCharacter:**
- `bIsLockedOn` (Boolean) - Lock state
- `LockedTarget` (Scene Component Reference) - Target component

**BP_EnemyBase:**
- `LockOnTarget` (Scene Component) - Aim point at chest height

**CharacterMovement:**
- `Orient Rotation to Movement` - Toggled by lock state

---

### Key Functions/Macros

**BP_ThirdPersonCharacter:**
- IA_LockOn Started event - Toggle logic
- FindNearestEnemy macro - Enemy detection
- LockCamera macro - Camera control (called in Tick)

---

## Architectural Principles

**Single Responsibility:**
- Lock-on handles camera aim only
- Doesn't know about combat, damage, or animations
- Other systems check bIsLockedOn for context

**Component-Based:**
- LockOnTarget = reusable component
- Any enemy can have lock target
- Position per-enemy via component offset

**Toggle State Pattern:**
- Boolean flag (bIsLockedOn)
- Toggle input (Tab)
- State persists until toggled again

**Loose Coupling:**
- Dodge system checks lock state, doesn't control it
- Block system checks lock state, doesn't control it
- Lock-on doesn't know about dodge/block
- Clean separation of concerns

---

## Design Philosophy

**This system embodies:**

✅ **Intimate Combat** - Close camera, always facing enemy  
✅ **Tactical Movement** - Strafing enables positioning strategy  
✅ **Player Agency** - Toggle when needed, not forced  
✅ **Clear Feedback** - Character body language shows lock state  
✅ **System Synergy** - Enables directional dodges and guards

**Inspired by:**
- **God of War 2018** - Soft lock, fluid combat, over-shoulder intimacy
- **Monster Hunter** - Target lock for deliberate boss fights
- **Resident Evil 2 Remake** - Close camera, claustrophobic feel

---

## Session Notes

**Session 8-9.11.2025 (Initial):**
- Created lock-on toggle system
- Implemented FindNearestEnemy
- Added Orient Rotation to Movement toggle
- Integrated with dodge/block animation logic

**Session 21.11.2025 (Chest Targeting Fix):**
- Duration: ~30 minutes
- Bug: Lock-on targeted actor root (feet level, bad angles)
- Fix: Added LockOnTarget Scene Component to BP_EnemyBase
- Fix: Changed LockedTarget variable type to Scene Component Reference
- Fix: Updated lock-on logic to get component from enemy
- Fix: Updated LockCamera to use Get World Location on component
- Result: Camera now aims at chest height (good framing)

**Deferred (Smart Decision):**
- Camera pitch clamp (needs real boss to tune)
- Camera smoothing (polish phase)
- Lock-on UI (visual polish)

---

*Last Updated: 21.11.2025*  
*Status: Complete with chest targeting fix*  
*Next: Camera pitch clamp (when real boss added)*
