# BP_EnemyBase Documentation

Parent class for all boss enemies. Contains core systems for geometry detection, attack selection, hitbox traces, rotation, and damage handling.

---

## Functions

### Debug/

| Function | Description | Input | Output |
|----------|-------------|-------|--------|
| ShowDebugGeometry | Master toggle for debug visualization | None | None |
| DrawDebugSectorGrid | Draws distance circles and cone boundary lines | None | None |
| DrawDebugActiveHighlight | Draws wedge showing player's current sector and precision | None | None |
| DebugCallAttackFromButton | Test harness - triggers SelectAttackFromIdle() and plays montage | None | None |
| DebugCallRotationFromButton | Test harness - triggers SelectTurnOnAngle() and plays turn montage | None | None |

### Player Geometry/

| Function | Description | Input | Output |
|----------|-------------|-------|--------|
| IsAngleInCone | Checks if angle falls within cone bounds (handles Back cone wrapping) | Angle (Float), MinAngle (Float), MaxAngle (Float) | Boolean |
| GetPlayerSector | Calculates player distance band and directional cone from data tables | None | Sets PlayerLocationSector, PlayerLocationCone |
| CalculatePlayerAngle | Returns yaw and pitch delta angles to player | None | AngleYaw (Float), AnglePitch (Float) |
| GetPlayerOptimalAngle | Determines angle precision level based on offset from cone center | None | Sets PlayerLocationOptimalAngle |
| IsPlayerInRange | Checks if player is within specified range | PlayerRef (Actor), BossRef (Actor), Range (Float) | bIsInRange (Boolean) |

### Attack Selection/

| Function | Description | Input | Output |
|----------|-------------|-------|--------|
| IsAttackValidForGeometry | Three-part validation: distance band + damage cone + optimal angle | AttackData (S_BossAttackData_V2) | Boolean |
| SelectAttackFromIdle | Queries Idle context transitions, filters by current geometry, returns first valid | None | S_BossAttackData_V2 (or None) |
| SelectComboAttack | Queries Combo context transitions from CurrentAttackData, filters by geometry | None | S_BossAttackData_V2 (or None) |

### Attack Execution/

| Function | Description | Input | Output |
|----------|-------------|-------|--------|
| PerformAttackTrace | Dual-socket sphere trace using CurrentAttackData hitbox settings | None | Processes hits via DeliverHitboxDamage |
| DeliverHitboxDamage | Validates target, checks AlreadyHitActors, applies damage via interface | OtherActor (Actor), HitComponent (PrimitiveComponent) | None |

### Rotation/

| Function | Description | Input | Output |
|----------|-------------|-------|--------|
| SelectTurnOnAngle | Selects 90° or 180° turn montage based on angle magnitude and direction | AngleToPlayer (Float) | AnimMontage (or None if <45°) |
| PerformBossRotationTracking | Rate-limited rotation toward player during ANS_AllowBossRotation window | None | None |
| InterpPlayerAngles | Smoothly interpolates LookAtYaw/Pitch for head tracking | Yaw (Float), Pitch (Float) | None (sets LookAtYaw, LookAtPitch) |

### Damage/

| Function | Description | Input | Output |
|----------|-------------|-------|--------|
| ProcessDamage | Handles boss HP reduction, controls Dead state, plays hit sounds | IncomingDamage (Float) | None |
| ProcessFlinch | Applies flinch damage, activates flinch regeneration, triggers Flinch state | FlinchDamage (Float) | None |
| ClearHitReact | Resets bJustGotHit to false | None | None |
| CalculateFlinchRegen | Handles flinch resistance regeneration (needs refactor) | None | None |

### AI Core/

| Function | Description | Input | Output |
|----------|-------------|-------|--------|
| TickBossAI_V2 | Main AI state machine loop (pending implementation) | None | None |

### Legacy (DELETE)

| Function | Description | Status |
|----------|-------------|--------|
| TickBossAI_V1 | Old AI loop | Delete |
| SelectNewCombatBehavior | Old behavior selection | Delete |
| ProcessThreat | Old threat accumulation before attack | Delete |
| OnAttackAnimationComplete | Old V1 callback | Delete |
| RotationHandler | Old V1 rotation | Delete |

---

## Variables

### Health/

| Variable | Description | Type |
|----------|-------------|------|
| MaxHealth | Maximum health value | Float |
| CurrentHealth | Current health value | Float |
| bIsDead | Death state flag | Boolean |

### Flinch Resistance/

| Variable | Description | Type |
|----------|-------------|------|
| MaxFlinchResistance | Maximum flinch resistance (resets to this after flinch) | Float |
| CurrentFlinchResistance | Current resistance (reduced by damage, flinch at 0) | Float |
| FlinchRegenRate | Resistance regeneration per second | Float |
| FlinchRegenDelay | Delay before regeneration starts after damage | Float |
| LastFlinchDamageTime | Timestamp of last damage received | Float |
| bShouldFlinch | Flag indicating flinch state triggered | Boolean |

### Hit Reaction/

