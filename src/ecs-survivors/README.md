# ECS Survivors

Unity 2D survivor-like prototype built around an ECS gameplay core. The project uses Entitas for entity/component/system composition and Zenject for dependency injection, scene bootstrap, factories, and service lifetime management.

## Project Status

This repository is a Unity project, not a standalone .NET package. Open the project from this directory:

```text
src/ecs-survivors
```

The codebase already contains generated Entitas contexts and components under `Assets/Code/Generated`. Treat that directory as generated output: do not manually edit it unless you are intentionally patching generated code and understand the regeneration impact.

## Tech Stack

- Unity `2022.3.62f3`
- C# / Unity MonoBehaviour runtime
- Entitas ECS
- Zenject DI
- DOTween settings in `Assets/Resources/DOTweenSettings.asset`
- TextMesh Pro
- Unity 2D feature package
- Newtonsoft.Json for serialization

## Main Scenes

| Scene | Purpose |
| --- | --- |
| `Assets/Scenes/Boot.unity` | Entry scene. Owns project bootstrap and DI startup. |
| `Assets/Scenes/HomeScreen.unity` | Meta/home UI scene. Shop, storage, start flow. |
| `Assets/Scenes/Meadow.unity` | Gameplay level scene. Current battle scene. |

Always start from `Boot.unity` unless you are debugging a scene-specific issue. The bootstrap path initializes services, loads progress, actualizes offline/meta simulation, then moves into the home screen.

## Quick Start

1. Install Unity `2022.3.62f3` or a compatible `2022.3 LTS` editor.
2. Open `src/ecs-survivors` through Unity Hub.
3. Let Unity restore packages from `Packages/manifest.json`.
4. Open `Assets/Scenes/Boot.unity`.
5. Enter Play Mode.

If Unity asks to reimport assets, allow it. `Library`, `Temp`, `Logs`, `obj`, and user IDE files should remain local artifacts and should not be treated as source.

## Repository Layout

```text
Assets/
  Code/
    Common/          Shared ECS components, helpers, destruct flow, entity utilities.
    Generated/       Entitas-generated contexts, entities, matchers, components.
    Gameplay/        Battle runtime: hero, enemies, movement, abilities, loot, effects.
    Infrastructure/  DI, state machine, scene loading, asset loading, view binding.
    Meta/            Home screen, shop, storage, offline simulation/meta systems.
    Progress/        Progress data, provider, save/load services.
  Resources/         Runtime-loaded configs, prefabs, UI, ProjectContext.
  ResourcesStatic/   Static environment/level assets and editor-facing resources.
  Scenes/            Boot, HomeScreen, Meadow.
Packages/            Unity package manifest.
ProjectSettings/     Unity project settings and editor version.
```

## Runtime Architecture

### Bootstrap

`BootstrapInstaller` is the composition root. It binds:

- Entitas contexts: `GameContext`, `InputContext`, `MetaContext`
- infrastructure services: scene loading, asset provider, identifiers, system factory
- gameplay services: static data, level data, statuses, level-up, ability upgrades
- factories: hero, enemy, armament, ability, effects, statuses, loot, UI/shop items
- UI services and UI factories
- progress services
- game states

After installation, `BootstrapInstaller.Initialize()` enters `BootstrapState`.

### State Machine Flow

The high-level lifecycle is:

```text
BootstrapState
  -> LoadProgressState
  -> ActualizeProgressState
  -> LoadingHomeScreenState
  -> HomeScreenState
  -> LoadingBattleState
  -> BattleEnterState
  -> BattleLoopState
  -> GameOverState
```

Important behavior:

- `BootstrapState` loads static data.
- `LoadProgressState` loads existing progress or creates new meta storage.
- `ActualizeProgressState` simulates offline progress with a two-day cap.
- `LoadingHomeScreenState` loads `HomeScreen`.
- `LoadingBattleState` loads a battle scene payload, currently `Meadow`.
- `BattleEnterState` places the hero at the level start point.
- `BattleLoopState` owns `BattleFeature` initialization, execution, cleanup, and teardown.

### Battle Feature

`BattleFeature` is the gameplay system graph. Current ordering:

```text
InputFeature
BindViewFeature
HeroFeature
EnemyFeature
DeathFeature
LootingFeature
LevelUpFeature
MovementFeature
AbilityFeature
ArmamentFeature
CollectTargetsFeature
EffectApplicationFeature
EnchantFeature
EffectFeature
StatusFeature
StatsFeature
GameOverOnHeroDeathSystem
ProcessDestructedFeature
```

