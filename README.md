 Mafia Game Manager Android

A single-device Android manager for an offline Mafia game.

## Status
V1 architecture and game rules are designed. Implementation has not started yet.

## Modules
- `:core`: pure JVM game engine, no Android dependency
- `:app`: Kotlin + Jetpack Compose Android app
- Dependency direction: `app -> core`

## Source of truth
Read `docs/ANDROID_APP_DESIGN.md` before implementing game logic.

## Reference
Previous Java algorithm repository:
https://github.com/HunterYim/MafiaGameAlgorithm_java

Use it as a logic reference, but do not copy the old blocking UI/input structure into the Android app.
