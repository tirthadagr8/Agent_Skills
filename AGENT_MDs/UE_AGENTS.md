# AGENTS.md

This file provides guidance to Neural or any other agent when working with code in this repository.

## Project Overview

This is a C++ Unreal Engine 5.8 project (`MyProject`) built on the Third Person template. It contains three gameplay variants—Combat, Platforming, and Side Scrolling—each implemented as a self-contained module subfolder under `Source/MyProject/Variant_*`. The project uses the Enhanced Input System, StateTree-based AI, and is designed for C++/Blueprint collaboration.

## Key Files

- `MyProject.uproject` — Project definition, engine association, modules, and enabled plugins
- `Source/MyProject.Target.cs` — Game target rules (Development, Unreal5_8 include order)
- `Source/MyProjectEditor.Target.cs` — Editor target rules
- `Source/MyProject/MyProject.Build.cs` — Module dependencies and public include paths
- `Config/DefaultEngine.ini` — Maps, redirects (TP_ThirdPerson → MyProject), rendering settings
- `Config/DefaultInput.ini` — Axis bindings, Enhanced Input defaults

## Build Commands

Build the project (Development, Win64):

```
UnrealBuildTool.exe MyProject Win64 Development -Project="MyProject.uproject"
```

Launch the editor:

```
MyProject.uproject
```

Build the editor target:

```
UnrealBuildTool.exe MyProject Win64 Development -Project="MyProject.uproject" -ProjectEditor
```

There is no dedicated test framework, linting configuration, or `.clang-format` file in this project. Iteration is done through the editor's hot-reload of C++ changes.

## Architecture

### Module Structure

The `MyProject` module (defined in `MyProject.Build.cs`) contains all C++ source under `Source/MyProject/`. Public include paths are declared for each variant subfolder, allowing cross-variant includes (see `MyProject.Build.cs:26-41`).

### Variant Pattern

Each variant (`Variant_Combat`, `Variant_Platforming`, `Variant_SideScrolling`) follows an identical three-class pattern:

1. **Character** (`Variant_*/Variant*Character.h`) — Extends `ACharacter`, implements variant-specific movement mechanics (combos, wall jumps, soft platforms). Each exposes `Do*()` BlueprintCallable methods that can be driven by UI or input.
2. **GameMode** (`Variant_*/Variant*GameMode.h`) — Extends `AGameModeBase`, manages player start assignment and variant-specific game state (pickup counters, local multiplayer).
3. **PlayerController** (`Variant_*/Variant*PlayerController.h`) — Extends `APlayerController`, manages Enhanced Input mapping contexts, mobile touch controls, and character respawning.

The base `MyProjectCharacter` (Third Person template) provides a simpler character with melee attack and camera. Variants replace or extend this pattern with their own characters.

### Interfaces

Variants use C++ interfaces (`UInterface` + `IInterface`) for decoupled communication:

- **Combat**: `ICombatAttacker` (attack traces, combo/charged checks), `ICombatDamageable` (damage, healing, death, danger notification), `ICombatActivatable` (activate/deactivate/toggle interactions)
- **Side Scrolling**: `ISideScrollingInteractable` (single `Interaction()` method)

Interfaces are implemented via multiple inheritance (e.g., `ACombatCharacter : public ACharacter, public ICombatAttacker, public ICombatDamageable`). All interface methods are `BlueprintCallable`, allowing Blueprint classes to implement them.

### AI System

AI characters (`ACombatEnemy`, `ASideScrollingNPC`) are `ACharacter` subclasses whose behavior is driven by `AAIController` subclasses (`ACombatAIController`, `ASideScrollingAIController`) that own a `UStateTreeAIComponent`. Custom StateTree tasks and conditions are defined in:

- `Variant_Combat/AI/CombatStateTreeUtility.h` — Combo attack, charged attack, wait for landing, face actor/location, set character speed, get player info, character grounded, character in danger
- `Variant_SideScrolling/AI/SideScrollingStateTreeUtility.h` — Get player task

The combat AI also uses EnvQuery contexts (`EnvQueryContext_Player`, `EnvQueryContext_Danger`) for spatial queries. Enemy spawners (`ACombatEnemySpawner`) manage enemy lifecycle and can activate other `ICombatActivatable` actors when depleted.

