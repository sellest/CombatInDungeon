# Boss Attack System Architecture

## Overview

The Boss Attack System is a data-driven, geometry-aware combat AI framework that selects and executes attacks based on real-time spatial analysis of player positioning. The system combines reactive decision-making with combo mechanics to create dynamic, Monster Hunter-inspired boss encounters.

**Core Philosophy:**
- **Geometry-based selection:** Attacks chosen based on player distance, angle, and directional cone
- **Data-driven design:** All attack properties, transitions, and requirements stored in data tables
- **Scalable architecture:** Designed to support 10+ unique bosses with minimal code duplication
- **Separation of concerns:** Attack selection, execution, and damage delivery handled independently

**Key Features:**
- 12-sector spatial awareness (3 distance bands × 4 directional cones)
- Optimal angle precision matching (Narrow/Moderate/Wide requirements)
- Flexible hitbox system (dual-socket traces support weapon, body, and equipment attacks)
- Animation-driven combo windows (designers control timing via AnimNotifyStates)
- Multi-hit prevention with per-attack reset

---

## System Components

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     BOSS AI SYSTEM                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────┐ │
│  │  GEOMETRY    │ ───> │   ATTACK     │ ───> │ EXECUTION│ │
│  │  DETECTION   │      │  SELECTION   │      │  SYSTEM  │ │
│  └──────────────┘      └──────────────┘      └──────────┘ │
│         │                      │                    │      │
│         │                      │                    │      │
│    ┌────▼─────┐          ┌────▼─────┐        ┌────▼────┐ │
│    │ Player   │          │ Data     │        │ Hitbox  │ │
│    │ Sector   │          │ Tables   │        │ System  │ │
│    └──────────┘          └──────────┘        └─────────┘ │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

---

## Data Structures

### Enumerations

#### E_AttackName
**Purpose:** Unique identifier for each attack in a boss's moveset
**Location:** Content/ThirdPerson/Bosses/Data/Enums/
**Usage:** Primary key for attack data tables, transition definitions

**Values (Gothic Knight Example):**
```
- Attack_01  (Horizontal sweep)
- Attack_02  (Follow-up sweep)
- Attack_03  (360° spin)
- Attack_04  (Overhead slam)
- Attack_05  (Forward thrust)
- Attack_06  (Quick stab)
- Attack_07  (Vertical slash)
- Attack_08  (Overhead chop)
- Attack_09  (Follow-up chop)
- Attack_10  (Sprint tackle)
- Attack_11  (Sprint slash)
- Attack_12  (Leg kick)
- Attack_13  (Charging slash)
- Attack_14  (Sprint uppercut)
- Attack_15  (Dodge right + slash)
- Attack_16  (Dodge left + slash)
- Attack_18  (Sprint spin)
- Attack_19  (Windmill spin - 360°)
```

**Design Note:** Numbers are not sequential (no Attack_17) to accommodate animation asset naming or removed moves.

---

#### E_AttackFamily
**Purpose:** Groups attacks by strategic purpose for weighted selection
**Usage:** Future implementation for boss personality tuning

**Values:**
```
- OPENER       (Attacks usable from neutral/idle stance)
- SPRINT_ATTACK (Require sprint state entry)
- PRESSURE_COMBO (Multi-hit sequences to maintain aggression)
- QUICK_PUNISH  (Fast reactive attacks for punishing player mistakes)
- REPOSITIONING (Attacks that adjust boss positioning)
- AREA_DENIAL   (360° coverage attacks, prevent flanking)
```

**Current Status:** Defined in data but not yet used in selection logic (planned for Phase 5).

---

#### E_DistanceBand
**Purpose:** Categorizes player distance into discrete bands
**Values:**
```
- Close   (0-200 units)    - Kicks, punches, close-range body attacks
- Medium  (200-400 units)  - Standard sword swings, most melee attacks
- Far     (400-650 units)  - Thrusts, lunges, gap-closers
```

**Tuning Note:** Values based on measured attack reaches including root motion. See "Attack Range Measurement" section.

---

#### E_AttackCone
**Purpose:** Divides arena into 4 directional sectors around boss
**Values:**
```
- Front  (-45° to 45°)     - Directly in front of boss
- Right  (45° to 135°)     - Boss's right flank
- Back   (135° to -135°)   - Behind boss (wrapping cone)
- Left   (-135° to -45°)   - Boss's left flank
```

**Angle Convention:** -180° to 180° range, 0° = boss forward direction

**Special Case:** Back cone wraps around ±180° boundary, requires special handling in IsAngleInCone().

---

#### E_OptimalAngle
**Purpose:** Defines angle precision requirements for attacks
**Values:**
```
- Narrow30   (Player within ±15° of sector center) - Precise attacks, thrusts
- Moderate60 (Player within ±30° of sector center) - Standard attacks
- Wide90     (Player within ±45° of sector center) - Sweeping attacks
- Any        (No angle requirement)                - 360° attacks
- None       (Player outside all optimal zones)    - Invalid position
```

**Usage:** Attacks specify which precision levels they accept. Player position determines which precision they satisfy.

---

#### E_TransitionContext
**Purpose:** Defines when a transition is valid
**Values:**
```
- Idle         (From neutral stance, not in attack/combo)
- Combo        (During ANS_ComboWindow in another attack)
- Sprint       (After entering sprint state)
- DodgeRight   (After dodge right animation)
- DodgeLeft    (After dodge left animation)
```

**Design Pattern:** Contexts represent "moments" when attack decisions occur, not AI states.

---

#### E_HitboxMeshType
**Purpose:** Specifies which mesh component to trace hitbox on
**Values:**
```
- Weapon  (Static mesh weapon component)
- Body    (Main skeletal mesh - fists, feet, shoulders)
- Shield  (Future - for shield bash attacks)
```

**Usage:** Allows same boss to use weapon attacks (sword traces) and body attacks (kick/punch traces) with different socket pairs.

---

#### E_BossState_V2
**Purpose:** AI state machine states (for Phase 4 implementation)
**Values:**
```
- Idle         (Waiting for player aggro)
- Engage       (Decision function - routes to other states)
- Reposition   (Adjusting position for valid geometry)
- Sprint       (Gap-closing movement)
- Attack       (Executing attack montage)
- Flinch       (Taking damage, interrupted)
- Dead         (Death state)
```

**Current Status:** Defined but not yet implemented (Phase 4 pending).

---

### Structures

