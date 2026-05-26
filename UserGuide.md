# Modular Player Control System — User Guide

## Overview

MPCS is an Unreal Engine 5 plugin for managing input mode transitions: activate a named layer and MPCS handles adding its mapping context, wiring its response logic, and notifying observers — without you touching the Player Controller or Enhanced Input Component directly.

MPCS layers on top of Unreal's Enhanced Input system. The goal is to let
designers swap large groups of input behavior at runtime — switching to UI
input during a menu, locking input during a cutscene, temporarily overriding
the camera during a QTE — without rewriting the Player Controller or the
Enhanced Input Component every time. It does this by splitting "which keys
does the engine recognize right now" (the *layer*) from "which logic responds
to those keys right now" (the *binding channel*), and using a small gameplay-tag
vocabulary for both.

The runtime has five moving pieces. External code reaches the orchestrator
and registry through a LocalPlayer subsystem rather than walking the component
tree; the InputRootComponent itself is found via `FindComponentByClass` on the
PlayerController, but nothing else needs to do that lookup directly:

| Piece | Role |
|---|---|
| `UMPCS_InputRootComponent` | Composition root on the Player Controller. Owns the other pieces. |
| `UMPCS_InputLayerHostSlot` | Manages per-layer `InputLayer` instances and the active channel set. |
| `UMPCS_InputOrchestratorSlot` | Authority over which layers are active. Handles transactions, gates, and the `ActivateLayer` / `DeactivateLayer` API. |
| `UMPCS_BindingRegistry` | Channel → blocks index. Populated by providers, queried on refresh. |
| `UMPCS_InputOrchestratorLocatorSubsystem` | `ULocalPlayerSubsystem` used by outside code to find the orchestrator and registry. |

Note that the orchestrator is **not** itself a subsystem — it is a `UObject`
slot created by the InputRootComponent. The locator subsystem holds weak
references to it. This keeps the orchestrator subclassable per-game while
still giving external code a cheap, standard discovery path.

The rest of this guide covers the four building-block concepts, then the
concrete class anatomy, then a step-by-step walkthrough for setting up a
project from scratch, and finally a reference for common runtime workflows.

---

## MPCS vs. Raw Enhanced Input

Enhanced Input handles *what a key does*. MPCS handles *which keys are active right now and why*. The two layers complement each other; MPCS does not replace Enhanced Input, it sits on top.

**Stick with raw Enhanced Input when:**
- Your game has one or two fixed input modes that never change at runtime.
- You switch contexts in at most two or three places and are comfortable tracking that state yourself.
- No actor other than the Player Controller contributes input blocks.

**Reach for MPCS when:**
- You have three or more input modes that transition at runtime (locomotion, vehicle, UI, dialogue, cutscene…).
- Multiple actors contribute input — a pawn, a possessed vehicle, an ability component — and you want them to register and unregister automatically.
- You need a hard block during cutscenes or loading screens (`PushGate` / `PopGate`).
- UI or game systems need to observe input-mode changes without polling (`OnLayerActivated` / `OnLayerDeactivated` delegates).
- You want the orchestration bookkeeping — which contexts are active, in what priority order, how to unwind on transition — handled once rather than re-implemented per project.

The practical threshold: if you have found yourself writing a `bIsInUIMode` boolean and checking it in three different places, MPCS pays for its setup cost.

---

## Concepts

Before touching the editor it helps to understand the four things MPCS manages
and how they relate to each other.

### Input Layer
An **Input Layer** is the physical side of the system. It owns a
`UInputMappingContext` and a priority value. When a layer is activated, its
mapping context is added to the Enhanced Input subsystem; when deactivated, it
is removed. Layers are identified by a Gameplay Tag (`LayerId`).

### Binding Channel
A **Binding Channel** is the logical side of the system. It is a Gameplay Tag
that groups a set of Binding Blocks together. When a channel is active, all
blocks registered under it are live. A channel has no asset of its own — it is
just a tag.