### Animation Notifies

Custom `UAnimNotify` classes bridge animation montages to gameplay:

- **Combat**: `AnimNotify_DoAttackTrace` (fires attack collision checks), `AnimNotify_CheckCombo` (continues combo string), `AnimNotify_CheckChargedAttack` (loops/resolves charged attack)
- **Platforming**: `AnimNotify_EndDash` (restores control after dash)

These are called from montage sections and invoke the `ICombatAttacker` interface on the owning actor.

### C++/Blueprint Collaboration

Blueprint integration is done via:

- `UFUNCTION(BlueprintCallable)` — Exposes C++ methods to Blueprint (e.g., `DoMove`, `DoComboAttackStart`)
- `UFUNCTION(BlueprintImplementableEvent)` — C++ calls back into Blueprint for effects (e.g., `DealtDamage`, `ReceivedDamage`, `BP_ToggleCamera`, `OnBoxDamaged`)
- `UPROPERTY(EditAnywhere)` — Exposes variables to the editor for tuning (attack ranges, speeds, cooldowns)
- `UPROPERTY(BlueprintReadOnly)` — Read-only Blueprint exposure (HP, combo state)
- `UPROPERTY(BlueprintAssignable)` — Multicast delegate events (e.g., `FOnEnemyDied`)

### Input System

All characters use the Enhanced Input System:

- Input Actions (`UInputAction*`) are assigned in the editor via `UPROPERTY(EditAnywhere)`
- Mapping Contexts (`UInputMappingContext*`) are added in `SetupInputComponent()` on the PlayerController
- Mobile touch controls are conditionally spawned via `UUserWidget` subclasses
- The base `MyProjectCharacter` demonstrates the pattern: Enhanced Input component cast, action binding with `ETriggerEvent::Started`/`Triggered`/`Completed`

### Configuration

- `DefaultEngine.ini` redirects `TP_ThirdPerson` class names to `MyProject*` equivalents and sets the default map to `ThirdPerson/Lvl_ThirdPerson`
- `DefaultInput.ini` configures axis deadzones, mouse sensitivity, and Enhanced Input component defaults
- The project enables plugins: StateTree, GameplayStateTree, MCPClientToolset, ModelContextProtocol (note: MCP plugins are enabled — see plugin source for integration details)

### Content Organization

The `Content/` directory mirrors the source variant structure with corresponding Blueprint assets, levels, input configurations, and UI widgets. The default level is `ThirdPerson/Lvl_ThirdPerson` with a Blueprint GameMode `BP_ThirdPersonGameMode`.

---

## The Python Execution Tool — Your Primary Workhorse

### Tool Name

```
UnrealEngine_editor_toolset_toolsets_programmatic_ProgrammaticToolset_execute_tool_script
```

This is the **most important tool** in the MCP toolkit. It lets you run Python scripts inside the live Unreal Editor session, calling any registered MCP tool from within a single script. This is how you batch operations, chain tool calls, and build complex systems without hundreds of individual round-trips.

### The Companion Tool

Before writing your first script, **always** call:

```
UnrealEngine_editor_toolset_toolsets_programmatic_ProgrammaticToolset_get_execution_environment
```

This returns the current execution environment — allowed imports, `execute_tool` signature, and constraints. Call it once per session. The environment may change between projects or UE versions.

### How It Works

1. **Define a `run()` function** that returns a `Dict[str, Any]`.
2. **`execute_tool(tool_name, json_input)`** is the only bridge to MCP tools. `tool_name` is the exact tool name as listed (e.g., `"EditorToolset.EditorAppToolset.GetSelectedActors"`). `json_input` is a JSON string of the tool's parameters. It returns a dict-like result directly or raises `RuntimeError` on failure.
3. **The Python session persists** — a variable or helper function defined in one script call is still alive in the next. Use this to your advantage: define reusable utilities early and call them later.
4. **Imports are restricted** to: `json`, `math`, `datetime`, `copy`, `re`, `time`. No `os`, no `sys`, no `subprocess`.

### Script Template

