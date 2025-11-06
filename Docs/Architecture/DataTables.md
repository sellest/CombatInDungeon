# Data Table Architecture

## Philosophy

Normalized database design - separate attack properties from combo transitions.

## DT_Attacks

**Purpose:** Define attack properties

| Column | Type | Description |
|--------|------|-------------|
| RowName | Name | Unique identifier (matches enum) |
| AttackName | Enum | E_AttackName enum value |
| AttackMontage | Animation Montage | Animation to play |
| BaseDamage | Float | Base damage value |
| AttackType | Enum | Light/Heavy/Special |
| HitboxDelay | Float | Time before hitbox active |
| HitboxDuration | Float | How long hitbox stays active |
| CameraShakeScale | Float | Intensity of camera shake |

## DT_Combos

**Purpose:** Define valid combo transitions

| Column | Type | Description |
|--------|------|-------------|
| RowName | Name | Descriptive (e.g., "Combo_1to2") |
| FromAttack | Enum | Starting attack state |
| ToAttack | Enum | Destination attack state |
| InputAction | Enum | Required input (Y_Tap, B_Hold, etc.) |

## Enums

### E_AttackName

- None
- Attack_1
- Attack_2
- Attack_Turn (loop variant)
- Attack_Funisher
- [Expand as needed]

### E_InputAction

- LMB
- RMB
- [Future inputs]

## Flow

1. Player input → Determine InputAction
2. Query DT_Combos: FromAttack=CurrentState, InputAction=Input
3. Get ToAttack from result
4. Query DT_Attacks: AttackName=ToAttack
5. Execute attack with properties
6. Update CurrentState = ToAttack

## Benefits

- ✅ Add new attacks = add rows (no code)
- ✅ Type-safe (enums prevent typos)
- ✅ Reusable attacks (same attack in multiple combos)
- ✅ Easy to visualize (export to spreadsheet)