The reason layers and channels are separate is that the same set of blocks may
need to respond to inputs from multiple different layers (e.g. a movement block
that is active both when on foot and when swimming), and a single layer may need
to activate multiple independent groups of blocks. The
`UMPCS_InputLayerDefinition` asset is what bridges the two: it says "when this
layer is on, these channels become active."

### Binding Block
A **Binding Block** is a `UObject` Blueprint subclass that owns one
`UInputAction`, declares which trigger events it cares about, and implements the
response logic in `OnTriggered`. A block declares which channel it belongs to
via its `BindingChannelId` property. Blocks are created at runtime by a
Binding Provider Component on any actor in the world.

### Orchestrator
The **Orchestrator** (`UMPCS_InputOrchestratorSlot`) is the authority over which
layers are active. It exposes `ActivateLayer` / `DeactivateLayer` Blueprint
functions and handles transaction batching, gate (hard-block) logic, and state
change delegates. Gameplay code that wants to switch input modes calls the
orchestrator rather than touching layers or channels directly.

---

## System Anatomy

The plugin's runtime code lives in `MPCS_Core` and is grouped into four source
subfolders by concern — each folder answers *who physically owns this at
runtime?*

- `Actor/` — components and UObjects that live on ordinary actors (anything
  that contributes blocks).
- `Controller/` — things that live on or under the Player Controller.
- `Data/` — data assets and data-asset-instantiated UObjects.
- `Shared/` — plugin-wide helpers and native tag declarations.

### Controller side (lives on the Player Controller)

**`AMPCS_ModularPlayerController`** — Thin C++ `APlayerController` base.
Overrides `SetupInputComponent`: after the parent call it looks up the
`UMPCS_InputRootComponent` on itself and calls `BindToInput` on it with the
cast `UEnhancedInputComponent`. Inherit from this for the zero-wiring path,
or replicate the override in your own controller.

**`UMPCS_InputRootComponent`** — Composition root. Not user-facing at runtime;
creates the three slot objects plus the registry in `OnRegister`, wires the
orchestrator and registry into the locator subsystem in `BeginPlay`, and
drives `BindToInput` in response to `SetupInputComponent`. Designers configure
two things on it: `HostSlotClass` and `OrchestratorClass` (the Blueprints you
create in Steps 4 and 5). Exposes `ResetAllRuntimeState` for save/load reset
points.

**`UMPCS_InputSlotBase`** — Shared base for the three slot UObjects. Provides a
weak reference to the owning `APlayerController`. Not typically subclassed by
users.

**`UMPCS_InputLayerHostSlot`** — Owns the per-tag `UMPCS_InputLayer` instances
and the `UMPCS_InputBindingsSlot`. On `ActivateInputLayer(tag)` it adds the
layer's mapping context to Enhanced Input, recomputes the active binding
channel set from each active layer's `BindingChannels` declaration, and calls
`RefreshBindings` to ask the registry for the blocks that should be live.
Designers subclass this as a Blueprint to set `InitialLayerDefinitions`.

**`UMPCS_InputBindingsSlot`** — The bridge to Enhanced Input. Holds the current
`RuntimeInputBlocks` list and a dispatch map from `UInputAction` to the blocks
that want it. `BindToInput` walks the blocks, calls
`UEnhancedInputComponent::BindAction` once per unique `(Action, Trigger)` pair,
and records the binding handles so it can unbind cleanly on rebind. A single
handler (`HandleBoundAction`) fans out to every block that asked for the
firing trigger. Also exposes `AddRuntimeInputBlock` / `RemoveRuntimeInputBlock`
for programmatic blocks that bypass the registry.