System order matters. If a feature depends on components produced by another feature in the same frame, place it after the producer.

## ECS Conventions

- Components live near the feature they belong to, usually in `*Components.cs` files.
- Systems are grouped under feature folders and composed through `*Feature.cs` classes.
- Use `GameContext` for battle runtime entities.
- Use `InputContext` for frame input signals.
- Use `MetaContext` for home/meta/progress simulation entities.
- Use `isDestructed` instead of destroying runtime entities directly from arbitrary systems.
- Cleanup is centralized through `ProcessDestructedFeature` and related cleanup systems.
- Entity creation helpers are in `Assets/Code/Common/Entity`.

## Views And Prefabs

Runtime views are ECS-bound through the infrastructure view layer:

- `EntityViewFactory` creates and binds Unity views.
- `BindViewFeature` resolves pending view bindings.
- Entity view registrars attach Unity-side references back into ECS components.

Most runtime assets are loaded from `Assets/Resources`, including:

- gameplay prefabs: hero, enemies, armaments, loot, effects
- UI prefabs: root, home HUD, shop, level-up, game-over, ability cards
- configs: abilities, enchants, loot, level-up, shop items, AFK gain

When adding a new runtime prefab/config, keep the path stable if it is referenced by static data or factories.

## Static Data

Static gameplay/meta data is loaded by `IStaticDataService` during bootstrap. Current config groups include:

- abilities: Garlic Aura, Orbiting Mushroom, Vegetable Bolt
- enchants: explosive and poison armaments
- loot: experience, healing, buff items
- shop items: boosters
- level-up options
- AFK gain
- window/UI configuration

Prefer adding new tunables as ScriptableObject configs under `Assets/Resources/Configs` rather than hardcoding values into systems.

## Progress And Save/Load

Progress is handled by:

- `IProgressProvider`
- `ISaveLoadService`
- progress DTOs under `Assets/Code/Progress`
- JSON serialization infrastructure

New progress fields should be added through data objects first, then wired into creation, loading, save, and actualization paths. Keep backward compatibility in mind when changing serialized structure.

## Adding Gameplay Content

### New Ability

1. Add an ability id.
2. Add or extend ability config data.
3. Create/update the ability factory branch.
4. Add systems under `Gameplay/Features/Abilities` or related armament/effect feature.
5. Register systems in the correct feature order.
6. Add config asset under `Assets/Resources/Configs/Abilities`.
7. Add prefabs under `Assets/Resources/Gameplay/Abilities` if a view is required.

### New Enemy

1. Add enemy type/config data.
2. Extend `EnemyFactory` and static data loading.
3. Add prefab under `Assets/Resources/Gameplay/Enemies`.
4. Ensure movement, lifetime, stats, view binding, and collision components are assigned consistently.

### New Loot Item

1. Add loot config.
2. Extend `LootFactory` if behavior differs from existing loot.
3. Add pickup/effect systems if the item introduces a new effect path.
4. Add prefab under `Assets/Resources/Gameplay/Loot`.

## Development Notes

- Keep gameplay logic in ECS systems, not in MonoBehaviours.
- Keep MonoBehaviours focused on view, registration, animation hooks, or Unity integration.
- Keep services deterministic where practical; inject `ITimeService` and `IRandomService` instead of using Unity globals directly in systems.
- Do not bypass factories for runtime entity creation unless the entity is intentionally trivial and local to a system.
- Keep system execution order explicit in feature classes.
- Avoid manual changes in `Assets/Code/Generated` unless regeneration is not available and the change is deliberate.

## Tests And Validation

The project includes Unity Test Framework as a package, but no project-specific automated test suite was identified in the current tree.

Minimum validation before merging gameplay changes:

- Open `Boot.unity` and enter Play Mode.
- Start a battle from the home screen.
- Verify hero spawn, movement, enemy spawn, damage/death, loot pickup, level-up, and game-over flow.
- Check Unity Console for DI binding errors, missing Resources paths, and Entitas cleanup errors.

## Known Constraints

- `Assets/Code/Generated` is checked into the project and should be considered generated Entitas code.
- Scene constants are partially centralized; `Meadow` is in `Scenes`, while `HomeScreen` is currently hardcoded in `LoadingHomeScreenState`.
- The current README documents the observed project structure and runtime flow; it does not define build/release automation because no project-specific build scripts were found.
