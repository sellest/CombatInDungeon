## Gothic Knight Attack Family Table

| Attack | Family | Damage Cones | Optimal Angle | WindUp | Distance | Transitions From | Transitions To | Notes |
|--------|--------|--------------|---------------|--------|----------|------------------|----------------|-------|
| **Attack_01** | OPENER | R→F→L (270°) | Front 90° | Long (30f) | Medium | 2H-B_Idle | 02, 06 | Wide sweep, use when player roughly in front |
| **Attack_02** | PRESSURE COMBO | F→R (180°) | Front 90° | Short | Medium | Attack_01 | 03, 07 | Follow-up slash |
| **Attack_03** | AREA DENIAL | 360° | Any | Long | Close | Attack_02 | 04, 11 | Don't care about angle |
| **Attack_04** | PRESSURE COMBO | R→F, F | Front 60° | Medium | Medium | Attack_03 | - | Gap-close finisher |
| **Attack_05** | OPENER | F (narrow) | Front 30° | Long (30f) | Far | 2H-B_Idle | - | Thrust, player must be directly ahead |
| **Attack_06** | QUICK PUNISH | F (narrow) | Front 30° | Short (15f) | Close | Attack_01 | - | Uppercut, precise aim |
| **Attack_07** | PRESSURE COMBO | F (narrow) | Front 30° | Long (30f) | Medium | Attack_02 | Dead-end | Advancing uppercut |
| **Attack_08** | OPENER | F (narrow) | Front 30° | Long (30f) | Medium | 2H-B_Idle | 09 | Overhead, precise vertical |
| **Attack_09** | PRESSURE COMBO | F (narrow) | Front 30° | Very Long (45f) | Medium | Attack_08 | - | Heavy overhead |
| **Attack_10** | SPRINT ATTACK | F (narrow) | Front 30° | Fast | Far | 2H-B_Jog_F | - | Jump overhead |
| **Attack_11** | PRESSURE COMBO | F (narrow) | Front 30° | Very Long (45f) | Medium | Attack_03 | - | Advancing overhead |
| **Attack_12** | OPENER | F | Front 60° | Medium (25f) | Close | 2H-B_Idle | - | Kick |
| **Attack_13** | QUICK PUNISH | F | Front 60° | Very Short (10f) | Medium | 2H-B_Idle | - | Shoulder charge |
| **Attack_14** | SPRINT ATTACK | R→F (180°) | Front 90° | Short (15f) | Far | 2H-B_Jog_F | - | Sprint slash |
| **Attack_15** | REPOSITIONING | L→F→R (270°) | Front 90° | Long (25f) | Medium | 2H-B_Dodge_R | - | Sidestep right |
| **Attack_16** | REPOSITIONING | R→F→L (270°) | Front 90° | Long (25f) | Medium | 2H-B_Dodge_L | - | Sidestep left |
| **Attack_18** | SPRINT ATTACK | L→F→R (270°) | Front 90° | Short (15f) | Far | 2H-B_Jog_F | - | Sprint wide |
| **Attack_19** | AREA DENIAL | 360° | Any | Charged | Close | 2H-B_Idle | - | Windmill |

## Optimal Angle Categories

- **Front 30°:** Narrow attacks requiring precise aim (thrusts, overheads, uppercuts)
- **Front 60°:** Moderate width attacks (kicks, charges)
- **Front 90°:** Wide sweeps and slashes
- **Any:** 360° attacks, angle doesn't matter
```

## Distance Bands (Starting Values)
```
Close:  0-300 units
Medium: 300-600 units
Far:    600-1000 units
```
## Transition Sources Summary

### **From 2H-B_Idle (Openers):**
- Attack_01, 05, 08, 12, 13, 19

### **From 2H-B_Jog_F (Sprint):**
- Attack_10, 14, 18

### **From 2H-B_Dodge_R/L (Dodge Counters):**
- Attack_15 (from Dodge_R)
- Attack_16 (from Dodge_L)

### **From Other Attacks (Combo Only):**
- Attack_02 (from 01)
- Attack_03 (from 02)
- Attack_04 (from 03)
- Attack_06 (from 01)
- Attack_07 (from 02)
- Attack_09 (from 08)
- Attack_11 (from 03)