**`UMPCS_InputOrchestratorSlot`** — Authority over which layers are active.
Exposes `ActivateLayer`, `DeactivateLayer`, `ActivateMany`, `DeactivateMany`,
`ApplyActiveLayers`, plus the transaction API (`BeginTransaction` /
`EndTransaction`) and the gate API (`PushGate` / `PopGate`). Requests go
through a pending-ops map and flush on transaction commit so the host sees a
single ordered batch. Fires `OnLayerActivated` / `OnLayerDeactivated` per
layer that actually changed state. Has a `OnOrchestratorReady`
BlueprintImplementableEvent called in `Initialize()` before defaults apply,
for BP-side setup hooks. Debug logging can be toggled with `bEnableDebugLogs`
(on by default). Designers subclass this as a Blueprint to set
`DefaultActiveLayers`.

**`UMPCS_BindingRegistry`** — Weak-pointer
`TMap<FGameplayTag, TArray<TWeakObjectPtr<Block>>>` keyed by `BindingChannelId`.
Providers register and unregister; the host slot queries `GetBlocksForChannel`.
Stale weak entries are pruned lazily. Skips any block whose `BindingChannelId`
tag is invalid (the most common cause of "my block isn't firing").

**`UMPCS_InputOrchestratorLocatorSubsystem`** — A `ULocalPlayerSubsystem`.
Holds weak pointers to the orchestrator and registry. External code that wants
to call `ActivateLayer` or look up blocks should fetch this subsystem from the
LocalPlayer and ask it for what it needs. It is the discovery point by
design — nothing outside `UMPCS_InputRootComponent` should walk the component
tree.

### Actor side (lives on any contributing actor)

**`UMPCS_BindingProviderComponent`** — Attach one to any actor that owns input
behavior: the Player Controller, a Pawn, a vehicle, an ability component.
Designers fill `BindingBlockClasses` with the block Blueprints this actor
contributes. On `OnRegister` the component instantiates one block object per
class. On `BeginPlay` it looks up the registry via the locator subsystem and
registers the blocks. If the registry is not yet available it retries every
0.25 seconds. On `EndPlay` / `OnUnregister` it unregisters, preserving the
registry from stale weak pointers.

**`UMPCS_InputBindingBlockBase`** — Abstract base for all binding blocks. Holds
the `BindingChannelId` tag, a weak pointer to the owning provider, and the
`BindBlock` / `UnbindBlock` virtuals. Rarely subclassed directly — use
`UMPCS_InputBindingBlock` instead.

**`UMPCS_InputBindingBlock`** — The user-facing block class. Adds `Action` (the
`UInputAction` this block responds to), `Triggers` (the `ETriggerEvent`s it
cares about), and a `BlueprintImplementableEvent OnTriggered(PC, Value, Trigger)`.
Designers subclass this as a Blueprint (`BB_<Name>`) and override
`OnTriggered` for the response logic.

### Data side (data assets)

**`UMPCS_InputLayer`** — The runtime side of a layer: a `UObject` with a
`MappingContext` and a `Priority`. `OnLayerActivated` adds the mapping context
to the Enhanced Input subsystem; `OnLayerDeactivated` removes it. Designers
subclass this as a Blueprint (`BP_Layer_<Name>`) and set the two properties.

**`UMPCS_InputLayerDefinition`** — The data-asset side of a layer: a
`UDataAsset` with `LayerId` (the tag that identifies this layer), `LayerClass`
(the `BP_Layer_*` to instantiate), `BindingChannels` (the channel tags that
become active when this layer is on), and optional `MetaTags`. One asset per
layer; the list of all layer definitions lives on
`UMPCS_InputLayerHostSlot::InitialLayerDefinitions`.

### Shared

**`MPCS_NativeTags`** — The plugin's own native tag declarations. Currently
defines `TAG_MPCS_Input_Layer_Debug`. Add your own plugin-owned tags here if
you want them available as C++ symbols.

**`UMPCS_OrchestratorBlueprintLibrary`** — Blueprint function library.
Currently exposes `GetInputRootFromPlayerController` for BP access to the
root component when the locator path is not convenient (e.g. when you want
to call `ResetAllRuntimeState`).

---

## Prerequisites

