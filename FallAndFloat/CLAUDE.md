# FallAndFloat

2D Unity mobile-style arcade game. Meteors fall on a village of five houses; the player holds the screen to make a mage blow a bubble that grows while held. Releasing it catches a meteor if the bubble size is within 0.25 of the meteor size (`Meteor.IsFittingSize`), otherwise the bubble pops. Meteor hits set houses on fire; when all burn → game over → main menu. Score increments per catch.

- Unity **6000.6.0f1**, **URP 17.6.0** with 2D Renderer (`Assets/Settings`).
- Key packages: VContainer 1.19.0 (DI), UniTask, PrimeTween, TextMesh Pro, uGUI. Legacy `UnityEngine.Input` (no Input System package).

## Exploration rules

- **Skip by default**: `Library/`, `Packages/`, `Assets/Packages/` (NuGet), `Temp/`, `Logs/`, `obj/`, `UserSettings/`, `.idea/`, `*.meta`, `*.csproj`. Only look there when explicitly asked.
- Third-party in `Assets/Plugins/` (PrimeTween, vHierarchy, vInspector, IngameDebugConsole, AssetContract, TextMesh Pro) — don't analyze. Exception: `Assets/Plugins/YolarUtils/` is the in-house utility library.
- Don't read scene/prefab YAML unless needed.

## Assemblies

| asmdef | Assembly | Contents | References |
|---|---|---|---|
| `Assets/Core/game.core.asmdef` | `game` | Audio, Infrastructure (bootstrap, state machine, input, events), StaticData, Utils, GameLoop | game.plugins |
| `Assets/Features/game.feature.asmdef` | `game.feature` | Abilities, Bubbles, Health, Mage, Meteors, StaticData, vContainer scopes/installers | game, game.plugins |
| `Assets/UI/game.ui.asmdef` | `game.ui` | Menus, HUD, AbilitiesWindow, PrimitiveMvc | game, game.feature, game.plugins |
| `Assets/Plugins/YolarUtils/game.plugins.asmdef` | `game.plugins` | Extensions, Postponer, SceneManagement, SerializableDictionary, SmartLogger (`SLogger`), StateMachine | — |
| `Assets/Core/Editor/game.editor.asmdef` | `game.editor` | Editor tools (CanvasCameraBinderEditor) | game |

## Boot flow & scenes

Build order: `scene_bootstrap` → `scene_mainMenu` → `scene_game`.

VContainer root scope `Features/vContainer/ProjectScope.cs` (prefab `Assets/Prefabs/Infrastructure/ProjectScope.prefab`, set in `Assets/Settings/VContainerSettings.asset`) registers singletons (state machine, `IMeteorFactory`, `IInputHandler`, `IAudioService`, `ISceneLoader`, configs) and entry point `Core/Infrastructure/GameBootstrapper.cs` → `BootstrapState` → `LoadLevelState(scene_mainMenu)` → `GameLoopState` (`Core/Infrastructure/StateMachine/`).

Note: `UI/MainMenuController.cs` loads `Scenes.Game` directly via `ISceneLoader`, bypassing the state machine; `GameOverScreen`/`PauseMenu` go back through `LoadLevelState`.

Scene scope: `Features/vContainer/SceneScope.cs` runs `MonoInstaller[]` (`LevelInstaller`, `TutorialInstaller`, `UiInstaller` — currently empty `Install` bodies).

## Main systems

- **Events**: `Core/Infrastructure/GameEvents.cs` — static C# event bus (`OnBubbleStartBlow`, `OnBubbleEndBlow`, `OnScoreChanged`, `OnDamage`, `OnBubblePop`, `OnGameOver`).
- **Audio**: `Core/Audio/` — `AudioService` (`IAudioService.PlayOneShot`), `AudioSourcePool`, `AudioSourceFactory`, `AudioEntity` (AudioSource wrapper), `RandomAudioProviderSo`, `AudioServiceConfigSo` (SFX/Music mixers).
- **Input**: `Core/Infrastructure/Input/InputHandler.cs` (ITickable, Escape/Back → `OnBack`); `UI/UiInputReader.cs` (pointer down/up for blowing).
- **Meteors**: `Features/Meteors/Scripts/` — `Meteor` base + `MeteorFiery` (extinguish first), `MeteorIcy` (remove ice first), `MeteorSplitting` (splits after 5–8 s); `MeteorFactory` (resolver-injected), `MeteorConfigSo` (prefabs by type), `MeteorSpawner` (coroutine, every 3 s default).
- **Bubbles / Mage**: `Features/Bubbles/` (`Bubble`, `BubbleSpawner`), `Features/Mage/Mage.cs`.
- **Health**: `Features/Health/` — `House`, `HealthHouses`, `HealthRemoveNotifier`.
- **Abilities**: `Features/Abilities/` — abstract button-based `Ability`; `PerfectCatch`, `BubbleBarrier` (temp no damage), `Rebuild` (repair one house).
- **Tutorial**: `Core/GameLoop/GameTutorial.cs`, gated by PlayerPrefs key `"Tutorial"`.
- **UI**: `UI/` — `MainMenuController`, `PauseMenu` (timescale 0.15), `GameOverScreen`, `GameScore`, `AbilitiesWindow/`, `PrimitiveMvc/GameWindow.cs` (window base with PrimeTween).
- **Static data**: `Core/StaticData/Scenes.cs`, `Tags.cs`; `Features/StaticData/ObjectSize.cs`.
- **Configs** (`Assets/Configs`): `audio-service-config`, `canvas-scaler-config`, `game-textures-config`, `meteor-config`.

## Conventions

- Namespaces mirror folders: `Core.*`, `Features.*`, `UI.*`. One class per file, PascalCase.
- DI: VContainer; MonoBehaviours get dependencies via `[Inject] private void Construct(...)`.
- Fields: `[SerializeField] private` or `[field: SerializeField]` auto-properties.
- ScriptableObjects: `*So` suffix, `[CreateAssetMenu(menuName = "Fall And Float/Configs/...")]`.
- Assets: prefabs `p-`/`p_` prefix, configs `*-config.asset`, scenes `scene_*`.
- Async: UniTask for scene flow, coroutines in gameplay, PrimeTween for tweens, Postponer for sequencing.
- Logging: `SLogger` (YolarUtils). Indentation is tabs.