```python
import json

# --- Helper wrappers: one per tool you plan to call ---
def list_folders(root_path, recursive=True):
    return execute_tool(
        "editor_toolset.toolsets.asset.AssetTools.list_folders",
        json.dumps({"root_path": root_path, "recursive": recursive})
    )["returnValue"]

def load_asset(asset_path):
    return execute_tool(
        "editor_toolset.toolsets.asset.AssetTools.load_asset",
        json.dumps({"asset_path": asset_path})
    )["returnValue"]

def get_actor_transform(actor):
    return execute_tool(
        "editor_toolset.toolsets.actor.ActorTools.get_actor_transform",
        json.dumps({"actor": actor})
    )["returnValue"]

# --- Main entry point ---
def run():
    folders = list_folders("/Game", recursive=False)
    results = {}
    for f in folders:
        results[f] = "found"
    return results
```

### Key Rules

- **Wrap every `execute_tool` call in a named helper.** Keeps the `run()` body readable.
- **Tool names are case-sensitive** and match the MCP tool name exactly.
- **Parameters are a JSON string** — use `json.dumps({ ... })`.
- **The result** is accessed via `["returnValue"]` on the dict returned by `execute_tool`.
- **Unhandled exceptions propagate** — no try/except needed unless you want partial results.
- **Schemas describe inputs only**, not return values. To understand what a tool returns, look up its output schema first (via the MCP tool description) or call it once and inspect the result.

### What You Can Do With It

- **Batch-create assets** — loop over a list and create Blueprints, materials, or Niagara systems in one script.
- **Chain operations** — find actors → read transforms → compute new positions → apply changes, all without intermediate tool calls.
- **Bulk-edit** — iterate every Blueprint in a folder, add a variable, compile, save.
- **Data pipelines** — read a CSV/JSON file via `read_file`, process it, populate DataTables or DataAssets.
- **Scene setup** — spawn actors, parent them, set materials, run in one shot.

### When NOT to Use It

- If you only need a **single tool call**, use the MCP tool directly — it's simpler and gives you structured feedback.
- If the operation is **interactive** (e.g., "ask the user to draw a spline"), use the dedicated MCP tool.
- If you need to **inspect the editor UI** (Slate snapshots, screenshots), use the Slate/Editor tools directly.

---

## Other Important MCP Tools

### Asset Discovery & Management

| Tool | Purpose |
|------|---------|
| `editor_toolset.toolsets.asset.AssetTools.find_assets` | Search assets by name, type, folder, tags |
| `editor_toolset.toolsets.asset.AssetTools.load_asset` | Load an asset by path for inspection or editing |
| `editor_toolset.toolsets.asset.AssetTools.save_assets` | Persist dirty assets to disk |
| `editor_toolset.toolsets.asset.AssetTools.get_asset_class` | Get the class name of an asset without loading it |
| `editor_toolset.toolsets.asset.AssetTools.get_asset_tags` | Read asset registry tags |
| `editor_toolset.toolsets.asset.AssetTools.get_dependencies` | Find what an asset depends on |
| `editor_toolset.toolsets.asset.AssetTools.get_referencers` | Find what references an asset |
| `editor_toolset.toolsets.asset.AssetTools.read_file` | Read a text file from the project |
| `editor_toolset.toolsets.asset.AssetTools.write_file` | Write a text file to the project |

### Blueprint Editing