1. **Enhanced Input** must be enabled in the project. In *Project Settings →
   Input*, set the *Default Player Input Class* to `EnhancedPlayerInput` and
   the *Default Input Component Class* to `EnhancedInputComponent`.
2. **GameplayTags** must be enabled (they are on by default in UE5 projects).
3. The **MPCS_Core** plugin must be enabled in the `.uproject` file and the
   editor must have been restarted after enabling it.

---

## Step 1 — Set Up Gameplay Tags

MPCS uses Gameplay Tags as stable identifiers for layers and channels. Define
these tags before creating any assets so the drop-downs are populated when you
need them.

Open *Project Settings → GameplayTags* and add the following tag families.
Adjust the names to suit your project; only the structure matters.

```
MPCS.Input.Layer.<YourLayerName>     — one per input layer
MPCS.Input.Channel.<YourChannelName> — one per binding channel
```

**Example tags for a basic top-down game:**

```
MPCS.Input.Layer.Locomotion
MPCS.Input.Layer.UI
MPCS.Input.Channel.Movement
MPCS.Input.Channel.Camera
MPCS.Input.Channel.Selection
```

Tags can be added directly in *Project Settings* or in a
`Config/DefaultGameplayTags.ini` file. It does not matter which approach you
use; MPCS only looks them up by name at runtime.

---

## Step 2 — Set Up the Player Controller

MPCS works through an `UMPCS_InputRootComponent` that lives on the Player
Controller. The plugin ships with `AMPCS_ModularPlayerController`, a thin
C++ base class whose only job is to find that component in
`SetupInputComponent` and call `BindToInput` on it.

### Option A — Use the provided base class (recommended)

1. Create a new Blueprint class. Set its parent to
   `MPCS_ModularPlayerController`.
2. Add an `MPCS_InputRootComponent` to the Blueprint's component list.
3. Set the Blueprint as the *Player Controller Class* in your Game Mode.

That is all that is needed. The C++ base handles the `SetupInputComponent`
wiring automatically.

### Option B — Use your own Player Controller base class

If you already have a C++ or Blueprint player controller that cannot inherit
from `MPCS_ModularPlayerController`:

1. Add an `MPCS_InputRootComponent` to the controller.
2. Override `SetupInputComponent`. Inside the override, after calling the
   parent, find the `MPCS_InputRootComponent` and call `BindToInput` on it,
   passing the `InputComponent` cast to `UEnhancedInputComponent`.

```
// Blueprint pseudocode — do this after calling the Parent node
InputRoot = GetComponentByClass(MPCS_InputRootComponent)
InputRoot → BindToInput(InputComponent cast to EnhancedInputComponent)
```

---

## Step 3 — Create an Input Layer

Each input mode in your game (locomotion, UI, vehicle, dialogue, etc.) needs
three assets: a **Layer Blueprint**, a **Layer Definition**, and a
**Mapping Context**. The mapping context is a standard Unreal asset; the other
two are MPCS-specific.

### 3a — Create the Mapping Context

1. In the Content Browser, right-click → *Input → Input Mapping Context*.
2. Name it something like `IMC_Locomotion`.
3. Open it and add the raw key bindings your layer needs (W/A/S/D, mouse axes,
   etc.). Do not worry about binding actions to Blueprint logic here; that is
   handled by Binding Blocks in a later step.

### 3b — Create the Layer Blueprint

The Layer Blueprint is the object that adds and removes the mapping context
from the Enhanced Input subsystem when the layer is activated or deactivated.

1. In the Content Browser, right-click → *Blueprint Class*.
2. Search for `MPCS_InputLayer` and select it as the parent class.
3. Name it something like `BP_Layer_Locomotion`.
4. Open it. In the *Class Defaults*, you will see two properties:
   - **Mapping Context** — assign `IMC_Locomotion` here.
   - **Priority** — set the priority for this context. Higher numbers override
     lower ones when the same key is bound in multiple active contexts.
     A value of `0` is fine for a basic single-layer setup.
