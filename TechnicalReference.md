# Modular Player Control System — Technical Reference

C++ reference for `MPCS_Core`. For step-by-step setup and concept overview see `UserGuide.md`.

---

## Contents

- [Build Integration](#build-integration)
- [Class Hierarchy](#class-hierarchy)
- [Initialization and Teardown Sequence](#initialization-and-teardown-sequence)
- [Class Reference](#class-reference)
- [Key Contracts and Invariants](#key-contracts-and-invariants)
- [Extension Points](#extension-points)
- [Automation Tests](#automation-tests)
- [Known Issues](#known-issues)

---

## Build Integration

The plugin exposes one public module: `MPCS_Core`. Add it to your module's `.Build.cs`:

```csharp
PublicDependencyModuleNames.AddRange(new string[] { "MPCS_Core", "EnhancedInput", "GameplayTags" });
```

If depending from another plugin, declare the dependency in your `.uplugin`:

```json
"Plugins": [
    { "Name": "ModularPlayerControlSystem", "Enabled": true }
]
```

Headers are organized under `Source/MPCS_Core/Public/`:

| Folder | Contents |
|---|---|
| `Actor/` | `UMPCS_BindingProviderComponent`, `UMPCS_InputBindingBlockBase`, `UMPCS_InputBindingBlock` |
| `Controller/` | Slots, registry, locator subsystem, root component, convenience PC |
| `Data/` | `UMPCS_InputLayer`, `UMPCS_InputLayerDefinition` |
| `Shared/` | `MPCS_NativeTags.h`, `MPCS_CoreLog.h`, `MPCS_OrchestratorBlueprintLibrary.h` |

---

## Class Hierarchy

```
UObject
├── UMPCS_InputSlotBase  (Abstract)
│   ├── UMPCS_InputOrchestratorSlot
│   └── UMPCS_InputBindingsSlot
├── UMPCS_InputLayerHostSlot
├── UMPCS_BindingRegistry
├── UMPCS_InputBindingBlockBase  (Abstract)
│   └── UMPCS_InputBindingBlock  (Abstract — subclass in Blueprint as BB_<Name>)
└── UMPCS_InputLayer  (Abstract — subclass in Blueprint as BP_Layer_<Name>)

UDataAsset
└── UMPCS_InputLayerDefinition

UActorComponent
├── UMPCS_InputRootComponent
└── UMPCS_BindingProviderComponent

ULocalPlayerSubsystem
└── UMPCS_InputOrchestratorLocatorSubsystem

APlayerController
└── AMPCS_ModularPlayerController

UBlueprintFunctionLibrary
└── UMPCS_OrchestratorBlueprintLibrary
```

---

## Initialization and Teardown Sequence

### Phase 1 — Component registration (`OnRegister`)

`UMPCS_InputRootComponent::OnRegister` creates all runtime objects before BeginPlay and before the input component exists:

1. Instantiates `HostSlot` as `NewObject<UMPCS_InputLayerHostSlot>(this, HostSlotClass)`.
2. Instantiates `OrchestratorSlot` as `NewObject<UMPCS_InputOrchestratorSlot>(this, OrchestratorClass)`.
3. Instantiates `BindingRegistry` as `NewObject<UMPCS_BindingRegistry>(this)`.
4. Calls `WireSlotsToOwner()` — sets the owning `APlayerController` on the slots, hands the registry to the host slot, and gives the host slot pointer to the orchestrator.

`BindToInput` is **not** called here.

### Phase 2 — World start (`BeginPlay`)

`UMPCS_InputRootComponent::BeginPlay`:

1. Fetches `UMPCS_InputOrchestratorLocatorSubsystem` from the owning local player and publishes weak refs to the orchestrator and registry (`BindOrchestratorToLocator`). External code can discover MPCS via the subsystem from this point.
2. Calls `HostSlot::InstantiateLayers()` — creates one `UMPCS_InputLayer` instance per entry in `InitialLayerDefinitions`, keyed by `LayerId`.
3. Calls `OrchestratorSlot::Initialize()`:
   - Fires `OnOrchestratorReady()` before any default layers are applied. Use this for BP-side setup that must precede the initial layer state.
   - If `bApplyDefaultsOnInitialize` is true, calls `ApplyActiveLayers(DefaultActiveLayers, false)`.

`UMPCS_BindingProviderComponent::BeginPlay` (on any contributing actor):

1. Binds `OnOwnerControllerChanged` to `APawn::ReceiveControllerChangedDelegate` if the owner is a pawn and `RegistrationMode` is `Auto`.
2. Calls `TryRegisterAll()` — see [Provider Registration](#provider-registration).

### Phase 3 — Input component creation (`SetupInputComponent`)

`AMPCS_ModularPlayerController::SetupInputComponent` (or your equivalent override):

1. Calls `Super::SetupInputComponent()`.
2. Finds `UMPCS_InputRootComponent` via `FindComponentByClass`.
3. Calls `BindToInput(Cast<UEnhancedInputComponent>(InputComponent))`.

`UMPCS_InputRootComponent::BindToInput` cascades:
`HostSlot::BindToInput` → `BindingsSlot::BindToInput` → `BuildDispatchAndBind`

`BuildDispatchAndBind` iterates live blocks, calls `BindBlock` on each, and records the returned binding handles. It also builds a `TMap<(UInputAction*, ETriggerEvent) → TArray<Block*>>` dispatch map so `HandleBoundAction` can fan out to every block that requested a given action+trigger pair.

### Provider Registration

`TryRegisterAll()` calls `ResolveTargetControllers()` to determine which PCs to register with, then for each calls `FindBindingRegistryForPC(PC)` (via the locator subsystem) and `RegisterWithController(PC)`. If the registry is not yet available, it schedules a 0.25-second retry with no maximum retry count. In the common case (PC `BeginPlay` already ran) the first attempt succeeds immediately.

Possession changes fire `OnOwnerControllerChanged`, which calls `UnregisterFromAll()` then `TryRegisterAll()` so the block set stays consistent with the current possessor.

### Teardown Sequence

`UMPCS_InputRootComponent::EndPlay`:

1. Sets `bIsShuttingDown = true` on itself, the host slot, and the orchestrator.
2. Calls `ResetAllRuntimeState()`:
   - `OrchestratorSlot::ResetOrchestrator()` — clears intent and applied layer sets, discards the gate stack.
   - `HostSlot::ResetHost()` — deactivates all layers (skipping `RefreshBindings` because `bIsShuttingDown` is set), destroys layer instances.
   - Clears the locator subsystem's weak refs.

`UMPCS_BindingProviderComponent::EndPlay` / `OnUnregister` — calls `UnregisterFromAll()`, iterating `RegisteredControllers` and calling `UnregisterFromController(PC)` for each.

---

## Class Reference

### `UMPCS_InputRootComponent`

Composition root on the Player Controller. Owns the three slot UObjects and the binding registry. Not subclassed by users; configure it via its UPROPERTY fields in the Player Controller Blueprint.

**Public API:**

| Method | Description |
|---|---|
| `BindToInput(UEnhancedInputComponent&)` | Called by the PC's `SetupInputComponent` override. Must be called every time the Enhanced Input Component is (re)created. Cascades to host → bindings slot. |
| `ResetAllRuntimeState()` | Tears down and re-establishes all runtime slot state. Safe to call at save/load boundaries. `BlueprintCallable`. |
| `GetHostSlotRuntime() const` | Returns the live `UMPCS_InputLayerHostSlot*`. Plain C++ getter; not a `UFUNCTION`. |
| `GetBindingRegistry() const` | Returns the live `UMPCS_BindingRegistry*`. Plain C++ getter. |

**Configurable properties (set on the component in the Player Controller BP):**

| Property | Type | Description |
|---|---|---|
| `HostSlotClass` | `TSubclassOf<UMPCS_InputLayerHostSlot>` | Blueprint subclass that carries `InitialLayerDefinitions`. |
| `OrchestratorClass` | `TSubclassOf<UMPCS_InputOrchestratorSlot>` | Blueprint subclass that carries `DefaultActiveLayers`. |
| `DefaultBindingInputBlockClasses` | `TArray<TSubclassOf<UMPCS_InputBindingBlock>>` | Block classes seeded into the bindings slot at startup, bypassing the registry. Stay live regardless of channel state. |

---

### `UMPCS_InputOrchestratorSlot`

Authority over which layers are currently active. Exposes the transaction and gate API. Intended to be subclassed in Blueprint (`BP_Orchestrator_Default`).

#### State layers

Two sets are kept in sync; they diverge during transactions and while gated:

| Set | Meaning | Updated when |
|---|---|---|
| `ActiveLayers` | **Intent set** — what has been requested | Every `ActivateLayer` / `DeactivateLayer` call, even mid-transaction |
| `AppliedActiveInputLayers` | **Applied set** — what the host has actually been told | On transaction commit only |

`IsLayerActive` queries the **intent** set. During a gate or open transaction, the two sets may differ — do not use `AppliedActiveInputLayers` for game-logic decisions.

#### BlueprintCallable API

| Method | Notes |
|---|---|
| `ActivateLayer(Tag)` | Adds to intent, stages an activate op, commits immediately if no open transaction. Returns `true` if the host accepted. |
| `DeactivateLayer(Tag)` | Symmetric. |
| `ActivateMany(Tags)` | Wraps all tags in a single transaction; all-or-nothing commit. |
| `DeactivateMany(Tags)` | Symmetric. |
| `ApplyActiveLayers(Tags, bDeactivateOthers)` | Activates all listed tags; when `bDeactivateOthers` is true, deactivates any currently active tag not in the list. Useful for modal transitions. |
| `IsLayerActive(Tag) const` | Queries intent set. `BlueprintPure`. |
| `BeginTransaction(Reason)` | Opens a transaction. Subsequent ops stage into `PendingInputLayerOps` without flushing to the host. |
| `EndTransaction()` | Commits pending ops to the host. Staging is **last-write-wins per tag**: if a tag was activated then deactivated in the same transaction, only the deactivate is applied. Only the outermost `EndTransaction` commits; nested calls decrement a depth counter. |
| `PushGate(ForcedLayers, Reason)` | Snapshots current state into a `FGateFrame`, applies `ForcedLayers`, and blocks all subsequent Activate/Deactivate/Apply calls. Returns `false` if already gated at the current depth. |
| `PopGate(Reason)` | Restores the layer set from the matching `PushGate` snapshot. Requests that arrived while gated are **discarded, not replayed**. |
| `IsGated() const` | True when the gate stack is non-empty. `BlueprintPure`. |

#### BlueprintImplementableEvent

`OnOrchestratorReady()` — fires in `Initialize()` before `DefaultActiveLayers` are applied. Override in the orchestrator Blueprint for setup logic that must precede the initial layer state.

#### Configurable properties

| Property | Default | Description |
|---|---|---|
| `DefaultActiveLayers` | `[]` | Tags activated at `Initialize()` when `bApplyDefaultsOnInitialize` is true. |
| `bApplyDefaultsOnInitialize` | `true` | Whether to activate `DefaultActiveLayers` on startup. |
| `bEnableDebugLogs` | `true` | Gates transaction-commit summaries and gate-blocked-request warnings. Disable in the orchestrator Blueprint for shipping. |

---

### `UMPCS_InputLayerHostSlot`

Owns layer instances and the derived active binding channel set. Called by the orchestrator on every layer state change; external code rarely calls this directly. Intended to be subclassed in Blueprint (`BP_HostSlot_Default`).

**Key methods:**

| Method | Notes |
|---|---|
| `InstantiateLayers()` | Called by root component `BeginPlay`. Creates one `UMPCS_InputLayer` per `InitialLayerDefinitions` entry. |
| `ActivateInputLayer(Tag)` | Called by orchestrator on commit. Adds the layer's mapping context via `UEnhancedInputLocalPlayerSubsystem`, recomputes `ActiveBindingChannels`, calls `RefreshBindings()`. Returns `false` if no definition exists for the tag. |
| `DeactivateInputLayer(Tag)` | Removes the mapping context, recomputes channels, refreshes. Skips `RefreshBindings()` when `bIsShuttingDown` is set. |
| `RefreshBindings()` | Full replace of the live block set: queries the registry for every active channel tag, collects all results, unbinds and rebinds the bindings slot from scratch. |
| `AddRuntimeInputBindingBlockByClass(Class)` | Instantiates and adds a block that bypasses the registry. Stays live until explicitly removed. Not `BlueprintCallable`. |
| `RemoveRuntimeInputBindingBlock(Block)` | Symmetric. |

**Configurable property:**

| Property | Description |
|---|---|
| `InitialLayerDefinitions` | `TArray<UMPCS_InputLayerDefinition*>`. Fill in the Blueprint subclass's class defaults. A tag absent from this list causes `ActivateInputLayer` to return `false`. |

---

### `UMPCS_InputBindingsSlot`

Bridge to Enhanced Input. Holds the live block list and a dispatch map from `(UInputAction*, ETriggerEvent)` to `TArray<Block*>`. Managed entirely by the host slot; external code does not call this directly except via the runtime-block API.

`BindToInput(EIC)` (re)builds the dispatch map and calls `UEnhancedInputComponent::BindAction` once per unique `(Action, Trigger)` pair across all live blocks. Previously registered handles are removed first via `UnbindFromLastInputComponent`. `HandleBoundAction` fans out to every block that subscribed to the firing pair.

Blocks are divided into two sets:
- **Registry-fed** — computed by `RefreshBindings` from the current active channel set.
- **Runtime** — added via `AddRuntimeInputBlock` / `RemoveRuntimeInputBlock`. Not channel-gated; stay live until explicitly removed.

**BlueprintCallable runtime-block API:**

| Method | Description |
|---|---|
| `AddRuntimeInputBlock(Block)` | Adds a block outside the registry. Triggers a rebind. |
| `RemoveRuntimeInputBlock(Block)` | Removes a runtime block. |
| `ClearRuntimeInputBlocks()` | Removes all runtime blocks. |
| `GetRuntimeInputBlockCount() const` | Returns live runtime block count. `BlueprintPure`. |

Note: the bindings slot is not currently reachable via Blueprint from the public API on `UMPCS_InputRootComponent` or the locator subsystem. To expose this path to Blueprint, add a `BlueprintCallable` wrapper on `UMPCS_InputRootComponent` or extend `UMPCS_OrchestratorBlueprintLibrary`.

---

### `UMPCS_BindingRegistry`

Channel-to-block index. Storage: `TMap<FGameplayTag, TArray<TWeakObjectPtr<UMPCS_InputBindingBlock>>>`.

Populated by `UMPCS_BindingProviderComponent` on BeginPlay; queried by the host slot on every `RefreshBindings`. Holds **weak references** and self-heals via lazy null pruning during both `UnregisterBlocks` and `GetBlocksForChannel`.

**API:**

| Method | Notes |
|---|---|
| `RegisterBlocks(Blocks)` | Inserts weak refs keyed by each block's `BindingChannelId`. Blocks with an invalid channel tag are silently skipped. |
| `UnregisterBlocks(Blocks)` | Removes entries and prunes nulls in the same pass. |
| `GetBlocksForChannel(Tag)` | Returns the live block list for a channel, with nulls stripped before returning. |
| `Reset()` | Clears the entire map. Called by `ResetAllRuntimeState`. |

---

### `UMPCS_InputOrchestratorLocatorSubsystem`

`ULocalPlayerSubsystem`. The single well-known discovery point for external code. Holds weak pointers published by `InputRootComponent::BeginPlay` and cleared by `EndPlay`. Both `GetOrchestrator()` and `GetBindingRegistry()` may return `nullptr` before the PC's `BeginPlay` has run.

```cpp
auto* LP      = GetWorld()->GetFirstLocalPlayerFromController();
auto* Locator = LP->GetSubsystem<UMPCS_InputOrchestratorLocatorSubsystem>();
auto* Orch    = Locator->GetOrchestrator();   // BlueprintPure
auto* Reg     = Locator->GetBindingRegistry(); // BlueprintPure
```

Nothing outside `UMPCS_InputRootComponent` should walk the component tree for MPCS objects — use the locator subsystem instead.

---

### `UMPCS_BindingProviderComponent`

Attach to any actor that contributes input blocks. Instantiates one `UMPCS_InputBindingBlock` object per class in `BindingBlockClasses` during `OnRegister`; registers those blocks into the binding registry during `BeginPlay`.

**`EProviderRegistrationMode`:**

| Value | Behaviour |
|---|---|
| `Auto` | Possessed pawn → registers with the possessor only. Unregisters and re-registers on possession change. Non-pawn or unpossessed → registers with all local PCs. |
| `AllLocalPlayers` | Registers with every local PC unconditionally. |
| `Manual` | No automatic registration. Call `RegisterWithPC` / `UnregisterFromPC` directly. |

**Public API:**

| Method | Notes |
|---|---|
| `RegisterWithPC(PC)` | `Manual` mode: register all blocks with the given PC's registry. No-op if already registered with that PC. `BlueprintCallable`. |
| `UnregisterFromPC(PC)` | `Manual` mode: unregister. `BlueprintCallable`. |
| `IsRegisteredWithPC(PC) const` | True if `PC` is in `RegisteredControllers`. `BlueprintCallable`. |
| `GetBindingBlocks() const` | Returns the instantiated block list. `BlueprintCallable`. |

---

### `UMPCS_InputBindingBlockBase` / `UMPCS_InputBindingBlock`

`UMPCS_InputBindingBlockBase` — abstract base. Provides `BindingChannelId`, provider weak ref, `BindBlock` / `UnbindBlock` virtuals, and `GetOwningActor`. `GetOwningActor` fallback chain: provider owner → outer component owner → `nullptr`. Logs each resolution step at `Warning` in dev builds (see [Known Issues](#known-issues)).

`UMPCS_InputBindingBlock` — user-facing concrete base. Adds `Action` (`TObjectPtr<UInputAction>`), `Triggers` (`TArray<ETriggerEvent>`), and the `OnTriggered(PC, Value, Trigger)` `BlueprintImplementableEvent`. Subclass in Blueprint (`BB_<Name>`), set the three class-default properties, and override `OnTriggered` for the response logic.

For C++-only blocks, subclass `UMPCS_InputBindingBlockBase` and override `BindBlock` / `UnbindBlock` directly.

---

### `UMPCS_InputLayer` / `UMPCS_InputLayerDefinition`

`UMPCS_InputLayer` — abstract `UObject`. Subclass in Blueprint (`BP_Layer_<Name>`). Set `MappingContext` and `Priority`. The two C++ virtuals (`OnLayerActivated`, `OnLayerDeactivated`) are the only code that touches `UEnhancedInputLocalPlayerSubsystem` directly; both receive a `UEnhancedInputLocalPlayerSubsystem&` reference.

`UMPCS_InputLayerDefinition` — `UDataAsset`. One per layer. Properties:

| Property | Type | Description |
|---|---|---|
| `LayerId` | `FGameplayTag` | The stable tag identity for this layer. Used as the key throughout the runtime. |
| `LayerClass` | `TSubclassOf<UMPCS_InputLayer>` | The `BP_Layer_*` to instantiate at `InstantiateLayers` time. |
| `BindingChannels` | `TArray<FGameplayTag>` | Channel tags that become active when this layer is on. |
| `MetaTags` | `FGameplayTagContainer` | Freeform; not read by MPCS at runtime. |

---

### `AMPCS_ModularPlayerController`

Thin `APlayerController` subclass. Its only override is `SetupInputComponent`, which finds `UMPCS_InputRootComponent` via `FindComponentByClass` and calls `BindToInput`. Inherit from this for the zero-wiring path, or replicate the two-line override in your own controller class.

---

### `UMPCS_OrchestratorBlueprintLibrary`

Single static function:

```cpp
static UMPCS_InputRootComponent* GetInputRootFromPlayerController(APlayerController* PlayerController);
```

Use this when you have a PC reference and need the root component directly (e.g., to call `ResetAllRuntimeState`). For the orchestrator or registry, prefer the locator-subsystem path.

---

## Key Contracts and Invariants

**`ActiveLayers` vs. `AppliedActiveInputLayers`**  
`IsLayerActive` queries the intent set (`ActiveLayers`), not the applied set. They diverge during an open transaction and while a gate is active. Do not use `AppliedActiveInputLayers` for game-logic decisions.

**Transaction staging is last-write-wins per tag**  
If a tag is activated and then deactivated within the same `BeginTransaction` / `EndTransaction` block, only the final op (deactivate) is committed. Nested `BeginTransaction` calls increment a depth counter; only the outermost `EndTransaction` flushes.

**Gate is a hard block**  
`ActivateLayer`, `DeactivateLayer`, and `ApplyActiveLayers` are silently rejected while a gate is active (logged when `bEnableDebugLogs` is on). `PopGate` is always accepted. On `PopGate`, the state is restored from the snapshot saved by the matching `PushGate` — ops queued during the gate are **not replayed**. Gates nest in LIFO order.

**`RefreshBindings` is a full replace**  
The host slot does not compute a delta. Every call to `RefreshBindings` fetches the full block set for all active channels from the registry, then unbinds and rebinds the bindings slot from scratch. Partial updates are not supported.

**Registry holds weak references; self-heals lazily**  
Blocks that are GC'd between registration and the next query are pruned during `GetBlocksForChannel` and `UnregisterBlocks`. Explicit unregistration is cleaner but not required for correctness — the registry will not return a dangling pointer.

**`bIsShuttingDown` guard**  
`UMPCS_InputLayerHostSlot::DeactivateInputLayer` skips `RefreshBindings` when `bIsShuttingDown` is true. This prevents the bindings slot from touching a `UEnhancedInputComponent` that may be in a deinitialized state during PIE stop or world shutdown. See `UE teardown — IsValid can lie during PIE stop` in project memory for the broader pattern.

**Provider retry has no maximum count**  
`TryRegisterAll` keeps retrying at 0.25-second intervals until `FindBindingRegistryForPC` returns a valid registry. In the common case (PC `BeginPlay` already ran) the first attempt succeeds.

---

## Extension Points

**Subclass `UMPCS_InputOrchestratorSlot`**  
Primary extension point for project-specific orchestration. Set `DefaultActiveLayers` in the Blueprint subclass's class defaults. Override `OnOrchestratorReady` for BP-side setup that must precede the initial layer state. Subclass in C++ to intercept `ActivateLayer` / `DeactivateLayer` or to add custom transaction/gate logic.

**Subclass `UMPCS_InputLayerHostSlot`**  
Primary extension point for the layer definition list. Fill `InitialLayerDefinitions` in the Blueprint subclass. Subclass in C++ to intercept `ActivateInputLayer` / `DeactivateInputLayer` for project-specific validation or side effects.

**Subclass `UMPCS_InputLayer`**  
Override `OnLayerActivated` and `OnLayerDeactivated` in Blueprint or C++ to run logic beyond mapping context management when a layer transitions (e.g., locking the camera, setting a game state flag).

**Subclass `UMPCS_InputBindingBlock`**  
Normal path: Blueprint subclass (`BB_<Name>`), `OnTriggered` implemented in the event graph. For C++-only blocks, subclass `UMPCS_InputBindingBlockBase` and override `BindBlock` / `UnbindBlock` directly.

**Registry-bypassing (programmatic) blocks**  
`UMPCS_InputLayerHostSlot::AddRuntimeInputBindingBlockByClass(Class)` instantiates and wires a block into `BindingsSlot::RuntimeInputBlocks` without touching the registry or any channel. The block stays live until `RemoveRuntimeInputBindingBlock` or `ClearRuntimeInputBlocks`. Use for scope-bound utilities (debug bindings, single-span QTE input) rather than persistent gameplay behavior.

---

## Automation Tests

19 C++ tests under `Source/MPCS_Core/Private/Tests/`:

| File | Suite | Tests | Coverage |
|---|---|---|---|
| `MPCS_BindingRegistryTests.cpp` | `MPCS.BindingRegistry.*` | 10 | Register, unregister, per-channel routing, duplicate-register, null-safety, lazy weak-ref pruning |
| `MPCS_OrchestratorSlotTests.cpp` | `MPCS.OrchestratorSlot.*` | 9 | Intent-layer tracking (5), gate push/pop / depth / blocked-request suppression (4) |

`MPCSTestHelpers.h/.cpp` defines `UMPCS_TestInputBindingBlock`: a concrete `UCLASS()` subclass of `UMPCS_InputBindingBlockBase` that exposes `SetChannelIdForTest(FGameplayTag)` to assign `BindingChannelId` directly, bypassing the provider lifecycle. Use this for any test that needs a block in the registry without a live world.

**Pattern for orchestrator/gate tests without a full PIE world:**

```cpp
// Both objects kept on the stack in RunTest to prevent GC.
UMPCS_InputOrchestratorSlot* Orch = NewObject<UMPCS_InputOrchestratorSlot>();
UMPCS_InputLayerHostSlot*    Host = NewObject<UMPCS_InputLayerHostSlot>();
Orch->SetHostSlot(Host);

// Host is valid → PushGate / PopGate succeed.
// No PlayerController → EnhancedInputSubsystem is null →
// AppliedActiveInputLayers never commits to the EI stack.
// ActiveLayers (intent) and gate depth are fully testable without PIE.
```

---

## Known Issues

| Location | Issue | Severity |
|---|---|---|
| `UMPCS_InputBindingBlockBase::GetOwningActor` | Logs each fallback step at `Warning`. Acceptable in development. | Should drop to `Verbose` before shipping. |
| `UMPCS_InputBindingsSlot::HandleBoundAction` | Logs every fired input event at `Warning`. Fires per-frame on held inputs. | Should drop to `VeryVerbose` before shipping. |
| `UMPCS_InputOrchestratorSlot::ActivateLayer` | Dead `auto notNeeded = ActiveLayers.Contains(Identifier)` — vestige of a removed short-circuit. | Cosmetic; documented in the `.cpp`. |
