# Kingdom Rush Battles - Battle Damage Calculation Research

**Author:** LogicBr3ak3r
**Contact:** Telegram - [@LogicBr3ak3r](https://t.me/LogicBr3ak3r)

> **Note on Tools & Scripts:** All scripts and automation tools developed during this research - including `auto_farm.py` (the full economy farming bot), the PvP headless client, the savegame editor, and supporting utilities - will be released publicly once these reports gain sufficient attention. Follow the Telegram contact above to stay notified.

## Overview

All findings from reverse engineering Kingdom Rush Alliance's battle simulation via IDA and captured API traffic. Covers the full damage pipeline, stat tracking, card/level scaling, AI behavior, and every relevant binary address.

---

## 1. Fixed-Point Arithmetic (Q16.16 FP)

ALL damage, position, and stat values in the Quantum ECS simulation use **Q16.16 fixed-point**:

```
realValue = RawValue / 65536
RawValue = int(realValue * 65536)
```

- Internal type: `FP` (64-bit int64 storing the raw fixed-point value)
- `FPVector2` = two FP values (X, Y) = 128 bits total
- BitStream encoding: `WriteULong(FP.RawValue, 64)` -- raw int64, 64 bits

**IDA addresses for FP serialization:**

| Address | Function | Purpose |
|---------|----------|---------|
| `0x20EA29C` | `BitStream.Serialize(ref FP)` | FP.RawValue as int64 |
| `0x20EA2DC` | `BitStream.Serialize(ref FPVector2)` | X then Y, each as int64 |
| `0x20DC58C` | `BitStream.Serialize(ref int)` | 4-byte LE int32 |
| `0x20EA21C` | `BitStream.Serialize(ref long)` | 8-byte int64 via WriteULong(64) |

---

## 2. DailyMissions ECS Component (56 bytes)

This is the in-match stat tracker. The client sends accumulated values to the server via `finish-game` API.

```
DailyMissions (56 bytes):
  +0x00: Creeps.DeathCount        (uint32)  -- incremented per WaveCreep killed
  +0x08: Hero.DamageDealt         (FP/int64) -- accumulated post-resistance hero damage
  +0x10: Spells.CastCount         (uint32)  -- incremented per spell cast
  +0x18: Spells.DamageDealt       (FP/int64) -- accumulated post-resistance spell damage
  +0x20: Spells.UnitSpawnedCount  (uint32)  -- per successful spell unit spawn
  +0x28: Towers.BuildCount        (uint32)  -- per tower purchased
  +0x2C: Towers.SkillCount        (uint32)  -- per tower skill unlocked
  +0x30: Towers.UpgradeCount      (uint32)  -- per tower upgrade purchased
```

### Signal Handlers (DailyMissionsSystem)

| Address | Function | Trigger | Action |
|---------|----------|---------|--------|
| `0xb20b0c` | `OnDeath` | WaveCreep killed by opponent | `deathCount++` |
| `0xb20ba4` | `OnHeroDamage` | Hero deals damage | `hero.damageDealt += amount` |
| `0xb20c0c` | `OnHealthChanged` | Any entity health changes | Identifies source by SpawnerType (Hero=1, Tower=2) |
| `0xb20e34` | `OnSpellDamage` | Spell deals damage | `spells.damageDealt += amount` |
| `0xb20e9c` | `OnSpellUnitSpawned` | Spell spawns a unit | `unitSpawnedCount++` |
| `0xb20fa8` | `OnSpellCasted` | Spell is cast | `castCount++` |

Note: Hero damage is tracked by BOTH `OnHeroDamage` and `OnHealthChanged` (which identifies source as Hero via SpawnerType index = 1).

---

## 3. Damage Pipeline

### Full Formula

```
1. Roll baseDamage = Random(minDamage, maxDamage)     -- from card level config lookup table
2. sourceMult = UnitStats.GetFinalDamageMultiplier()   -- source unit's damage multiplier
3. resistanceMult = 1.0 - max(0, resistance - piercing) / 100.0
4. finalDamage = baseDamage * sourceMult * resistanceMult
```

### Key Addresses in the Pipeline

| Address | Function | Role |
|---------|----------|------|
| `0xbe46c0` | `TargetEffect_Damage.Configure` | Reads 12 config keys per damage effect |
| `0xbecd88` | `TargetEffect_Damage.ApplyEffectToEntity` | Full damage pipeline execution |
| `0xbec940` | `UnitStats.GetFinalDamageMultiplier` | Source damage multiplier |
| `0xdcd9d8` | `Resistances.GetResistanceMultiplier` | Resistance calculation |
| `0x83e358` | `AIManager.UpdateDamageMultipliers` | Bot difficulty scaling per wave |
| `0x1e79104` | `RuntimePlayer.GetAvgCardLevel` | Average deck card level |

### 12 Config Keys per Damage Effect (TargetEffect_Damage)

Set via `TargetEffect_Damage.Configure` @ `0xbe46c0`:

| Key | Type | Description |
|-----|------|-------------|
| `abilityType` | enum | Type of ability (attack, spell, etc.) |
| `rangeType` | enum | Melee vs ranged |
| `damageType` | enum | Physical vs magical |
| `minDamage` | FP | Minimum base damage roll |
| `maxDamage` | FP | Maximum base damage roll |
| `armorPiercing` | FP | Ignores this much armor |
| `magicPenetration` | FP | Ignores this much magic resistance |
| `chancePercentage` | FP | Chance to apply (100 = always) |
| `damagePercentage` | FP | Damage multiplier percentage |
| `extraMovementDamage` | FP | Bonus damage based on target movement |
| `damageMovementPercentage` | FP | Movement damage multiplier |
| `areaRange` | FP | AoE radius |

### Resistance Formula

```python
def get_resistance_multiplier(resistance, piercing):
    effective = max(0, resistance - piercing)
    return 1.0 - effective / 100.0
```

- If piercing >= resistance: full damage (multiplier = 1.0)
- 50 effective resistance = 50% damage reduction
- 100 effective resistance = 0% damage (full block)
- Armor blocks physical, magic resistance blocks magical

---

## 4. Card Level Scaling System

### How It Works

Card level does NOT use a formula. It uses **lookup tables** stored in serialized asset files:

```
statsPerLevel[level - 1] -> stat values for that level
```

If level exceeds array length, clamps to last element.

These tables are in `MetagameConfigurationSettings` assets (Quantum serialized binary data), NOT hardcoded in the binary. The binary only contains the lookup logic.

### API Data Structure

**Example card collection (deck composition):**

| Card Name | Type | Level | Worn Copies |
|-----------|------|-------|-------------|
| MG_GeraldData (Grawl) | Hero | 4 | 10 |
| MG_TowerArtillery | Tower | 4 | 10 |
| MG_TowerMage | Tower | 2 | 2 |
| MG_TowerArcher | Tower | 3 | 4 |
| MG_RainOfFire | Spell | 4 | 2 |
| MG_TowerBarrack | Tower | 2 | 2 |
| MG_Reinforcements | Spell | 5 | 20 |
| MG_AlleriaCardData (Alleria) | Hero | 2 | 2 |
| MG_TowerBarrack2 | Tower | 1 | 0 |
| MG_Stun | Spell | 5 | 20 |
| MG_HealingBlessing | Spell | 1 | 0 |
| MG_TowerRangerHideout | Tower | 3 | 0 |
| MG_EarthQuake | Spell | 1 | 0 |
| MG_TowerPoisonArcher | Tower | 1 | 0 |
| MG_TowerGnoll | Tower | 1 | 0 |
| MG_TowerGolem | Tower | 1 | 0 |
| MG_TowerCrossbowHunter | Tower | 1 | 0 |
| MG_BoosterHealingRegeneration | Booster | 1 | 0 |

**Card reference database:** 30 total cards (15 towers, 5 heroes, 10 spells)

**Card system parameters (admin-settings):**
- `maxLevelOfCards`: 15
- `initialLevelOfCards`: [1, 1, 1, 1, 3]
- PvP card level cap: `pvpCardLevelCap: 7`
- Card upgrade cost: 5 gold per level-up

### Event Card Level Caps

| Event Type | cardLevelCap | creepLevel | boosterLevelCap | botDifficulty |
|------------|-------------|------------|-----------------|---------------|
| TROPHYROAD (PvP) | 7 | 7 | 6 | N/A |
| COINFARM | 5 | 3 | 5 | 0 (Easy) |
| ONSLAUGHT | 5 | 3 | 5 | 0 (Easy) |
| ONSLAUGHT_2 | 9 | 19 | 7 | 0 (Easy) |

Card levels are clamped to `cardLevelCap` for each mode. A level 10 tower in your collection plays as level 5 in COINFARM.

### CreepLevelMultipliers

Enemy creep stats (health, damage, armor, magic resistance) scale based on:
1. The `creepLevel` setting for the mode
2. The average card level of the player's deck (`RuntimePlayer.GetAvgCardLevel` @ `0x1e79104`)
3. Lookup tables in StageWaveSettings assets

Higher creepLevel = tankier enemies with more damage/resistance.

---

## 5. UnitStats Structure (488 bytes)

Each unit (tower, hero, creep) has 16 stat fields across 488 bytes. Per-field breakdown not fully mapped but includes:
- Health / MaxHealth
- Damage (min/max)
- Attack speed
- Range
- Armor / MagicResistance
- Movement speed (creeps/heroes)
- Various multipliers

### Tower-Specific

**TowerSettings** extends **UnitSettings** (240 bytes):
- `TowerType` field
- 3 in-match upgrade tiers: `Level1`, `Level2`, `Level3` (base -> first upgrade -> second upgrade)
- 2 skills per tower: `Skill1`, `Skill2` (require upgraded tower to unlock)
- Upgrade settings per tier

### Map Constraints

| Constraint | Value | Source |
|-----------|-------|--------|
| Tower slots per map | 5-12 | Data-driven per StageWaveSettings |
| Max upgrade tiers | 2 per tower | base -> Level2 -> Level3 |
| Max skills per tower | 2 | Skill1, Skill2 |
| Deck towers | 3 | Fixed deck composition |
| Deck spells | 3 | Fixed deck composition |
| Deck heroes | 1 | Fixed deck composition |
| Deck boosters | 1 | Fixed deck composition |

---

## 6. AI Damage Multipliers (Bot Difficulty)

### Bot Difficulty Levels

| Value | Level | Description |
|-------|-------|-------------|
| 0 | Easy | Default for events |
| 1 | Normal | Standard PvP bots |
| 2 | Hard | Scaled up |
| 3 | Maximum | Full multipliers |

### `AIManager.UpdateDamageMultipliers` @ `0x83e358`

Adjusts tower and hero damage output per wave based on bot difficulty settings. Higher difficulty = higher multipliers = bots deal more damage.

### AI Decision Pipeline

```
1. AIThreatLedger updates threat scores per lane
2. WaveModifierAI selects boosters
3. UtilityReasoner evaluates tower placement scores
4. UtilityReasoner evaluates tower upgrade scores
5. HeroAI repositions hero based on threat
6. SpellAI targets spells at high-threat clusters
7. Highest-scoring action is executed as a Command
```

### AI Tower Placement Scoring (weights)

| Factor | Weight | Description |
|--------|--------|-------------|
| Path coverage | 40% | Segments within tower range |
| Multi-path intersection | 20% | Bonus for covering multiple paths |
| Exit proximity | 15% | Last-chance defensive value |
| Node density | 15% | Turn/waypoint proximity |
| Strategic centrality | 10% | Map center positioning |

### AI Utility Functions

| Address | Function | Purpose |
|---------|----------|---------|
| `0x82D9CC` | `AIUtils.GetBestBuyTowerUtility` | Iterates tower types, picks highest utility |
| `0x82E1CC` | `AIUtils.GetPurchaseUtility` | Multi-factor utility score per tower |
| `0x82ED28` | `AIUtils.GetCoverageMultiplier` | Coverage-based multiplier |
| `0x82FCD8` | `AIUtils.GetTileWeight` | Directional tile weight scoring |

---

## 7. Wave / Creep System

### Wave Hierarchy

```
StageWaveSettings
  -> Waves[]              (array of wave definitions)
    -> Subwaves[]         (groups within a wave)
      -> Spawns[]         (individual spawn groups)
        -> Count: uint8   (creeps per spawn)
```

### Key Fields

| Field | Type | Location |
|-------|------|----------|
| `TotalWaves` | uint8 | `WaveManager` component |
| `CurrentWaveIndex` | int8 | `WaveManager` component |
| `DeathCount` | uint32 | `CreepsMissionInfo` (DailyMissions+0x00) |

### Wave Manager

- `WaveManager.Initialize` @ `0x2759df4` -- sets up wave count from stage data
- Analytics show wave 6 is specifically tracked = ~6+ waves standard per map
- Up to ~15 waves in longer matches

### End-of-Match Onslaught

When a player wins, remaining enemy creeps are instakilled. These kills STILL count toward `deathCount`. This means a win always has inflated kill counts compared to what the player actually fought through.

### Typical Wave Kill Counts

| Scenario | Expected Kills | Notes |
|----------|---------------|-------|
| Win (all waves) | 30-65 | Includes onslaught cleanup |
| Early loss (wave 1-2) | 5-15 | Real captured loss: 15 |
| Mid loss (wave 3-4) | 15-30 | Survived a few waves |
| Late loss (wave 5+) | 25-40 | Almost won |

---

## 8. Spell System Details

### Spell Types in Game

| Spell Card | Code | Type | Damage Profile |
|-----------|------|------|---------------|
| MG_RainOfFire | s0 | Damage | High AoE burst, one-shot |
| MG_Reinforcements | s2 | SpawnUnit | Spawns soldiers, their damage accumulates as spell damage |
| MG_Stun | s3 | CC | Stun + minor damage |
| MG_HealingBlessing | s4 | Heal | No damage |
| MG_EarthQuake | s5 | Damage | AoE damage + slow |

### Spell Damage Tracking

- `OnSpellCasted` fires once per spell use -> `castCount++`
- `OnSpellDamage` fires per damage tick -> `spells.damageDealt += amount`
- `OnSpellUnitSpawned` fires per summoned unit -> `unitSpawnedCount++`
- Reinforcements is `SpawnUnit` type -- spawns soldiers whose individual attacks accumulate as spell damage
- Healing spells (HealingBlessing) don't add to damageDealt

### Spell Cooldowns

Data-driven per spell, level-scaled. From observation:
- Typical match allows 2-6 casts per spell
- 3 spells in deck = 6-18 total casts in a full match (theoretical max)
- Realistic: 2-8 total casts across all spells in a win

### Spell Damage Per Cast (estimated from captured data)

| Context | Damage Per Cast | Notes |
|---------|----------------|-------|
| Low level (1-3) | 100-400 | Basic damage |
| Mid level (4-6) | 200-700 | Typical range |
| High level (7-9) | 400-1200 | Event/PvP cap |
| Reinforcements | Varies | Depends on spawned unit lifetime and targets |

---

## 9. Hero System Details

### Heroes in Game

| Hero Card | Name | Notes |
|-----------|------|-------|
| MG_GeraldData | Grawl | Melee, starter hero |
| MG_AlleriaCardData | Alleria Wildcat | Ranged, gacha hero (pity at 40 pulls) |
| (3 more heroes) | Unknown | Not in captured collection |

### Hero Damage Tracking

- Hero is always present on the battlefield (auto-deployed)
- Continuously attacks enemies within range
- Damage tracked per tick via `OnHeroDamage` and `OnHealthChanged`
- All damage is post-resistance (after armor/magic resist reduction)

### Hero DPS Estimates (from captured data)

**Real captured match:** Grawl level 4, 919 damage in 56 seconds = **~16.4 DPS**

| Card Level | Estimated DPS | Match Duration Range | Total Damage Range |
|-----------|--------------|---------------------|-------------------|
| 1-2 | 8-15 | 45-210s | 360-3150 |
| 3-4 | 12-22 | 45-210s | 540-4620 |
| 5-6 | 18-30 | 45-210s | 810-6300 |
| 7-9 | 25-40 | 45-210s | 1125-8400 |
| 10+ | 35-55 | 45-210s | 1575-11550 |

Note: These are rough extrapolations. Actual DPS depends on:
- Hero card level (statsPerLevel lookup)
- Target resistance values
- Hero attack speed
- Time spent walking vs attacking
- Number of targets in range

---

## 10. Tower System Details

### Tower Build Mechanics

- Player has 3 tower cards in deck
- Map has 5-12 tower "holder" slots (building positions)
- Buying a tower: `BuyTowerCommand` -> `buildCount++`
- Each tower can be upgraded twice: base -> Level2 -> Level3
- Each upgrade: `UpgradeTowerCommand` -> `upgradeCount++`
- Skills (2 per tower, require upgraded tower): `UnlockTowerSkillCommand` -> `skillCount++`

### Tower Types in Game

| Tower Card | Code | Type | Notes |
|-----------|------|------|-------|
| MG_TowerArcher | t0 | Ranged DPS | Fast attack, low damage |
| MG_TowerArtillery | t1 | AoE DPS | Slow attack, area damage |
| MG_TowerBarrack | t2 | Melee/Tank | Spawns soldiers to block |
| MG_TowerMage | t3 | Magic DPS | Magic damage, bypasses armor |
| MG_TowerRangerHideout | t4 | Ranged | Stealth/ambush variant |
| MG_TowerBarrack2 | t5 | Melee variant | Different soldier type |
| MG_TowerPoisonArcher | t6 | Ranged + DoT | Poison damage over time |
| MG_TowerGnoll | t7 | Melee variant | Gnoll soldiers |
| MG_TowerGolem | t8 | Melee/Tank | Golem tank unit |
| MG_TowerCrossbowHunter | t9 | Ranged | High single-target damage |
| (5 more towers) | -- | Various | Not in captured collection |

### Tower Stat Scaling

Tower damage/health/range are determined by:
1. Card level -> `statsPerLevel[level]` lookup table (in asset files)
2. In-match upgrade tier (Level1/Level2/Level3) -> different stat blocks
3. AI damage multiplier (if bot)
4. No global formula -- each tower has unique stat progression

---

## 11. Server-Side Behavior (What's Validated vs Trusted)

### Fully Trusted by Server (No Validation)

| Field | Notes |
|-------|-------|
| `missions.towers.buildCount` | Any value accepted |
| `missions.towers.upgradeCount` | Any value accepted |
| `missions.towers.skillCount` | Any value accepted |
| `missions.spells.damageDealt` | Any value accepted |
| `missions.spells.castCount` | Any value accepted |
| `missions.spells.unitSpawnedCount` | Any value accepted |
| `missions.hero.damageDealt` | Any value accepted |
| `missions.creeps.deathCount` | Any value accepted |
| `win` | "true"/"false" - server trusts outcome |
| Star conditions | All 6 star booleans trusted |
| Match duration | No minimum enforced (0s matches give full rewards) |

### Server-Calculated (Not Client-Controllable)

| Field | Notes |
|-------|-------|
| `trophies` delta | Calculated from trophiesBot delta and win/loss |
| `gold` reward | Calculated by server based on match type and win/loss |
| `stars` points | Calculated from star conditions sent (applies point weights) |
| `playerXP` | Server adds fixed amount per match |
| `eventRewardIds` | Server determines event rewards |

### Anti-Cheat Status: DISABLED

- `ChecksumInterval` explicitly set to 0 in `DeterministicSessionConfigAsset.GetConfig` @ `0x29BA3FC`
- The checksum verification code in `DeterministicSimulator.SimulateVerified` @ `0x20D9284` is dead code (never executes)
- "Cheater detected" callback at `0x29BCC14` can never fire
- No root detection, no hook detection, no emulator detection
- `IntegrityValidateService` exists at `0xDEB7EC` but doesn't block

---

## 12. Simulation Timing

| Parameter | Value | Source |
|-----------|-------|--------|
| Tick rate | 60 ticks/second | Deterministic simulation config |
| Tick duration | 16.67ms | 1000/60 |
| Max ticks (PvP) | 54,000 | 15 minutes at 60 TPS |
| Bot fill timeout | 6 seconds | MatchmakingSettings.botTimeout |
| forceBotMinDelay | 2 seconds | Matchmaking config |
| forceBotMaxDelay | 5 seconds | Matchmaking config |
| Matchmaking timeout | 90 seconds | Client-side timeout |

### Real Match Duration Data

| Match Type | Outcome | Duration | Source |
|-----------|---------|----------|--------|
| TROPHYROAD vs Bot | LOSS | 56 seconds | Captured traffic |
| TROPHYROAD (spoofed) | WIN | 0 seconds | PoC test (accepted!) |
| Typical real win | ~90-210 seconds | Estimated from wave counts |
| Typical real loss | ~45-120 seconds | Estimated from captured data |

---

## 13. Real Match Data (Captured Traffic)

### Only Captured Real Match -- LOSS on TROPHYROAD vs Bot

| Stat | Value | Analysis |
|------|-------|----------|
| buildCount | 2 | 1 tower + hero placement |
| upgradeCount | 0 | Lost too fast |
| skillCount | 0 | No skills |
| spells.damageDealt | 0 | No spells cast |
| spells.castCount | 0 | No spells cast |
| spells.unitSpawnedCount | 0 | No summons |
| hero.damageDealt | 919 | ~16.4 DPS * 56s |
| creeps.deathCount | 15 | Lost on wave 1 |

**Match Context:**
- Stage: Forest 3
- Hero: Grawl (level 4)
- Deck towers: archer(lv3), artillery(lv4), barbarian(lv4), rangershideout(lv3)
- Deck spells: rainoffire(lv4), reinforcements(lv4), stun(lv5)
- Average deck level: 4
- Trophies result: -9
- Gold result: 3

### Trophy/Gold Rewards Observed

| Scenario | Trophies | Gold |
|----------|----------|------|
| Real loss (trophiesBot=500) | -9 | 3 |
| Spoofed win (trophiesBot=500) | +48 | ~5-20 |
| Fast spoofed win (0s delay) | +7-8 | ~5 |
| COINFARM win | 0 | 80 (softCoinWin) |
| COINFARM loss | 0 | 40 (softCoinLose) |
| ONSLAUGHT win | 0 | 100 + 1 summon scroll |
| ONSLAUGHT_2 win | 0 | 100 + 6 summon scrolls |

---

## 14. Mission System Integration

### How Missions Connect to finish-game Stats

| Mission Type | maxAmount | Feeds From | Field |
|-------------|----------|-----------|-------|
| DefeatCreeps | 200 | missions.creeps.deathCount | Cumulative across matches |
| TowerBuild | 20 | missions.towers.buildCount | Cumulative |
| TowerUpgrade | 30 | missions.towers.upgradeCount | Cumulative |
| SpellDamage | 30,000 | missions.spells.damageDealt | Cumulative |
| HeroDamage | 15,000 | missions.hero.damageDealt | Cumulative |
| PlayGames | 2 | Any finish-game call | Server counts |
| StarsEarn | 6 | Star conditions in finish-game | Server calculates points |
| LoginsDone | 1 | Auto on login | Server tracks |
| ChestOpen | 3 | open-chests endpoint | Server tracks |
| PurchaseDailyDeal | 2 | Purchase endpoint | Server tracks |
| SpendGems | 18 | Gem spending endpoint | Server tracks |
| UnlockChestWithGems | 1 | Gem chest skip | Server tracks |

8 of 12 missions are client-controllable via finish-game missions data.

---

## 15. Deterministic Command Wire Formats

All 15 in-match commands and their serialization:

| Command | ID | Fields | Wire Size |
|---------|----|---------|----|
| `PlayerReadyCommand` | 0 | *(empty)* | 0 bits |
| `UseCardCommand` | 1 | `byte CardIndex` + `FPVector2 Position` | 136 bits |
| `UseMercenaryCardCommand` | 2 | `byte CardIndex` + `FPVector2 Position` | 136 bits |
| `UseEmoteCommand` | 3 | `byte EmoteIndex` | 8 bits |
| `BuyTowerCommand` | 4 | `EntityRef Holder` + `AssetRefTowerSettings TowerRef` | 128 bits |
| `SellTowerCommand` | 5 | `EntityRef Tower` | 64 bits |
| `UpgradeTowerCommand` | 6 | `EntityRef Tower` | 64 bits |
| `BuyHolderCommand` | 7 | `EntityRef Holder` | 64 bits |
| `UnlockTowerSkillCommand` | 8 | `EntityRef Tower` + `int SkillIndex` | 96 bits |
| `TriggerTowerSkillCommand` | 9 | `EntityRef Tower` + `int SkillIndex` | 96 bits |
| `DragDraggableCommand` | 10 | `EntityRef Entity` + `FPVector2 WorldPos` + `int Tile` | 224 bits |
| `SelectWaveModifierCommand` | 11 | `PlayerRef Player` + `byte ModifierId` | 40 bits |
| `TapCommand` | 12 | `EntityRef Entity` + `FPVector2 WorldPos` | 192 bits |
| `TutorialCommand` | 13 | `int Type` + `FP Value` + `int Player` + `EntityRef Entity` + `AssetRef AssetRef` + `FPVector2 Position` | 384 bits |
| `TutorialBotAICommand` | 14 | `bool DisableUnitsAI` | ~8 bits |

**Type sizes:**
- `EntityRef` = Index(int32) + Version(int32) = 64 bits
- `PlayerRef` = _index(int32) = 32 bits
- `FP` = RawValue(int64) = 64 bits
- `FPVector2` = X(int64) + Y(int64) = 128 bits
- `AssetRef` / `AssetGuid` = Value(int64) = 64 bits

---

## 16. Complete IDA Pro Address Reference

### DailyMissions / Stats Tracking

| Address | Function |
|---------|----------|
| `0xb20b0c` | `DailyMissionsSystem.OnDeath` |
| `0xb20ba4` | `DailyMissionsSystem.OnHeroDamage` |
| `0xb20c0c` | `DailyMissionsSystem.OnHealthChanged` |
| `0xb20e34` | `DailyMissionsSystem.OnSpellDamage` |
| `0xb20e9c` | `DailyMissionsSystem.OnSpellUnitSpawned` |
| `0xb20fa8` | `DailyMissionsSystem.OnSpellCasted` |

### Damage Pipeline

| Address | Function |
|---------|----------|
| `0xbe46c0` | `TargetEffect_Damage.Configure` |
| `0xbecd88` | `TargetEffect_Damage.ApplyEffectToEntity` |
| `0xbec940` | `UnitStats.GetFinalDamageMultiplier` |
| `0xdcd9d8` | `Resistances.GetResistanceMultiplier` |
| `0x83e358` | `AIManager.UpdateDamageMultipliers` |
| `0x1e79104` | `RuntimePlayer.GetAvgCardLevel` |

### AI System

| Address | Function |
|---------|----------|
| `0x82D9CC` | `AIUtils.GetBestBuyTowerUtility` |
| `0x82E1CC` | `AIUtils.GetPurchaseUtility` |
| `0x82ED28` | `AIUtils.GetCoverageMultiplier` |
| `0x82FCD8` | `AIUtils.GetTileWeight` |

### Wave System

| Address | Function |
|---------|----------|
| `0x2759df4` | `WaveManager.Initialize` |

### Anti-Cheat (All Disabled/Dead)

| Address | Function |
|---------|----------|
| `0x29BA3FC` | `DeterministicSessionConfigAsset.GetConfig` (ChecksumInterval=0) |
| `0x20D9284` | `DeterministicSimulator.SimulateVerified` (dead code) |
| `0x29BCC14` | `QuantumCallbackHandler_FrameDiffer` (cheater popup, never fires) |
| `0xDEB7EC` | `IntegrityValidateService.Initialize` |
| `0xDEB91C` | `IntegrityValidateService.IsAppValid` |
| `0xDEBA34` | `IntegrityValidateService.IsAndroidValid` |
| `0x9D82C8` | `ErrorHandlerUtils.DisplayBannedAccount` |

### BitStream Serialization

| Address | Function | Format |
|---------|----------|--------|
| `0x20DC58C` | `BitStream.Serialize(ref int)` | 4-byte LE int32 |
| `0x20EA21C` | `BitStream.Serialize(ref long)` | 8-byte int64 |
| `0x20EA29C` | `BitStream.Serialize(ref FP)` | FP.RawValue as int64 |
| `0x20EA2DC` | `BitStream.Serialize(ref FPVector2)` | X,Y each as int64 |
| `0x1340444` | `BitStreamExtensionsCore.Serialize(EntityRef)` | Index+Version |
| `0x1340574` | `BitStreamExtensionsCore.Serialize(PlayerRef)` | _index(int32) |
| `0x133A818` | `BitStreamExtensionsCore.Serialize(AssetGuid)` | Value(int64) |
| `0x1340630` | `BitStreamExtensionsCore.Serialize(AssetRef)` | AssetGuid(int64) |

### Matchmaking

| Address | Function |
|---------|----------|
| `0x17A90EC` | `MatchManager.StartMatch` |
| `0x17AAAFC` | `MatchManager.CreateMatchRequest` |
| `0x1796A1C` | `Matchmaking.SynchronizeRoom` |
| `0x1797FC0` | `Matchmaking.TryJoinOrCreateRandomRoom` |
| `0x1798A44` | `Matchmaking.GenerateCustomRoomProperties` |
| `0x1798EFC` | `Matchmaking.GenerateCustomRoomPropertiesForLobby` |
| `0x17991BC` | `Matchmaking.GenerateExpectedRoomProperties` |
| `0x17984F8` | `Matchmaking.GeneratePlayerProperties` |

### Savegame / Auth

| Address | Function |
|---------|----------|
| `0x2412DB4` | `DictionaryFile.Crypto` (DES-CBC, key="muKzm2cr", IV="ZCyNhQEz") |
| `0x24124C0` | `DictionaryFile.Save` |
| `0x2B886C0` | `DataManager.GetToken` |
| `0x2B88A3C` | `DataManager.SetUserData` |
| `0x3060F70` | `SavedAuth.GetInstance` (AES key from deviceUniqueIdentifier) |
| `0x3064D08` | `SavedAuth.Save` |

---

## 17. What's Missing (Needs Asset Extraction)

The following data exists in serialized Quantum asset files (.asset / .bytes), NOT in the binary:

1. **`statsPerLevel` tables** per card -- the actual damage/health/range numbers at each level
2. **Stage wave definitions** -- exact creep counts, types, spawn timing per map
3. **CreepLevelMultiplier tables** -- exact health/damage/armor scaling per creep level
4. **Spell cooldown timers** per spell per level
5. **Tower range values** per tower per upgrade tier
6. **Hero base stats** per hero per level (attack speed, damage, health, range)
7. **AssetGuid mappings** -- which int64 ID maps to which tower/spell/hero

### Extraction Approach

These assets are stored in Unity AssetBundles (`.assets` files in the APK or downloaded via CDN). Options include runtime instrumentation, asset extraction tools, or memory inspection of the loaded Quantum `Frame` after match initialization. This data is required to build a truly card-level-aware stat simulation instead of the current static ranges.