5. Save and close.

### 3c — Create the Layer Definition

The Layer Definition is a Data Asset that binds the layer's tag identity to its
Blueprint class and declares which binding channels become active when this
layer is on.

1. In the Content Browser, right-click → *Miscellaneous → Data Asset*.
2. Select `MPCS_InputLayerDefinition` as the class.
3. Name it something like `DA_Layer_Locomotion`.
4. Open it. Fill in the three properties:
   - **Layer Id** — pick `MPCS.Input.Layer.Locomotion` from the tag picker.
   - **Layer Class** — assign `BP_Layer_Locomotion`.
   - **Binding Channels** — add the channel tags whose blocks should be live
     while this layer is on. For a locomotion layer you might add
     `MPCS.Input.Channel.Movement` and `MPCS.Input.Channel.Camera`.
5. Save and close.

**MetaTags** is an optional freeform container for custom metadata. It has no
effect on the runtime and is not required.

Repeat steps 3a–3c for every layer in your game.

---

## Step 4 — Create the Host Slot Blueprint

The Host Slot manages the list of known layer definitions and is responsible for
adding and removing mapping contexts. It needs to be configured with your layer
definitions via a Blueprint subclass so the editor can display the properties.

1. In the Content Browser, right-click → *Blueprint Class*.
2. Search for `MPCS_InputLayerHostSlot` and select it as the parent class.
3. Name it something like `BP_HostSlot_Default`.
4. Open it. In the *Class Defaults*, find **Initial Layer Definitions** (under
   the `MPCS|Input` category).
5. Add one entry per layer you created in Step 3. Assign each
   `DA_Layer_Locomotion`, `DA_Layer_UI`, etc.
6. Save and close.

This Blueprint class is what you will assign to the Input Root Component in
Step 6.

---

## Step 5 — Create the Orchestrator Blueprint

The Orchestrator tracks which layers are currently active and exposes the API
that gameplay code calls to switch between them. Like the Host Slot, it needs
a Blueprint subclass so you can configure the default active layers.

1. In the Content Browser, right-click → *Blueprint Class*.
2. Search for `MPCS_InputOrchestratorSlot` and select it as the parent class.
3. Name it something like `BP_Orchestrator_Default`.
4. Open it. In the *Class Defaults*, find **Default Active Layers** (under
   `MPCS|Input|Orchestrator`).
5. Add the layer tags that should be active when the game starts. For a
   top-down game this is typically just `MPCS.Input.Layer.Locomotion`.
6. **Apply Defaults On Initialize** is checked by default. Leave it checked;
   it is what causes those default layers to activate at BeginPlay.
7. Save and close.

---

## Step 6 — Wire Up the Input Root Component

Now connect the Host Slot and Orchestrator Blueprints to the player controller.

1. Open your Player Controller Blueprint.
2. Select the `MPCS_InputRootComponent` in the component list.
3. In the *Details* panel find the `MPCS|Input` categories:
   - **Host Slot Class** — assign `BP_HostSlot_Default`.
   - **Orchestrator Class** — assign `BP_Orchestrator_Default`.
4. Compile and save.

At this point the layer system is complete. If you play in the editor, the
default layers will activate at BeginPlay and their mapping contexts will be
added to the Enhanced Input subsystem. No blocks will be live yet because none
have been created.

---

## Step 7 — Create a Binding Block

A Binding Block is the object that actually responds to player input. One block
covers one Input Action. Create as many as you need.

### 7a — Create the Input Action

1. In the Content Browser, right-click → *Input → Input Action*.
2. Name it `IA_MoveForward`, `IA_Look`, etc.
3. Set the *Value Type* to match the expected input shape (`Axis1D`, `Axis2D`,
   `Digital`, etc.).

### 7b — Create the Block Blueprint

