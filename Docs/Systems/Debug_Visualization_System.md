# Debug Visualization System - Boss AI Geometry

## Overview

The debug visualization system provides real-time visual feedback of the boss AI's spatial awareness and geometry-based decision making. It renders geometric zones, attack ranges, and player position data to aid in development, testing, and tuning.

**Purpose:**
- Visualize the 12 geometric sectors (3 distance bands × 4 directional cones)
- Show which sector the player currently occupies
- Display optimal angle precision zones
- Debug attack selection geometry
- Validate hitbox traces
- Tune distance bands and angle requirements

---

## Enabling/Disabling Debug Mode

**Master Toggle:**
- **Variable:** `bShowDebugGeometry` (Boolean) in BP_EnemyBase
- **Default:** False (disabled)
- **Set at runtime:** Can be toggled via Blueprint logic or debug console

**What Gets Drawn When Enabled:**
1. Distance band circles (3 rings)
2. Cone boundary lines (4 radial lines)
3. Dynamic optimal angle wedge (changes with player position)
4. Weapon trace visualization (red/green spheres during attacks)
5. Socket location spheres (during attacks)

---

## Visualization Components

### 1. Distance Band Circles

**Function:** `DrawDebugSectorGrid()` (Part 1)
**Location:** BP_EnemyBase
**Called From:** Event Tick (when bShowDebugGeometry = True)

**What It Draws:**
Three concentric circles representing distance thresholds:

| Circle | Radius | Color | Represents |
|--------|--------|-------|------------|
| Inner | 200 units | Cyan (semi-transparent) | Close band boundary |
| Middle | 400 units | Yellow (semi-transparent) | Medium band boundary |
| Outer | 650 units | Red (semi-transparent) | Far band boundary |

**Implementation:**
```
ForEach row in DT_DistanceBands:
  Draw Debug Circle:
    - Center: Boss location
    - Radius: MaxDistance from table
    - Color: DebugColor from table
    - Thickness: 2
    - Duration: 0 (one frame)
    - Z offset: 10 (ground level)
```

**Data Source:** DT_DistanceBands
- Close: 0-200 units
- Medium: 200-400 units  
- Far: 400-650 units

---

### 2. Cone Boundary Lines

**Function:** `DrawDebugSectorGrid()` (Part 2)
**Location:** BP_EnemyBase
**Called From:** Event Tick (when bShowDebugGeometry = True)

**What It Draws:**
Four radial lines dividing the arena into directional cones:

| Line | Angle | Color | Represents |
|------|-------|-------|------------|
| Front/Right boundary | 45° (adjusted +180°) | Green | Right cone starts |
| Right/Back boundary | 135° (adjusted +180°) | Yellow | Back cone starts |
| Back/Left boundary | -135° (adjusted +180°) | Red | Left cone starts |
| Left/Front boundary | -45° (adjusted +180°) | Blue | Front cone starts |

**Implementation:**
```
ForEach cone in DT_AttackCones:
  Rotate forward vector by (MinAngle + 180°)
  Draw Debug Line:
    - Start: Boss location (Z=10)
    - End: Start + (Direction × 1000 units)
    - Color: DebugColor from table
    - Thickness: 3
    - Duration: 0
```

**Critical Detail:**
- +180° offset required due to Unreal's forward vector convention
- Without offset, lines point backward from boss perspective

**Data Source:** DT_AttackCones
- Front: -45° to 45°
- Right: 45° to 135°
- Back: 135° to -135° (wrapping)
- Left: -135° to -45°

---

### 3. Dynamic Optimal Angle Wedge

**Function:** `DrawDebugActiveHighlight()`
**Location:** BP_EnemyBase
**Called From:** Event Tick (when bShowDebugGeometry = True)

**What It Draws:**
A colored wedge showing:
- **Color:** Which sector player is in (Blue/Green/Red/Yellow)
- **Size:** Which optimal angle precision player satisfies (30°/60°/90°)

**Wedge Behavior:**

| Player Position | Wedge Size | Meaning |
|----------------|-----------|---------|
| ±0-15° from sector center | 30° narrow | Player in Narrow30 zone |
| ±15-30° from sector center | 60° moderate | Player in Moderate60 zone |
| ±30-45° from sector center | 90° wide | Player in Wide90 zone |
| >45° from sector center | No wedge | Player outside optimal angles |

**Color Coding:**

| Sector | Color | RGB Value |
|--------|-------|-----------|
| Front | Blue | (0, 0, 1, 0.5) |
| Right | Green | (0, 1, 0, 0.5) |
| Back | Red | (1, 0, 0, 0.5) |
| Left | Yellow | (1, 1, 0, 0.5) |