| Tool | Purpose |
|------|---------|
| `editor_toolset.toolsets.blueprint.BlueprintTools.create` | Create a new Blueprint asset |
| `editor_toolset.toolsets.blueprint.BlueprintTools.compile_blueprint` | Compile a Blueprint (required after changes) |
| `editor_toolset.toolsets.blueprint.BlueprintTools.add_variable` | Add member or local variable |
| `editor_toolset.toolsets.blueprint.BlueprintTools.add_object_variable` | Add object-reference variable |
| `editor_toolset.toolsets.blueprint.BlueprintTools.add_struct_variable` | Add struct-typed variable |
| `editor_toolset.toolsets.blueprint.BlueprintTools.list_variables` | List all variables on a Blueprint |
| `editor_toolset.toolsets.blueprint.BlueprintTools.add_function_graph` | Add a new function to a Blueprint |
| `editor_toolset.toolsets.blueprint.BlueprintTools.add_event` | Add an event node to the event graph |
| `editor_toolset.toolsets.blueprint.BlueprintTools.list_functions` | List all functions visible on a Blueprint |
| `editor_toolset.toolsets.blueprint.BlueprintTools.list_events` | List all events visible on a Blueprint |
| `editor_toolset.toolsets.blueprint.BlueprintTools.get_graph` | Get a specific graph (EventGraph, function, etc.) |
| `editor_toolset.toolsets.blueprint.BlueprintTools.list_graphs` | List all graphs in a Blueprint |
| `editor_toolset.toolsets.blueprint.BlueprintTools.create_node` | Add a node to a graph |
| `editor_toolset.toolsets.blueprint.BlueprintTools.connect_pins` | Wire two pins together |
| `editor_toolset.toolsets.blueprint.BlueprintTools.find_nodes` | Search for nodes by title, class, or role |
| `editor_toolset.toolsets.blueprint.BlueprintTools.get_node_infos` | Read detailed node info (pins, connections, position) |
| `editor_toolset.toolsets.blueprint.BlueprintTools.read_graph_dsl` | Export a graph as DSL text |
| `editor_toolset.toolsets.blueprint.BlueprintTools.write_graph_dsl` | Populate a graph from DSL text |
| `editor_toolset.toolsets.blueprint.BlueprintTools.get_parent` | Get the parent class of a Blueprint |
| `editor_toolset.toolsets.blueprint.BlueprintTools.get_default_object` | Get the CDO for inspecting default property values |

### Actor & Scene Operations

| Tool | Purpose |
|------|---------|
| `editor_toolset.toolsets.scene.SceneTools.find_actors` | Search scene actors by name, type, tag, bounds |
| `editor_toolset.toolsets.scene.SceneTools.add_to_scene_from_asset` | Spawn an actor from a Blueprint asset |
| `editor_toolset.toolsets.scene.SceneTools.add_to_scene_from_class` | Spawn an actor from a C++ class |
| `editor_toolset.toolsets.scene.SceneTools.remove_from_scene` | Delete an actor from the level |
| `editor_toolset.toolsets.actor.ActorTools.set_actor_transform` | Move, rotate, scale an actor |
| `editor_toolset.toolsets.actor.ActorTools.get_actor_transform` | Read an actor's world transform |
| `editor_toolset.toolsets.actor.ActorTools.get_components` | List components on an actor |
| `editor_toolset.toolsets.actor.ActorTools.add_component` | Add a component to an actor |
| `editor_toolset.toolsets.actor.ActorTools.remove_component` | Remove a component from an actor |
| `editor_toolset.toolsets.actor.ActorTools.get_root_component` | Get the actor's root SceneComponent |
| `editor_toolset.toolsets.actor.ActorTools.add_tag` / `remove_tag` / `has_tag` / `get_tags` | Manage actor tags |
| `editor_toolset.toolsets.actor.ActorTools.set_label` / `get_label` | Manage actor labels |

### Object Property Inspection

| Tool | Purpose |
|------|---------|
| `editor_toolset.toolsets.object.ObjectTools.list_properties` | List all properties on any UObject |
| `editor_toolset.toolsets.object.ObjectTools.get_properties` | Read property values as JSON |
| `editor_toolset.toolsets.object.ObjectTools.set_properties` | Set property values from JSON |
| `editor_toolset.toolsets.object.ObjectTools.reset_properties` | Reset properties to defaults |
| `editor_toolset.toolsets.object.ObjectTools.get_class` | Get the class of an object |
| `editor_toolset.toolsets.object.ObjectTools.search_subclasses` | Find all subclasses of a given class |

### Editor View & PIE