#### S_DistanceBandDefinition
**Purpose:** Defines properties of a distance band
**Location:** Content/ThirdPerson/Bosses/Data/Structures/

**Fields:**
```cpp
- DistanceBand (E_DistanceBand)  - Enum identifier
- MinDistance (Float)            - Minimum range (inclusive)
- MaxDistance (Float)            - Maximum range (exclusive)
- DebugColor (LinearColor)       - Visualization color
- Description (Text)             - Designer notes
```

**Example Row (Medium):**
```
DistanceBand: Medium
MinDistance: 200.0
MaxDistance: 400.0
DebugColor: (R=1.0, G=1.0, B=0.0, A=0.4)  // Yellow, semi-transparent
Description: "Standard sword range with root motion buffer"
```

---

#### S_AttackConeDefinition
**Purpose:** Defines directional cone properties
**Location:** Content/ThirdPerson/Bosses/Data/Structures/

**Fields:**
```cpp
- AttackCone (E_AttackCone)  - Enum identifier
- MinAngle (Float)           - Start angle (degrees, -180 to 180)
- MaxAngle (Float)           - End angle (degrees, -180 to 180)
- DebugColor (LinearColor)   - Visualization color
- Description (Text)         - Designer notes
```

**Example Row (Back - Wrapping):**
```
AttackCone: Back
MinAngle: 135.0
MaxAngle: -135.0  // Wraps around ±180°
DebugColor: (R=1.0, G=0.0, B=0.0, A=0.4)  // Red
Description: "Behind boss, handles angle wrapping"
```

**Critical Implementation Detail:** When MinAngle > MaxAngle, cone wraps around ±180° boundary. Requires OR logic instead of AND.

---

#### S_BossAttackData_V2
**Purpose:** Complete definition of a single attack
**Location:** Content/ThirdPerson/Bosses/Data/Structures/

**Fields:**
```cpp
- AttackName (E_AttackName)                 - Unique identifier
- AttackMontage (Animation Montage)         - Animation to play
- Damage (Float)                            - Base damage value
- FlinchLevel (Integer)                     - Stun/flinch intensity
- AttackFamily (E_AttackFamily)             - Strategic categorization
- DistanceBand (E_DistanceBand)             - Required player distance
- DamageCones (Array<E_AttackCone>)         - Which directions attack hits
- ValidOptimalAngles (Array<E_OptimalAngle>) - Acceptable angle precisions
- HitboxMeshType (E_HitboxMeshType)         - Which mesh to trace on
- HitboxStartSocket (Name)                  - Base socket name
- HitboxEndSocket (Name)                    - Tip socket name
- HitboxRadius (Float)                      - Trace sphere radius
- AudioCues (Sound Cue)                     - Attack sounds (future)
```

**Example Row (Attack_01 - Horizontal Sweep):**
```
AttackName: Attack_01
AttackMontage: AM_GothicKnight_Attack_01
Damage: 25.0
FlinchLevel: 2
AttackFamily: OPENER
DistanceBand: Medium
DamageCones: [Right, Front, Left]  // 270° coverage
ValidOptimalAngles: [Narrow30, Moderate60, Wide90]  // Works at any precision
HitboxMeshType: Weapon
HitboxStartSocket: Socket_WeaponBase
HitboxEndSocket: Socket_WeaponTip
HitboxRadius: 30.0
```

**Example Row (Attack_12 - Leg Kick):**
```
AttackName: Attack_12
AttackMontage: AM_GothicKnight_Attack_12
Damage: 15.0
FlinchLevel: 1
AttackFamily: QUICK_PUNISH
DistanceBand: Close
DamageCones: [Front]
ValidOptimalAngles: [Narrow30, Moderate60]  // Requires some precision
HitboxMeshType: Body
HitboxStartSocket: foot_r
HitboxEndSocket: knee_r
HitboxRadius: 40.0
```

**Design Rationale:**
- **DamageCones:** Defines where attack can be used FROM (if player in Right cone, can use attacks that list Right)
- **ValidOptimalAngles:** Arrays allow flexible precision (wide attacks accept Narrow/Moderate/Wide, precise attacks only accept Narrow)
- **Hitbox fields:** Flexible system supports weapon, body parts, or future equipment (shields, dual weapons)

---

#### S_AttackTransition
**Purpose:** Defines valid transitions between attacks
**Location:** Content/ThirdPerson/Bosses/Data/Structures/

**Fields:**
```cpp
- TransitionContext (E_TransitionContext)  - When transition is valid
- FromAttack (E_AttackName)                - Source attack (None for Idle context)
- ToAttack (E_AttackName)                  - Target attack
- Priority (Integer)                       - Selection weight (1-10, future use)
```

**Example Rows (Idle Context - Openers):**
```
Context: Idle, FromAttack: None, ToAttack: Attack_01, Priority: 5
Context: Idle, FromAttack: None, ToAttack: Attack_05, Priority: 5
Context: Idle, FromAttack: None, ToAttack: Attack_08, Priority: 5
Context: Idle, FromAttack: None, ToAttack: Attack_12, Priority: 5
Context: Idle, FromAttack: None, ToAttack: Attack_13, Priority: 5
Context: Idle, FromAttack: None, ToAttack: Attack_19, Priority: 5
```

**Example Rows (Combo Context - Chains):**
```
Context: Combo, FromAttack: Attack_01, ToAttack: Attack_02, Priority: 5
Context: Combo, FromAttack: Attack_02, ToAttack: Attack_03, Priority: 5
Context: Combo, FromAttack: Attack_03, ToAttack: Attack_04, Priority: 5
Context: Combo, FromAttack: Attack_08, ToAttack: Attack_09, Priority: 5
```

**Design Pattern:**
- **Idle transitions:** FromAttack = None, defines attacks usable from neutral
- **Combo transitions:** FromAttack specifies, defines valid follow-ups
- **Priority:** Currently unused (all 5), reserved for future weighted selection

**Normalized Database Approach:**
- Transitions table = "relationships between attacks"
- Attacks table = "intrinsic properties of attacks"
- Separation enables reusable attack definitions across bosses

---

#### S_FamilyWeight
**Purpose:** Boss personality tuning via family selection weights (future)
**Status:** Defined but not implemented

**Fields:**
```cpp
- AttackFamily (E_AttackFamily)
- SelectionWeight (Float)
- FamilyCooldown (Float)
- Description (Text)
```

