# Communication Guide for AI Assistants

This document defines the preferred communication style and approach when assisting with this project.

---

## Developer Profile

**Background:**
- PhD in Nuclear Physics
- Solution Architect at Alfabank
- Strong systems thinking and architectural skills
- ~2 months into game development, rapidly advancing
- Values pragmatic, scalable solutions

**Strengths:**
- System architecture and design
- Problem decomposition
- Data-driven approaches
- Debugging methodology
- Logical reasoning
- Learning from reference games (2000+ hours Monster Hunter)

**Budget Strategy:**
- $10k/year for assets (art, animation, sound)
- Focus development time on systems, not art creation
- Buy assets to cover weaknesses, build custom systems for strengths

---

## Communication Preferences

### ✅ DO

**1. Give ONE Best Practice Approach First**
- Don't offer 5 different options upfront
- Lead with the industry-standard / best-practice solution
- Only present alternatives if the first approach has issues

**2. Be Direct and Systematic**
- Step-by-step instructions with complete node details
- Clear, complete examples (not "add the node and connect it...")
- Assume technical competence but verify UE5 specifics

**3. Explain the "Why"**
- Not just "do this" but "do this because..."
- Connect to larger architectural patterns
- Relate to software engineering principles when relevant

**4. Focus on Scalability**
- Data-driven solutions over hardcoding
- Reusable systems over one-off implementations
- "Will this work when you have 10 bosses, not just 1?"

**5. Acknowledge When Lost**
- If the conversation gets confused, STOP and reset
- Restate the current state and what we're building
- Don't keep layering confusion on confusion

**6. Document As You Go**
- Suggest documentation when progress was significant
- Ask if you think that it's time for documenting
- Update documentation when systems are complete
- Create handoff letters for session continuity on demand
- Keep BP_EnemyBase_Documentation.md current

---

### ❌ DON'T

**1. Present Multiple Options Simultaneously**
```
❌ "You could do A, B, C, D, or E..."
✅ "The best approach is A. If you hit issues, we'll try B."
```

**2. Over-Explain Basic Concepts**
```
❌ "A variable is a container that stores..."
✅ "Create variable: bIsDodging (Boolean)"
```

**3. Suggest Placeholder Art/Animation Creation**
```
❌ "You could learn Blender and model the sword..."
✅ "Buy sword pack from asset store, focus on systems"
```

**4. Provide Incomplete Node Instructions**
```
❌ "Add the node and connect it... [rest is obvious]"
✅ "Get Unit Direction: From = Get Actor Location (Self), To = Get Actor Location (PlayerRef)"
```

**5. Keep Building When Architecture Is Unclear**
```
❌ Adding more features when basic flow is confused
✅ "Let me reset — here's what exists and what we're building"
```

**6. Duplicate Logic Across Functions**
```
❌ Calculate direction in ExecuteMovement AND Event Tick
✅ Clear ownership: ExecuteMovement sets state, Event Tick executes
```

---

## Project Context

### Core Vision
Monster Hunter-style boss rush game with "cornered rat" player fantasy — visceral survival combat, not heroic power fantasy. Close over-the-shoulder camera, commitment-based mechanics, perfect timing as core pillar.

### Current State (December 2025)
- **Player:** Combat complete (attacks, dodge, block, i-frames)
- **Boss AI V2:** Geometry-aware, reactive, threatening
- **Gothic Knight:** 9 of 18 attacks implemented
- **Feel:** Boss is "T-1000 killing machine" — needs exhaustion system for rhythm

### Architecture Pattern
```
Event Tick
    └─ RepositionForAttack() — DECIDES
            ├─ ExecuteAttack() — Attack state
            └─ ExecuteMovement() — Movement state
```

### Key Systems Built
- 12-sector geometry detection (4 bands × 4 cones)
- Randomized attack selection with geometry validation
- Movement system (Sprint, Approach, Circle, Backstep, Turn180)
- Head tracking with smooth interpolation
- Dual-socket hitbox traces
- Combo chains with geometry re-validation

---

## Session Structure

### When Starting a Session
1. Read provided documentation (handoff letter, relevant .md files)
2. Ask clarifying questions about goals
3. Confirm current state before building
4. Reference existing systems to maintain consistency

### When Building Features
1. State the architecture clearly before coding
2. Build in testable increments
3. Use debug visualization and prints
4. Test each piece before integration

### When Confused or Frustrated
- Developer says "that's stupid" or "I'm confused" → STOP
- Restate what exists and what we're building
- Look for simpler solution
- Don't keep layering complexity

### When Documenting
- Update BP_EnemyBase_Documentation.md for new functions/variables
- Create system documentation when complete
- Write handoff letter at session end

---

## Technical Patterns

### Blueprint Organization
- **Functions for reusable logic** (SelectAttackFromIdle, ExecuteMovement)
- **Clear ownership** (who calculates what, who stores what)
- **State flags over enums** for simple states (bIsExecutingAttack)
- **Enums for complex states** (E_BossMovementType)

### Naming Conventions
- **Blueprints:** BP_ThirdPersonCharacter, BP_EnemyBase
- **Functions:** PascalCase (ExecuteMovement, RepositionForAttack)
- **Variables:** bPrefix for Boolean, others PascalCase
- **Enums:** E_AttackName, E_BossMovementType
- **Data Tables:** DT_GothicKnightAttacks_V2
- **AnimNotifyStates:** ANS_ComboWindow, ANS_EnableEnemyHitbox

### Data Tables
- Enum-based row names (type-safe)
- Normalized structure (transitions separate from attack properties)
- Boss-specific tables (DT_GothicKnightAttacks_V2, not DT_BossAttacks)

### Common Gotchas
- **Orient Rotation to Movement** = FALSE when doing manual rotation
- **RInterpTo** for smooth rotation (not direct Set Actor Rotation)
- **Get Unit Direction** = cleaner than subtract + normalize
- **Cross Product** order matters: (A, B) vs (B, A) = opposite directions

---

## Key Learnings (Updated December 2025)

### Technical
- Geometry-based AI > scripted patterns
- Movement direction must update every frame for tracking
- Engine settings (Orient Rotation) can override manual control
- Head tracking makes boss feel alive with minimal effort

### Design
- "Working systems ≠ good game" — feel matters
- Study reference games actively (Doshaguma research)
- Attacks should hide rotation, not turn-then-attack
- Combat needs rhythm — restless boss is exhausting, not fun

### Process
- Test in isolation before integration (learned from V1 failure)
- Document when complete, not "later"
- Reset and restate when confused
- Simple random before weighted random

---

## Immediate Priorities (January Playtest)

| Priority | Task | Status |
|----------|------|--------|
| 1 | Boss exhaustion system | Not started |
| 2 | Player block animation fix | Not started |
| 3 | Simple arena | Not started |
| 4 | Attack tuning | Ongoing |

**Not needed for playtest:** Additional bosses, part-break, UI polish, sound polish, multiple arenas.

---

*Last Updated: December 11, 2025*
*Session: Claude-13*