| Variable | Description | Type |
|----------|-------------|------|
| bJustGotHit | Prevents multiple flinch animations from same hit | Boolean |
| LastHitAngle | Directional angle for hit reaction animations | Float |

### Hitbox/

| Variable | Description | Type |
|----------|-------------|------|
| WeaponHitboxSocketName | Socket name for weapon attachment (set in Construction Script) | Name |
| AlreadyHitActors | Actors hit during current attack (prevents multi-hit) | Array(Actor) |

### Player Geometry/

| Variable | Description | Type |
|----------|-------------|------|
| PlayerRef | Reference to player character | BP_ThirdPersonCharacter |
| PlayerLocationSector | Current player distance band (Close/Medium/Far/Out) | E_DistanceBand |
| PlayerLocationCone | Current player directional cone (Front/Right/Back/Left) | E_AttackCone |
| PlayerLocationAngle | Raw yaw angle to player (-180 to 180) | Float |
| PlayerLocationOptimalAngle | Current angle precision (Narrow30/Moderate60/Wide90/None) | E_OptimalAngle |

### Head Tracking/

| Variable | Description | Type |
|----------|-------------|------|
| LookAtYaw | Interpolated yaw for head tracking blendspace | Float |
| LookAtPitch | Interpolated pitch for head tracking blendspace | Float |

### Rotation/

| Variable | Description | Type |
|----------|-------------|------|
| MaxRotationSpeed | Maximum rotation speed during tracking (degrees/sec) | Float |
| bCanRotate | Flag set by ANS_AllowBossRotation | Boolean |
| TotalRotationThisAttack | Accumulated rotation during current attack (budget limit) | Float |

### Movement/

| Variable | Description | Type |
|----------|-------------|------|
| MovementTargetLocation | Target position for movement | Vector |
| bIsMoving | Currently executing movement | Boolean |

### Attack State/

| Variable | Description | Type |
|----------|-------------|------|
| bIsExecutingAttack | Currently in attack animation | Boolean |
| CurrentAttackData_V2 | Active attack properties from data table | S_BossAttackData_V2 |
| LastAttackTime | Timestamp of last attack start | Float |
| bInComboWindow | Flag set by ANS_ComboWindow | Boolean |

### Data Tables/

| Variable | Description | Type |
|----------|-------------|------|
| DistanceBandsTable | Reference to DT_DistanceBands | DataTable |
| AttackConesTable | Reference to DT_AttackCones | DataTable |
| OptimalAnglesTable | Reference to DT_OptimalAngles | DataTable |
| AttackTransitionsTable | Reference to DT_[Boss]Transitions | DataTable |
| BossAttackTable | Reference to DT_[Boss]Attacks_V2 | DataTable |

### Targeting/

| Variable | Description | Type |
|----------|-------------|------|
| LockOnTargetSocketName | Socket name for player lock-on targeting | Name |

### AI State/

| Variable | Description | Type |
|----------|-------------|------|
| CurrentBossState_V2 | Current AI state (pending implementation) | E_BossState_V2 |

### Debug/

| Variable | Description | Type |
|----------|-------------|------|
| bShowDebugGeometry | Enables debug visualization | Boolean |

### Legacy (DELETE)

| Variable | Status |
|----------|--------|
| CurrentBossState | Delete - V1 state |
| CurrentAttackData | Delete - V1 attack data |
| AttackDataTable | Delete - V1 |
| CurrentAttackCooldown | Delete - V1 |
| AttackRangeBuffer | Delete - V1 |
| DangerZoneRange | Delete - V1 |
| CombatBehaviorTable | Delete - V1 |
| CurrentCombatBehavior | Delete - V1 |
| CurrentMovementType | Delete - V1 |
| CombatBehaviorEndTime | Delete - V1 |
| TotalWeight | Delete - V1 random selection |
| RandomValue | Delete - V1 random selection |
| AccumulatedWeight | Delete - V1 random selection |
| LastCombatBehavior | Delete - V1 |
| CurrentMovementType_V2 | Delete - unused |
| ThreatLevel | Delete - V1 threat system |
| ThreatThreshold | Delete - V1 threat system |
| ThreatAccumulationRate | Delete - V1 threat system |
| CurrentPoise | Delete - duplicate of FlinchResistance |
| PlayerYaw | Delete - use PlayerLocationAngle |
| PlayerPitch | Delete - unused |
| CurrentRotationSpeed | Delete - unused |
| ApproachSpeed | Delete - V1 |
| CombatMovementSpeed | Delete - V1 |
| Target | Delete - unknown purpose |
| AggroRange | Delete - V1 |
| AttackRange | Delete - V1 |
| AttackHitboxes | Delete - old hitbox system |

---

## Summary

**Active Systems:** 5 function categories, 12 variable categories

**Legacy to Delete:** 5 functions, 28 variables

**Pending Implementation:**
- TickBossAI_V2 (AI state machine)
- Movement system (Approach, Circle, Backstep, Sprint)
- Turn-in-place execution (montages exist, code pending)
- RepositionForAttack (movement brain)

---

*Document Version: 1.1*
*Last Updated: December 2025*