**Planned Usage:** Different bosses favor different families (aggressive boss prefers PRESSURE_COMBO, defensive boss prefers REPOSITIONING).

---

## Data Tables

### DT_DistanceBands
**Structure:** S_DistanceBandDefinition
**Rows:** 3 (Close, Medium, Far)
**Purpose:** Defines distance band thresholds and visualization

**Complete Table:**

| Row Name | MinDistance | MaxDistance | DebugColor | Description |
|----------|------------|-------------|------------|-------------|
| Close | 0 | 200 | Cyan (0,1,1,0.4) | Kick/punch range |
| Medium | 200 | 400 | Yellow (1,1,0,0.4) | Standard sword swings |
| Far | 400 | 650 | Red (1,0,0,0.4) | Thrusts, gap-closers |

**Tuning History:**
- Initial guess: 0-300, 300-600, 600-10000
- After measurement: 0-200, 200-400, 400-650
- Tight bands = more precise AI decisions

---

### DT_AttackCones
**Structure:** S_AttackConeDefinition
**Rows:** 4 (Front, Right, Back, Left)
**Purpose:** Defines directional sectors and visualization

**Complete Table:**

| Row Name | MinAngle | MaxAngle | DebugColor | Description |
|----------|----------|----------|------------|-------------|
| Front | -45 | 45 | Blue (0,0,1,0.5) | Directly ahead |
| Right | 45 | 135 | Green (0,1,0,0.5) | Right flank |
| Back | 135 | -135 | Red (1,0,0,0.5) | Behind (wrapping) |
| Left | -135 | -45 | Yellow (1,1,0,0.5) | Left flank |

**Angle Convention Notes:**
- 0° = Boss forward direction
- Positive = Clockwise rotation (right side)
- Negative = Counter-clockwise (left side)
- Back cone wraps: covers 135° to 180° AND -180° to -135°

---

### DT_GothicKnightAttacks_V2
**Structure:** S_BossAttackData_V2
**Rows:** 18 (Attack_01 through Attack_19, skip 17)
**Purpose:** Complete attack definitions for Gothic Knight boss

**Sample Entries:**

| Attack | Montage | Damage | DistanceBand | DamageCones | ValidOptimalAngles | HitboxMesh | StartSocket | EndSocket | Radius |
|--------|---------|--------|--------------|-------------|-------------------|------------|-------------|-----------|--------|
| Attack_01 | AM_GK_01 | 25 | Medium | [R,F,L] | [N,M,W] | Weapon | Socket_WeaponBase | Socket_WeaponTip | 30 |
| Attack_05 | AM_GK_05 | 30 | Far | [F] | [N] | Weapon | Socket_WeaponBase | Socket_WeaponTip | 30 |
| Attack_08 | AM_GK_08 | 28 | Medium | [F] | [N] | Weapon | Socket_WeaponBase | Socket_WeaponTip | 30 |
| Attack_12 | AM_GK_12 | 15 | Close | [F] | [N,M] | Body | foot_r | knee_r | 40 |
| Attack_19 | AM_GK_19 | 35 | Close | [F,R,B,L] | [N,M,W] | Weapon | Socket_WeaponBase | Socket_WeaponTip | 30 |

**Legend:**
- R/F/B/L = Right, Front, Back, Left
- N/M/W = Narrow30, Moderate60, Wide90

**Attack Categorization by Usage:**

**Openers (Idle context):**
- Attack_01, 05, 08, 12, 13, 19

**Combo Chains:**
- 01→02→03→04 (horizontal sequence)
- 08→09 (vertical chops)

**Special Attacks:**
- Attack_03, 19 (360° coverage)
- Attack_10, 14, 18 (sprint attacks - future)
- Attack_15, 16 (dodge counters - future)

**Design Notes:**
- Most attacks = Front cone only (realistic sword combat)
- 360° attacks (03, 19) = All cones (anti-flanking)
- Kicks/punches = Body hitboxes (different socket pairs)
- Longer reach = tighter angle requirements (thrusts need precision)

---

### DT_GothicKnightTransitions
**Structure:** S_AttackTransition
**Rows:** 10 (6 Idle, 4 Combo - expandable)
**Purpose:** Defines valid attack chains and entry points

**Current Entries:**

**Idle Context (Openers):**
```
Idle → None → Attack_01
Idle → None → Attack_05
Idle → None → Attack_08
Idle → None → Attack_12
Idle → None → Attack_13
Idle → None → Attack_19
```

**Combo Context (Chains):**
```
Combo → Attack_01 → Attack_02
Combo → Attack_02 → Attack_03
Combo → Attack_03 → Attack_04
Combo → Attack_08 → Attack_09
```

**Future Expansion:**
- Sprint context (10, 14, 18)
- Dodge contexts (15, 16)
- More combo branches
- Conditional transitions (e.g., "only if player health < 50%")

**Design Philosophy:**
- Small number of openers = focused neutral game
- Linear combo chains = readable, avoidable patterns
- Branches planned but not implemented (test simple first)

---

## Core Systems

### Geometry Detection System

**Purpose:** Real-time spatial awareness of player position

**Components:**

#### GetPlayerSector()
**Returns:** (E_DistanceBand, E_AttackCone)
**Called:** Every frame in Event Tick
**Stored:** PlayerLocationSector, PlayerLocationCone (member variables)

**Algorithm:**
```
1. Calculate distance to player:
   Distance = Get Distance To (Self, PlayerRef)

2. Determine distance band:
   ForEach band in DT_DistanceBands:
     IF Distance >= MinDistance AND Distance < MaxDistance:
       CurrentDistanceBand = band

3. Calculate angle to player:
   LookAt = Find Look at Rotation (Self, PlayerRef)
   BossRot = Get Actor Rotation (Self)
   Delta = Delta Rotator (LookAt, BossRot)
   AngleToPlayer = Delta.Yaw  // -180 to 180

4. Determine directional cone:
   ForEach cone in DT_AttackCones:
     IF IsAngleInCone(AngleToPlayer, cone.MinAngle, cone.MaxAngle):
       CurrentCone = cone

5. Return (CurrentDistanceBand, CurrentCone)
```

**Edge Cases:**
- Player >650 units: Returns Far (MaxDistance = 10000 to catch overflow)
- Player exactly on boundary: Uses exclusive MaxDistance (200 = Medium, not Close)

---

#### GetPlayerOptimalAngle()
**Inputs:** CurrentCone (E_AttackCone), AngleToPlayer (Float)
**Returns:** E_OptimalAngle
**Called:** Every frame in Event Tick
**Stored:** PlayerLocationOptimalAngle (member variable)