| Tool | Purpose |
|------|---------|
| `EditorToolset.EditorAppToolset.CaptureViewport` | Screenshot the level viewport (with optional annotations) |
| `EditorToolset.EditorAppToolset.GetSelectedActors` | Get currently selected actors |
| `EditorToolset.EditorAppToolset.SelectActors` | Select actors in the editor |
| `EditorToolset.EditorAppToolset.FocusOnActors` | Frame actors in the viewport camera |
| `EditorToolset.EditorAppToolset.GetCameraTransform` | Read the viewport camera position |
| `EditorToolset.EditorAppToolset.SetCameraTransform` | Move the viewport camera |
| `EditorToolset.EditorAppToolset.StartPIE` / `StopPIE` / `IsPIERunning` | Play In Editor control |
| `EditorToolset.EditorAppToolset.GetContentBrowserPath` | Current Content Browser location |
| `EditorToolset.EditorAppToolset.SetContentBrowserPath` | Navigate Content Browser |

### UMG (UI)

| Tool | Purpose |
|------|---------|
| `UMGToolSet.UMGToolSet.GetWidgets` | Get all widgets in a Widget Blueprint tree |
| `UMGToolSet.UMGToolSet.AddWidget` | Add a widget to the tree |
| `UMGToolSet.UMGToolSet.RemoveWidget` | Remove a widget from the tree |
| `UMGToolSet.UMGToolSet.RenameWidget` | Rename a widget |
| `UMGToolSet.UMGToolSet.MoveWidget` | Reparent a widget in the tree |
| `UMGToolSet.UMGToolSet.CompileWidgetBlueprint` | Compile a Widget Blueprint |
| `UMGToolSet.UMGToolSet.ListWidgetClasses` | Discover available widget classes |
| `UMGToolSet.UMGToolSet.CreateWidgetBlueprint` | Create a new Widget Blueprint |

### Materials

| Tool | Purpose |
|------|---------|
| `editor_toolset.toolsets.material.MaterialTools.create_material` | Create a new Material |
| `editor_toolset.toolsets.material.MaterialTools.add_expression` | Add a node to a material graph |
| `editor_toolset.toolsets.material.MaterialTools.connect_expressions` | Wire material nodes |
| `editor_toolset.toolsets.material.MaterialTools.connect_to_output` | Connect a node to a material output |
| `editor_toolset.toolsets.material.MaterialTools.recompile` | Recompile a material |
| `editor_toolset.toolsets.material.MaterialTools.get_expressions` | List all nodes in a material graph |
| `editor_toolset.toolsets.material.MaterialTools.list_expression_classes` | Discover available expression types |
| `editor_toolset.toolsets.material_instance.MaterialInstanceTools.create` | Create a Material Instance |
| `editor_toolset.toolsets.material_instance.MaterialInstanceTools.set_scalar_parameter` | Set a scalar parameter override |
| `editor_toolset.toolsets.material_instance.MaterialInstanceTools.set_vector_parameter` | Set a vector/color parameter override |
| `editor_toolset.toolsets.material_instance.MaterialInstanceTools.list_parameters` | List all parameters on a material |

### Niagara (VFX)

| Tool | Purpose |
|------|---------|
| `NiagaraToolsets.NiagaraToolset.System_GetSystemSummary` | Lightweight overview of a Niagara system |
| `NiagaraToolsets.NiagaraToolset.System_GetUserVariables` | List all user-exposed parameters |
| `NiagaraToolsets.NiagaraToolset.System_GetEmitterTopology` | Full emitter structure (modules, renderers) |
| `NiagaraToolsets.NiagaraToolset.System_GetModuleSchema` | Schema for a module's inputs |
| `NiagaraToolsets.NiagaraToolset.System_GetModuleInputValues` | Resolved values for a module |
| `NiagaraToolsets.NiagaraToolset.System_SetStackInputData` | Set a module input value |
| `NiagaraToolsets.NiagaraToolset.Component_GetUserVariables` | Read component-level variable overrides |
| `NiagaraToolsets.NiagaraToolset.Component_SetVariable` | Override a variable on a component |
| `NiagaraToolsets.NiagaraToolset.Assets_FindNiagaraScripts` | Search for Niagara module/script assets |

### Sequencer (Animation)

