# Perfect Timing System

**Status:** ✅ Complete  
**Dependencies:** Combat System, Animation System, Material System  

---

## Overview

The Perfect Timing system rewards precise player input during combo windows with enhanced visual and audio feedback. It's the core skill expression mechanic of the game.

**Design Goal:** Create rhythm-based combat where mastery comes from learning attack timings, not button mashing.

---

## Components

### AnimNotifyState

**File:** `ANS_PerfectWindow`  
**Location:** `/Content/Animations/Notifies/`  

**Purpose:** Defines the timing window within attack animations where perfect inputs are recognized.

**Implementation:**

- **Notify Begin:** Sets `bPerfectWindowActive = True`
- **Notify End:** Sets `bPerfectWindowActive = False`
- Typical duration: 0.15-0.25 seconds
- Placement: Based on Developer's feel and experience, should be set precisely via playtest

**Usage:** Place in attack montages at the precise timing window for each attack.

---

### Blueprint Variables

**In BP_ThirdPersonCharacter:**

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `bPerfectWindowActive` | Boolean | False | Is timing window currently open? |
| `bLastAttackWasPerfect` | Boolean | False | Was last attack input perfect? (for damage calc) |
| `SwordMaterialDynamic` | Material Instance Dynamic | None | Runtime-modifiable weapon material |

---

### Blueprint Functions

#### `ExecutePerfectStrikeFX()`

**Returns:** Void

```
Triggers all perfect timing feedback:
- Plays success sound (ding)
- Triggers weapon glow timeline
- (Future: camera pulse, particles, etc.)
```

**Location:** BP_ThirdPersonCharacter  
**Called By:** Input events when CheckPerfectTiming() returns true

**Implementation:**

```
1. Play Sound 2D: SFX_PerfectHit
2. Play from Start: TL_WeaponGlow (timeline)
3. Timeline Update pin → Set Scalar Parameter Value (EmissiveIntensity)
```

---

### Timeline

**Name:** `TL_WeaponGlow`  
**Location:** Event Graph (cannot be in functions)  
**Duration:** 0.3 seconds  
**Looping:** No  

**Float Track: GlowIntensity**

| Time (s) | Value | Description |
|----------|-------|-------------|
| 0.0 | 0.0 | Start: No glow |
| 0.05 | 0.25 | Peak: Bright flash |
| 0.3 | 0.0 | End: Fade complete |

**Curve Shape:** Quick spike up, smooth ease-out down

**Wired To:**

- Update pin → Set Scalar Parameter Value
  - Target: SwordMaterialDynamic
  - Parameter Name: "EmissiveIntensity"
  - Value: GlowIntensity (track output)

---

### Material System

#### Sword Material Setup

**Base Material:** `Sword2` (or current weapon material)  
**Location:** Varies per weapon asset  

**Required Parameters:**

- `EmissiveIntensity` (Scalar Parameter, default 0.0)
- `EmissiveColor` (Vector Parameter, optional - default white)

**Graph Structure:**

```
[EmissiveIntensity (Scalar)] ──┐
                                ├──> [Multiply] ──> Emissive Color
[EmissiveColor (Vector)]    ────┘
```

**Dynamic Instance Creation:**

In `Event BeginPlay`:

```
1. Get SwordMesh component
2. Get Material (Index 0)
3. Create Dynamic Material Instance (Parent: base material)
4. Set SwordMaterialDynamic variable
5. Set Material (apply dynamic instance to mesh)
```

**Critical:** Dynamic material must be applied back to mesh with `Set Material`, otherwise changes won't be visible.

---

### Audio

**Sound:** `SFX_PerfectHit`  
**Location:** `/Content/Audio/` (placeholder)  
**Source:** Music sample library (glockenspiel C6)  
**Duration:** ~0.3 seconds  
**Volume:** 0.7 (70% max)  

**Characteristics:**

- High pitch (success feeling)
- Short, bright
- Clean decay
- Not harsh or piercing

**Future:** Different sounds per weapon type or combo position.

---

## Data Flow

### Execution Sequence

```
1. Attack Animation Playing
   └─ Montage timeline reaches ANS_PerfectWindow

2. ANS_PerfectWindow Notify Begin
   └─ Cast to BP_ThirdPersonCharacter
      └─ Set bPerfectWindowActive = True

3. Player Presses Attack Input (LMB/RMB/etc)
   └─ CheckPerfectTiming() called
      └─ Returns: bPerfectWindowActive (True)

4. Input Handler Detects Perfect Timing
   ├─ Set bLastAttackWasPerfect = True
   ├─ ExecutePerfectStrikeFX()
   │  ├─ Play Sound: SFX_PerfectHit
   │  └─ Play Timeline: TL_WeaponGlow
   └─ AttemptCombo() (normal combo logic)

5. Timeline Updates (every frame for 0.3s)
   └─ Set Scalar Parameter Value
      └─ SwordMaterialDynamic.EmissiveIntensity = GlowIntensity

6. ANS_PerfectWindow Notify End
   └─ Set bPerfectWindowActive = False

7. Ready for Next Window
```

---

## Configuration & Tuning

### Window Duration

**Adjust:** Move ANS_PerfectWindow edges in montage timeline  
**Current:** ~0.2 seconds  
**Feel:** Tight but achievable with practice  

**Too Easy:** Window > 0.3s (feels forgiving, less rewarding)  
**Too Hard:** Window < 0.15s (frustrating, feel unfair)  

---

## Integration Points

### With Combat System

**File:** `AttemptCombo()` function  
**Connection:** Receives `bLastAttackWasPerfect` for future damage calculations  

Currently timing doesn't affect combo flow (all transitions valid regardless of timing).

---

### With Damage System