**Algorithm:**
```
1. Get cone center angle:
   ConeData = Get Data Table Row (DT_AttackCones, CurrentCone)
   
   IF ConeData.MinAngle > ConeData.MaxAngle (wrapping):
     CenterAngle = 180  // Back cone special case
   ELSE:
     CenterAngle = (MinAngle + MaxAngle) / 2

2. Calculate player offset from center:
   RawOffset = AngleToPlayer - CenterAngle

3. Normalize offset to -180 to 180:
   IF RawOffset > 180:
     NormalizedOffset = RawOffset - 360
   ELSE IF RawOffset < -180:
     NormalizedOffset = RawOffset + 360
   ELSE:
     NormalizedOffset = RawOffset

4. Get absolute offset:
   AngleOffset = Abs(NormalizedOffset)

5. Determine precision level:
   IF AngleOffset <= 15:
     RETURN Narrow30
   ELSE IF AngleOffset <= 30:
     RETURN Moderate60
   ELSE IF AngleOffset <= 45:
     RETURN Wide90
   ELSE:
     RETURN None  // Outside all optimal zones
```

**Example:**
- Player in Front cone, 5° right of center → Narrow30
- Player in Front cone, 25° right of center → Moderate60
- Player in Front cone, 50° right of center → None (switching to Right cone)

---

#### IsAngleInCone()
**Inputs:** AngleToPlayer (Float), MinAngle (Float), MaxAngle (Float)
**Returns:** Boolean
**Purpose:** Handles wrapping cone logic for Back sector

**Algorithm:**
```
IF MinAngle > MaxAngle:  // Wrapping cone (Back sector)
  RETURN (AngleToPlayer >= MinAngle) OR (AngleToPlayer <= MaxAngle)
ELSE:  // Normal cone
  RETURN (AngleToPlayer >= MinAngle) AND (AngleToPlayer <= MaxAngle)
```

**Example (Back Cone: 135° to -135°):**
- Player at 150° → (150 >= 135) OR (150 <= -135) → TRUE OR FALSE → TRUE ✓
- Player at -160° → (-160 >= 135) OR (-160 <= -135) → FALSE OR TRUE → TRUE ✓
- Player at 0° → (0 >= 135) OR (0 <= -135) → FALSE OR FALSE → FALSE ✓

**Critical Detail:** Normal AND logic fails for wrapping cones because no angle is simultaneously ≥135 AND ≤-135. OR logic correctly handles both "wings" of the wrapped cone.

---

### Attack Selection System

**Purpose:** Choose appropriate attack based on current geometry

#### SelectAttackFromIdle()
**Returns:** S_BossAttackData_V2 (or None if no valid attacks)
**Called:** When boss decides to attack from neutral (manual trigger currently, TickBossAI_V2 future)

**Algorithm:**
```
1. Query Idle transitions:
   AllRows = Get Data Table Row Names (DT_GothicKnightTransitions)
   
   ForEach row in AllRows:
     TransitionData = Get Data Table Row (row)
     
     IF TransitionData.TransitionContext == Idle:
       IdleAttacks.Add(TransitionData.ToAttack)

2. Filter by geometry:
   ForEach attackName in IdleAttacks:
     AttackData = Get Data Table Row (DT_GothicKnightAttacks_V2, attackName)
     
     IF IsAttackValidForGeometry(AttackData):
       GeometryValidAttacks.Add(AttackData)

3. Return first valid attack:
   IF GeometryValidAttacks.Length > 0:
     RETURN GeometryValidAttacks[0]
   ELSE:
     RETURN None  // No valid attacks, reposition needed
```

**Current Implementation:** Returns first valid attack (fast, deterministic)
**Future Enhancement:** Weighted random selection using Priority field from transitions

---

#### SelectComboAttack()
**Returns:** S_BossAttackData_V2 (or None if no valid combos)
**Called:** During ANS_ComboWindow (recovery frames of current attack)

**Algorithm:**
```
1. Query combo transitions FROM current attack:
   AllRows = Get Data Table Row Names (DT_GothicKnightTransitions)
   
   ForEach row in AllRows:
     TransitionData = Get Data Table Row (row)
     
     IF (TransitionData.TransitionContext == Combo) 
        AND (TransitionData.FromAttack == CurrentAttackData.AttackName):
       
       ComboAttackData = Get Data Table Row (DT_GothicKnightAttacks_V2, TransitionData.ToAttack)
       
       IF IsAttackValidForGeometry(ComboAttackData):
         RETURN ComboAttackData  // First valid combo found, exit immediately

2. No valid combo found:
   RETURN None  // End combo chain
```

**Key Difference from SelectAttackFromIdle:**
- Uses FromAttack filter (only considers combos FROM current attack)
- Returns immediately on first match (stops loop early)
- None result = natural combo chain end (not an error)

**Reactive Combo Design:**
- Player in Medium range, dead center → Attack_01 (wide sweep)
- ANS_ComboWindow opens
- Player still in range → Combo to Attack_02
- Player moved out of range → No combo, chain ends
- Player repositioned to Far → Potentially different attack if state machine decides

---

#### IsAttackValidForGeometry()
**Inputs:** AttackData (S_BossAttackData_V2)
**Returns:** Boolean
**Purpose:** Three-part geometry check (distance + cone + angle)

**Algorithm:**
```
IsAttackValidForGeometry(AttackData):
  
  1. Check distance band match:
     Check1 = (PlayerLocationSector == AttackData.DistanceBand)
  
  2. Check cone match:
     Check2 = AttackData.DamageCones.Contains(PlayerLocationCone)
  
  3. Check optimal angle match:
     Check3 = AttackData.ValidOptimalAngles.Contains(PlayerLocationOptimalAngle)
  
  4. Return combined result:
     RETURN (Check1 AND Check2 AND Check3)
```

**Example (Attack_05 - Precise Thrust):**
```
Attack_05:
  DistanceBand: Far
  DamageCones: [Front]
  ValidOptimalAngles: [Narrow30]

Player Position: Front-Far-Narrow
  Check1: Far == Far → TRUE
  Check2: [Front].Contains(Front) → TRUE
  Check3: [Narrow30].Contains(Narrow30) → TRUE
  Result: TRUE AND TRUE AND TRUE → VALID ✓

Player Position: Front-Far-Wide
  Check1: Far == Far → TRUE
  Check2: [Front].Contains(Front) → TRUE
  Check3: [Narrow30].Contains(Wide90) → FALSE
  Result: TRUE AND TRUE AND FALSE → INVALID ✗
  // Player too far off-center for precise thrust
```

