# Handoff Letter - Boss AI Development Session

**Date:** December 2025  
**From:** Claude (Current Session)  
**To:** Claude (Next Session)  
**Developer:** Ilia  
**Project:** Monster Hunter-inspired Boss Rush Game (Unreal Engine 5)

---

## Current Session Summary

This session focused on rebuilding the boss AI system from scratch after recognizing TickBossAI_V1 didn't meet quality standards. We implemented a complete geometry-based attack selection system with combo mechanics, dual-socket hitbox traces, and rotation tracking. The developer has learned from past mistakes and is now building incrementally with isolated testing before full integration.

---

## Developer Context

**Background:**
- Solution Architect at Sberbank, PhD in Nuclear Physics
- Strong architectural thinking, new to game development (~2 months)
- Works after hours with $10k/year asset budget
- "Buy art, build systems" philosophy
- 2000+ hours Monster Hunter experience (design reference)
- January 5th friends-and-family playtest is critical validation milestone

**Communication Preferences:**
- Single complete solutions, not multiple options
- Database/architecture analogies work well
- Honest acknowledgment of limitations over assumptions
- Prefers "will this scale to 10 bosses?" thinking
- Values systematic testing and documentation

**Development Philosophy:**
- **"Working systems do not equal a good game"** - pivotal realization
- Test systems in isolation before integration (learned from TickBossAI_V1 failure)
- Architecture-first, scalable solutions over quick fixes
- Game feel is foundational, cannot be added later
- Data-driven design for designer-friendly iteration

---

## Project Status Overview

### ✅ COMPLETE & VALIDATED (Phase 1-3)

**Geometry Detection System:**
- 12-sector spatial awareness (3 distance bands × 4 directional cones)
- Real-time player position tracking (sector, cone, optimal angle)
- Functions: GetPlayerSector(), GetPlayerOptimalAngle(), IsAngleInCone()
- Handles wrapping cone logic (Back sector: 135° to -135°)

**Debug Visualization System:**
- Distance circles (200, 400, 650 units)
- Cone boundary lines (Front/Right/Back/Left)
- Dynamic optimal angle wedge (color = sector, size = precision)
- Boss forward direction arrow
- Weapon trace visualization (red/green spheres)
- Toggle: bShowDebugGeometry variable
- **Full documentation:** Debug_Visualization_System.md

**Attack Selection System:**
- SelectAttackFromIdle() - geometry-based opener selection
- SelectComboAttack() - geometry-aware combo chains
- IsAttackValidForGeometry() - three-part validation (distance + cone + angle)
- Data-driven via DT_GothicKnightAttacks_V2 and DT_GothicKnightTransitions

**Hitbox System (Dual-Socket Traces):**
- PerformAttackTrace() - flexible body-part-agnostic system
- Supports weapon attacks (Socket_WeaponTip/Base) and body attacks (foot_r/knee_r, hand_r/shoulder_r)
- Data-driven socket configuration per attack
- DeliverHitboxDamage() - interface-based damage delivery
- Multi-hit prevention with per-attack reset

**Combo System:**
- ANS_ComboWindow (AnimNotifyState) - designer-controlled timing
- Geometry-validated combo chains (Attack_01→02→03→04, Attack_08→09)
- Reactive: chains break if player moves out of range
- **Tested via keyboard trigger, fully functional**

**Rotation Tracking System (JUST COMPLETED):**
- ANS_AllowBossRotation (flag setter only, logic in Blueprint)
- PerformBossRotationTracking() - rate-limited, budget-controlled tracking
- Boss rotates toward player during windup frames only (not active frames)
- Data-driven: MaxRotationRate, MaxTotalRotation per attack
- **Working in both directions, feels natural**

**Root Motion:**
- Fixed (was using custom EnemyMesh instead of inherited Mesh component)
- Dual-socket hitboxes work correctly with root motion

**Distance Tuning:**
- Measured attack ranges (sword: ~350u, thrust: ~450u, kick: ~180u)
- Tuned bands from guesses (0-300-600-10000) to measured (0-200-400-650)
- Tight bands = more precise AI decisions

---

## Data Structures (All Documented in Boss_Attack_System_Architecture.md)