**Implementation:**
```
1. Get player sector and angle offset from sector center
2. Determine optimal angle met (Narrow/Moderate/Wide)
3. Calculate wedge direction (pointing at player)
4. Draw Debug Cone:
   - Origin: Boss location (Z=10)
   - Direction: Toward player
   - Angle Width: 30/60/90 based on precision
   - Color: Sector color
   - Alpha: 0.5 (semi-transparent)
```

**Use Case:**
- Instantly see which attacks are geometrically valid
- Narrow wedge = precise attacks available
- Wide wedge = only loose-angle attacks available
- No wedge = reposition needed

---

### 4. Sector Identification (HUD)

**Display Location:** On-screen HUD (if implemented)
**Updates:** Every frame

**Information Shown:**
```
Current Sector: [Front/Right/Back/Left]
Distance Band: [Close/Medium/Far]
Optimal Angle: [Narrow30/Moderate60/Wide90/None]
Distance: [XXX] units
```

**Variables Read:**
- `PlayerLocationCone` (E_AttackCone)
- `PlayerLocationSector` (E_DistanceBand)
- `PlayerLocationOptimalAngle` (E_OptimalAngle)
- `PlayerLocationAngle` (Float, -180 to 180)

---

### 5. Weapon Trace Visualization

**Function:** `PerformAttackTrace()`
**Location:** BP_EnemyBase
**Active:** During ANS_EnableEnemyHitbox (attack active frames only)

**What It Draws:**

**Sphere Trace:**
- **Red line:** Trace path from base socket to tip socket
- **Red spheres:** Trace radius (start/end points)
- **Green hit markers:** Where trace detected collision
- **Duration:** 0.1 seconds per frame

**Socket Debug (if enabled separately):**
- **Blue sphere:** Base socket location (weapon guard/foot/shoulder base)
- **Yellow sphere:** Tip socket location (weapon tip/toes/fist)
- **Radius:** 50 units (large for visibility)
- **Duration:** 2 seconds

**Configuration:**
```
Sphere Trace by Channel:
  - Draw Debug Type: For Duration (if bShowDebugGeometry)
  - Draw Debug Type: None (if bShowDebugGeometry = False)
  - Draw Time: 0.1
  - Trace Color: Red
  - Trace Hit Color: Green
```

**Use Case:**
- Verify hitboxes follow animations correctly
- Debug missed hits (trace not intersecting player)
- Validate socket placement on weapons/body parts
- Tune trace radius per attack type

---

## System Architecture

### Execution Flow

```
Event Tick (BP_EnemyBase):
  ↓
  Branch: bShowDebugGeometry?
  └─ True:
      ↓
      Update Player Position Variables:
        - GetPlayerSector() → PlayerLocationCone, PlayerLocationSector
        - GetPlayerOptimalAngle() → PlayerLocationOptimalAngle
        - Calculate PlayerLocationAngle
      ↓
      DrawDebugSectorGrid():
        - Draw 3 distance circles
        - Draw 4 cone boundary lines
      ↓
      DrawDebugActiveHighlight():
        - Calculate angle offset
        - Determine wedge size
        - Draw colored wedge
      ↓
      [Optional: Draw HUD text]

During Attack (ANS_EnableEnemyHitbox active):
  ↓
  Received_NotifyTick:
    ↓
    PerformAttackTrace():
      ↓
      Get socket locations
      ↓
      Sphere Trace (with debug drawing if enabled)
```

---

## Key Functions Reference

### GetPlayerSector()
**Returns:** (E_DistanceBand, E_AttackCone)
**Purpose:** Determines which of 12 base sectors player occupies
**Logic:**
1. Calculate distance to player
2. Query DT_DistanceBands to find matching band
3. Calculate angle to player (-180 to 180)
4. Query DT_AttackCones, use IsAngleInCone for each
5. Return both values

---

### GetPlayerOptimalAngle()
**Inputs:** CurrentCone (E_AttackCone), AngleToPlayer (Float)
**Returns:** E_OptimalAngle (Narrow30/Moderate60/Wide90/None)
**Purpose:** Determines angle precision player satisfies
**Logic:**
1. Get cone center angle from DT_AttackCones
2. Calculate player's offset from cone center
3. Normalize offset to -180 to 180 range
4. Check absolute offset against thresholds:
   - ≤15° → Narrow30
   - ≤30° → Moderate60
   - ≤45° → Wide90
   - >45° → None

---

### IsAngleInCone()
**Inputs:** AngleToPlayer (Float), MinAngle (Float), MaxAngle (Float)
**Returns:** Boolean
**Purpose:** Checks if angle falls within cone (handles wrapping)
**Special Case:** Back cone (135° to -135°) wraps around ±180° boundary
**Logic:**
```
IF MinAngle > MaxAngle (wrapping cone):
  RETURN (AngleToPlayer >= MinAngle) OR (AngleToPlayer <= MaxAngle)
ELSE (normal cone):
  RETURN (AngleToPlayer >= MinAngle) AND (AngleToPlayer <= MaxAngle)
```