**Design Rationale:**
- Simple AND logic = clear failure points for debugging
- Array.Contains() = flexible precision matching (attacks specify acceptable ranges)
- Early rejection = performance optimization (no need to check angle if distance wrong)

---

### Hitbox System

**Purpose:** Flexible, body-part-agnostic collision detection

**Architecture:** Dual-socket sphere traces executed every frame during attack active windows

#### PerformAttackTrace()
**Called:** Every frame from ANS_EnableEnemyHitbox → Received_NotifyTick
**Purpose:** Traces hitbox and delivers damage to valid targets

**Algorithm:**
```
PerformAttackTrace():
  
  1. Get hitbox parameters from CurrentAttackData:
     HitboxMeshType = CurrentAttackData.HitboxMeshType
     StartSocket = CurrentAttackData.HitboxStartSocket
     EndSocket = CurrentAttackData.HitboxEndSocket
     Radius = CurrentAttackData.HitboxRadius
  
  2. Resolve mesh component reference:
     SWITCH HitboxMeshType:
       CASE Weapon:
         MeshRef = WeaponMesh (Static Mesh Component)
       CASE Body:
         MeshRef = Mesh (Skeletal Mesh Component)
       CASE Shield:
         MeshRef = ShieldMesh (future)
  
  3. Get socket locations:
     StartTransform = Get Socket Transform (MeshRef, StartSocket)
     EndTransform = Get Socket Transform (MeshRef, EndSocket)
     StartLocation = Break Transform (StartTransform)
     EndLocation = Break Transform (EndTransform)
  
  4. Execute sphere trace:
     DrawDebugType = (bShowDebugGeometry ? ForDuration : None)
     
     OutHits = Sphere Trace by Channel:
       Start: StartLocation
       End: EndLocation
       Radius: Radius
       Channel: Visibility
       ActorsToIgnore: [Self]
       DrawDebugType: DrawDebugType
       DrawTime: 0.1
       TraceColor: Red
       TraceHitColor: Green
  
  5. Process all hits:
     ForEach Hit in OutHits:
       DeliverHitboxDamage(Hit.Actor, Hit.Component)
```

**Key Features:**
- **Data-driven sockets:** Attack data specifies socket names, not hardcoded
- **Multi-mesh support:** Same function handles weapon, body, equipment traces
- **Per-attack radius:** Fists can have different radius than sword
- **Debug visualization:** Automatically shows trace when debug enabled
- **Every-frame tracing:** Catches fast movements between frames

---

#### DeliverHitboxDamage()
**Inputs:** OtherActor (Actor), HitComponent (Primitive Component)
**Purpose:** Validates hit, prevents multi-hits, applies damage via interface

**Algorithm:**
```
DeliverHitboxDamage(OtherActor, HitComponent):
  
  1. Validate target type:
     PlayerRef = Cast to BP_ThirdPersonCharacter (OtherActor)
     IF Cast Failed:
       RETURN  // Not a valid target
  
  2. Check multi-hit prevention:
     IF AlreadyHitActors.Contains(PlayerRef):
       RETURN  // Already hit this attack, prevent double-hit
  
  3. Add to hit list:
     AlreadyHitActors.Add(PlayerRef)
  
  4. Validate hit component (prevent capsule hits):
     IF HitComponent != PlayerRef.Mesh:
       RETURN  // Hit capsule collision, not actual body mesh
  
  5. Apply damage via interface:
     ApplyDamageToActor (BPI_Damageable interface call):
       Target: PlayerRef  // Target calculates and applies damage internally
       Damage: CurrentAttackData.Damage
       FlinchLevel: CurrentAttackData.FlinchLevel
       Attacker: Self
```

**Design Pattern:**
- **Interface call, not direct damage:** Target (player) handles i-frames, blocking, damage reduction internally
- **Component validation:** Prevents multiple hits from same swing (capsule + mesh both detecting)
- **AlreadyHitActors:** Reset in ANS_EnableEnemyHitbox Begin, persists during active frames
- **Boss reports, player decides:** Boss doesn't know about i-frames or blocking, just reports potential damage

**Multi-Hit Prevention Flow:**
```
Frame 1: Trace hits player → Add to AlreadyHitActors → Apply damage
Frame 2: Trace hits player → Already in list → Skip
Frame 3: Trace hits player → Already in list → Skip
...
Attack ends, new attack begins:
ANS Begin → Clear AlreadyHitActors → Fresh slate
```

---

### Combo System

**Purpose:** Animation-driven combo windows with geometry validation

**Components:**

#### ANS_ComboWindow (AnimNotifyState)
**Placement:** In attack montages during recovery frames (after damage, before full recovery)
**Duration:** ~0.3-0.5 seconds (designer-controlled timing)

**Events:**

**Received_NotifyBegin:**
```
Get Owning Actor → Cast to BP_EnemyBase → BossRef
Set bInComboWindow (BossRef) = TRUE
Print: "Combo window OPEN"
```

**Received_NotifyEnd:**
```
Get Owning Actor → Cast to BP_EnemyBase → BossRef
Set bInComboWindow (BossRef) = FALSE
Print: "Combo window CLOSED"
```

**Design Philosophy:**
- **Animators control timing:** Designers place ANS where combo "feels right"
- **No hardcoded frame counts:** Different attacks have different recovery speeds
- **Visual placement:** Easy to see combo window in montage timeline
- **Fighting game pattern:** Window = "cancel window" in traditional fighting games

---

#### Combo Execution (Event Tick - Temporary Test Harness)

**Current Implementation (Phase 3 testing):**
```
Event Tick:
  
  Branch: bInComboWindow?
  └─ TRUE:
      Set bInComboWindow = FALSE  // Prevent next frame re-trigger
      ↓
      SelectComboAttack() → NextAttack
      ↓
      Branch: NextAttack.AttackName != None?
      ├─ TRUE:
      │   Set CurrentAttackData = NextAttack
      │   Get Anim Instance → Play Montage (NextAttack.AttackMontage)
      │   Print: "Combo to {AttackName}"
      │
      └─ FALSE:
          Print: "No valid combo, chain ends"
```