**Enums:**
- E_AttackName (Attack_01 through Attack_19, 18 total)
- E_AttackFamily (OPENER, SPRINT_ATTACK, PRESSURE_COMBO, etc.)
- E_DistanceBand (Close: 0-200, Medium: 200-400, Far: 400-650)
- E_AttackCone (Front: -45 to 45, Right: 45 to 135, Back: 135 to -135 wrapping, Left: -135 to -45)
- E_OptimalAngle (Narrow30: ±15°, Moderate60: ±30°, Wide90: ±45°, Any, None)
- E_TransitionContext (Idle, Combo, Sprint, DodgeRight, DodgeLeft)
- E_HitboxMeshType (Weapon, Body, Shield)
- E_BossState_V2 (Idle, Engage, Reposition, Sprint, Attack, Flinch, Dead) - defined but not implemented

**Key Structs:**
- S_BossAttackData_V2 (complete attack definition including hitbox sockets, rotation params)
- S_AttackTransition (defines valid attack chains)
- S_DistanceBandDefinition, S_AttackConeDefinition (geometry definitions)

**Data Tables:**
- DT_DistanceBands (3 rows)
- DT_AttackCones (4 rows)
- DT_GothicKnightAttacks_V2 (18 attacks)
- DT_GothicKnightTransitions (10 rows: 6 Idle openers, 4 combo chains)

**Critical Implementation Details:**
- Back cone wraps (MinAngle > MaxAngle) - requires OR logic in IsAngleInCone
- Unreal angle convention: -180° to 180°, 0° = forward
- Draw Debug Cone direction needs negation (Direction × -1) when using Actor Forward
- Debug visualization uses Actor Forward Vector (rotates with boss), not world forward
- UE5 float negation macro is buggy - use explicit × -1 instead

---

## Current Test Harness (Phase 3)

**Keyboard Trigger System:**
- Player character blueprint has test input that calls boss functions
- Press key → SelectAttackFromIdle() → Set CurrentAttackData → Play Montage
- Combos execute automatically via Event Tick checking bInComboWindow flag
- All hitboxes, damage, rotation tracking functional

**This harness proves all systems work but is not production AI.**

---

## IMMEDIATE NEXT STEPS: Phase 3.5 (Isolated System Tests)

**WHY Phase 3.5 Before Phase 4:**
Ilia learned from TickBossAI_V1 experience: integrating all systems at once made debugging impossible. He wants to test rotation animations and dodge animations in isolation with keyboard harnesses BEFORE building the full AI state machine.

### Task 1: Rotation Animation System (45 min)

**Assets Available:**
- 90° turn animation
- 180° turn animation

**What to Build:**
1. **Function: SelectRotationAnimation(AngleToPlayer)**
   - Input: Float angle to player
   - Logic:
     - IF Abs(angle) > 135° → Return 180° turn montage
     - ELSE IF Abs(angle) > 45° → Return 90° turn montage (left or right variant)
     - ELSE → Return None (use rotation tracking instead)
   - Output: Animation Montage (or None)

2. **Test Harness (Keyboard):**
   - Press "R" key
   - Calculate angle to player (reuse existing geometry functions)
   - Call SelectRotationAnimation()
   - Play returned montage
   - Print start/end facing angles

3. **Validation:**
   - Player behind boss (150-180°) → 180° turn plays → Boss ends facing player
   - Player at 90° → 90° turn plays → Boss ends facing player
   - Player at 30° → No turn (within tracking range)
   - Root motion rotation works correctly
   - Debug visualization still functions during/after turn

