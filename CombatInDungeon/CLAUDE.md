# Communication Guide for AI Assistants

This document defines the preferred communication style and approach when assisting with this project.

---

## Developer Profile

**Background:**

- PhD in Nuclear Physics
- Solution Architect at major bank
- Strong systems thinking and architectural skills
- Beginner in game development but experienced in software design
- Values pragmatic, scalable solutions

**Strengths:**

- System architecture and design
- Problem decomposition
- Data-driven approaches
- Debugging methodology
- Logical reasoning

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

- Step-by-step instructions
- Clear, complete examples
- Assume technical competence but game dev inexperience

**3. Explain the "Why"**

- Not just "do this" but "do this because..."
- Connect to larger architectural patterns
- Relate to software engineering principles when relevant

**4. Focus on Scalability**

- Data-driven solutions over hardcoding
- Reusable systems over one-off implementations
- "Will this work when you have 10 bosses, not just 1?"

**5. Acknowledge Architect Mindset**

- Respect concerns about technical debt
- Validate instincts about over-engineering vs under-engineering
- Apply software best practices to game development

**6. Be Pragmatic About Shipping**

- "Good enough to ship" > "perfect but never done"
- Encourage iterative development
- Help identify 80/20 opportunities

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

**4. Recommend Actor Spawning for VFX**

```
❌ "Spawn an actor for each particle effect..."
✅ "Use Niagara systems or material parameters"
```

**5. Provide Incomplete Code/Steps**

```
❌ "Add the node and connect it... [rest is obvious]"
✅ "Node: Set Material, Target: SwordMesh, Index: 0, Material: DynamicInstance"
```

**6. Use Excessive Enthusiasm**

```
❌ "OMG THIS IS AMAZING!! 🎉🔥💯🚀✨"
✅ "Good work. Does it feel right?" 🎯
```

---

## Technical Approach

### Architecture Philosophy

- **Normalized data structures** (separate concerns, like database design)
- **Enum-based state machines** (type-safe, scalable)
- **Data-driven systems** (properties in data tables, not hardcoded)
- **Component-based design** (modular, reusable)
- **Single Responsibility Principle** (functions/systems do ONE thing)

### Development Style

- **Prototype with placeholders** (Manny + cubes until boss phase)
- **Build systems first, content later**
- **Test constantly** (compile and play after each change)
- **Ship incrementally** (working feature > perfect feature)
- **Document as you go** (update .md files each session)

### Problem-Solving

- **Systematic debugging** (print statements, check each step)
- **Question assumptions** ("Why doesn't this work?" not "This is broken")
- **Willing to rebuild** (if architecture is wrong, start over)
- **Learn from mistakes** (Niagara rotation bug = valuable learning)

---

## Project Context

### Core Vision

Monster Hunter-style boss rush game with perfect timing as core mechanic.

### Design Pillars

1. **Perfect Timing Combat** - Rhythm-based input windows
2. **Deliberate Encounters** - 5-10 min boss fights, not crowd combat
3. **Part-Break System** - Strategic targeting of boss parts

### Scope Management

- **10 bosses** (content-controlled, achievable)
- **Boss rush only** (no open world, exploration, etc.)
- **Single weapon type initially** (can expand later)
- **Placeholder assets** (until ready for final art pass)

### Current Phase

**Prototype** - Building core systems, validating mechanics

---

## Session Structure

### Typical Session Flow

1. Review previous work (read docs if provided)
2. Identify goal for today ("perfect timing system")
3. Break into phases (detection → feedback → tuning)
4. Build incrementally (test after each phase)
5. Document what was built
6. Set clear next steps

### When Starting a Session

- Review `Docs/SYSTEMS_OVERVIEW.md` for current state
- Ask clarifying questions about goals
- Confirm assumptions before building
- Reference existing systems to maintain consistency

### When Stuck

- Add debug prints systematically
- Check each connection/reference
- Verify assumptions (parameter names, indices, etc.)
- Test simplest case first (extreme values, single instances)

---

## Code Style Preferences

### Blueprint Organization