1. In the Content Browser, right-click → *Blueprint Class*.
2. Search for `MPCS_InputBindingBlock` and select it as the parent class.
3. Name it something like `BB_MoveForward`.
4. Open it. In the *Class Defaults*, fill in:
   - **Binding Channel Id** — the channel this block belongs to. For a
     movement block set this to `MPCS.Input.Channel.Movement`. This tag must
     appear in the *Binding Channels* array of at least one Layer Definition
     or the block will never be live.
   - **Action** — assign `IA_MoveForward`.
   - **Triggers** — add the trigger events you want to respond to. For a
     movement action that should fire every tick while held, add `Triggered`.
     For a single press, add `Started`.
5. Override the **On Triggered** event (under *Functions → Override*). This is
   where you write the response logic. The event receives:
   - `PC` — the owning Player Controller.
   - `Value` — the `FInputActionValue` (call `Get Axis 1D`, `Get Axis 2D`, or
     `Get Digital` depending on the action type to unpack it).
   - `Trigger` — the specific `ETriggerEvent` that fired.

   Example for movement:
   ```
   On Triggered →
       Value.GetAxis2D → AddMovementInput (using PC → GetPawn)
   ```
6. Compile and save.

Repeat for every input action in the channel.

---

## Step 8 — Add a Binding Provider to an Actor

Blocks are instantiated at runtime by a `UMPCS_BindingProviderComponent`.
Add this component to any actor whose Blueprint subclass should contribute
input behavior. The most common location is on the Player Controller itself
(alongside the Input Root Component), but it can live on any actor — a pawn,
a vehicle, an ability component, etc.

1. Open the actor Blueprint.
2. In the *Components* panel, click *Add* and search for
   `MPCS_BindingProviderComponent`. Add it.
3. Select the component and look at the *Details* panel.
4. Find **Binding Block Classes** and add an entry for each block Blueprint
   you created in Step 7 that this actor should own. For example, on the
   Player Controller you might add `BB_MoveForward`, `BB_MoveRight`,
   `BB_Look`, etc.
5. Compile and save.

At BeginPlay the provider will instantiate one object per class, call
`SetBindingProvider` on each, and register them into the
`UMPCS_BindingRegistry` under their `BindingChannelId`. The next time the host
slot calls `RefreshBindings` for a channel those blocks belong to, they will
become live and start receiving input.

**Multiple providers on different actors are fully supported.** A unit actor
could have its own provider with unit-specific actions, and the Player
Controller could have a provider with camera and selection actions. They all
end up in the same registry and share the channel namespace.

---

## Step 9 — Controlling Layers at Runtime

All runtime layer control goes through the Orchestrator, reached via the
`UMPCS_InputOrchestratorLocatorSubsystem`.

### Getting the Orchestrator in Blueprint

From any Blueprint that has access to a Player Controller:

```
Get Local Player (from Player Controller)
    → Get Subsystem (class: MPCS_InputOrchestratorLocatorSubsystem)
    → Get Orchestrator
```

The subsystem is a `ULocalPlayerSubsystem`, so it is also accessible from any
widget or HUD that has a local player reference.

### Activating and Deactivating Layers

```
Orchestrator → Activate Layer (Identifier: MPCS.Input.Layer.UI)
Orchestrator → Deactivate Layer (Identifier: MPCS.Input.Layer.Locomotion)
```

Both calls are immediate and trigger a `RefreshBindings` internally, so the
correct blocks become live on the same frame.

### Applying an Exclusive Layer Set

`Apply Active Layers` activates a list of layers and optionally deactivates
everything else in one atomic transaction:

```
Orchestrator → Apply Active Layers
    Identifiers: [MPCS.Input.Layer.UI]
    Deactivate Others: true
```

This is useful for modal states (menus, cutscenes) where you want to replace
whatever is active with a known set.

### Reacting to layer changes and gating

The Orchestrator broadcasts `OnLayerActivated` / `OnLayerDeactivated` delegates
for observing state changes, and exposes `Push Gate` / `Pop Gate` for
cutscene-style hard blocks. See **Extended Workflows** below for the full
treatment of both.