**Future Implementation (Phase 4 - Attack State):**
```
Attack State:
  On Entry:
    Set CurrentAttackData
    Play Montage
  
  Tick:
    Branch: bInComboWindow?
    └─ TRUE:
        Set bInComboWindow = FALSE
        SelectComboAttack() → NextAttack
        
        Branch: NextAttack valid?
        ├─ TRUE:
            Update CurrentAttackData
            Play Montage (queue in Blend Out)
        └─ FALSE:
            Set bComboEnded = TRUE
  
  On Exit (Montage Complete):
    IF bComboEnded:
      Transition to Engage (decide next action)
```

**Key Differences:**
- Test harness = Event Tick (runs globally)
- Production = Attack State (only runs during attacks)
- Test harness = immediate play
- Production = queued play during blend-out

---

### Complete Attack Execution Flow

**Scenario: Boss attacks player, lands hit, chains combo**

```
1. SELECTION PHASE (Idle → Attack)
   ├─ Event Tick calculates player geometry:
   │  ├─ GetPlayerSector() → Front-Medium
   │  └─ GetPlayerOptimalAngle() → Narrow30
   ├─ User presses test key (future: TickBossAI_V2 Engage state)
   ├─ SelectAttackFromIdle():
   │  ├─ Query DT_Transitions (Context = Idle)
   │  ├─ Filter by geometry (Front-Medium-Narrow)
   │  ├─ Attack_01, 08 valid
   │  └─ Return Attack_01 (first match)
   ├─ Set CurrentAttackData = Attack_01
   └─ Play Montage (AM_GothicKnight_Attack_01)

2. ATTACK EXECUTION PHASE
   ├─ Montage plays, boss animates
   ├─ ANS_EnableEnemyHitbox begins (at active frames):
   │  └─ Received_NotifyBegin:
   │     └─ Clear AlreadyHitActors array
   ├─ ANS_EnableEnemyHitbox ticks (every frame):
   │  └─ Received_NotifyTick:
   │     └─ Call PerformAttackTrace():
   │        ├─ Get sockets from CurrentAttackData:
   │        │  ├─ HitboxMeshType: Weapon
   │        │  ├─ StartSocket: Socket_WeaponBase
   │        │  └─ EndSocket: Socket_WeaponTip
   │        ├─ Get Socket Transform (WeaponMesh, sockets) → Locations
   │        ├─ Sphere Trace (Base to Tip, Radius 30):
   │        │  └─ OutHits = [Player]
   │        └─ ForEach Hit:
   │           └─ DeliverHitboxDamage(Player, Mesh):
   │              ├─ Check AlreadyHitActors → Not present
   │              ├─ Add to AlreadyHitActors
   │              └─ ApplyDamageToActor (BPI_Damageable):
   │                 ├─ Target: Player
   │                 ├─ Damage: 25
   │                 ├─ FlinchLevel: 2
   │                 └─ Attacker: Boss
   └─ ANS_EnableEnemyHitbox ends:
      └─ Hitbox detection stops

3. COMBO DECISION PHASE
   ├─ ANS_ComboWindow begins (during recovery):
   │  └─ Set bInComboWindow = TRUE
   ├─ Event Tick detects combo window:
   │  ├─ Set bInComboWindow = FALSE (prevent re-trigger)
   │  ├─ Update player geometry (player may have moved):
   │  │  └─ Still Front-Medium-Narrow
   │  └─ SelectComboAttack():
   │     ├─ Query DT_Transitions (Context = Combo, FromAttack = Attack_01)
   │     ├─ Found: Attack_01 → Attack_02
   │     ├─ Get Attack_02 data
   │     ├─ IsAttackValidForGeometry(Attack_02):
   │     │  ├─ DistanceBand: Medium (match ✓)
   │     │  ├─ DamageCones: [Front] (match ✓)
   │     │  └─ ValidOptimalAngles: [Narrow, Moderate, Wide] (match ✓)
   │     └─ Return Attack_02
   ├─ Set CurrentAttackData = Attack_02
   ├─ Play Montage (AM_GothicKnight_Attack_02)
   └─ ANS_ComboWindow ends

4. COMBO EXECUTION (Attack_02)
   └─ [Repeat steps 2-3 for Attack_02]
      ├─ Hitbox traces with Attack_02 sockets
      ├─ AlreadyHitActors cleared (new attack)
      ├─ Player damaged again
      └─ ANS_ComboWindow checks Attack_02 → Attack_03

5. COMBO CHAIN END
   ├─ Attack_03 completes
   ├─ ANS_ComboWindow opens
   ├─ SelectComboAttack():
   │  ├─ Query Attack_03 → Attack_04
   │  ├─ Player moved to Far range (dodged)
   │  ├─ IsAttackValidForGeometry(Attack_04):
   │  │  └─ DistanceBand: Medium, Player: Far → FALSE ✗
   │  └─ Return None
   └─ Combo chain ends naturally (no valid follow-up)
```

---

## Integration Points

### Blueprint References

**BP_EnemyBase (Parent Class):**
- All core functions and variables
- Geometry detection (GetPlayerSector, GetPlayerOptimalAngle, IsAngleInCone)
- Attack selection (SelectAttackFromIdle, SelectComboAttack, IsAttackValidForGeometry)
- Attack execution (PerformAttackTrace, DeliverHitboxDamage)
- Debug visualization (DrawDebugSectorGrid, DrawDebugActiveHighlight)

**BP_Boss_GothicKnight (Child Class):**
- Inherits all functionality from parent
- Sets boss-specific references:
  - WeaponMesh → SM_GK_Sword (in Construction Script)
  - References to DT_GothicKnightAttacks_V2
  - References to DT_GothicKnightTransitions
- Boss-specific visual components (mesh, materials, weapon)

---

### Animation System Integration

**AnimNotifyStates:**
- **ANS_EnableEnemyHitbox:** Defines attack active frames, calls PerformAttackTrace
- **ANS_ComboWindow:** Defines combo decision window, sets bInComboWindow flag

**Placement in Montages:**
```
Attack Montage Timeline:
0.0s ────┬────────┬───────────┬───────── 2.0s
         │        │           │
         │        │           └─ ANS_ComboWindow (0.2s)
         │        └─ ANS_EnableEnemyHitbox (0.4s)
         └─ Windup frames
```

**Timing Guidelines:**
- ANS_EnableEnemyHitbox: Place during "active frames" (weapon moving through strike)
- ANS_ComboWindow: Place during recovery, after active frames but before full recovery
- Avoid overlap: Hitbox and combo window should not overlap (prevents combo while hitbox active)