---

### DrawDebugSectorGrid()
**Purpose:** Draws base geometric framework (circles + lines)
**Performance:** Lightweight, ~7 draw calls per frame
**Visual Impact:** Static background grid

---

### DrawDebugActiveHighlight()
**Purpose:** Draws dynamic player position indicator
**Performance:** Moderate, ~1-2 draw calls per frame
**Visual Impact:** Shows real-time position and precision

---

## Data Table Dependencies

### DT_DistanceBands
**Structure:** S_DistanceBandDefinition
**Used For:** Circle radii and colors

| Row | MinDistance | MaxDistance | DebugColor |
|-----|------------|-------------|------------|
| Close | 0 | 200 | Cyan |
| Medium | 200 | 400 | Yellow |
| Far | 400 | 650 | Red |

---

### DT_AttackCones
**Structure:** S_AttackConeDefinition
**Used For:** Line angles and colors

| Row | MinAngle | MaxAngle | DebugColor |
|-----|----------|----------|------------|
| Front | -45 | 45 | Blue |
| Right | 45 | 135 | Green |
| Back | 135 | -135 | Red |
| Left | -135 | -45 | Yellow |

---

## Performance Notes

**Frame Cost:**
- Distance circles: ~3 draw calls
- Cone lines: ~4 draw calls
- Active wedge: ~1 draw call
- Trace visualization: ~1 draw call (only during attacks)
- **Total:** ~8-9 draw calls per frame (negligible impact)

**Optimization:**
- All debug draws use Duration = 0 (one frame persistence)
- No persistent geometry (no memory accumulation)
- Disabled entirely when bShowDebugGeometry = False
- Trace debug only active during attack windows

**Recommended Usage:**
- Enable during development/testing
- Disable in shipping builds (Blueprint compile-out)
- Safe to leave enabled in PIE sessions

---

## Common Issues & Solutions

### Issue: Lines/circles not visible
**Solution:** Check bShowDebugGeometry is True, verify Z offset = 10 (ground level)

### Issue: Wedge pointing wrong direction
**Solution:** Verify +180° offset applied to angle calculations

### Issue: Wedge not appearing
**Solution:** Player might be >45° from sector center (outside optimal angles)

### Issue: Colors wrong or overlapping
**Solution:** Check DebugColor alpha values in data tables (should be 0.3-0.5)

### Issue: Trace not showing hits
**Solution:** Verify Draw Debug Type = "For Duration" in Sphere Trace node

### Issue: Performance drop with debug enabled
**Solution:** Reduce Draw Time duration or disable during intensive scenes

---

## Testing Checklist

**Visual Validation:**
- [ ] Distance circles visible and concentric
- [ ] Cone lines divide arena into 4 equal sectors
- [ ] Wedge changes color when crossing sector boundaries
- [ ] Wedge changes size when moving closer/farther from center
- [ ] Wedge disappears >45° from center
- [ ] Trace shows red line during attacks
- [ ] Trace shows green hit markers on player contact

**Functional Validation:**
- [ ] Front sector = Blue wedge
- [ ] Right sector = Green wedge
- [ ] Back sector = Red wedge
- [ ] Left sector = Yellow wedge
- [ ] Dead center = Narrow wedge (30°)
- [ ] Medium offset = Moderate wedge (60°)
- [ ] Edge offset = Wide wedge (90°)

**Edge Cases:**
- [ ] Back sector (wrapping cone) works correctly
- [ ] Transitions between sectors are smooth
- [ ] Very close range (<50 units) still renders
- [ ] Very far range (>1000 units) handles gracefully

---

## Future Enhancements

**Potential Additions:**
- Attack range visualization (draw optimal distance for selected attack)
- Valid attack indicators (show which attacks are currently available)
- Combo chain prediction (visualize potential combo paths)
- State machine status (show current AI state)
- Aggression meter (visual representation of pressure tracking)
- Attack telegraph indicators (show incoming attack zones)

**Configuration Options:**
- Toggle individual components (circles only, lines only, etc.)
- Adjustable transparency/brightness
- Color customization per boss
- Performance-optimized "minimal" mode

---

## Conclusion

The debug visualization system provides comprehensive real-time insight into boss AI geometry awareness. By rendering the 12-sector grid, dynamic player positioning, and attack trace data, developers can validate spatial logic, tune attack ranges, and debug selection issues without guesswork.

**Key Takeaway:** The visual feedback transforms abstract geometric calculations into immediately understandable visual information, drastically reducing iteration time during combat tuning.

---

**Document Version:** 1.0
**Last Updated:** December 2025
**Maintained By:** Project Team
