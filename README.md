# Boss Rush Action Game

Unreal Engine 5 boss rush game focusing on Monster Hunter-style combat with close over-the-shoulder camera (God of War 2018 style). The game rewards thoughtful input and punishes button mashing. Core fantasy: "cornered rat" — visceral survival combat, not heroic power fantasy.

## Project Status

- **Engine:** Unreal Engine 5.4
- **Development Stage:** Prototype (Pre-Alpha)
- **Target:** January 5th Friends & Family Playtest

## Core Pillars

1. **Perfect Timing Combat** — Deliberate, commitment-based mechanics
2. **Monster Hunter Style** — Tactical boss encounters (5-10 min each)
3. **Part-Break System** — Strategic targeting of boss parts (planned)
4. **Intimate Combat** — Close over-the-shoulder camera, visceral feel

## Current Features

### Player Systems ✅

- Data Driven Combo System
- Perfect timing with sound + glow feedback
- Directional dodge with i-frames (God of War style)
- Block with locomotion blend space
- Lock-on camera system
- Player damage system with mitigation (block, i-frames)

### Boss AI (Gothic Knight) ✅

- **Geometry-based attack selection** — 12-sector spatial awareness (4 distance bands × 4 directional cones)
- **Reactive decision making** — Boss evaluates attack options every frame
- **Movement system** — Sprint, Approach, Circle Left/Right, Backstep, Turn180
- **Combo chains** — Geometry-validated follow-up attacks
- **Dual-socket hitboxes** — Supports weapon and body attacks
- **Head tracking** — Boss watches player when not attacking
- **Rotation tracking** — Smooth rotation during movement and attack windup
- **9 attacks implemented** (of 18 defined)

### VFX ✅(not built yet)

- Sword trails (Niagara)
- Weapon glow (dynamic materials)
- Hit effects

### Audio ✅ (not built yet)

- Data-driven hit sounds per surface type
- Sound concurrency management

## Architecture

### Data-Driven Design

- **DT_Attacks** — Player attack properties
- **DT_Combos** — Player combo transitions
- **DT_GothicKnightAttacks_V2** — Boss attack definitions (18 rows)
- **DT_GothicKnightTransitions** — Boss attack chains
- **DT_DistanceBands** — Distance thresholds (Close/Medium/Far/Out)
- **DT_AttackCones** — Directional sectors (Front/Right/Back/Left)

### Key Blueprints

- **BP_ThirdPersonCharacter** — Player with combat systems
- **BP_EnemyBase** — Parent class for all bosses
- **BP_Boss_GothicKnight** — First boss implementation

### Boss AI V2 Architecture

```
Event Tick
    │
    ├─ bIsExecutingAttack? → Skip (montage playing)
    ├─ bIsExecutingRotation? → Skip (turn playing)
    │
    └─ RepositionForAttack()
            ├─ Attack valid? → ExecuteAttack()
            └─ Movement needed? → ExecuteMovement()
```

## Documentation

- [Systems Overview](Docs/SYSTEMS_OVERVIEW.md)
- [Boss Attack Architecture](Docs/Architecture/BossAttack_SystemArchitecture.md)
- [BP_EnemyBase Documentation](Docs/BP_EnemyBase_Documentation.md)
- [BossTickAI V2 Documentation](Docs/BossTickAI_V2_Documentation.md)

## Tech Stack

- **Engine:** Unreal Engine 5.6
- **Scripting:** Blueprints (100%)
- **Assets:** Marketplace + Mixamo (placeholder phase)
- **Budget:** $10k/year for production assets

## Development Philosophy

- **Buy art, build systems** — Compensate for art weakness with strong architecture
- **Data-driven** — Properties in tables, not hardcoded
- **Scalable** — Systems designed for 10+ bosses
- **Ship incrementally** — Working feature > perfect feature
- **Quality over features** — Willing to rebuild if feel is wrong

## Roadmap

### Done

- [x] Player combat (attacks, dodge, block)
- [x] Perfect timing system <-- Outdated
- [x] Lock-on camera <-- works, needs a lot of work to build camera logic per action
- [x] Boss AI V2 (geometry-aware, reactive)
- [x] Boss movement system
- [x] Combo chains (player + boss)
- [x] Hitbox system (player + boss)
- [x] Boss Head tracking

### In Progress

- [ ] Boss exhaustion system (combat rhythm)
- [ ] Player block animation polish
- [ ] Attack tuning (distances, damage, timing)

### Planned

- [ ] Boss flinch/stagger states
- [ ] Additional boss attacks (9 remaining)
- [ ] Simple arena level
- [ ] Boss phase 2 (werewolf moveset)
- [ ] Part-break system (probably for future bosses)

## Quick Links

- [Communication Guide (CLAUDE.md)](CLAUDE.md) — AI assistant preferences
- [Session Handoff Letters](Docs/Handoffs/) — Context for AI sessions

---

*Last Updated: December 11, 2025*
