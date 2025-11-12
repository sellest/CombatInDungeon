## Hit Sound System

**Status:** ✅ Complete
**Session:** 16

### Components

- **BPL_AudioHelpers** - Function library for audio utilities
- **PlayRandomHitSound** - Plays random sound from data table at location
- **SC_HitSounds** - Sound Concurrency (max 3 instances, stop oldest)

### Data Tables

**Structure:** DT_[WeaponType]HitSounds_[SurfaceType]

Example: DT_SwordHitSounds_Wood

| Column | Type | Description |
|--------|------|-------------|
| Row Name | Name | Individual sound identifier |
| Sound | Sound Base | Audio file reference |

### Usage

Enemies call `PlayRandomHitSound` in Event AnyDamage:
- HitSoundTable: Reference to appropriate DT
- HitLocation: Where to play sound
- WorldContextObject: Self

### Future Expansion

- DT_SwordHitSounds_Steel (for armored enemies)
- DT_SwordHitSounds_Flesh (for organic enemies)
- DT_ShieldHitSounds_[Surface] variants