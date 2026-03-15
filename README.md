# Olympus of the Heavens

> **Isometric co-op action RPG** — Unreal Engine 5.3 · C++ · Solo Development  
> [Steam Page](https://store.steampowered.com/app/3358020/Olympus_of_the_Heavens) · [ArtStation](https://www.artstation.com/kubrik)

![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3358020/5835d3278c7a8838d878ada77a514f6e0a468b4b/library_header.jpg?t=1739608354)

## Overview

Olympus of the Heavens is an isometric co-op action RPG built in Unreal Engine 5.3 using C++. Players ascend through a series of god-specific temples, each housing one of the 12 Olympian bosses, defeat them to seize their sacred flames, and build toward a final ascension. The game is structured around a **boss-rush loop** with deep character build customization — skill trees, equipment layering, artifact systems, and a crafting pipeline — combined with a procedural generation system that ensures structural variety across runs.

Multiplayer co-op is implemented via the Steam Online Subsystem with full session management, server-authoritative gameplay state, and synchronized procedural seed distribution. All gameplay systems, boss AI, environment assets, shaders, UI, and tooling were developed by a single developer.

---

## Engine & Technical Stack

| Layer | Technology |
|---|---|
| Engine | Unreal Engine 5.3 |
| Primary Language | C++ (gameplay, AI, networking, systems) |
| Scripting / Prototyping | Unreal Blueprint (UI, Sequencer cutscenes) |
| Rendering | Lumen (dynamic GI), custom isometric camera rig |
| AI | Unreal Behavior Tree + custom C++ task/decorator nodes |
| Networking | Steam Online Subsystem — listen-server, session management |
| Physics | Chaos — physics-driven combat props, destructibles |
| Replication | UE Actor Replication, `UPROPERTY(Replicated)`, RPCs |
| Platform | PC (Win64/Linux), Steam SDK |
| 3D Pipeline | ZBrush → Maya → Substance Painter → UE5 |
| Shader Authoring | UE Material Editor + HLSL custom nodes |

---

## Architecture Overview

```
OlympusOfTheHeavens/
├── Source/
│   ├── Core/
│   │   ├── OlympusCharacter.h/.cpp            # Base player character
│   │   ├── OlympusGameMode.h/.cpp             # Session management, temple sequencing
│   │   ├── OlympusGameState.h/.cpp            # Shared state: flames, god defeats, seed
│   │   └── OlympusPlayerState.h/.cpp          # Per-player: stats, inventory, flames
│   ├── Systems/
│   │   ├── FlameSystem/                        # Divine flame acquisition, ascension power
│   │   ├── SkillTree/                          # Node graph, unlock logic, stat routing
│   │   ├── EquipmentSystem/                    # Weapons, armor, artifact slots
│   │   ├── CraftingSystem/                     # Recipe DB, material requirements, output
│   │   ├── LootSystem/                         # Drop tables, rarity tiers, world loot
│   │   ├── ProceduralGen/                      # Temple layout, encounter, loot seeding
│   │   ├── CombatSystem/                       # Combo chains, dodge, hit detection
│   │   ├── AIController/                       # Boss AI, phase system, add spawning
│   │   ├── NetworkSubsystem/                   # Steam session, lobby, replication layer
│   │   └── ProgressionSystem/                  # Level gating, flame count, ascension state
│   ├── Bosses/
│   │   ├── OlympianBossBase.h/.cpp            # Shared boss framework
│   │   ├── Zeus/                               # Boss-specific logic
│   │   ├── Hades/
│   │   ├── Poseidon/
│   │   └── [...9 additional Olympians]
│   ├── Environment/
│   │   ├── TempleBase/                         # Shared modular temple components
│   │   ├── FloatingLands/                      # Overworld traversal areas
│   │   └── GodDomains/                         # Per-god biome overlays
│   └── UI/
│       ├── HUD/                                # Isometric HUD, skill bar, health/mana
│       ├── SkillTreeWidget/                    # Node graph UI
│       ├── InventoryWidget/                    # Equipment, artifacts, crafting
│       └── LobbyWidget/                        # Co-op session, player readiness
```

---

## Core Systems: Technical Detail

### 1. Divine Flame System

The Flame System is the primary progression currency and power amplifier. Each of the 12 Olympian gods drops a `FDivineFlame` on defeat — a typed resource representing the god's domain.

**Flame Architecture:**
- `FDivineFlame` struct: `EGodType GodOwner`, `float FlamePower`, `TArray<FFlameEffect> GrantedEffects`.
- Flame effects are data-driven via `UFlameEffectDataAsset` — each defines a stat modifier, a passive ability unlock, or a combat property override.
- Flames are stored in `UFlameInventoryComponent` on the player, replicated via `UPROPERTY(Replicated)`.
- **Ignition mechanic**: Once a player holds a minimum flame count threshold, they can spend flames to activate the personal ascension state — a time-limited power amplification mode that scales with total collected flame power.

```cpp
void UFlameInventoryComponent::IgniteAscension()
{
    if (CollectedFlames.Num() < AscensionMinFlames) return;

    float TotalPower = 0.f;
    for (const FDivineFlame& Flame : CollectedFlames)
    {
        TotalPower += Flame.FlamePower;
        ApplyFlameEffects(Flame.GrantedEffects);
    }

    AscensionPowerLevel = TotalPower;
    bAscensionActive = true;
    GetWorld()->GetTimerManager().SetTimer(
        AscensionTimer, this,
        &UFlameInventoryComponent::DeactivateAscension,
        AscensionDuration, false
    );
}
```

**Co-op Flame Handling:**
- Flames are player-owned resources — each co-op participant collects independently.
- On boss defeat, server broadcasts `OnGodDefeated` via `AOlympusGameState`; each client's `UFlameInventoryComponent` receives the flame grant via client RPC.
- `AOlympusGameState::DefeatedGods` is a replicated `TArray<EGodType>` used to gate temple access globally for all players in session.

---

### 2. Procedural Generation System
![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3358020/ss_f0ac4bb109e2a53d83bd3e8519ba083bbb0b6bf7.800x600.jpg)

Temple layouts, encounter configurations, loot distributions, and floating land structures are generated using a seeded procedural system.

**Seed Distribution in Co-op:**
- `GlobalSeed` is generated by the server host at session start and stored in `AOlympusGameState` as a replicated `int32`.
- All clients receive the same seed on join; procedural generation runs identically on all machines — deterministic, no generation data transmitted after seed sync.
- `FRandomStream` instances are derived per-zone: `FRandomStream TempleStream(GlobalSeed + (int32)GodType)`.

**Temple Layout Generation:**
- Each temple is built from modular room segments defined in `UTempleBiomeDataAsset` — room prefabs, connector pieces, dead ends, and arena chambers.
- A recursive corridor generation algorithm selects rooms from the biome asset's weighted mesh table, placing and connecting them until a minimum room count and mandatory arena chamber are satisfied.
- Room connections validated against an adjacency matrix to prevent isolated segments.

```cpp
void ATempleGenerator::GenerateLayout(int32 Seed, UTempleBiomeDataAsset* Biome)
{
    FRandomStream Stream(Seed);
    TArray<FRoomNode> RoomGraph;

    FRoomNode StartRoom = SpawnRoom(Biome->EntryRoom, FVector::ZeroVector);
    RoomGraph.Add(StartRoom);

    while (RoomGraph.Num() < Biome->MinRoomCount || !bArenaPlaced)
    {
        FRoomNode Parent = RoomGraph[Stream.RandRange(0, RoomGraph.Num() - 1)];
        URoomDataAsset* NextRoom = SelectWeightedRoom(Biome->RoomTable, Stream, bArenaPlaced);
        FVector Offset = ComputeConnectionOffset(Parent, NextRoom, Stream);

        if (ValidateAdjacency(Parent, NextRoom, RoomGraph))
        {
            RoomGraph.Add(SpawnRoom(NextRoom, Parent.WorldLocation + Offset));
            if (NextRoom->bIsArena) bArenaPlaced = true;
        }
    }
}
```

**Encounter & Loot Seeding:**
- Enemy spawn tables selected from `UEncounterDataAsset` weighted arrays using per-room stream offsets.
- Loot containers distributed via `FLootSpawnConfig` — seeded placement within room bounds, rarity tier selection using `FWeightedRandomSampler`.
- Hidden treasure locations seeded separately: `FRandomStream HiddenStream(GlobalSeed + RoomIndex + 9999)`.

---

### 3. Combat System
![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3358020/ss_e63f7a98f3be75a3d2f100bccfd56872d73cf086.800x600.jpg)

Combat is designed for isometric twin-stick style — responsive, combo-driven, with dodge as a core defensive tool. All combat logic is server-authoritative in co-op.

**Attack Chain System:**
- Combo sequences defined as `TArray<UAnimMontage*>` per weapon type.
- Input buffer window: attack input within the configured frame window queues the next combo step.
- Chain advancement is notify-driven: `AnimNotify_ComboOpen` opens the buffer; `AnimNotify_ComboClose` resets if no input received.
- Combo branch points allow directional divergence — holding a direction during the combo window selects an alternate montage branch.

**Dodge System:**
- Dodge is an i-frame window implemented via `bInvulnerable` flag on `UCombatComponent`, timed via `FTimerHandle`.
- Dodge direction computed from player input vector at dodge initiation; applies a physics impulse via `UCharacterMovementComponent::AddImpulse`.
- Dodge cooldown enforced per-player; in co-op, cooldown state is not replicated (client-local) but hit resolution is server-authoritative.

**Hit Detection:**
- Melee: swept capsule/box trace per attack frame, defined by weapon's `FHitboxConfig` (shape, offset, extent, active frame range).
- Ranged/Magic: `AProjectileBase` subclasses with `UProjectileMovementComponent`; hit resolved on server.
- Damage pipeline: `UGameplayStatics::ApplyDamage` → boss/enemy `TakeDamage` override → `UCombatComponent::ResolveDamage` applies resistance, elemental modifiers, and crit roll.

**Elemental & Status Effects:**
- Damage types mapped to `EElementType` enum: Physical, Lightning (Zeus), Shadow (Hades), Water (Poseidon), Fire (Hephaestus), etc.
- Each boss and enemy has `TMap<EElementType, float> ElementalResistance` — multipliers applied at damage resolution.
- Status effects (burn, stun, slow) implemented as `UStatusEffectComponent` on target; each effect is a `FStatusEffectData` struct with duration, tick rate, and tick effect.

---

### 4. Boss System — The 12 Olympians
![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3358020/ss_daf1b09592236f1dbc224665536f2e0e8e03cc5f.800x600.jpg)

Each boss (`AOlympianBossBase`) is a subclass with individual behavior trees, phase configurations, and attack pattern sets. The framework is shared; boss-specific logic is contained in overrides and data assets.

**Phase Architecture:**
- Health percentage thresholds define phase transitions, stored in `TArray<FBossPhaseConfig>` on each boss.
- `OnPhaseChange` delegate fires at threshold; injects new BT subtree via `UBehaviorTreeComponent::SetDynamicSubtree`.
- Phase transitions trigger: stat rescaling, new attack pattern activation, visual VFX (Niagara), arena modification events.

**Attack Pattern System:**
- Attack patterns are `FAttackPatternConfig` data assets: projectile type, count, arc, rotation speed, fire interval, activation condition.
- `AAttackPatternActor` spawned by AI during combat phases; manages its own spawn loop via `FTimerHandle`.
- Patterns vary by domain: Zeus patterns use scatter lightning strikes (randomized target points within arena bounds); Hades uses pursuing shadow orbs (`AActor` with custom tick-based tracking); Poseidon uses wave sweep patterns with timed AoE zones.

**Add Spawning:**
- Some bosses spawn minion waves at phase transitions.
- `ABossArenaManager` monitors the arena and triggers `SpawnEncounterWave` on phase change.
- Minion counts and types sourced from `FWaveConfig` data assets — compatible with the procedural encounter system.

**God-Specific Temple Design:**
- Each god's temple uses a dedicated `UTempleBiomeDataAsset` controlling room mesh sets, lighting color grading, material parameter overrides, and ambient Niagara systems.
- Zeus temple: exposed storm platforms, high vertical ceiling, lightning strike hazard zones as `ATriggerVolume` with damage-over-time.
- Hades temple: enclosed shadowed corridors, low visibility (Lumen exposure tuned down), shadow entity adds.
- Biome overlays applied via `APostProcessVolume` swaps on zone entry — each god domain has a unique color grade and fog configuration.

---

### 5. Skill Tree System
![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3358020/ss_df37c4f8a9a3635f22820f362029547a436e2b0f.800x600.jpg)

The skill tree is a node graph structure allowing player-directed build customization across combat style archetypes: melee, ranged, and magic.

**Node Graph Architecture:**
- Skill nodes defined in `USkillNodeDataAsset`: node type (passive stat, active ability, combat modifier), unlock cost (flame/currency), prerequisite node list, and effect config.
- `USkillTreeComponent` maintains the player's node graph state as a `TMap<FName, FSkillNodeState>` — keyed by node ID, value tracking unlock status.
- Prerequisite validation: on unlock attempt, component traverses `PrerequisiteNodes` array and confirms all are unlocked before granting.
- Unlocked passives are accumulated into a `FStatModifierSet` applied to the player's `UStatComponent` — all stat derivations read from this set rather than discrete variables.

**Active Abilities:**
- Active skill nodes grant abilities managed by `UAbilityManagerComponent`.
- Abilities are `UAbilityBase` subclasses — implementing `Activate()`, `OnCooldownEnd()`, and `GetAbilityInfo()`.
- Ability slots on the HUD are bound to input actions; slot count scales with player progression.

**Build Persistence:**
- Skill tree state serialized into `FOlympusPlayerSaveData` alongside equipment loadout.
- In co-op, skill tree is local to each player — not synchronized across session (each player builds independently).

---

### 6. Equipment & Artifact System
![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3358020/ss_a9ff185aa054e66af4bf37558cf827475819e1a7.800x600.jpg)

**Equipment Slots:**

| Category | Slots | Notes |
|---|---|---|
| Weapons | 2 (main + off-hand) | Melee, ranged, or magic staves |
| Armor | 4 (head, chest, legs, hands) | Stat bonuses + set bonuses |
| Artifacts | 3 | Unique passive effects, god-domain affinity |
| Accessories | 2 | Ring/amulet slots |

- All items defined via `UItemDataAsset`: stat block, item type tags, mesh reference, rarity tier, and effect list.
- Set bonuses: `UEquipmentComponent` evaluates active set tags on equip/unequip; broadcasts `OnSetBonusActivated` when threshold count reached.
- **Artifacts**: Rarer items with unique `FArtifactEffect` structs — may override combat behaviors (e.g., projectile bouncing, AoE on dodge, elemental conversion).

**Loot Rarity Tiers:**

```
Common → Uncommon → Rare → Epic → Legendary → Divine (god-drop exclusive)
```

- Rarity determined at drop time using weighted `FLootRarityTable` seeded from room/encounter stream.
- Divine-tier items only drop from Olympian boss defeats — one per boss, guaranteed.

---

### 7. Crafting System

- Recipes defined in `UCraftingRecipeDataAsset`: required `TArray<FMaterialRequirement>` (item type + count), output item data asset reference, and crafting station type tag.
- `UCraftingComponent` on the player evaluates `UInventoryComponent` contents against recipe requirements.
- Crafting station actors (`ACraftingStationActor`) placed in floating land areas and temple approach zones — interact to open `UCraftingWidget`.
- Material items are world-drop and boss-drop sourced; rarer materials gated behind higher temple progression.
- Upgrade recipes allow existing equipment to be enhanced: `FUpgradeConfig` on item data defines material cost per upgrade tier, stat multipliers per tier, and max tier.

---

### 8. Networking & Co-op Architecture

**Session Management:**
- `UOnlineSessionClient` + custom `USessionManagerComponent` handle lobby creation, discovery, and join flow.
- Host creates a Steam session; other players discover and join via Steam invite or lobby browser.
- Session configuration: max 4 players, server-authoritative game mode, late-join disabled after first boss engagement.

**State Replication:**
- `AOlympusGameState`: replicated `TArray<EGodType> DefeatedGods`, `int32 GlobalSeed`, `int32 CurrentTempleIndex`.
- `AOlympusPlayerState`: replicated `UFlameInventoryComponent` data, player level, current stats.
- Equipment loadout replicated via `UPROPERTY(Replicated)` on `UEquipmentComponent` — changes broadcast to all clients.

**Combat Replication:**
- Attack inputs are local-predicted; damage resolution is server-authoritative.
- `Server_RequestAttack` RPC validates input on server, applies damage, replicates result to all clients.
- Boss health and phase state maintained on server in `AOlympianBossBase`; replicated to all clients via `UPROPERTY(ReplicatedUsing = OnRep_BossPhase)`.

**Movement Replication:**
- Standard `UCharacterMovementComponent` replication with custom `FSavedMove` extension for dodge state and ability activation flags.
- Isometric camera rig is client-local — not replicated; each player manages their own camera independently.

**Procedural Sync:**
- `GlobalSeed` transmitted to joining clients via `AOlympusGameState` replication — all world generation runs client-side from the same seed.
- No geometry data transmitted; determinism guarantees identical world layout on all machines.

---

### 9. Isometric Camera System

The camera is a dedicated `AOlympusCameraActor` managed by a custom `UPlayerCameraManager` override.

- Fixed isometric angle: 45° horizontal rotation, 60° vertical pitch — configurable per temple via `FCameraConfig` data asset.
- Camera position follows player with configurable `LagSpeed` via `UCameraLagComponent`.
- In co-op: camera targets the average position of all living players within a maximum spread radius; if players exceed spread limit, each gets an independent camera instance.
- Temple boss arenas lock camera to a fixed position/angle for the encounter — `ABossArenaManager` triggers `AOlympusCameraActor::SetArenaMode` on boss room entry.
- Smooth interpolation between free-follow and arena-locked modes via `FInterpTo` on camera transform.

---

### 10. Floating Lands & Overworld

Floating land areas serve as the overworld between temples — exploration zones with hidden loot, crafting stations, and optional encounters.

- Land platforms generated via seeded `AFloatingLandGenerator`: platform count, size, elevation, and bridge connections seeded from `GlobalSeed + LandIndex`.
- Each land area has a biome type drawn from `UFloatingLandBiomeDataAsset` — determines mesh sets, foliage density, ambient entities, and loot table.
- Hidden treasures: `AHiddenTreasureActor` instances placed at seeded locations; require interaction or solving a minor environmental puzzle to access.
- Traversal: platforms connected by bridges or jump distances calibrated to character movement range; no flying or grapple — pure platforming navigation.

---

### 11. Save & Progression Persistence

- `FOlympusPlayerSaveData`: player stats, skill tree state, inventory, equipped items, collected flames, defeated gods list, crafting unlocks.
- `FOlympusSessionSaveData`: global seed, current temple index, world loot state (picked-up item IDs), boss defeat order.
- Async save on boss defeat and on return to overworld hub.
- Co-op save: host's session save is authoritative for world state; each player saves their own character data independently. On session resume, host's `FOlympusSessionSaveData` initializes the world; players load their own character states.

---

## Performance Targets & Optimization

| Target | Approach |
|---|---|
| 60 fps (PC, 1080p+, co-op 4-player) | LOD chains on all character and boss meshes; Nanite on static temple geo |
| Network | Minimal RPC traffic — movement replicated via CMC; only attack/ability RPCs sent |
| Procedural gen | All generation runs at session start, not at runtime during play; async loading for new temple zones |
| AI | Behavior Tree updates throttled for off-screen adds; full tick only for boss and nearby enemies |
| Lumen | Hardware RT on supported GPUs; software fallback tuned per biome for consistent performance |
| Physics | Chaos only on destructible arena props and projectiles; no persistent simulation on terrain |

---

## Development Scope

| Category | Detail |
|---|---|
| Developer count | 1 (solo) |
| Engine | Unreal Engine 5.3 |
| Languages | C++, HLSL |
| 3D Assets | All original — modeled, textured, rigged, animated by developer |
| Boss count | 12 Olympian gods + minion variants |
| Gameplay systems | 11+ discrete systems (see above) |
| Multiplayer | Steam co-op, up to 4 players |
| Platform | PC Windows / Linux (Steam) |
| Development tools | UE5 Editor, ZBrush, Maya, Blender, Substance Painter, Photoshop, After Effects |

---

## Related Projects

| Project | Description |
|---|---|
| [TIME SOUL](https://store.steampowered.com/app/2928270/TIME_SOUL) | Souls-like action platformer; parkour, time-as-resource, procedural gen — UE5.1 |
| [U.N. Owen Was Her](https://store.steampowered.com/app/3420540/UN_Owen_Was_Her) | Third-person horror; hunger/transformation AI, bullet-hell boss, seal progression — UE5.3 |
| [Blood Garden](https://kubrik.itch.io/bloodgarden) | Souls-like melee combat; stamina system, parry, enemy AI |
| [Royal Jump](https://play.google.com/store/apps/details?id=com.Kubrick.RoyalJump) | Mobile platformer; touch controls, physics movement, mobile optimization |
| [ArtStation Portfolio](https://www.artstation.com/kubrik) | 3D modeling — characters, creatures, props, environments |

---

## Developer

**Kubrik** — Developer & 3D Artist  
9 years web development · 7 years 3D modeling · 5 years Unreal Engine C++  
5 shipped commercial games as sole developer.

[Steam](https://store.steampowered.com/search/?developer=Kubrik) · [ArtStation](https://www.artstation.com/kubrik) · [itch.io](https://kubrik.itch.io)

![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3358020/702941c01097adfa11c36d6a16a2b609942d80c4/library_hero.jpg?t=1739608354)
---

*All code, design, and custom assets produced by a single developer. No third-party gameplay code used in core systems.*