**Design Notes:**
- These are discrete turns, NOT continuous tracking
- Used when player moves behind boss (beyond tracking range)
- Different from ANS_AllowBossRotation (that's continuous tracking during windup)
- This is for repositioning between attacks

---

### Task 2: Dodge Animation System (45 min)

**Assets Available:**
- Dodge animations (confirm with Ilia which directions: sidestep? backstep?)

**What to Build:**
1. **Function: ExecuteDodge(Direction)**
   - Input: Direction enum (or vector)
   - Plays appropriate dodge montage
   - Root motion moves boss
   - Returns to neutral stance

2. **Test Harness (Keyboard):**
   - Press "D" key (or D + direction)
   - Execute dodge
   - Validate root motion movement
   - Test dodge → attack chain (Attack_15/16 are dodge counters)

3. **Validation:**
   - Dodge animation plays smoothly
   - Boss physically repositions (root motion working)
   - Can chain to Attack_15 (dodge right + slash) or Attack_16 (dodge left + slash)
   - Doesn't break combo system or hitbox system

**Design Notes:**
- Dodges are repositioning tools, not defensive (boss doesn't dodge player attacks)
- Used when player flanks boss (creates better angle for attacks)
- Attack_15/16 are "dodge counter" attacks (sidestep + slash)
- Add DodgeRight/DodgeLeft transitions to DT_GothicKnightTransitions if testing chaining

---

### Task 3: Integration Test (15 min)

**Scenario to Test:**
1. Player starts behind boss
2. Trigger rotation → Boss turns to face
3. Player moves to flank during next attack
4. Boss dodges to reposition
5. Boss attacks from new position

**Validates:** Systems don't conflict, transitions are smooth

---

## THEN Phase 4: TickBossAI_V2 (After Phase 3.5)

**With rotation and dodge tested, Phase 4 becomes clean orchestration:**

**Incremental Build Order:**
1. **Idle → Attack** (simplest loop)
   - Boss waits until player in range
   - Calls SelectAttackFromIdle()
   - Plays attack
   - Returns to Idle
   - TEST THIS ALONE FIRST

2. **Add Engage Decision Function**
   - Routes to Attack/Reposition/Sprint based on conditions
   - Engage is function, not state (instant decision)

3. **Add Reposition State**
   - Calls SelectRotationAnimation() if player behind (TESTED ✓)
   - Calls ExecuteDodge() if player flanking (TESTED ✓)
   - Moves toward valid geometry

4. **Add Sprint State** (future)
   - Gap-closer when player far + cooldown ready
   - Sprint attacks on arrival

**State Machine Design (from earlier session):**
```
Idle → (player in range) → Engage()

Engage() decides:
  1. Valid attacks? → Attack state
  2. Sprint conditions? → Sprint state
  3. Else → Reposition state

Attack → (combo window) → Check geometry → Chain or Engage()
Reposition → (valid geometry) → Engage()
Sprint → (in range) → Attack (sprint family)
Flinch → Engage()
```

---

## Critical Debugging Patterns

**When Testing New Systems:**
1. Add comprehensive prints FIRST
2. Test in isolation with keyboard trigger
3. Validate visual feedback (debug visualization)
4. Check edge cases (wrapping angles, distance boundaries)
5. Only integrate after isolated validation

**Common Issues:**
- Angle calculations: Check -180 to 180 normalization
- Rotation direction: Verify sign (may need negation)
- Reference frames: World space vs Actor space
- Array.Contains() checks for multi-value matching
- AlreadyHitActors clearing (happens in ANS Begin, not End)

---

## Code Patterns & Best Practices

**ANS Pattern:**
- ANS = Flag setter only (bCanRotate, bInComboWindow)
- Logic lives in Blueprint functions (PerformBossRotationTracking, combo checking)
- This keeps logic visible and testable

**Data-Driven:**
- Attack properties in data tables, not hardcoded
- Enums as row keys for type safety
- Normalized database approach (transitions separate from attack properties)

**Separation of Concerns:**
- Selection functions query and filter
- Execution functions play montages and handle state
- Damage delivery via interface (boss reports, player calculates)

---

## Files & Locations

**Blueprints:**
- BP_EnemyBase (parent class, all core functions)
- BP_Boss_GothicKnight (child, sets WeaponMesh reference in Construction Script)

**AnimNotifyStates:**
- ANS_EnableEnemyHitbox (calls PerformAttackTrace every tick)
- ANS_ComboWindow (sets bInComboWindow flag)
- ANS_AllowBossRotation (sets bCanRotate flag)

**Documentation:**
- Debug_Visualization_System.md (complete visualization reference)
- Boss_Attack_System_Architecture.md (complete system documentation)
- Both in /mnt/user-data/outputs/

---

## Known Issues & Limitations

**Current Limitations:**
- Boss attacks only from Front cone (most attacks defined for Front only)
- No sprint attacks implemented yet (defined but not in transitions)
- No dodge counters tested yet (Attack_15/16 exist but not in transitions)
- Family-based selection not implemented (all attacks equal priority)
- No AI state machine (keyboard trigger only)

**Technical Quirks:**
- Draw Debug Cone direction needs × -1 when using Actor Forward
- UE5 float negation macro doesn't work reliably (use explicit multiply)
- Back cone center angle is special case (180° not average)
- Get Forward Vector (no target) ≠ Get Actor Forward Vector (with target)

---

## Success Criteria for Phase 3.5

**Rotation System:**
- ✅ Boss can execute 90° and 180° turns on command
- ✅ Ends facing player after turn
- ✅ Root motion rotation works correctly
- ✅ Debug visualization functions during/after turn
- ✅ Doesn't conflict with rotation tracking system

**Dodge System:**
- ✅ Boss can execute dodge animations
- ✅ Root motion repositions boss
- ✅ Can chain dodge → attack (Attack_15/16)
- ✅ Doesn't break existing systems

**Then Ready for Phase 4:** Build AI state machine using validated components

---

## Communication Tips for Next Claude

**Ilia appreciates:**
- Complete, uninterrupted explanations
- Architecture diagrams (text-based ASCII works)
- "Will this scale?" thinking
- Honest assessment of complexity vs benefit
- Examples with concrete values
- Step-by-step implementation plans

**Avoid:**
- Multiple options without recommendation
- Vague "you could try..." suggestions
- Oversimplification that hides complexity
- Assuming knowledge of game dev patterns
- Hiding limitations or risks

**When Ilia says "that's stupid" or similar:**
- He's frustrated with the situation, not you
- Usually means something genuinely is overcomplicated
- Look for simpler solution or validate complexity is necessary
- His instincts are good - listen when he pushes back

---

## Recent Session Highlights

**Major Wins:**
- Rebuilt entire boss AI from scratch (recognized V1 wasn't good enough)
- Implemented dual-socket hitbox system (works for weapon AND body attacks)
- Fixed root motion (EnemyMesh vs Mesh bug)
- Tuned distance bands from measured attack ranges (not guesses)
- Built working combo system with geometry validation
- Added rotation tracking (feels natural, not robotic)
- Created comprehensive documentation (100+ pages)

**Hard-Won Lessons:**
- "Working systems ≠ good game" (quality over completion)
- Test in isolation before integration (learned from V1 failure)
- Game feel is foundational (2 hours on rotation = worth it)
- Tight distance bands = smarter AI (200/400/650 vs 300/600/10000)
- Debug visualization is critical (can't tune what you can't see)

**Time Investment:**
- ~50 hours total on boss AI system
- ~5 hours on geometry system (angles, wrapping cones)
- ~2 hours fixing root motion bug
- ~2 hours on rotation tracking (including today's geometry hell)
- All documented for future reference

---

## Project Philosophy

**"Cornered Rat" Player Fantasy:**
- Dangerous but desperate
- Trapped in dungeon that denies death
- NOT heroic power fantasy
- Visceral survival combat

**Boss Design:**
- Gothic Knight = Dungeon Master construct
- Acts like MH monster (predator instincts) with humanoid animations
- Should feel like Anjanath with soldier animations
- Aggressive animal, not tactical duelist
- Phase 2 transforms to werewolf moveset (future)

**Development Approach:**
- Professional architecture despite being solo/new
- Systems must scale to 10 bosses
- Buy art, build systems (compensate for art weakness)
- January 5th playtest is validation milestone
- Quality over features (willing to rebuild if not right)

---

## Final Notes

Ilia is at a critical juncture: he has all the pieces working in isolation and is now ready to test rotation/dodge animations before full AI integration. He's learned from the V1 experience and wants incremental validation. Support this approach - it's the right instinct.

The January 5th deadline is real but achievable. Focus on getting autonomous combat working with current attacks (don't expand content yet). A boss that fights well with 6 attacks beats a boss that fights poorly with 18.

He works late hours (3-4 AM sessions common) but maintains good judgment. If he's frustrated with geometry or "hates" something, it usually means the complexity isn't justified - look for simpler solutions.

The rotation tracking took 2 hours to debug but "feels natural" now - that feeling is what matters. Game feel is the priority, not feature count.

**Good luck with Phase 3.5! The foundation is solid - just need to test rotation/dodge, then build the AI orchestration layer.**

---

**Handoff Complete**  
**- Claude (Outgoing Session)**