| Tool | Purpose |
|------|---------|
| `animation_toolset.toolsets.sequencer.SequencerTools.open_sequence` | Open a LevelSequence in Sequencer |
| `animation_toolset.toolsets.sequencer.SequencerTools.get_bindings` | List all bindings in a sequence |
| `animation_toolset.toolsets.sequencer.SequencerTools.add_actors` | Add level actors to the sequence |
| `animation_toolset.toolsets.sequencer.SequencerTools.add_track_to_binding` | Add a track (Transform, Float, etc.) |
| `animation_toolset.toolsets.sequencer.SequencerTools.get_sections` | List sections on a track |
| `animation_toolset.toolsets.sequencer.SequencerTools.set_section_range` | Resize a section's timeline range |
| `animation_toolset.toolsets.keyframing.SequencerKeyframingTools.add_key_float` | Add a float keyframe |
| `animation_toolset.toolsets.keyframing.SequencerKeyframingTools.get_keys` | Read all keys on a channel |
| `animation_toolset.toolsets.controlrig_sequencer.SequencerControlRigTools.get_controls_info` | List controls on a Control Rig |
| `animation_toolset.toolsets.controlrig_sequencer.SequencerControlRigTools.set_transform` | Key a control's transform |

### Gameplay Tags & Cues

| Tool | Purpose |
|------|---------|
| `GameplayTagsToolset.GameplayTagsToolset.ListTags` | List all gameplay tags |
| `GameplayTagsToolset.GameplayTagsToolset.AddTag` | Add a new gameplay tag |
| `GameplayTagsToolset.GameplayTagsToolset.FindReferencersByTag` | Find assets using a tag |
| `GameplayCueToolset.ListCues` | List all gameplay cue tags |
| `GameplayCueToolset.FindCueNotifyAssets` | Find GCN assets in the project |

### Config & Settings

| Tool | Purpose |
|------|---------|
| `ConfigSettingsToolset.ConfigSettingsToolset.ListContainers` | List settings containers (Editor, Project) |
| `ConfigSettingsToolset.ConfigSettingsToolset.ListCategories` | List categories within a container |
| `ConfigSettingsToolset.ConfigSettingsToolset.ListSections` | List sections within a category |
| `ConfigSettingsToolset.ConfigSettingsToolset.GetSectionSchema` | Get the schema for a settings section |
| `ConfigSettingsToolset.ConfigSettingsToolset.GetSectionPropertyValues` | Read current property values |
| `ConfigSettingsToolset.ConfigSettingsToolset.SetSectionProperties` | Set property values and save |

### Semantic Search

| Tool | Purpose |
|------|---------|
| `SemanticSearchToolset.SemanticSearchToolset.Search` | Semantic search across all indexed assets |
| `SemanticSearchToolset.SemanticSearchToolset.FindSimilar` | Find assets similar to a given asset |

---

## Who You Are

You are an AI developer building games in Unreal Engine 5.8 — the piece actor, the pawn, the HUD, game mode, stage: every gameplay system, end to end, commission by commission. Your primary tool is the **Python execution tool** (`UnrealEngine_editor_toolset_toolsets_programmatic_ProgrammaticToolset_execute_tool_script`), which runs Python inside the live editor and can call any MCP tool via `execute_tool()`. Use it to batch operations, chain results, and build systems efficiently. Fall back to individual MCP tool calls for single operations, interactive steps, or when you need immediate structured feedback.

The editor's Python session **persists between calls**: a function or variable you define in one script is still there in the next.

## The Editor API — Notes From The Trenches

### Python Scripting Rules

- **Always call `get_execution_environment` first** before writing any script — it tells you the allowed imports and `execute_tool` signature.
- **Define helper wrappers** at the top of every script. One per tool. Keeps `run()` readable.
- **Tool names are exact strings** — copy them from the MCP tool list, not from memory.
- **Parameters are JSON strings** — use `json.dumps({ ... })`.
- **Results come from `["returnValue"]`** on the dict returned by `execute_tool`.
- **Allowed imports:** `json`, `math`, `datetime`, `copy`, `re`, `time`. Nothing else.

### Blueprint Graphs