---

## Quick Reference

### Asset checklist per layer

| Asset | Class | Key properties to set |
|---|---|---|
| `IMC_<Name>` | `UInputMappingContext` | Raw key → action bindings |
| `BP_Layer_<Name>` | `MPCS_InputLayer` | Mapping Context, Priority |
| `DA_Layer_<Name>` | `MPCS_InputLayerDefinition` | Layer Id, Layer Class, Binding Channels |

### Asset checklist per binding block

| Asset | Class | Key properties to set |
|---|---|---|
| `IA_<Name>` | `UInputAction` | Value Type |
| `BB_<Name>` | `MPCS_InputBindingBlock` | Binding Channel Id, Action, Triggers |

### One-time project assets

| Asset | Class | Key properties to set |
|---|---|---|
| `BP_HostSlot_Default` | `MPCS_InputLayerHostSlot` | Initial Layer Definitions |
| `BP_Orchestrator_Default` | `MPCS_InputOrchestratorSlot` | Default Active Layers |
| `BP_PlayerController` | `AMPCS_ModularPlayerController` | Host Slot Class, Orchestrator Class (on the InputRootComponent) |

### Tag naming conventions

| Purpose | Suggested namespace |
|---|---|
| Layer identity | `MPCS.Input.Layer.*` |
| Binding channel | `MPCS.Input.Channel.*` |

### How a block becomes live — the full chain

```
Player presses a key
    ↓
Enhanced Input subsystem evaluates the active mapping contexts
(added by whichever Input Layers are currently active)
    ↓
The Input Action fires
    ↓
MPCS_InputBindingsSlot has the block bound to that action
(because the block's channel was active when RefreshBindings last ran)
    ↓
Block::OnTriggered is called with PC, Value, and TriggerEvent
```

A block is included in `RefreshBindings` when:
1. Its `BindingChannelId` tag is listed in **Binding Channels** on at least one
   active Layer Definition.
2. It has been registered in the registry by a Binding Provider Component.

Both conditions must be true simultaneously.

---

## Extended Workflows

Step 9 covered the basic activate/deactivate loop. The patterns below go a
little further and assume Steps 1–8 are in place.

### Reacting to layer changes

The Orchestrator broadcasts two Blueprint-assignable delegates after every
transaction commit: `OnLayerActivated(FGameplayTag LayerId)` and
`OnLayerDeactivated(FGameplayTag LayerId)`. Bind to them from any BP that can
reach the orchestrator — typically a UMG widget, a HUD, or a Player State — to
drive UI state or animation in lockstep with the input mode.

```
Orchestrator → OnLayerActivated → BindEvent → [your handler]
```

Each delegate fires once per layer that actually changed state: if
`ApplyActiveLayers` activates three layers at once you will get three
invocations on the same commit.

### Adding a runtime block programmatically (C++)

The normal path is a `UMPCS_BindingProviderComponent` with `BindingBlockClasses`
set in the editor. For blocks that need to exist only briefly — e.g. a debug
HUD that wants to own a "press F3 for diagnostics" binding while it is on
screen — the host slot exposes a C++ API:

- **`UMPCS_InputLayerHostSlot::AddRuntimeInputBindingBlockByClass(TSubclassOf<UMPCS_InputBindingBlock>)`** —
  instantiates the class, wires it into the bindings slot, and returns the
  instance so you can keep a reference.
- **`UMPCS_InputLayerHostSlot::RemoveRuntimeInputBindingBlock(UMPCS_InputBindingBlock*)`** —
  removes a previously added runtime block.

These host-slot entry points are not currently Blueprint-callable. The
`UMPCS_InputBindingsSlot` itself does expose
`BlueprintCallable AddRuntimeInputBlock` / `RemoveRuntimeInputBlock`, but the
bindings slot is not reachable from Blueprint via the current public API
(`UMPCS_InputRootComponent::GetHostSlotRuntime` and
`UMPCS_InputLayerHostSlot::GetInputBindingsSlot` are plain C++ getters, not
`UFUNCTION`). If you need this path from Blueprint, add a `BlueprintCallable`
wrapper on `UMPCS_InputRootComponent` or extend
`UMPCS_OrchestratorBlueprintLibrary` with a helper.