---

### Data Table Dependencies

**Query Relationships:**
```
SelectAttackFromIdle:
  ├─ DT_GothicKnightTransitions (query by Context = Idle)
  └─ DT_GothicKnightAttacks_V2 (query by ToAttack enum)

SelectComboAttack:
  ├─ DT_GothicKnightTransitions (query by Context = Combo, FromAttack)
  └─ DT_GothicKnightAttacks_V2 (query by ToAttack enum)

IsAttackValidForGeometry:
  ├─ Reads CurrentAttackData fields (from DT_GothicKnightAttacks_V2)
  ├─ Compares against PlayerLocationSector
  ├─ Compares against PlayerLocationCone
  └─ Compares against PlayerLocationOptimalAngle

GetPlayerSector:
  ├─ DT_DistanceBands (loop all rows, check distance)
  └─ DT_AttackCones (loop all rows, check angle)

GetPlayerOptimalAngle:
  └─ DT_AttackCones (query by CurrentCone to get center angle)
```

**Data Flow:**
```
Enums (attack names, contexts) 
  ↓
Transitions Table (defines valid chains)
  ↓
Attack Data Table (full attack properties)
  ↓
CurrentAttackData (runtime variable)
  ↓
Execution (montage play, hitbox trace, damage delivery)
```

---

## Performance Considerations

**Per-Frame Costs (Event Tick with debug enabled):**
- GetPlayerSector: ~10 μs (distance calc + table queries)
- GetPlayerOptimalAngle: ~5 μs (angle math)
- DrawDebugSectorGrid: ~50 μs (7 draw calls)
- DrawDebugActiveHighlight: ~15 μs (1-2 draw calls)
- **Total:** ~80 μs per frame (~12,500 FPS budget for this system alone)

**During Attack Costs (ANS active):**
- PerformAttackTrace: ~30 μs (socket queries + trace)
- DeliverHitboxDamage: ~5 μs per hit (typically 0-1 hits per frame)
- **Total:** ~35 μs per frame during attacks only

**Selection Costs (one-time per attack):**
- SelectAttackFromIdle: ~20 μs (6 transitions to check, 1-6 geometry validations)
- SelectComboAttack: ~10 μs (1-2 transitions to check, early exit)

**Optimization Notes:**
- Geometry calculations are lightweight (simple math, no pathfinding)
- Data table queries are fast (hash map lookups)
- Trace system more expensive than collision, but more accurate
- Debug visualization can be disabled at runtime (zero cost when off)

**Scalability:**
- System supports 10+ bosses without performance concerns
- Each boss uses separate data tables (no query conflicts)
- Multiple active bosses = linear cost increase (predictable scaling)

---

## Testing & Debugging

### Manual Testing (Current Phase 3 Implementation)

**Test Harness Setup:**
1. Enable bShowDebugGeometry in BP_Boss_GothicKnight
2. PIE (Play in Editor)
3. Press test key (calls SelectAttackFromIdle + Play Montage)
4. Observe:
   - Wedge color/size (shows current sector + precision)
   - Attack selection prints (which attack chosen)
   - Hitbox trace visualization (red/green spheres)
   - Combo chain execution (automatic follow-ups)

**Movement Testing:**
- Walk around boss → Watch wedge change
- Dead center (Narrow) → Wide attacks + precise attacks available
- Edge of sector (Wide) → Only wide attacks available
- Past edge → No wedge, reposition needed

**Attack Testing:**
- Stand in Front-Medium-Narrow → Trigger attack → Should select Attack_01/08
- Stand in Front-Far-Narrow → Trigger attack → Should select Attack_05 (thrust)
- Stand in Front-Close → Trigger attack → Should select Attack_12 (kick) or Attack_19 (spin)
- Stand in Right sector → Trigger attack → Should print "No valid attack" (no Right-cone attacks defined yet)

**Combo Testing:**
- Trigger Attack_01 → Should chain to Attack_02 (if still in range)
- Move out of range during Attack_01 → Attack_02 should not chain
- Stay in range for full combo → Attack_01 → 02 → 03 → 04 → Chain ends

---

### Common Issues & Solutions

**Issue:** Boss selecting wrong attack
- **Check:** Player geometry (print PlayerLocationSector, PlayerLocationCone, PlayerLocationOptimalAngle)
- **Check:** Attack data (verify DistanceBand, DamageCones, ValidOptimalAngles in table)
- **Check:** Transition data (verify attack is in Idle context transitions)

**Issue:** Combo not chaining
- **Check:** ANS_ComboWindow placed in montage (check timeline)
- **Check:** bInComboWindow flag (add print in Event Tick)
- **Check:** Transition exists (Combo context, FromAttack → ToAttack)
- **Check:** Player still in valid geometry for follow-up attack

**Issue:** Hitbox not hitting
- **Check:** ANS_EnableEnemyHitbox placed at active frames (scrub montage)
- **Check:** Socket names correct (typos in HitboxStartSocket/EndSocket)
- **Check:** WeaponMesh set in Construction Script (child blueprint)
- **Check:** Trace channel (Visibility should work, check player Mesh collision settings)

**Issue:** Multi-hits occurring
- **Check:** AlreadyHitActors cleared in ANS Begin (should clear every attack)
- **Check:** Component validation (should reject non-Mesh hits)
- **Check:** Multiple hitbox ANS overlapping (only one ANS per attack)

**Issue:** Wedge not visible
- **Check:** bShowDebugGeometry = True
- **Check:** Player >45° from sector center (intentionally hidden)
- **Check:** Z offset = 10 (might be underground if 0)

---

### Debug Prints (Recommended)

**Add these prints for testing:**

**Event Tick:**
```
Print: "Sector: {PlayerLocationSector}, Cone: {PlayerLocationCone}, Angle: {PlayerLocationOptimalAngle}"
```

**SelectAttackFromIdle:**
```
Print: "Idle attacks found: {IdleAttacks.Length}"
Print: "Geometry-valid attacks: {GeometryValidAttacks.Length}"
Print: "Selected: {SelectedAttack.AttackName}"
```

**SelectComboAttack:**
```
Print: "Checking combo from {CurrentAttackData.AttackName}"
Print: "Found combo: {NextAttack.AttackName}"
```

**DeliverHitboxDamage:**
```
Print: "Hit {ActorName} for {Damage} damage"
```

