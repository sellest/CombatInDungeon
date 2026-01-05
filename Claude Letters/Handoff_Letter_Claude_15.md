# Handoff Letter: Claude-15 → Claude-16

**Date:** January 5, 2026  
**Sessions Covered:** December 27, 2025 - January 5, 2026  
**Developer:** Ilia  
**Project:** Monster Hunter-inspired boss rush roguelite in UE5

---

## Executive Summary

This was a massive session spanning multiple days. We completed the entire player character locomotion foundation including armed/unarmed states, sheathe system, block with locomotion, idle turns, and roll/evasion system. The character is now ~90% playtest-ready - only flinch reactions and stamina remain.

---

## What Was Built

### 1. Locomotion System (Complete)

**Physics-based acceleration system:**
- TargetSpeed (intent: 0, 250, 600) vs CurrentSpeed (actual, interpolated)
- MaxAcceleration/MaxDeceleration for weighted feel
- No root motion for locomotion - all velocity-driven

**State machine (JogSequence) with 20+ states:**
- Unarmed: Idle, Turn, Walk_Start, Run_Start, WalkRun_Loop, Walk_Stop, Run_Stop
- Armed: Full duplicate of unarmed states
- Block: Block_Idle, Guard_Walk_Start, Guard_Walk_Loop, Guard_Walk_Stop
- Sheathe/Unsheathe transitions between armed/unarmed

**Input commitment system:**
- ANS_LockMovementInput on all Start/Stop animations
- bInputReleasedDuringLock for deferred stops
- bLocomotionFinished for outer state coordination

### 2. Turn System (Complete)

**Idle turns only** (motion turns disabled - no animations):
- BS_Turn and BS_Turn_Armed BlendSpaces
- Angle snapping to discrete values (-90, 90, 180)
- CalculateTurnAngle() - the most guarded function (7 guards)

**Guards in CalculateTurnAngle:**
1. SmoothedX/Y == 0 → RETURN
2. bWantsTurn == True → RETURN
3. bLockMovementInput == True → RETURN
4. CurrentSpeed > 50 → clear flag, RETURN
5. VectorLength < 0.5 → RETURN
6. bIsExecutingAttack == True → clear flag, RETURN
7. bIsDodging == True → clear flag, RETURN

**AN_ClearTurnIntent** on all Start/Stop animations prevents stale turns.

### 3. Block Locomotion (Complete)

**Block integrated into JogSequence:**
- Block_Start → Block_Idle → Block_End sequence in Idle state
- Guard_Walk states for movement while blocking
- Walk only, no run while blocking

**bBlockEstablished flag:**
- Set True: end of Block_Start, or when pressing block while moving
- Set False: when block released
- Routes correctly: fresh block → Block_Start, returning from movement → Block_Idle

### 4. Roll System (Complete)

**Top-level ABP state** (can interrupt most actions):
- BS_Roll and BS_Roll_Armed BlendSpaces
- Loop=True critical (prevents spam stuck state)

**Two modes:**
- Locked on: Directional rolls (Atan2 of LastMoveRight/Forward)
- Free camera: Rotate-then-roll (fast interp to target yaw, then forward roll)

**Animation timing (66 frames):**
- 0-10: Windup (vulnerable)
- 10-45: Active roll (I-frames via ANS_IFrames)
- 45-50: Early recovery (locked)
- 50-65: Late recovery (cancel window - locks end at 50)
- AN_DodgeComplete at frame 49
- AN_RollIdleExit at frame 65 (for clean idle exit if no input)

**Rotate-then-roll variables:**
- bRotatingIntoRoll, TargetRollRotation
- FInterpTo at speed 35.0 in Event Tick
- Only sets bWantsRoll = True when rotation complete

### 5. Attack Integration

**bIsExecutingAttack flag:**
- Set by ANS_ExecutingAttack on attack montages
- Event Tick forces TargetSpeed = 0 while True
- Forces locomotion to Idle underneath montages
- Clean transition when attack ends

---

## Key Technical Patterns Established

### Intent Flag Pattern
Used throughout: bWantsTurn, bWantsRoll, bWantsSheathe, bBlockEstablished
- Intent set on input
- State changes when animation completes
- Prevents state desync between BP and ABP

### Guard-Heavy Functions
CalculateTurnAngle has 7 guards - each one exists because we found a bug without it. This is robust production code, not over-engineering.

### BlendSpace Settings
- Loop=True prevents spam issues (learned from Armed_Roll bug)
- Angle snapping prevents interpolation between unintended samples

### Transition Timing
- Time Remaining < 0.1 for most exits
- Roll → Idle blend = 0.0 (prevents momentum carry from root motion)
- AN placement must consider blend timing

---

## Known Issues / Polish Items

1. **Sharp direction snap when locked on** - BlendSpace instantly switches L↔R strafe. Attempted rotation smoothing but issue is BlendSpace input, not rotation. Noted for polish.

2. **Minor transition twitches** - Various blend timing issues throughout locomotion. Not critical for playtest.

3. **Roll → Run transition** - Slightly rough, would benefit from longer blend or going through Walk_Start.

4. **Motion turns disabled** - No 180° turn animations while moving. BlendSpace handles direction changes, turns only from idle.

---

## Documentation Updated

1. **LocomotionSystem.md** - Complete rewrite with Turn System, Block Locomotion sections
2. **RollSystem.md** - New file, complete roll documentation
3. **SYSTEMS_OVERVIEW.md** - Added Locomotion section, updated Roll/Block sections, updated priorities
4. **CLAUDE.md** - Updated current state, key learnings, priorities

---

## What's Next (Priority Order)

### 1. Flinch System (Next Priority)
Player hit reactions. Ilia has animations:
- Light hit (F/B/L/R) - quick torso flinch
- Large hit (F/B/L/R) - big stagger
- Knockdown + get up - full floor recovery
- Block light hit - chip reaction
- Block break - guard broken

**Suggested approach:**
- Top-level ABP state (like Roll)
- FlinchTier enum (Light, Heavy, Knockdown)
- Tier determined by boss attack data
- Direction from hit source

### 2. Stamina System
Resource for rolls, attacks, blocking. Not yet designed.

### 3. Boss Exhaustion
Combat rhythm - boss needs rest periods. Identified as needed but not built.

---

## Developer State

Ilia pushed through significant frustration ("why is basic movement so hard") but emerged with a production-quality character controller. Key realization: this IS hard, and the time spent is normal for quality results. Comparing to Mortal Shell (indie with AAA-experienced team, 4+ years dev) helped calibrate expectations.

Birthday was January 1st. Good way to start 2026 - working on the game.

---

## Files to Reference

- `/mnt/user-data/outputs/LocomotionSystem.md` - Complete locomotion documentation
- `/mnt/user-data/outputs/RollSystem.md` - Complete roll documentation
- `/mnt/user-data/outputs/SYSTEMS_OVERVIEW.md` - All systems overview
- `/mnt/user-data/outputs/CLAUDE.md` - Communication guide and project context

---

## Session Stats

- **Duration:** ~9 days (Dec 27 - Jan 5)
- **Major systems built:** 5 (Locomotion, Turns, Block Loco, Roll, Attack Integration)
- **States in JogSequence:** 20+
- **Guards in CalculateTurnAngle:** 7
- **Bugs fixed:** Too many to count
- **Documentation files updated:** 4

---

*Good luck, Claude-16. The character is almost done. Flinch, then stamina, then it's boss tuning and playtest.*