**Status:** 🔴 Not Yet Implemented  
**Planned:** Hitbox overlap event will check `bLastAttackWasPerfect` and apply damage multiplier if true.

---

### With Animation System

**File:** All attack montages  
**Requirement:** Each attack needs ANS_PerfectWindow placed at appropriate timing  

**Current Coverage:**

- AM_Attack_01: ✅ Has window
- AM_Attack_02: ✅ Has window  
- AM_Attack_03: ✅ Has window
- AM_Attack_04: ✅ Has window

**Future Attacks:** Add ANS_PerfectWindow during animation setup.

---

## Known Issues

### Current

- None

### Resolved

- ~~Material parameter not updating~~ (Session 8)
  - **Cause:** Parameter name mismatch (included quotes: `"EmissiveIntensity"`)
  - **Fix:** Use parameter name without quotes: `EmissiveIntensity`
  
- ~~Glow not visible~~ (Session 8)
  - **Cause:** Dynamic material created but not applied to mesh
  - **Fix:** Added `Set Material` after creating dynamic instance

---

## Future Enhancements

### Planned

- [ ] **Camera Pulse** - Zoom/shake on perfect timing
- [ ] **Particle Burst** - Additional VFX on weapon
- [ ] **Damage Integration** - Apply 1.5x multiplier on perfect hits
- [ ] **Perfect Counter** - Track consecutive perfect hits for UI
- [ ] **Combo-Specific Feedback** - Different effects per attack position
- [ ] **Audio Variation** - Different pitch/sound per combo position

### Considered

- [ ] **Perfect Window Anticipation** - Subtle pre-window effect (teach timing)
- [ ] **Slow-mo on Perfect** - Brief time dilation (0.1s)
- [ ] **Screen Flash** - Subtle white flash on perfect
- [ ] **Controller Rumble** - Haptic feedback on perfect

### Rejected

- ❌ **Particle-based glow** - Too much visual noise
- ❌ **Multiple sounds** - One clear ding is enough
- ❌ **Extended glow duration** - Should be snappy, not lingering

---

## Testing Notes

### How to Test

1. Attack test dummy (Attack 1)
2. Wait for glow to appear on weapon (window open)
3. Press attack input during glow
4. Listen for ding sound
5. See weapon flash bright then fade

### What to Validate

- ✅ Window timing feels fair (not too tight/loose)
- ✅ Audio feedback is clear and satisfying
- ✅ Visual feedback is noticeable but not distracting
- ✅ System works across all 4 attacks
- ✅ Multiple perfect hits in combo work correctly

### Common Test Failures

**Window never opens:**

- Check ANS_PerfectWindow is in montage
- Verify BeginPlay creates dynamic material
- Print `bPerfectWindowActive` value

**Glow doesn't show:**

- Verify dynamic material applied to mesh
- Check parameter name matches exactly
- Test with extreme value (50.0) to confirm connection
- Check material has EmissiveIntensity parameter

**Sound doesn't play:**

- Verify sound asset imported
- Check sound reference in ExecutePerfectStrikeFX
- Test volume levels (might be too quiet)

---

## Performance Notes

**Cost:** Negligible  
**Timeline:** Single timeline running 0.3s, no performance impact  
**Material:** Dynamic material instance per character (standard practice)  
**Audio:** Single 2D sound, no spatialization needed  

**Scales well:** System works identically with 1 player or 100 NPCs.

---

## Maintenance

### When Adding New Attacks

1. Create attack animation
2. Add to DT_Attacks data table
3. **Place ANS_PerfectWindow in montage** ← Don't forget!
4. Test timing window placement
5. Adjust window position/duration for attack rhythm

### When Changing Weapons

1. Verify weapon material has `EmissiveIntensity` parameter
2. If not, add to material or create material instance
3. Test glow visibility (different weapons may need different intensity)
4. No code changes needed (system is weapon-agnostic)

### When Tuning Feel

1. Adjust ANS placement for window timing
2. Edit TL_WeaponGlow curve for visual feel
3. Swap SFX_PerfectHit for different success sound
4. Test extensively after changes (feel is subjective)

---

## Code References

### Key Files

- `BP_ThirdPersonCharacter` - Main character blueprint
- `ANS_PerfectWindow` - AnimNotifyState for timing windows
- `TL_WeaponGlow` - Timeline for glow fade (in Event Graph)
- Attack Montages (AM_Attack_01-04) - Have ANS placed

### Key Nodes

- `PerfectAttachCheck()` - macros to identify is input was perfect
- `ExecutePerfectStrikeFX()` - Triggers feedback
- `Set Scalar Parameter Value` - Updates material glow
- `Play Sound 2D` - Audio feedback

### Material Parameters

- `EmissiveIntensity` - Controls glow brightness (0-10)
- `EmissiveColor` - Glow color (optional, defaults white)

---

## Design Rationale

### Why AnimNotifyState?

**Alternatives considered:**

- ❌ Timer-based windows (less precise, not synced to animation)
- ❌ Frame-counted windows (brittle, animation speed changes break it)
- ✅ AnimNotifyState (frame-perfect, designer-friendly, visual in editor)

### Why Material Glow Over Particles?

**Testing showed:**

- ❌ Niagara particles: Visual noise, hard to tune, performance cost
- ✅ Material glow: Clean, precise, performant, easier to control

### Why Timeline Over Manual Interpolation?

**Alternatives considered:**

- ❌ Manual Lerp in Tick (messy, hard to tune)
- ❌ For Loop with delays (choppy, complex)
- ✅ Timeline (smooth, visual curve editor, standard practice)

### Why Single Sound Over Layered?

**Testing showed:**

- One clear, satisfying ding is more effective than multiple sounds
- Simpler = more responsive feeling
- Can layer later if needed, but starting simple worked best

---

*Last Updated: Session 8*  
*Next Review: After damage system integration*