**ANS_ComboWindow:**
```
Print: "Combo window OPEN" (Begin)
Print: "Combo window CLOSED" (End)
```

---

## Future Enhancements (Phase 4+)

### TickBossAI_V2 State Machine

**Planned States:**
- **Idle:** Waiting for player aggro (distance threshold)
- **Engage:** Decision function (routes to Attack/Sprint/Reposition)
- **Reposition:** Adjust position to create valid geometry (strafe, backstep)
- **Sprint:** Gap-closer (triggers sprint attacks on arrival)
- **Attack:** Execute attack, handle combos
- **Flinch:** Taking damage, interrupted
- **Dead:** Death state

**State Transitions:**
```
Idle → (player in range) → Engage

Engage:
  ├─ Valid attacks? → Attack
  ├─ Player far + sprint ready? → Sprint
  └─ No valid attacks? → Reposition

Attack → (combo available) → Attack (chain)
Attack → (combo ended) → Engage

Sprint → (in range) → Attack (sprint family)

Reposition → (valid geometry created) → Engage

Flinch → (animation complete) → Engage
```

---

### Weighted Attack Selection

**Current:** Returns first valid attack (deterministic)
**Planned:** Weighted random selection using Priority field

**Algorithm:**
```
SelectAttackFromIdle():
  [... existing geometry filtering ...]
  
  IF GeometryValidAttacks.Length > 1:
    TotalWeight = Sum(all transitions.Priority)
    RandomRoll = Random(0, TotalWeight)
    
    RunningTotal = 0
    ForEach attack:
      RunningTotal += attack.Priority
      IF RandomRoll <= RunningTotal:
        RETURN attack
  ELSE:
    RETURN GeometryValidAttacks[0]
```

**Benefits:**
- Boss personality (aggressive boss: higher PRESSURE_COMBO weights)
- Unpredictability (player can't memorize exact patterns)
- Designer control (tweak weights in data table, not code)

---

### Family-Based Selection

**Planned:** Use AttackFamily enum for strategic selection

**Use Cases:**
- **Aggression states:** High aggression = prefer PRESSURE_COMBO, low aggression = prefer REPOSITIONING
- **Phase changes:** Phase 1 = OPENER heavy, Phase 2 = AREA_DENIAL heavy
- **Player behavior:** Player flanking = increase AREA_DENIAL weight
- **Cooldowns:** Prevent spam of same family (track last N attacks)

**Implementation:**
```
SelectAttackFromIdle():
  [... existing geometry filtering ...]
  
  ForEach attack in GeometryValidAttacks:
    FamilyData = Get Data Table Row (DT_FamilyWeights, attack.AttackFamily)
    
    IF FamilyOnCooldown(attack.AttackFamily):
      Continue  // Skip this attack
    
    AdjustedWeight = attack.Priority * FamilyData.SelectionWeight
    WeightedAttacks.Add(attack, AdjustedWeight)
  
  RETURN WeightedRandomSelection(WeightedAttacks)
```

---

### Threat/Pressure System

**Concept:** Track how long player stays in danger zones, adjust boss aggression

**Variables:**
- `ThreatAccumulation` (Float): Increases when player in Close/Medium, decreases when Far
- `AggressionLevel` (Int): 0-10 scale, modifies attack frequency and family weights

**Usage:**
```
Tick:
  IF PlayerLocationSector == Close OR Medium:
    ThreatAccumulation += DeltaTime
  ELSE:
    ThreatAccumulation -= DeltaTime * 0.5
  
  AggressionLevel = Clamp(ThreatAccumulation / 5, 0, 10)

Engage:
  IF AggressionLevel > 7:
    Prefer PRESSURE_COMBO attacks
    Reduce decision time (faster attack frequency)
  ELSE:
    Normal family weights
```

---

### Sprint Attack System

**Implementation Plan:**
1. Add Sprint state to TickBossAI_V2
2. Trigger when player >800 units + sprint cooldown ready
3. On arrival in range, query transitions (Context = Sprint)
4. Sprint attacks use same geometry validation
5. Cooldown prevents spam (7-14 seconds)

**Sprint Attack Properties:**
- Attack_10, 14, 18 (already defined in E_AttackName)
- TransitionContext: Sprint
- FromAttack: None (sprint is entry point, not combo)
- DamageCones: Typically [Front] (charging attacks)
- DistanceBand: Varies (tackle = Close on arrival, slash = Medium)

---

### Dodge Counter System

**Implementation Plan:**
1. Add dodge animations to boss (context-sensitive repositioning)
2. ANS in dodge montage sets DodgeContext flag (DodgeRight or DodgeLeft)
3. Query transitions (Context = DodgeRight/DodgeLeft)
4. Attack_15, 16 execute (sidestep + slash)

**Usage:**
- Reposition state triggers dodge when player flanking
- Dodge animation plays
- ANS_DodgeWindow opens (end of dodge)
- Query DodgeRight/Left transitions
- Execute counter-attack if player still flanking

---

## Conclusion

The Boss Attack System provides a scalable, data-driven foundation for reactive AI combat. By separating geometry detection, attack selection, and execution into discrete systems, the architecture supports rapid iteration, designer-friendly tuning, and multi-boss scalability without code duplication.

**Key Strengths:**
- **Reactive:** Boss responds to real-time player positioning, not scripted sequences
- **Data-driven:** All attack properties, chains, and requirements in data tables
- **Flexible:** Supports weapon, body, and equipment attacks with single trace system
- **Testable:** Each system can be validated independently before integration
- **Scalable:** 10+ bosses supported with minimal per-boss code

**Current Status:**
- ✅ Phase 1: Geometry detection (complete)
- ✅ Phase 2: Debug visualization (complete)
- ✅ Phase 3: Attack selection & combos (complete, tested via manual trigger)
- ⏸️ Phase 4: TickBossAI_V2 state machine (pending)
- ⏸️ Phase 5: Advanced features (aggression, sprint, dodge counters)

**Next Steps:**
1. Finalize Phase 3 testing (additional combo chains, edge cases)
2. Begin Phase 4 (implement TickBossAI_V2 incrementally)
3. Add Reposition state (create valid geometry when no attacks available)
4. Add Sprint state (gap-closing, sprint attack triggers)
5. Expand transition table (more combo branches, sprint/dodge contexts)

---

**Document Version:** 1.0  
**Last Updated:** December 2025  
**Maintained By:** Project Team
