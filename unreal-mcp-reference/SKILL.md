---
name: unreal-mcp-reference
description: Comprehensive reference for the Unreal MCP server's toolset API — all 27+ toolsets covering asset operations, Blueprint editing, actor/scene manipulation, materials, Niagara VFX, Sequencer animations, Control Rig, Data Tables, String Tables, plugin management, gameplay tags, automation tests, and editor state control. Use when Codex needs to discover available MCP tools, confirm exact tool input/output schemas, or execute any Unreal Editor operation programmatically via MCP. Also covers the three meta-tools (list_toolsets, describe_toolset, call_tool) for tool discovery and invocation.
---

# Unreal MCP Server — Toolset Reference

## Overview

This skill provides the complete reference documentation for the Unreal MCP server's toolsets. The MCP server exposes Unreal Editor functionality through structured tool calls — each toolset is a group of related tools for a specific domain (assets, Blueprints, materials, Niagara, Sequencer, etc.).

**Full documentation:** See [`references/unreal_mcp_toolset.md`](references/unreal_mcp_toolset.md) for complete tool schemas, examples, and workflows.

## Core Discovery Workflow

Use these three meta-tools first in any session:

1. **`list_toolsets`** — enumerate all registered toolsets on the server
2. **`describe_toolset`** — get full schema for one toolset (every tool's input/output)
3. **`call_tool`** — invoke a concrete tool

**Critical:** `call_tool` requires `toolset_name` (full dotted path) + `tool_name` (short name only). Never pass the fully-qualified name as `tool_name`.

## Toolset Index

| Domain | Toolset | Reference |
|--------|---------|-----------|
| Asset ops | `editor_toolset.toolsets.asset.AssetTools` | Section 2 |
| Object properties | `editor_toolset.toolsets.object.ObjectTools` | Section 3 |
| Blueprint edits | `editor_toolset.toolsets.blueprint.BlueprintTools` | Section 4 |
| Skeletal mesh | `editor_toolset.toolsets.skeletal_mesh.SkeletalMeshTools` | Section 5 |
| Actor manipulation | `editor_toolset.toolsets.actor.ActorTools` | Section 8.1 |
| Scene/level | `editor_toolset.toolsets.scene.SceneTools` | Section 8.2 |
| Materials | `editor_toolset.toolsets.material.MaterialTools` | Section 8.3 |
| Material instances | `editor_toolset.toolsets.material_instance.MaterialInstanceTools` | Section 8.4 |
| Static mesh | `editor_toolset.toolsets.static_mesh.StaticMeshTools` | Section 8.5 |
| Primitives | `editor_toolset.toolsets.primitive.PrimitiveTools` | Section 8.6 |
| Textures | `editor_toolset.toolsets.texture.TextureTools` | Section 8.7 |
| Data Tables | `editor_toolset.toolsets.data_table.DataTableTools` | Section 8.8 |
| Curve Tables | `editor_toolset.toolsets.curve_table.CurveTableTools` | Section 8.9 |
| Data Assets | `editor_toolset.toolsets.data_asset.DataAssetTools` | Section 8.10 |
| String Tables | `editor_toolset.toolsets.string_table.StringTableTools` | Section 8.11 |
| Scripted batch | `editor_toolset.toolsets.programmatic.ProgrammaticToolset` | Section 8.12 |
| Editor state | `EditorToolset.EditorAppToolset` | Section 8.13 |
| Sequencer | `animation_toolset.toolsets.sequencer.SequencerTools` | Section 8.14 |
| Keyframe anim | `animation_toolset.toolsets.keyframing.SequencerKeyframingTools` | Section 8.15 |
| Control Rig | `animation_toolset.toolsets.controlrig.ControlRigTools` | Section 8.16 |
| Niagara system | `NiagaraToolsets.NiagaraToolset_System` | Section 8.17 |
| Niagara component | `NiagaraToolsets.NiagaraToolset_Component` | Section 8.18 |
| Niagara assets | `NiagaraToolsets.NiagaraToolset_Assets` | Section 8.19 |
| Niagara BP wrapper | `NiagaraToolsets.NiagaraToolset_Blueprint` | Section 8.20 |
| Editor logs | `EditorToolset.LogsToolset` | Section 8.21 |
| Config settings | `ConfigSettingsToolset.ConfigSettingsToolset` | Section 8.22 |
| Data registries | `DataRegistryToolset.DataRegistryTools` | Section 8.23 |
| Gameplay tags | `GameplayTagsToolset.GameplayTagsToolset` | Section 8.24 |
| Automation tests | `AutomationTestToolset.AutomationTestToolset` | Section 8.25 |
| Plugins | `PluginToolset.PluginToolset` | Section 8.26 |
| Semantic search | `SemanticSearchToolset.SemanticSearchToolset` | Section 8.27 |
| Slate UI automation | `SlateInspectorToolset.SlateInspectorToolset` | Section 8.28 |
| Gameplay cues | `GASToolsets.GameplayCueToolset` | Section 8.29 |
| Attribute sets | `GASToolsets.AttributeSetToolset` | Section 8.30 |
| Ability Inspector | `GASToolsets.AbilitySystemInspectorToolset` | Section 8.31 |

## Key Data Conventions

- **Object references:** `{"refPath": "/Game/Path/Asset.Asset"}` — used in tool inputs and nested property values
- **Blueprint CDOs:** Use `Default__<ClassName>_C` pattern; `get_default_object` returns the CDO refPath
- **Property names:** Not UE display names — e.g. `skeletalMeshAsset`, not `"Skeletal Mesh"`. Always call `list_properties` first
- **Generated classes:** `_C` suffix for Blueprint-generated classes (e.g. `BP_Foo_C`)
- **`float` is `real`:** In UE5 reflection, typed float properties must be cast as `real` type
- **After `set_properties`:** Compile the Blueprint (`compile_blueprint`) so the CDO reflects changes

## Common Workflows

### Character/asset swap (mesh + anim blueprint)
1. `list_toolsets` → discover; `describe_toolset` on `AssetTools`, `ObjectTools`, `SkeletalMeshTools`, `BlueprintTools`
2. `get_asset_class` on candidate assets to confirm types
3. `load_asset` each asset; use returned `refPath` for ObjectTools
4. `get_skeleton` (mesh) vs `get_asset_tags` (`TargetSkeleton` on anim BP) → verify skeleton compatibility
5. `get_bounds` on old vs new mesh → compute scale ratio
6. `get_default_object` on the character BP → `list_properties` / `get_properties` on the mesh component
7. `set_properties` on the mesh component → new `skeletalMesh`, `animClass`, `relativeScale3D`, `relativeLocation`
8. `compile_blueprint` → bake changes; watch for downstream errors
9. `save_assets` → persist

### Setting collision on StaticMeshComponents
1. `find_actors` with name filter → get target actors
2. For each actor, `get_properties` → `staticMeshComponent` to get the component refPath
3. `get_properties` on the component → check `bUseDefaultCollision`
4. If `bUseDefaultCollision` is `true`, set it to `false` FIRST: `set_properties` with `{"bUseDefaultCollision":false}`
5. THEN set collision: `set_properties` with `{"BodyInstance":{"collisionEnabled":"NoCollision"}}`
6. For Actor-type trees (not StaticMeshActor), use `get_components` to find Foliage/Trunk components, skip DefaultSceneRoot/BillboardComponent_0
7. Use `ProgrammaticToolset.execute_tool_script` to batch across many actors

### Diagnosing an anim-BP compile error
1. Read the editor log for the error message
2. `read_graph_dsl` on the anim BP's `EventGraph` → see what variables it reads
3. `list_variables` on the character BP → empty means the variable lives in C++
4. `get_parent` on the character BP → reveals the C++ parent class
5. Add the missing `UPROPERTY(BlueprintReadWrite)` + override relevant methods in C++
6. Rebuild via `UnrealBuildTool`, reopen editor → error clears

## Pitfalls & Gotchas

1. `call_tool` needs short tool name + separate `toolset_name` — fully-qualified dotted name as `tool_name` fails
2. Property names are not display names — call `list_properties` first
3. `animClass` must reference the generated `_C` class — `/Game/.../MyAnim.MyAnim_C`
4. Object references are `{"refPath": "..."}` — for tool inputs and nested property values
5. Nested-struct properties may be unreadable by name — read the parent struct instead
6. Compile after mutating — `set_properties` dirties the BP; `compile_blueprint` is mandatory
7. Skeleton compatibility is the anim-BP gate — verify `get_skeleton` matches `TargetSkeleton` tag
8. Bounds reveal scale — compare `get_bounds` on old vs new mesh for scale ratio
9. Build commands require absolute `.uproject` path; editor build blocked while editor runs
10. Editor must be open — all tools run against the currently running editor
11. **`bUseDefaultCollision` overrides instance collision** — StaticMeshComponents default to `bUseDefaultCollision=true`, inheriting collision from the mesh asset. Setting `BodyInstance.collisionEnabled` does NOTHING until you first set `bUseDefaultCollision=false`. This is the #1 reason collision changes appear to succeed (returns `true`) but don't take effect
12. **Actor-type trees vs StaticMeshActor trees** — trees can be plain Actors (with Foliage/Trunk components) or StaticMeshActors (with StaticMeshComponent). Use `get_class` to distinguish, then apply collision to the right component types. Skip `DefaultSceneRoot` and `BillboardComponent_0` — they have no `BodyInstance`
13. **`find_actors` requires `collision_channels: []`** — even an empty array is mandatory in the input schema

## Searching the Reference

The full documentation is large. Search patterns for finding specific information:

- Find all tools in a toolset: search for `**Toolset:** ` + toolset name
- Find a specific tool's input schema: search for the tool name + `Input`
- Find a specific toolset's notes: search for `### 8` + section number
- Find material properties: search for `MP_` (e.g. `MP_EmissiveColor`, `MP_BaseColor`)
- Find Niagara script stacks: search for `SpawnScript` or `UpdateScript`
- Find UE data types: search for `refPath`, `real`, `_C` suffix