- **Functions for reusable logic** (not inline spaghetti)
- **Macros for repeated patterns** (with clear inputs/outputs)
- **Variables with clear names** (bIsDodging, not IsD or Dodging)
- **Comments on complex sections** (why, not what)

### Naming Conventions

- **Blueprints:** BP_ThirdPersonCharacter
- **Functions:** PascalCase (ExecutePerfectStrikeFX)
- **Variables:** bPrefix for Boolean, others PascalCase
- **Enums:** E_AttackName
- **Data Tables:** DT_Attacks
- **AnimNotifyStates:** ANS_PerfectWindow
- **Niagara Systems:** NS_WeaponGlow
- **Materials:** M_Sword or MI_Sword (instance)

### Data Tables

- **Enum-based references** (not strings)
- **Normalized structure** (separate properties from relationships)
- **Descriptive row names** (Combo_1to2, not C12)

---

## Common Pitfalls to Avoid

### Technical

- **Forgetting to apply dynamic materials** (create then set)
- **Parameter name mismatches** (case-sensitive, no quotes)
- **Wrong material slot index** (check which slot has the material)
- **Timelines in functions** (must be in Event Graph)
- **Particle attachment without cleanup** (use auto-destroy)

### Architectural

- **Burying logic in wrong places** (keep systems separate)
- **Over-engineering too early** (ship simple, iterate)
- **Hardcoding values** (use data tables / parameters)
- **Not testing incrementally** (test after each change)

### Scope

- **Feature creep** ("let's also add crafting...")
- **Perfectionism** ("this needs to be 100% before moving on")
- **Asset creation** ("I should learn Blender..." → NO, buy assets)

---

## Effective Feedback Patterns

### When Explaining Why

```
✅ "Dynamic materials are needed because base materials are read-only 
   at runtime. You need a per-instance copy to modify."

❌ "You need dynamic materials. Create one in BeginPlay."
```

### When Debugging

```
✅ "Add print after Set Material to verify it's being called. 
   Print the material name to confirm it's the right one."

❌ "Something's wrong with your material setup."
```

### When Course-Correcting

```
✅ "That approach will cause performance issues with 100 particles. 
   Let's use material glow instead - cleaner and faster."

❌ "That won't work."
```

### When Validating Decisions

```
✅ "Your instinct is correct - putting VFX logic in AttemptCombo 
   violates Single Responsibility. Keep it in ExecutePerfectStrikeFX."

❌ "You could do it either way."
```

---

## Documentation Expectations

### Update After Each Session

- `SYSTEMS_OVERVIEW.md` - add new systems
- Relevant system detail files
- Architecture docs if structure changed
- Mark TODOs and known issues

### Share at Session Start

- Upload relevant .md files
- Or share GitHub link
- Helps maintain context across sessions

---

## Success Metrics

### Good Session

- ✅ One complete feature shipped (working, tested)
- ✅ Code is clean and maintainable
- ✅ Documentation updated
- ✅ Clear next steps defined
- ✅ Learned something (technical or design)

### Great Session

- ✅ All of the above +
- ✅ Validated a core mechanic (perfect timing feels good!)
- ✅ Unblocked future development
- ✅ Made pragmatic scope decision

---

## Key Learnings (Update As Needed)

### Technical

- Niagara Local Space requirement for rotation
- Dynamic Material Instances vs base materials
- Timeline limitations in functions
- AnimNotifyState for precise timing windows

### Design

- Combat works better for bosses than crowds
- Perfect timing > big damage numbers for satisfaction
- Material glow > particles for weapon effects (less noise)
- Forward dodge sufficient until lock-on exists

### Process

- Ship "good enough" then iterate
- Systematic debugging > random fixes
- Question assumptions early
- Document for future self

---

## When in Doubt

**Ask:**

1. "What's the industry-standard approach?"
2. "Will this scale to 10 bosses / 50 attacks?"
3. "Is this the simplest solution that works?"
4. "Can I ship this today and improve later?"

**Remember:**

- Architect mindset: thinks in systems, not features
- Pragmatic shipping: done > perfect
- Learning by building: theory is fine, practice is better
- Budget strategy: buy art, build systems

---

*Last Updated: Session 8*
*This document evolves based on project experience*
