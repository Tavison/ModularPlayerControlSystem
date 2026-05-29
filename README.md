# Modular Player Control System (MPCS)

A tag-driven input orchestration plugin for Unreal Engine 5. MPCS sits on top of Enhanced Input and gives you a single API for activating and deactivating named input modes — locomotion, UI, vehicle, cutscene — rather than managing mapping contexts and response logic by hand each time a mode changes.

---

## Why MPCS

### Input logic belongs with the thing that owns it

In a standard Enhanced Input setup, all input handling gravitates toward the Player Controller. A vehicle that can be driven, a turret that can be manned, a terminal that can be interacted with — their input responses end up centralized, wired up and torn down by code that has to know about every actor type in the game.

MPCS inverts this. Any actor can carry its own input blocks. A vehicle Blueprint declares which layer activates its controls, which channel its blocks belong to, and implements the steering, throttle, and horn responses directly on the vehicle. When the player possesses it, the vehicle's provider component automatically registers those blocks with the player's input system. When the player leaves, it unregisters. The Player Controller never needs to know the vehicle exists.

### Smart Objects become fully self-describing

MPCS is particularly well-suited to Smart Object patterns. A puzzle panel, a mounted weapon, a door with a hold-to-open interaction — each object can own everything about its input interface:

- The **input layer** that activates while the player is engaged with it
- The **binding channel** its actions belong to
- The **action responses**, implemented in the object's own Blueprint

No other code needs to be written or modified to add a new interactive object to the game. Drop it in the level, add a provider component, and its input works. Remove it, and the input cleans itself up.

### One call to change everything

Without MPCS, switching input modes means manually adding and removing mapping contexts, rebinding handlers, and tracking what was active so you can restore it later. With MPCS, it is one orchestrator call:

```
Orchestrator → Activate Layer (MPCS.Input.Layer.Vehicle)
```

The orchestrator handles adding the mapping context to Enhanced Input, recomputing which channels are live, wiring the correct blocks, and broadcasting state-change events to any observers. The atomic transaction and gate APIs handle edge cases like simultaneous mode switches and cutscene hard blocks without additional code.

---

## Features

- **Layer system** — group an Input Mapping Context and its response logic under a Gameplay Tag. Activate or deactivate the whole group in one call.
- **Binding channels** — decouple which keys are active (layers) from which logic responds (channels). Multiple layers can share a channel; one layer can activate multiple channels.
- **Provider components** — attach to any actor to contribute input blocks. Blocks register and unregister automatically on spawn/despawn and possession change. No central registration needed.
- **Orchestrator** — single authority over all layer state. Blueprint-callable API with transaction batching and a `PushGate` / `PopGate` mechanism for hard blocks during cutscenes or loading screens.
- **Observable state** — `OnLayerActivated` / `OnLayerDeactivated` delegates let UI and game systems react to mode changes without polling.
- **Split-screen / local multiplayer** — provider components support per-possessor, all-local-players, and manual registration modes.

---

## When to Use MPCS vs. Raw Enhanced Input

Enhanced Input handles *what a key does*. MPCS handles *which keys are active right now and why*. The two complement each other.

**Raw Enhanced Input is sufficient when** your game has one or two fixed input modes, you switch contexts in a handful of places, and all input handling lives on the Player Controller.

**MPCS pays for its setup cost when:**
- Multiple actor types contribute input (pawns, vehicles, interactive objects)
- You want those actors to be self-contained — no external wiring when they appear or disappear
- You have three or more input modes that change at runtime
- You need hard blocks for cutscenes or loading screens
- UI or game systems need to observe input-mode changes without polling

See [UserGuide.md — MPCS vs. Raw Enhanced Input](UserGuide.md#mpcs-vs-raw-enhanced-input) for the full comparison.

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

The full walkthrough is in [UserGuide.md](UserGuide.md). The short version:

1. Enable Enhanced Input in *Project Settings → Input* (`EnhancedPlayerInput` + `EnhancedInputComponent`).
2. Add an `MPCS_InputRootComponent` to your Player Controller. Either inherit from `AMPCS_ModularPlayerController` or add a two-line `SetupInputComponent` override.
3. Define Gameplay Tags for your layers and channels, e.g.:
   ```
   MPCS.Input.Layer.Locomotion
   MPCS.Input.Channel.Movement
   ```
4. Create a Layer Blueprint (`BP_Layer_Locomotion`) and a Layer Definition Data Asset (`DA_Layer_Locomotion`) linking the tag, the layer class, and the channels it activates.
5. Create a Host Slot Blueprint (`BP_HostSlot_Default`) listing your layer definitions, and an Orchestrator Blueprint (`BP_Orchestrator_Default`) listing the layers active at startup.
6. Wire both to the `MPCS_InputRootComponent` on your Player Controller.
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

## Documentation

| Document | Audience | Contents |
|---|---|---|
| [UserGuide.md](UserGuide.md) | Designers and Blueprint users | Concepts, step-by-step setup, Blueprint workflows, extended patterns (gates, delegates, dynamic providers) |
| [TechnicalReference.md](TechnicalReference.md) | C++ developers | Full API surface, initialization sequence, contracts and invariants, extension points, test infrastructure |

---

## License

[LICENSE](LICENSE)