Runtime blocks added this way skip the registry and live only in the bindings
slot's `RuntimeInputBlocks`. They are not tied to a channel, so they stay
active until explicitly removed — use them for scope-bound utilities, not for
gameplay behavior.

### Spawning a provider on a dynamically spawned actor

Adding a `UMPCS_BindingProviderComponent` to an actor spawned mid-game works
without extra wiring. The sequence is:

1. Actor spawns; components register, provider included.
2. In `OnRegister`, the provider creates one block object per
   `BindingBlockClasses` entry.
3. In `BeginPlay`, the provider asks the locator subsystem for the registry
   and registers the blocks. If the registry is already bound (normal post-map-
   load case), registration succeeds immediately.
4. If the registry is not yet available — e.g. the Player Controller's
   InputRoot has not reached `BeginPlay` yet because actor init order is not
   guaranteed — the provider schedules a 0.25-second retry. It keeps retrying
   until registration succeeds.

No special ordering or deferred-spawn logic is needed on the caller's side.

### Gating for a cutscene or loading screen

`PushGate` is for states where *nothing* outside the gate should be able to
change the input set. It atomically replaces the current layer set with a
forced set and blocks every subsequent `ActivateLayer` / `DeactivateLayer` /
`ApplyActiveLayers` call until the matching `PopGate`.

```
OnCutsceneStart:
    Orchestrator → Push Gate
        Forced Active Input Layers: [MPCS.Input.Layer.Cutscene]
        Reason: "Cutscene_Start"

OnCutsceneEnd:
    Orchestrator → Pop Gate
        Reason: "Cutscene_End"
```

While a gate is active, blocked calls are silently ignored (logged when
`bEnableDebugLogs` is on). On `PopGate`, whatever layer state was applied
**before** the gate is restored; activation requests made while gated are
discarded, not replayed. Gates nest — pushing a second gate snapshots the
state between them and pops in LIFO order.

Use gates only for hard blocks. For soft, priority-based blocking (e.g. "menu
input should coexist with locomotion but override specific keys"), drive the
competing layers through `ApplyActiveLayers` with ordinary mapping-context
priorities instead.

### Multiple actors contributing to the same channel

Binding channels are a per-local-player namespace: any provider whose block
registers under `MPCS.Input.Channel.Combat` ends up in the same bucket. A unit
actor contributing a combat block and an ability component contributing another
combat block both fire when the channel is active. Dispatch order within a
single action fire is the order blocks were registered in the registry (which
is effectively actor-spawn order). If you need deterministic ordering across
runs, register all contributors on a single stable source (e.g. the Player
Controller) rather than on transient spawns.

### Looking up the InputRoot from Blueprint

If you have an `APlayerController` reference and want the root component
itself — e.g. to call `ResetAllRuntimeState` at a save/load boundary — use
the Blueprint function library:

```
UMPCS_OrchestratorBlueprintLibrary → Get Input Root From Player Controller(PC)
```

For the orchestrator or registry, prefer the locator-subsystem path shown in
Step 9.

### Disabling debug logging for shipping

The orchestrator's `bEnableDebugLogs` property is `true` by default. It gates
the transaction-commit summary and gate-blocked request logs. Turn it off in
`BP_Orchestrator_Default` → *MPCS | Input | Orchestrator | Debug* for
shipping builds.

The rest of the system (`InputRoot`, `HostSlot`, `BindingsSlot`,
`BindingRegistry`, `BindingProvider`) logs through the engine-level
`LogMPCS` category. To silence it for shipping, add to
`Config/DefaultEngine.ini`:

```ini
[Core.Log]
LogMPCS=NoLogging
```
