# Mafia Game Manager - Codex Working Guide

## Read first
1. `AGENTS.md`
2. `README.md`
3. `docs/ANDROID_APP_DESIGN.md`

`docs/ANDROID_APP_DESIGN.md` is the source of truth for V1 game rules and architecture.

## Project rules
- Repository: `MafiaGameManager_android`
- `:core` is a pure JVM game engine and must not depend on Android.
- `:app` uses Kotlin + Jetpack Compose and depends on `:core`.
- UI renders `GameUiState` and emits `GameEvent`; UI must not calculate game rules.
- `GameEngine` / Resolver logic is the game-rule source of truth.
- Preserve information asymmetry through UI projection.
- Do not over-restrict strategically bad but rule-legal choices.
- All player-chosen active abilities are skippable by default.
- Terrorist retaliation after actual daytime execution is mandatory.
- Automatic draw applies only to the explicit permanent final 1:1 stalemates documented in the design.
- Other unresolved or strategic stalemates use manual draw.
- Do not silently invent large systems or redesign settled rules.

## Reference implementation
Use `HunterYim/MafiaGameAlgorithm_java` branch `develop` as reference only.

## Initial implementation order
1. Gradle project with `:core` and `:app`
2. Core enums/data models
3. `GameState`, `PendingAction`, `GameEvent`
4. `GameEngine` progression skeleton
5. Basic full night/day cycle
6. Roles incrementally
7. Victory / final-1:1 stalemate handling
8. `GameState -> GameUiState` projection
9. Compose UI
10. Unit tests alongside rule implementation
