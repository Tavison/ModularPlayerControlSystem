# Modular Player Control System (MPCS)

A tag-driven input orchestration plugin for Unreal Engine 5. MPCS sits on top of Enhanced Input and gives you a single API for activating and deactivating named input modes — locomotion, UI, vehicle, cutscene — rather than managing mapping contexts and response logic by hand each time a mode changes.

---

## Features

- **Layer system** — group an Input Mapping Context and its response logic under a Gameplay Tag. Activate or deactivate the whole group in one call.
- **Binding channels** — decouple which keys are active (layers) from which logic responds (channels). Multiple layers can share a channel; one layer can activate multiple channels.
- **Orchestrator** — single authority over all layer state. Blueprint-callable API with transaction batching and a `PushGate` / `PopGate` mechanism for hard blocks during cutscenes or loading screens.
- **Provider components** — attach to any actor (pawn, vehicle, ability component) to contribute input blocks. Blocks register and unregister automatically on spawn/despawn and possession change.
- **Split-screen / local multiplayer** — provider components support per-possessor, all-local-players, and manual registration modes.
- **Observable state** — `OnLayerActivated` / `OnLayerDeactivated` delegates let UI and game systems react to mode changes without polling.

---

## Requirements

- Unreal Engine 5 (developed and tested on UE 5.7)
- Enhanced Input plugin enabled
- GameplayTags enabled (on by default in UE5 projects)

---

## Installation

### Project plugin

1. Copy the `ModularPlayerControlSystem` folder into your project's `Plugins/` directory.
2. Add it to your `.uproject`:
```json
{
  "Plugins": [
    { "Name": "ModularPlayerControlSystem", "Enabled": true }
  ]
}
```
3. Regenerate project files and rebuild.

### Plugin-to-plugin dependency

Add to your `.uplugin`:
```json
{
  "Plugins": [
    { "Name": "ModularPlayerControlSystem", "Enabled": true }
  ]
}
```

Add to your module's `.Build.cs`:
```csharp
PublicDependencyModuleNames.AddRange(new string[] { "MPCS_Core", "EnhancedInput", "GameplayTags" });
```

---

## Quick Start

The full walkthrough is in [Docs/UserGuide.md](Docs/UserGuide.md). The short version:

1. Enable Enhanced Input in *Project Settings → Input* (`EnhancedPlayerInput` + `EnhancedInputComponent`).
2. Add an `MPCS_InputRootComponent` to your Player Controller. Either inherit from `AMPCS_ModularPlayerController` or add a two-line `SetupInputComponent` override.
3. Define Gameplay Tags for your layers and channels, e.g.:
   ```
   MPCS.Input.Layer.Locomotion
   MPCS.Input.Channel.Movement
   ```
4. Create a Layer Blueprint (`BP_Layer_Locomotion`) and a Layer Definition Data Asset (`DA_Layer_Locomotion`) linking the tag, the layer class, and the channels it activates.
5. Create a Host Slot Blueprint (`BP_HostSlot_Default`) listing your layer definitions, and an Orchestrator Blueprint (`BP_Orchestrator_Default`) listing the layers that should be active at startup.
6. Wire both Blueprints to the `MPCS_InputRootComponent` on your Player Controller.
7. Create an Input Binding Block Blueprint (`BB_MoveForward`): set `BindingChannelId`, `Action`, and `Triggers`; implement `OnTriggered`.
8. Add an `MPCS_BindingProviderComponent` to any actor that owns blocks and fill in its `BindingBlockClasses` list.

At runtime, layer control goes through the orchestrator:
```
Get Local Player
    → Get Subsystem (MPCS_InputOrchestratorLocatorSubsystem)
    → Get Orchestrator
    → Activate Layer (MPCS.Input.Layer.UI)
```

---

## When to Use MPCS vs. Raw Enhanced Input

Enhanced Input handles *what a key does*. MPCS handles *which keys are active right now and why*. The two complement each other.

**Raw Enhanced Input is sufficient when** your game has one or two fixed input modes, you switch contexts in a handful of places, and no actor other than the Player Controller contributes input blocks.

**MPCS pays for its setup cost when** you have three or more input modes that change at runtime, multiple actors contribute input, you need hard blocks for cutscenes or loading, or you want observable mode-change events without building that infrastructure yourself.

See [Docs/UserGuide.md — MPCS vs. Raw Enhanced Input](Docs/UserGuide.md#mpcs-vs-raw-enhanced-input) for the full comparison.

---

## Documentation

| Document | Audience | Contents |
|---|---|---|
| [Docs/UserGuide.md](Docs/UserGuide.md) | Designers and Blueprint users | Concepts, step-by-step setup, Blueprint workflows, extended patterns (gates, delegates, dynamic providers) |
| [Docs/TechnicalReference.md](Docs/TechnicalReference.md) | C++ developers | Full API surface, initialization sequence, contracts and invariants, extension points, test infrastructure |

---

## License

[LICENSE](LICENSE)