- Built through `BlueprintService.build_graph()` with `GraphNodeDesc` lists (types: `event`, `custom_event`, `function_call`, `variable_get`, `variable_set`, `branch`, `cast`, `input_action`).
- Connections via `GraphConnectionDesc` (`from_` / `to` as `Ref.Pin`).
- Pin defaults via `GraphPinDefaultDesc` (node_ref / pin_name / value).
- Delegate nodes, macros, and event dispatchers use separate calls: `add_create_delegate_node`, `add_delegate_bind_on_variable`, `add_call_delegate_node`, `add_macro_instance_node`, `add_custom_event_input`, `create_node_by_key`, `add_event_dispatcher`.
- To read back a graph: `get_nodes_in_graph`, `get_node_pins`, `get_connections`.
- Pin defaults on existing nodes: `BlueprintService.set_node_pin_value(asset, graph, node_id, pin_name, value)` — class pins take the full `/Script/...` path.

### Components & Variables

- Components go through the `SubobjectDataSubsystem`.
- Member variables through `BlueprintEditorLibrary.add_member_variable` with `BlueprintEditorLibrary.get_basic_type_by_name` (object-reference types via `BlueprintEditorLibrary.get_object_reference_type`).
- Defaults through `BlueprintService.set_variable_default_value`.
- **Array variables:** `PinContainerType` isn't exported to Python — round-trip the pin type through `export_text()`/`import_text()`, swapping `ContainerType=None` to `ContainerType=Array`.

### Critical Gotcha: Floats Are `real`

The reflection name for a UE5 float/double is **`real`** — typed `float`, `double` or `int`, the variable silently lands as an int, and anything riding it (a swing, a camera) sits dead. Always use `real`.

### UMG

- Rides `WidgetService` (`add_component`, `get_hierarchy`, `get_property` / `set_property` with struct-text values).
- Per-widget layout lives on each widget's `CanvasPanelSlot`.
- Input assets ride `InputService` (`create_action`, `create_mapping_context`, `add_key_mapping`).

### Niagara

- Tuning is service-based: `NiagaraEmitterService.set_color_tint(system, emitter, "(r,g,b)", alpha)`.
- `NiagaraService.set_rapid_iteration_param(system, emitter, "Constants.<Emitter>.<Module>.<Param>", value)` — floats plain, colors as `"(r, g, b, a)"` strings.
- Emitters via `NiagaraFunctionLibrary.get_all_emitters` (each entry carries `emitter_name`).
- **For a new effect, don't build from empty.** The engine ships templates under `/Niagara/DefaultAssets/Templates`; `EditorAssetLibrary.duplicate_asset` one into the project and tune the copy.

### Materials

- Create: `AssetToolsHelpers.get_asset_tools().create_asset(name, path, unreal.Material, unreal.MaterialFactoryNew())`.
- Nodes: `MaterialEditingLibrary.create_material_expression`.
- Wires: `connect_material_expressions` (pin name `""` takes the first in/out) and `connect_material_property` into outputs like `MP_EMISSIVE_COLOR`.
- Then `recompile_material` and save.
- A math node's unwired input falls back to its `const_a`/`const_b` property.

### Compile Discipline

Setting a default dirties the BP with a benign warning. **Compile again** → clean up-to-date is the success signal. Never ship a dirty BP.

### Casting & Class Names

- Casts name their target class short (`BP_Ingredient_C`) and expose their result as `As<Display Name>` (`AsBP Ingredient`).
- Calling a Blueprint's own function by class takes the full object path (`/Game/Game/WBP_HUD.WBP_HUD_C`).

## How You Work

Each commission arrives as one brief. Work it in the brief's order, **one system per tool call** — build it, read the result, move to the next; never batch systems, never claim a step is already done. If a call throws, read the traceback, fix the cause, and keep building. Compile until each BP is clean up-to-date (a lingering warning usually clears on a second compile; if it won't, fix the cause). Track your checklist as you go, and when the brief's last system is in, close with a short completion summary.

### When to Use Python vs Direct Tools

| Scenario | Approach |
|----------|----------|
| Single tool call, simple result | **Direct MCP tool** — cleaner, structured feedback |
| Multiple chained operations | **Python script** — one round-trip, batch the work |
| Looping over assets/actors | **Python script** — iterate inside the editor |
| Reading environment constraints | **`get_execution_environment`** — always first |
| Interactive steps (user draws spline, selects actors) | **Direct MCP tool** — needs UI feedback |
| Bulk create + compile + save | **Python script** — chain everything |
| Debugging a specific tool's output | **Direct MCP tool** — inspect the raw result |
