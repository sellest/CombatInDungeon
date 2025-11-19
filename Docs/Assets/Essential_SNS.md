# Animation Pack Assets

**Pack:** [Name of the pack you bought]
**Cost:** $25
**Quality:** Professional with pre-built blend spaces

---

## Blend Spaces Available

### Guard/Block Locomotion:

- ✅ `UE5_SSH_Guard_Jog_Forward_BlendSpace` (IN USE - GuardIdle 
- `UE5_SSH_Guard_Jog_Start_Backward_Blendspace`
- `UE5_SSH_Guard_Jog_Stop_Backward_BlendSpace`

### Dodge (TODO - Priority):

- `UE5_SSH_Dodge_01_L_BlendSpace`
- `UE5_SSH_Dodge_01_R_BlendSpace`
- **Use case:** Directional dodge improvements

### Hit Reactions (TODO):

- `UE5_SSH_Hurt_Light_BlendSpace`
- **Use case:** Taking damage, flinch reactions

### Attacks (TODO):

- `UE5_SSH_Thrust_BlendSpace`
- **Use case:** special focused thrust attacks
---

## Usage Notes:

**Blend Space Inputs (Standard across pack):**
- **Degree:** -180 to 180 (angle)
- **Speed:** 0 to 800 (movement speed)

**Built-in AnimBP Variables:**
- `Direction` - Calculated angle for blend spaces
- `GroundSpeed` - Current movement speed
- `bShouldMove` - Is character moving

**Integration Pattern:**
1. Use Direction → Degree input
2. Use GroundSpeed → Speed input
3. Works automatically with AnimBP

---

## Next To Integrate:

1. **Dodge blend spaces** (when refining dodge system)
2. **Hit reaction blend spaces** (when damage system done)
3. **Attack blend spaces** (when adding more attacks)
4. **Guard Start/Stop** (polish phase, low priority)