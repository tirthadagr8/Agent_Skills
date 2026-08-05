# Unreal MCP Server — Toolset Documentation

> **Scope (pass 1):** Originally covered only tools exercised during the Valhalla character-swap commission. **Pass 2** expands to document every toolset described via `describe_toolset` — complete schema for each.
>
> **Session context:** UE 5.8, project `MyProject`, Windows. Pass 1 was the character-swap task; pass 2 is a systematic schema-documentation pass over the Unreal MCP server's toolset registry.

---

## Table of Contents

1. [MCP Protocol Meta-Tools](#1-mcp-protocol-meta-tools)
2. [AssetTools](#2-assettools)
3. [ObjectTools](#3-objecttools)
4. [BlueprintTools](#4-blueprinttools)
5. [SkeletalMeshTools](#5-skeletalmeshtools)
6. [Practical Workflows Learned](#6-practical-workflows)
7. [Common Pitfalls & Gotchas](#7-common-pitfalls)
8. [Pass 2 — New Toolsets](#8-pass-2--new-toolsets)
9. [Coverage Map & Status](#9-coverage-map)

---

## 1. MCP Protocol Meta-Tools

These three tools are the **entry layer** of the Unreal MCP server. Every concrete tool is reached through `call_tool`, and discovery happens through `list_toolsets` / `describe_toolset`.

### list_toolsets

- **Purpose:** Enumerate every registered toolset on the server.
- **Returns:** A flat list of `ToolsetRegistry.<Name>` entries, each with a short description.
- **Typical use:** First call in a session — establishes what is available.

**Example output:**
```
ToolsetRegistry.AgentSkillToolset
ConfigSettingsToolset.ConfigSettingsToolset
EditorToolset.EditorAppToolset
editor_toolset.toolsets.asset.AssetTools
editor_toolset.toolsets.blueprint.BlueprintTools
editor_toolset.toolsets.object.ObjectTools
editor_toolset.toolsets.skeletal_mesh.SkeletalMeshTools
... (many more)
```

**Notes:** The full list is large; use `describe_toolset` to drill into the ones you need.

### describe_toolset

- **Purpose:** Return the full schema for one toolset — every tool it exposes, plus each tool's input/output JSON schemas.
- **Input:** `toolset_name` — the full dotted name, e.g. `editor_toolset.toolsets.skeletal_mesh.SkeletalMeshTools`.
- **Returns:** `name`, `version`, `description`, and a `tools[]` array. Each tool entry includes `name`, `description`, `inputSchema`, and `outputSchema`.

**Notes:**
- Object references in schemas are serialized as `{"refPath": "/Game/Path/Asset.Asset"}`.
- `title` on a schema property often reveals the underlying UE type (e.g. `/Script/Engine.SkeletalMesh`).
- Use this to **confirm the exact property name** before calling `get_properties`/`set_properties`.

### call_tool

- **Purpose:** Invoke a concrete tool on a toolset.
- **Inputs:** `toolset_name`, `tool_name` (short name only), and `arguments` (JSON matching `inputSchema`).
- **Returns:** The tool's `returnValue` per its `outputSchema`.

**Critical:** `tool_name` must be the **short** name, not the fully-qualified dotted path.

- Wrong: `tool_name: "editor_toolset.toolsets.asset.AssetTools.get_asset_class"`
- Correct: `toolset_name: "editor_toolset.toolsets.asset.AssetTools"`, `tool_name: "get_asset_class"`

**Notes:** The server runs tools against the **currently running editor** — the editor must be open.

---

## 2. AssetTools

**Toolset:** `editor_toolset.toolsets.asset.AssetTools`
**Purpose:** Asset-level operations — class lookup, loading, dependency/tag introspection, saving, existence checks.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `get_asset_class` | Return class of an asset | `asset_path` | Class name string (e.g. `SkeletalMesh`, `BP_ThirdPersonCharacter_C`) |
| `load_asset` | Load asset into memory | `asset_path` | `{"refPath": "..."}` object reference |
| `get_dependencies` | List assets this asset depends on | `asset_path` | Array of content paths |
| `get_asset_tags` | Return asset registry tags | `asset_path` | Dict of tag keys to values |
| `save_assets` | Persist assets to disk | `asset_paths` (empty = all dirty) | `true`/`false` |
| `exists` | Check if asset/folder exists | `path` | `true`/`false` |

**Notes:**
- The `_C` suffix distinguishes a Blueprint's generated class from its asset object.
- `load_asset` returns the canonical `refPath` (`Package.Asset`) used by ObjectTools.
- Dependencies include native `/Script/...` entries — filter for `/Game/` paths for asset-level relationships.
- Tags like `TargetSkeleton` confirm anim BP skeleton compatibility.

---

## 3. ObjectTools

**Toolset:** `editor_toolset.toolsets.object.ObjectTools`
**Purpose:** Read/write properties on any `UObject` — including Blueprint CDOs and sub-components. The workhorse for inspecting and mutating the editor state.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `list_properties` | Full property schema of an object | `instance` | JSON string: type, title, description per property |
| `get_properties` | Read values of named properties | `instance` + `properties` array | JSON string mapping property names to values |
| `set_properties` | Set values of named properties | `instance` + `values` JSON string | `true` on success |

**Notes:**
- Property names are **not** UE display names (e.g. `skeletalMeshAsset`, not `Skeletal Mesh`). Always `list_properties` first.
- Object-reference values use `{"refPath": "..."}` form.
- Class-pin values (like `animClass`) must reference the generated `_C` class.
- Nested struct values are passed as nested JSON.
- After `set_properties`, **compile the Blueprint** so the CDO reflects the change.
- Some nested-struct properties (e.g. `collisionProfileName`) may be unreadable by name — read the parent struct instead.

---

## 4. BlueprintTools

**Toolset:** `editor_toolset.toolsets.blueprint.BlueprintTools`
**Purpose:** Blueprint-level operations — class defaults, variables, functions, graphs, compilation.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `get_default_object` | Return CDO for a Blueprint | `blueprint` refPath | CDO `{"refPath": "..."}` |
| `list_variables` | List member variables | `blueprint` (+ optional `graph`) | Array of variable names |
| `list_functions` | List functions/events | `blueprint` | Array of `{name, description, bIsImplemented}` |
| `get_parent` | Return parent class | `blueprint` | `{"refPath": "/Script/Module.Class"}` |
| `list_graphs` | List all graphs | `blueprint` | Array of graph object references |
| `read_graph_dsl` | Read graph as S-expression DSL | `graph` refPath | DSL text |
| `compile_blueprint` | Compile a Blueprint | `blueprint` (+ optional `warnings_as_errors`) | `null` on success |

**Notes:**
- CDO path pattern: `Default__<ClassName>_C`.
- `bIsImplemented` distinguishes overridable inherited functions from ones with actual graphs.
- For anim BPs, `list_graphs` reveals the state machine structure.
- Compiling surfaces downstream errors; a lingering warning usually clears on second compile.

---

## 5. SkeletalMeshTools

**Toolset:** `editor_toolset.toolsets.skeletal_mesh.SkeletalMeshTools`
**Purpose:** Inspect skeletal mesh assets — skeleton, materials, bones, physics, bounds.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `get_skeleton` | Return skeleton bound to mesh | `mesh` | Skeleton refPath |
| `get_material_slots` | List material slot names | `mesh` | Array of slot name strings |
| `get_material` | Return material for a slot | `mesh` + `slot_name` | Material refPath |
| `get_bone_names` | List all bone names (hierarchy order) | `mesh` | Array of bone name strings |
| `get_physics_asset` | Return physics asset | `mesh` | Physics asset refPath |
| `get_bounds` | Return local-space bounding volume | `mesh` | `{origin, boxExtent, sphereRadius}` |

**Notes:**
- Bounds are local-space reference-pose; use for size comparisons, not runtime.
- Verify mesh `get_skeleton` matches the anim BP's `TargetSkeleton` tag for compatibility.

---

## 6. Practical Workflows Learned

### A. Character/asset swap (mesh + anim blueprint)

1. `list_toolsets` → discover; `describe_toolset` on `AssetTools`, `ObjectTools`, `SkeletalMeshTools`, `BlueprintTools`.
2. `get_asset_class` on candidate mesh/skeleton/anim assets to confirm types.
3. `load_asset` each asset; use returned `refPath` for ObjectTools.
4. `get_skeleton` (mesh) vs `get_asset_tags` (`TargetSkeleton` on anim BP) → verify skeleton compatibility.
5. `get_bounds` on old vs new mesh → compute scale ratio.
6. `get_default_object` on the character BP → `list_properties` / `get_properties` on the mesh component to read current `skeletalMesh` / `animClass`.
7. `set_properties` on the mesh component → new `skeletalMesh`, `animClass`, `relativeScale3D`, `relativeLocation`.
8. `compile_blueprint` → bake changes; watch for downstream errors.
9. `save_assets` → persist.

### B. Diagnosing an anim-BP compile error

1. Read the editor log for the error message.
2. `read_graph_dsl` on the anim BP's `EventGraph` → see what variables it reads.
3. `list_variables` on the character BP → empty means the variable lives in C++.
4. `get_parent` on the character BP → reveals the C++ parent class.
5. Add the missing `UPROPERTY(BlueprintReadWrite)` + override relevant methods in C++.
6. Rebuild via `UnrealBuildTool`, reopen editor → error clears.

### C. Build commands

```powershell
# Game target
& "C:\Program Files\Epic Games\UE_5.8\Engine\Binaries\ThirdParty\DotNet\10.0\win-x64\dotnet.exe" `
  "C:\Program Files\Epic Games\UE_5.8\Engine\Binaries\DotNET\UnrealBuildTool\UnrealBuildTool.dll" `
  MyProject Win64 Development -Project="C:\...\MyProject.uproject"

# Editor target (editor must be CLOSED)
& "C:\Program Files\Epic Games\UE_5.8\Engine\Binaries\ThirdParty\DotNet\10.0\win-x64\dotnet.exe" `
  "C:\Program Files\Epic Games\UE_5.8\Engine\Binaries\DotNET\UnrealBuildTool\UnrealBuildTool.dll" `
  MyProjectEditor Win64 Development -Project="C:\...\MyProject.uproject"
```

**Notes:** Absolute `.uproject` path required; editor build blocked while editor runs.

---

## 7. Common Pitfalls & Gotchas

1. **`call_tool` needs short tool name + separate `toolset_name`.** Fully-qualified dotted name as `tool_name` fails.
2. **Property names are not display names.** `skeletalMeshAsset`, not `Skeletal Mesh`. Call `list_properties` first.
3. **`animClass` must reference the generated `_C` class.** `/Game/.../valhalla_anim.valhalla_anim_C`.
4. **Object references are `{"refPath": "..."}`.** For tool inputs and nested property values.
5. **Nested-struct properties may be unreadable by name.** Read the parent `bodyInstance` struct instead.
6. **Compile after mutating.** `set_properties` dirties the BP; `compile_blueprint` is mandatory.
7. **Skeleton compatibility is the anim-BP gate.** Verify `get_skeleton` matches `TargetSkeleton` tag.
8. **Bounds reveal scale.** Compare `get_bounds` on old vs new mesh for scale ratio.
9. **Anim-BP variable dependencies can live in C++ parents.** Fix in C++, not Blueprint.
10. **Build tooling quirks:** absolute path required; editor build blocked while running; engine-shipped .NET 10 needed.

---

## 8. Pass 2 — New Toolsets (schema-documented)

> **Note:** These toolsets were **described** via `describe_toolset` to capture their full schema. They have not all been exercised in-session; the documentation below reflects the authoritative JSON schemas returned by the server.

### 8.1 ActorTools

**Toolset:** `editor_toolset.toolsets.actor.ActorTools`
**Purpose:** Inspect and modify actors — transforms, labels, parent-child relationships, components, tags.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `add_component` | Add component to actor | `owner`, `component_type`, `name` | New `ActorComponent` refPath |
| `remove_component` | Remove component from actor | `component` | `true`/`false` |
| `get_components` | List components on actor | `actor`, optional `component_type` filter | Array of `ActorComponent` refPaths |
| `get_actor_bounds` | World-space bounding box | `actor` | `{min, max, isValid}` |
| `set_parent_component` | Reparent a SceneComponent | `component`, `parent` (null to detach) | `true`/`false` |
| `get_parent_component` | Get parent of SceneComponent | `component` | Parent `SceneComponent` refPath |
| `get_component_actor` | Get actor owning a component | `component` | Owner `Actor` refPath |
| `get_root_component` | Get root SceneComponent | `actor` | Root `SceneComponent` refPath |
| `look_at` | Rotate actor to face position | `actor`, `target` (Vector) | — |
| `set_actor_transform` | Set position/rotation/scale | `actor`, `xform`, `worldspace` | `true`/`false` |
| `get_actor_transform` | Get world-space transform | `actor` | `{location, rotation, scale}` |
| `add_tag` | Add tag to actor | `actor`, `tag` | — |
| `remove_tag` | Remove tag from actor | `actor`, `tag` | — |
| `has_tag` | Check if actor has tag | `actor`, `tag` | `true`/`false` |
| `get_tags` | List all tags on actor | `actor` | Array of tag strings |
| `set_label` | Set actor display name | `actor`, `label` | `true`/`false` |
| `get_label` | Get actor display name | `actor` | Label string |

**Key type:** `ToolsetTransform` = `{location: {x,y,z}, rotation: {pitch,yaw,roll}, scale: {x,y,z}}` — all optional; unset fields mean "identity" on create, "don't change" on modify.

### 8.2 SceneTools

**Toolset:** `editor_toolset.toolsets.scene.SceneTools`
**Purpose:** Work with the currently loaded level — load levels, place/remove actors, control camera, organize outliner, Level Instances.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `load_level` | Load a level | `level_path` | — |
| `get_current_level` | Get current level path | — | Level path string |
| `add_to_scene_from_asset` | Spawn actor from asset | `asset_path`, `name`, `xform`, optional `parent`, `snap_to_ground` | New `Actor` refPath |
| `add_to_scene_from_class` | Spawn actor from class | `actor_type`, `name`, `xform`, optional `parent`, `snap_to_ground` | New `Actor` refPath |
| `remove_from_scene` | Delete actor from scene | `actor` | `true`/`false` |
| `save_actor` | Save actor to disk | `actor` | — |
| `find_actors` | Search scene by criteria | optional: `root`, `name`, `actor_type`, `tag`, `bounds`, `collision_channels` | Array of `Actor` refPaths |
| `get_collision_channels` | List collision channels | — | Array of channel strings |
| `trace_world` | Line trace in world | `start`, `end` (Vectors) | Distance to hit or null |
| `create_level_instance` | Create Level Instance actor | `level_path`, `name`, `xform`, optional `parent` | New `LevelInstance` refPath |
| `edit_level_instance` | Open level instance for editing | `level_instance` | — |
| `commit_level_instance` | Save/discard level instance edits | `level_instance`, `discard` | — |
| `merge_actors` | Merge StaticMeshActors into one | `actors`, `output_path`, `name`, `destroy_source_actors` | Merged `StaticMeshActor` refPath |
| `get_folders` | List outliner folder paths | — | Array of folder path strings |
| `set_actor_folder` | Assign actor to folder | `actor`, `folder_path` | — |
| `get_actors_in_folder` | List actors in folder | `folder_path`, `recursive` | Array of `Actor` refPaths |
| `rename_folder` | Rename outliner folder | `old_path`, `new_path` | Count of updated actors |
| `delete_folder` | Delete outliner folder | `folder_path` | Count of moved actors |
| `can_edit` | Check if actor can be edited | `actor` | `true`/`false` |
| `is_checked_out` | Check source control checkout | `actor` | `true`/`false` |

### 8.3 MaterialTools

**Toolset:** `editor_toolset.toolsets.material.MaterialTools`
**Purpose:** Create and edit Material and MaterialFunction assets — graph nodes, connections, recompilation.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `create_material` | Create new empty Material | `folder_path`, `asset_name` | New `Material` refPath |
| `create_function` | Create new empty MaterialFunction | `folder_path`, `asset_name` | New `MaterialFunction` refPath |
| `create_parameter_collection` | Create new MPC asset | `folder_path`, `asset_name` | New `MaterialParameterCollection` refPath |
| `add_expression` | Add expression node to graph | `material_or_function`, `expression_class`, `x`, `y` | New `MaterialExpression` refPath |
| `delete_expression` | Remove expression node | `material_or_function`, `expression` | — |
| `get_expressions` | List all expression nodes | `material_or_function` | Array of `MaterialExpression` refPaths |
| `list_expression_classes` | Discover available expression types | `material_or_function`, optional `search` | Array of class refPaths |
| `connect_expressions` | Wire output pin to input pin | `from_expression`, `from_output_name`, `to_expression`, `to_input_name` | — |
| `disconnect_expressions` | Unwire an input pin | `to_expression`, `to_input_name` | — |
| `connect_to_output` | Wire expression to material output | `expression`, `output_name`, `material_property` | — |
| `disconnect_from_output` | Unwire a material output | `material`, `material_property` | — |
| `get_expression_inputs` | Get input wiring of expression | `material_or_function`, `expression` | Array of `{output_name, expression, input_name}` |
| `get_property_input` | Get expression feeding an output | `material`, `material_property` | `{output_name, expression, input_name}` |
| `get_expression_input_names` | List input pin names | `expression` | Array of pin name strings |
| `get_expression_output_names` | List output pin names | `expression` | Array of pin name strings |
| `recompile` | Recompile material/function | `material_or_function` | — |
| `delete_unused_expressions` | Remove disconnected nodes | `material` | — |
| `layout_expressions` | Auto-arrange graph nodes | `material_or_function` | — |
| `list_parameter_groups` | List parameter group names | `material_or_function` | Array of group name strings |
| `rename_parameter_group` | Rename a parameter group | `material_or_function`, `old_name`, `new_name` | — |
| `delete_parameter_group` | Remove a parameter group | `material_or_function`, `group_name` | — |
| `get_referencing_materials` | Find materials using a function | `material_function` | Array of AssetData |

**Notes:** Material property enum values include `MP_EmissiveColor`, `MP_BaseColor`, `MP_Metallic`, `MP_Roughness`, `MP_Normal`, etc. Empty string `""` uses the first/default output pin.

### 8.4 MaterialInstanceTools

**Toolset:** `editor_toolset.toolsets.material_instance.MaterialInstanceTools`
**Purpose:** Create and modify MaterialInstanceConstant assets — parameter overrides, parent chains.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `create` | Create new MI from parent | `folder_path`, `asset_name`, `parent` | New `MaterialInstanceConstant` refPath |
| `set_parent` | Change parent material | `instance`, `parent` | — |
| `clear_parameters` | Reset all overrides to parent | `instance` | — |
| `set_parameter_override` | Enable/disable parameter override | `instance`, `name`, `override` | — |
| `get_scalar_parameter` | Read scalar parameter value | `instance`, `name` | Float value |
| `set_scalar_parameter` | Write scalar parameter value | `instance`, `name`, `value` | — |
| `get_vector_parameter` | Read vector (color) parameter | `instance`, `name` | `{r, g, b, a}` LinearColor |
| `set_vector_parameter` | Write vector (color) parameter | `instance`, `name`, `value` | — |
| `get_texture_parameter` | Read texture parameter | `instance`, `name` | Texture refPath or null |
| `set_texture_parameter` | Write texture parameter | `instance`, `name`, `value` | — |
| `get_static_switch_parameter` | Read static switch value | `instance`, `name` | Boolean |
| `set_static_switch_parameter` | Write static switch value | `instance`, `name`, `value` | — |
| `list_parameters` | List all exposed parameters | `material` | Array of `{name, type}` (Scalar/Vector/Texture/StaticSwitch) |

**Notes:** Static switches preserve their value across toggle; non-static parameters lose their override value when disabled.

### 8.5 StaticMeshTools

**Toolset:** `editor_toolset.toolsets.static_mesh.StaticMeshTools`
**Purpose:** Inspect and modify static mesh assets — LODs, materials, collision, Nanite, bounds.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `get_bounds` | Local-space bounding box | `mesh` | `{min, max, isValid}` |
| `get_vertex_count` | Vertices in an LOD | `mesh`, optional `lod_index` | Integer |
| `get_triangle_count` | Triangles in an LOD | `mesh`, optional `lod_index` | Integer |
| `get_lod_count` | Number of LODs | `mesh` | Integer |
| `get_lod_thresholds` | Screen-size thresholds per LOD | `mesh` | Array of floats |
| `set_lod_thresholds` | Set screen-size thresholds | `mesh`, `thresholds` | `true`/`false` |
| `generate_lods` | Auto-generate LODs via reduction | `mesh`, `triangle_percents` | Total LOD count |
| `remove_lods` | Remove all auto-LODs, keep LOD0 | `mesh` | `true`/`false` |
| `get_material_slots` | List material slot names | `mesh` | Array of slot names |
| `get_material` | Get material for a slot | `mesh`, `slot_name` | MaterialInterface refPath |
| `set_material` | Assign material to a slot | `mesh`, `slot_name`, `material` | `true`/`false` |
| `generate_convex_collisions` | Generate convex hull collision | `mesh`, optional `hull_count`, `max_hull_verts`, `hull_precision` | `true`/`false` |
| `remove_collisions` | Remove all collision shapes | `mesh` | `true`/`false` |
| `is_nanite_enabled` | Check if Nanite is enabled | `mesh` | `true`/`false` |
| `set_nanite_enabled` | Enable/disable Nanite | `mesh`, `enabled` | — |
| `import_file` | Import mesh from disk | `folder_path`, `asset_name`, `source_file`, optional flags | Array of imported assets |

**Notes:** Screen size threshold is a ratio of mesh screen height to viewport height. Thresholds must be strictly descending (LOD0 largest).

### 8.6 PrimitiveTools

**Toolset:** `editor_toolset.toolsets.primitive.PrimitiveTools`
**Purpose:** Add primitive geometry components to actors.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `add_cube` | Add cube StaticMeshComponent | `actor`, `name`, optional `dimensions`, `local_transform` | New `StaticMeshComponent` refPath |
| `add_sphere` | Add sphere StaticMeshComponent | `actor`, `name`, optional `radius`, `local_transform` | New `StaticMeshComponent` refPath |
| `add_cone` | Add cone StaticMeshComponent | `actor`, `name`, optional `radius`, `height`, `local_transform` | New `StaticMeshComponent` refPath |
| `add_cylinder` | Add cylinder StaticMeshComponent | `actor`, `name`, optional `radius`, `height`, `local_transform` | New `StaticMeshComponent` refPath |

**Notes:** Default dimensions: cube 100x100x100, sphere radius 50, cone radius 50/height 100, cylinder radius 50/height 100.

### 8.7 TextureTools

**Toolset:** `editor_toolset.toolsets.texture.TextureTools`
**Purpose:** Work with Texture2D assets.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `get_size` | Get pixel dimensions | `texture` | `{x, y}` (width, height) |
| `import_file` | Import image from disk | `folder_path`, `asset_name`, `source_file` | Array of imported assets |

### 8.8 DataTableTools

**Toolset:** `editor_toolset.toolsets.data_table.DataTableTools`
**Purpose:** Create and edit DataTable assets — rows, columns, schema.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `create` | Create new DataTable | `folder_path`, `asset_name`, `schema` (TableRowBase struct) | New `DataTable` refPath |
| `import_file` | Import from disk | `folder_path`, `asset_name`, `source_file`, `schema` | Array of imported assets |
| `search_row_structs` | Find usable schema structs | optional `struct_name` (wildcard) | Array of ScriptStruct refPaths |
| `get_schema` | Get column schema | `data_table` | JSON string of column types |
| `list_rows` | List all row names | `data_table` | Array of row name strings |
| `add_rows` | Add rows with defaults | `data_table`, `row_names` | — |
| `remove_rows` | Remove rows | `data_table`, `row_names` | — |
| `rename_rows` | Rename rows | `data_table`, `renames` (old→new map) | — |
| `get_rows` | Get row values | `data_table`, `row_names` | JSON string of row data |
| `set_rows` | Set row values | `data_table`, `values` (JSON string) | — |

**Notes:** Schema must be a subclass of `TableRowBase`. Row values use camelCase property names.

### 8.9 CurveTableTools

**Toolset:** `editor_toolset.toolsets.curve_table.CurveTableTools`
**Purpose:** Create and edit CurveTable assets — rows, keys, interpolation.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `create` | Create new CurveTable | `folder_path`, `asset_name` | New `CurveTable` refPath |
| `import_file` | Import from disk | `folder_path`, `asset_name`, `source_file`, `interp_mode` | Array of imported assets |
| `list_rows` | List all row names | `curve_table` | Array of row name strings |
| `add_row` | Add new row | `curve_table`, `row_name`, optional `default_value` | — |
| `remove_row` | Remove row | `curve_table`, `row_name` | — |
| `rename_row` | Rename row | `curve_table`, `row_name`, `new_row_name` | — |
| `get_keys` | Get all keys for a row | `curve_table`, `row_name` | Array of `{time, value}` |
| `add_key` | Add key to row | `curve_table`, `row_name`, `key` `{time, value}` | `true`/`false` |
| `set_keys` | Replace all keys in row | `curve_table`, `row_name`, `keys` array | `true`/`false` |

**Notes:** `interp_mode` for import: `RCIM_Linear` or `RCIM_Constant`. Keys are `{time: number, value: number}`.

### 8.10 DataAssetTools

**Toolset:** `editor_toolset.toolsets.data_asset.DataAssetTools`
**Purpose:** Work with DataAsset objects — typed data containers.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `create` | Create new DataAsset | `folder_path`, `asset_name`, `asset_type` (DataClass) | New `DataAsset` refPath |

**Notes:** `asset_type` must be a specific DataClass subclass. Use ObjectTools to read/write properties on the created DataAsset.

### 8.11 StringTableTools

**Toolset:** `editor_toolset.toolsets.string_table.StringTableTools`
**Purpose:** Create and edit StringTable assets — localization, text properties.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `create` | Create new StringTable | `folder_path`, `asset_name` | New `StringTable` refPath |
| `import_file` | Import from disk (CSV/JSON) | `folder_path`, `asset_name`, `source_file` | Array of imported assets |
| `get_namespace` | Get table namespace | `string_table` | Namespace string |
| `get_table_id` | Get table ID for references | `string_table` | Table ID string |
| `list_keys` | List all entry keys | `string_table` | Array of key strings |
| `get_entry` | Get source string for key | `string_table`, `key` | Source string |
| `set_entry` | Add/update entry | `string_table`, `key`, `value` | — |
| `remove_entry` | Remove entry | `string_table`, `key` | — |

**Notes:** Import files must have header row with at least `Key` and `SourceString` columns.

### 8.12 ProgrammaticToolset

**Toolset:** `editor_toolset.toolsets.programmatic.ProgrammaticToolset`
**Purpose:** Batch multiple tool calls into a single Python script execution — reduces round-trips.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `get_execution_environment` | Get available modules & instructions | — | `{instructions, supported_modules, language}` |
| `execute_tool_script` | Run Python script against toolset APIs | `script` (must define `run()` returning dict) | JSON string of result |

**Notes:** 
- MUST call `get_execution_environment` first to learn available modules.
- Script must define a `run()` function that returns a `Dict[str, Any]`.
- Sandboxed — not general Python execution, tool orchestration only.
- Look up output schemas before writing scripts to know how to parse results.

### 8.13 EditorAppToolset

**Toolset:** `EditorToolset.EditorAppToolset`
**Purpose:** Query and modify Unreal Editor state — console variables, viewport, content browser, PIE.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `GetCameraTransform` | Get viewport camera transform | — | `{location, rotation, scale}` |
| `SetCameraTransform` | Set viewport camera transform | `transform` | — |
| `FocusOnActors` | Focus camera on actors | `actors` array | — |
| `GetSelectedActors` | Get selected actors in level | — | Array of `Actor` refPaths |
| `SelectActors` | Select actors in level | `actors` array | — |
| `GetSelectedAssets` | Get selected assets in content browser | — | Array of package path strings |
| `SelectAssets` | Select assets in content browser | `assetPaths` array | — |
| `GetContentBrowserPath` | Get current content browser path | — | Path string |
| `SetContentBrowserPath` | Navigate content browser | `path` | — |
| `GetOpenAssets` | Get assets open in editors | — | Array of package path strings |
| `OpenEditorForAsset` | Open asset editor | `assetPath` | — |
| `GetVisibleActors` | Get actors in viewport frustum | — | Array of `Actor` refPaths |
| `CaptureViewport` | Screenshot level viewport | optional `captureTransform`, `annotations`, `bShowUI` | `{image, cameraLocation, cameraRotation, cameraFOV, grid, labeledActors}` |
| `CaptureAssetImage` | Render asset thumbnail | `assetPath` | `{mimeType, data}` (base64 PNG) |
| `CaptureEditorImage` | Screenshot entire editor | — | `{mimeType, data}` (base64 PNG) |
| `WorldPosToScreenCoords` | Convert world pos to screen | `position` (Vector) | `{x, y}` normalized screen coords |
| `ScreenCoordsToWorld` | Convert screen coords to world | `coords` (Vector2D), optional `traceDistance` | World position Vector |
| `SearchCVars` | Search console variables | `name` (substring) | JSON string of matching cvars |
| `StartPIE` | Start Play-In-Editor | `options` `{bSimulate, playMode, startTransform, warmupSeconds}` | — |
| `StopPIE` | Stop Play-In-Editor | — | — |
| `IsPIERunning` | Check if PIE is active | — | `true`/`false` |

**Notes:**
- `CaptureViewport` with `annotations` overlays a 3D grid + actor labels — gives spatial awareness.
- `StartPIE` completes after `PostPIEStarted` + `warmupSeconds` have elapsed.
- `FocusOnActors` cannot be called while PIE is active.
- Images are returned as base64-encoded PNG with `mimeType: "image/png"`.

### 8.14 SequencerTools

**Toolset:** `animation_toolset.toolsets.sequencer.SequencerTools`
**Purpose:** Core tools for creating and editing Level Sequences — lifecycle, playback, bindings, tracks, sections, folders.

| Category | Tool | Purpose |
|----------|------|---------|
| **Lifecycle** | `create_level_sequence` | Create new LevelSequence asset |
| | `open_sequence` | Open sequence in Sequencer editor |
| | `close_sequence` | Close current sequence editor |
| | `get_current_sequence` | Get root sequence currently open |
| | `get_focused_sequence` | Get focused sequence (may be sub-sequence) |
| **Playback** | `play` | Start playback |
| | `pause` | Pause playback |
| | `is_playing` | Check if playing |
| | `play_to` | Play to specific frame then stop |
| | `set_playhead_frame` / `get_playhead_frame` | Set/get playhead position |
| | `set_playback_speed` / `get_playback_speed` | Set/get speed multiplier |
| | `set_loop_mode` / `get_loop_mode` | Enable/disable looping |
| **Properties** | `set_display_rate` / `get_display_rate` | Set/get frame rate |
| | `set_tick_resolution` / `get_tick_resolution` | Set/get internal tick rate |
| | `set_playback_range` / `get_playback_range` | Set/get playback start/end frames |
| | `set_work_range` / `get_work_range` | Set/get work range in seconds |
| | `set_view_range` / `get_view_range` | Set/get visible timeline range |
| | `get_clock_source` / `set_clock_source` | Get/set clock source (TICK, PLATFORM, etc.) |
| | `get_evaluation_type` / `set_evaluation_type` | Get/set evaluation type |
| **Bindings** | `get_bindings` | List all bindings in sequence |
| | `find_binding_by_name` | Find binding by display name |
| | `add_actors` | Add level actors as bindings |
| | `add_actors_by_name` | Add actors by label name |
| | `add_spawnable_from_class` / `add_spawnable_from_instance` | Create spawnable bindings |
| | `create_camera` | Create cine camera actor |
| | `remove_binding` | Remove a binding |
| | `get_binding_name` / `set_binding_name` | Get/set binding display name |
| | `get_bound_objects` | Get resolved objects for binding |
| | `get_child_possessables` | Get component bindings under actor binding |
| **Tracks** | `get_tracks_on_binding` / `get_tracks_on_sequence` | List tracks |
| | `add_track_to_binding` / `add_track_to_sequence` | Add track |
| | `remove_track` / `remove_track_from_sequence` | Remove track |
| | `find_tracks_by_type` | Find tracks by class |
| **Sections** | `get_sections` | List sections on track |
| | `add_section` / `remove_section` | Add/remove section |
| | `get_section_range` / `set_section_range` | Get/set frame range |
| | `get_section_blend_type` / `set_section_blend_type` | Get/set blend (Absolute, Additive, etc.) |
| | `get_section_completion_mode` / `set_section_completion_mode` | Get/set completion (KeepState, RestoreState) |
| | `get_section_ease_in` / `set_section_ease_in` | Get/set ease-in frames |
| | `get_section_ease_out` / `set_section_ease_out` | Get/set ease-out frames |
| | `get_section_pre_roll_frames` / `set_section_pre_roll_frames` | Get/set pre-roll |
| | `get_section_post_roll_frames` / `set_section_post_roll_frames` | Get/set post-roll |
| **Folders** | `get_root_folders` | List root folders |
| | `add_root_folder` / `remove_root_folder` | Add/remove folder |
| | `add_binding_to_folder` / `add_track_to_folder` | Organize into folder |
| | `copy_folders` / `paste_folders` | Copy/paste folders |
| **Sub-sequences** | `focus_sub_sequence` | Navigate into sub-sequence |
| | `focus_parent_sequence` | Navigate up one level |
| | `get_sub_sequence_hierarchy` | Get current hierarchy path |
| **Tags** | `tag_binding` / `untag_binding` | Add/remove tag |
| | `get_binding_tags` | Get tags on binding |
| | `find_binding_by_tag` / `find_bindings_by_tag` | Find by tag |
| | `get_all_binding_tags` | List all registered tags |
| **Selection** | `select_bindings` / `get_selected_bindings` | Set/get binding selection |
| | `select_tracks` / `get_selected_tracks` | Set/get track selection |
| | `select_sections` / `get_selected_sections` | Set/get section selection |
| | `select_folders` / `get_selected_folders` | Set/get folder selection |
| | `empty_selection` | Clear all selection |
| **Utility** | `bake_transform` | Bake transforms for bindings |
| | `refresh_sequence` | Force UI refresh |
| | `force_evaluate` | Force viewport update |
| | `set_sequence_locked` / `is_sequence_locked` | Lock/unlock sequence |
| | `set_camera_lock` / `is_camera_cut_locked` | Lock camera to viewport |
| | `set_selection_range` / `get_selection_range` | Set/get green bar |
| | `get_marked_frames` / `add_marked_frame` / `delete_marked_frame` | Manage bookmarks |
| | `set_property_name_and_path` | Bind property track to UProperty |
| | `set_section_animation` | Assign AnimSequence to section |
| | `set_byte_track_enum` | Configure enum type for byte track |
| | `add_event_trigger_section` / `add_event_repeater_section` | Add event sections |

### 8.15 SequencerKeyframingTools

**Toolset:** `animation_toolset.toolsets.keyframing.SequencerKeyframingTools`
**Purpose:** Keyframing and animating properties on the Sequencer timeline.

| Tool | Purpose |
|------|---------|
| `get_channel_names` | List all channels on a section |
| `get_keys` | Get all keys on a channel (JSON array) |
| `get_keys_by_index` | Get specific keys by index |
| `add_key_float` | Add float key with interpolation |
| `add_key_bool` | Add bool key |
| `add_key_integer` | Add integer key |
| `add_key_string` | Add string key |
| `remove_key_at_frame` | Remove key at frame |
| `bake_channel_keys` | Evaluate channel at every frame in range |
| `get_default_value` / `set_default_value` | Get/set channel default |
| `select_channels` / `get_selected_channels` | Set/get channel selection |
| `open_curve_editor` / `close_curve_editor` | Open/close Curve Editor |
| `is_curve_editor_open` | Check if Curve Editor is open |
| `show_curve` / `is_curve_shown` | Show/hide curve in editor |
| `curve_editor_select_keys` / `get_curve_editor_selected_keys` | Select/get selected keys |
| `get_selected_key_channels` | Get channels with selected keys |
| `curve_editor_empty_selection` | Clear key selection |

**Notes:** Interpolation modes for float keys: `"cubic"`, `"linear"`, `"constant"`, `"break"`, or `""` for default (smart auto).

### 8.16 ControlRigTools

**Toolset:** `animation_toolset.toolsets.controlrig.ControlRigTools`
**Purpose:** Create and edit Control Rig assets — hierarchy, graphs, nodes, pins.

| Category | Tool | Purpose |
|----------|------|---------|
| **Hierarchy** | `create` | Create new Control Rig |
| | `add_bone` / `add_null` / `add_control` | Add hierarchy element |
| | `import_bones_from_asset` | Import bones from skeletal mesh |
| | `get_elements` / `get_all_bones` / `get_all_nulls` / `get_all_controls` | Query elements |
| | `get_parent` / `get_children` | Traverse hierarchy |
| | `get_local_transform` / `set_local_transform` | Get/set local transform |
| | `get_global_transform` / `set_global_transform` | Get/set global transform |
| **Graphs** | `list_graphs` | List all graphs |
| | `get_graph` | Get graph by name |
| | `get_forward_solve_graph` / `get_backward_solve_graph` / `get_interaction_graph` | Get specific graph |
| | `add_graph` / `add_event_graph` / `add_backward_solve_graph` / `add_interaction_graph` | Create graph |
| **Nodes** | `list_nodes` | List nodes in graph |
| | `create_node` | Create RigUnit node |
| | `delete_node` | Delete node |
| | `get_node_position` / `set_node_position` | Get/set node position |
| | `add_variable_node` | Create variable getter/setter |
| | `add_event_node` | Create event node (FORWARD_SOLVE, BACKWARD_SOLVE, INTERACTION) |
| **Pins** | `list_pins` | List pins on node |
| | `get_pin_value` / `set_pin_value` | Get/set pin default value |
| | `connect_pins` / `disconnect_pins` | Wire/unwire pins |
| | `get_connected_pins` | Get pins connected to a pin |
| **Variables** | `list_variables` | List member variables |
| | `get_variable` | Get variable by name |
| | `add_variable` | Add member variable |
| | `remove_variable` | Remove member variable |
| | `change_variable_type` | Change variable type |

**Notes:** Element types: `Bone`, `Null`, `Space`, `Control`, `Curve`, `Physics`, `Reference`, `Connector`, `Socket`. Event types: `FORWARD_SOLVE`, `BACKWARD_SOLVE`, `INTERACTION`.

### 8.17 NiagaraToolset_System

**Toolset:** `NiagaraToolsets.NiagaraToolset_System`
**Purpose:** Comprehensive Niagara System editing — system, emitter, module, renderer, dynamic input operations.

| Category | Tool | Purpose |
|----------|------|---------|
| **System** | `GetSystemSummary` | Lightweight metadata: name, user vars, per-emitter summary |
| | `GetSystemData` | Get all system property values |
| | `SetSystemData` | Set system property values |
| | `GetSystemSchema` | Get system property schema |
| | `GetSystemDependencies` | Rollup of used renderers, modules, data interfaces, dynamic inputs |
| | `GetSystemCompileState` | Compile status + per-script events |
| | `GetStackIssues` | All stack issues (errors, warnings, infos) |
| | `ApplyStackIssueFix` | Apply a Fix-style issue fix programmatically |
| | `AddUserVariables` | Add/update user variables on system |
| | `GetUserVariables` | Get all user variables on system |
| | `RemoveUserVariables` | Remove user variables |
| | `CreateNiagaraSystem` | Create new system from template |
| **Emitter** | `GetEmitterSummary` | Lightweight emitter metadata |
| | `GetEmitterData` | Full emitter property values |
| | `SetEmitterData` | Set emitter property values |
| | `GetEmitterSchema` | Emitter property schema |
| | `GetEmitterTopology` | Full emitter structure: 4 script stacks + renderers |
| | `GetEmitterInputValues` | All resolved input values across all stacks |
| | `AddEmitter` | Add emitter from template asset |
| | `RemoveEmitter` | Remove emitter |
| **Script Stack** | `GetScriptStackTopology` | All modules and inputs in execution order |
| | `GetScriptStackInputValues` | All resolved input values for a stack |
| **Module** | `GetModuleTopology` | Module metadata + all inputs |
| | `GetModuleSchema` | Module input schema with metadata |
| | `GetModuleSchemaFromAsset` | Schema from standalone module asset |
| | `GetModuleInputValues` | Resolved input values for one module |
| | `AddModule` | Add module to script stack |
| | `RemoveModule` | Remove module from stack |
| | `SetModuleEnabled` | Enable/disable module |
| | `AddSetParametersModule` | Add SetParameters module dynamically |
| | `AddSetParameterEntry` | Add parameter to existing SetParameters |
| | `RemoveSetParameterEntry` | Remove parameter from SetParameters |
| **Renderer** | `GetRendererData` | Get renderer property values |
| | `SetRendererData` | Set renderer property values |
| | `GetRendererSchema` | Get renderer property schema |
| | `AddRenderer` | Add renderer to emitter |
| | `RemoveRenderer` | Remove renderer from emitter |
| **Dynamic Input** | `GetDynamicInputChain` | Full recursive chain of dynamic inputs |
| | `GetDynamicInputSchema` | Schema for dynamic input in stack |
| | `GetDynamicInputSchemaFromAsset` | Schema from standalone asset |
| | `GetAvailableDynamicInputs` | List compatible dynamic input assets |
| **Stack Input** | `GetStackInputTopology` | Input metadata: name, type, visibility, editability |
| | `GetStackInputSchema` | Full input schema with metadata |
| | `GetStackInputData` | Get resolved input value |
| | `SetStackInputData` | Set input value |
| **Data Interface** | `GetDataInterfaceSchema` | Get data interface property schema |

**Notes:**
- Uses `FNiagaraExt_StackItemReference` to identify items: `{system, emitterName, scriptName, moduleName, rendererIndex, inputNameStack}`.
- Script stacks: `SystemSpawnScript`, `SystemUpdateScript`, `EmitterSpawnScript`, `EmitterUpdateScript`, `ParticleSpawnScript`, `ParticleUpdateScript`.
- `GetSystemSummary` is the best first call for unfamiliar systems.
- Compile state includes per-script events with severity (`Log`, `Display`, `Warning`, `Error`).
- Stack issues have `Fix` (programmatically applicable) and `Link` (navigation hint) styles.

### 8.18 NiagaraToolset_Component

**Toolset:** `NiagaraToolsets.NiagaraToolset_Component`
**Purpose:** Runtime Niagara component operations — user variable overrides, system assignment.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `GetUserVariables` | Get all user variable values on component | `component` | Array of `{name, type, value}` |
| `GetVariable` | Get specific user variable value | `component`, `var` | `{name, type, value}` |
| `SetVariable` | Override user variable on component | `component`, `variable` | — |
| `SetSystem` | Assign Niagara System to component | `niagaraComponent`, `system`, `bResetExistingOverrideParameters` | — |

**Notes:**
- This is the primary toolset for **runtime** Niagara operations.
- User variables are parameters exposed for external control, overridable per component instance.
- `SetSystem` properly initializes the component (preferred over setting `Asset` property directly).

### 8.19 NiagaraToolset_Assets

**Toolset:** `NiagaraToolsets.NiagaraToolset_Assets`
**Purpose:** Discovery of Niagara script assets via asset registry — no `LoadObject` required.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `GetAssetDiscoveryInfo` | Get project-configured discovery groups | — | Array of `{description, paths}` |
| `FindNiagaraScripts` | Search Niagara scripts by filters | `folderPath`, `name`, `usages`, `visibilities`, `moduleUsageBitmask`, `bRecursive`, `bIncludeDeprecated` | Array of `AssetData` |
| `GetNiagaraScriptDigest` | Get decoded registry tags for one script | `objectPath` | `{usage, libraryVisibility, moduleUsageBitmask, category, description, keywords, bDeprecated, bSuggested}` |

**Notes:**
- All functions read from asset registry without `LoadObject` — fast.
- Pair with `NiagaraToolset_System.AddModule` to drop discovered modules into a stack.
- `FAssetData` JSON serialization does not include `TagsAndValues` map — use `GetNiagaraScriptDigest` to decode metadata.
- Script usages: `Function`, `Module`, `DynamicInput`, `ParticleSpawnScript`, `ParticleUpdateScript`, etc.
- Library visibilities: `Invalid`, `Unexposed`, `Library` (= "Exposed" in editor), `Hidden`.

### 8.20 NiagaraToolset_Blueprint

**Toolset:** `NiagaraToolsets.NiagaraToolset_Blueprint`
**Purpose:** Create Blueprint wrappers around Niagara Systems for reusable FX actors.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `ConstructNiagaraBPWrapperFromSystem` | Create BP from Niagara System | `newAssetPath`, `system`, `parentClass` | New `Blueprint` refPath |
| `ConstructNiagaraBPWrapperFromComponent` | Create BP from configured component | `newAssetPath`, `component`, `parentClass` | New `Blueprint` refPath |

**Notes:** Naming convention: `NS_MyEffect` → `B_MyEffect`. Second tool preserves all component property values and user variable overrides.

---

## 9. Coverage Map & Status

### Toolsets Documented (24 total)

| # | Toolset | Status | Docs |
|---|---------|--------|------|
| 1 | `editor_toolset.toolsets.asset.AssetTools` | ✅ Exercised + Schema | Section 2 |
| 2 | `editor_toolset.toolsets.object.ObjectTools` | ✅ Exercised + Schema | Section 3 |
| 3 | `editor_toolset.toolsets.blueprint.BlueprintTools` | ✅ Exercised + Schema | Section 4 |
| 4 | `editor_toolset.toolsets.skeletal_mesh.SkeletalMeshTools` | ✅ Exercised + Schema | Section 5 |
| 5 | `editor_toolset.toolsets.actor.ActorTools` | Schema only | Section 8.1 |
| 6 | `editor_toolset.toolsets.scene.SceneTools` | Schema only | Section 8.2 |
| 7 | `editor_toolset.toolsets.material.MaterialTools` | Schema only | Section 8.3 |
| 8 | `editor_toolset.toolsets.material_instance.MaterialInstanceTools` | Schema only | Section 8.4 |
| 9 | `editor_toolset.toolsets.static_mesh.StaticMeshTools` | Schema only | Section 8.5 |
| 10 | `editor_toolset.toolsets.primitive.PrimitiveTools` | Schema only | Section 8.6 |
| 11 | `editor_toolset.toolsets.texture.TextureTools` | Schema only | Section 8.7 |
| 12 | `editor_toolset.toolsets.data_table.DataTableTools` | Schema only | Section 8.8 |
| 13 | `editor_toolset.toolsets.curve_table.CurveTableTools` | Schema only | Section 8.9 |
| 14 | `editor_toolset.toolsets.data_asset.DataAssetTools` | Schema only | Section 8.10 |
| 15 | `editor_toolset.toolsets.string_table.StringTableTools` | Schema only | Section 8.11 |
| 16 | `editor_toolset.toolsets.programmatic.ProgrammaticToolset` | Schema only | Section 8.12 |
| 17 | `EditorToolset.EditorAppToolset` | Schema only | Section 8.13 |
| 18 | `animation_toolset.toolsets.sequencer.SequencerTools` | Schema only | Section 8.14 |
| 19 | `animation_toolset.toolsets.keyframing.SequencerKeyframingTools` | Schema only | Section 8.15 |
| 20 | `animation_toolset.toolsets.controlrig.ControlRigTools` | Schema only | Section 8.16 |
| 21 | `NiagaraToolsets.NiagaraToolset_System` | Schema only | Section 8.17 |
| 22 | `NiagaraToolsets.NiagaraToolset_Component` | Schema only | Section 8.18 |
| 23 | `NiagaraToolsets.NiagaraToolset_Assets` | Schema only | Section 8.19 |
| 24 | `NiagaraToolsets.NiagaraToolset_Blueprint` | Schema only | Section 8.20 |

### Toolsets Not Yet Documented

| Toolset | Purpose |
|---------|---------|
| `ToolsetRegistry.AgentSkillToolset` | List, read, create/update skills |
| `EditorToolset.LogsToolset` | Read UE output log, control verbosity |
| `ConfigSettingsToolset.ConfigSettingsToolset` | List, inspect, edit config settings |
| `AutomationTestToolset.AutomationTestToolset` | Discover and run automation tests |
| `DataRegistryToolset.DataRegistryTools` | Query and inspect Data Registries |
| `PCGToolset.PCGToolset` | Build and modify PCG graphs |
| `PCGToolset.PCGSpatialToolset` | PCG spatial operations |
| `DataflowAgent.DataflowAgentToolset` | Dataflow graph editing |
| `NiagaraToolsets.NiagaraToolset_Info` | Niagara enum lookups, guidance |
| `GameplayTagsToolset.GameplayTagsToolset` | Read/manage gameplay tags |
| `PhysicsToolsets.PhysicsAssetToolset` | Create/manage Physics Assets |
| `GameFeaturesToolset.GameFeaturesToolset` | List/activate/deactivate Game Features |
| `PluginToolset.PluginToolset` | Create/edit/enable/query plugins |
| `SemanticSearchToolset.SemanticSearchToolset` | Hybrid vector + BM25 asset search |
| `SlateInspectorToolset.SlateInspectorToolset` | Playwright-style Slate UI automation |
| `GASToolsets.GameplayCueToolset` | Inspect/execute/manage gameplay cues |
| `GASToolsets.AttributeSetToolset` | Discover AttributeSet classes and attributes |
| `GASToolsets.AbilitySystemInspectorToolset` | Inspect runtime AbilitySystemComponent state |
| `WorldConditionsToolset.WorldConditionTools` | Inspect WorldCondition structs |
| `UMGToolSet.UMGToolSet` | UMG widget creation and tree manipulation |
| `state_tree_toolset.toolsets.state_tree.StateTreeTools` | Inspect StateTree assets |
| `animation_toolset.toolsets.sequencer.SequencerControlRigTools` | Control Rig animation in Sequencer |
| `animation_toolset.toolsets.outliner.SequencerOutlinerTools` | Sequencer outliner inspection |
| `animation_toolset.toolsets.conditions.SequencerConditionTools` | Runtime conditions on Sequencer tracks |
| `animation_toolset.toolsets.custom_bindings.SequencerCustomBindingTools` | Custom binding type management |
| `animation_toolset.toolsets.import_export.SequencerImportExportTools` | FBX import/export for Sequencer |
| `conversation_toolset.toolsets.conversation.ConversationTools` | Inspect Conversation Graph assets |
| `editor_toolset.toolsets.string_table.StringTableTools` | Create/edit StringTable assets |
| `aimodule_toolset.toolsets.behavior_tree.BehaviorTreeTools` | Inspect BehaviorTree assets |

---

*Document generated from live Unreal MCP sessions on UE 5.8 (MyProject). Pass 1: exercised tools. Pass 2: described schemas. Toolsets marked "Not Yet Documented" remain for future passes.*

### 8.21 LogsToolset

**Toolset:** `EditorToolset.LogsToolset`
**Purpose:** Read the UE output log and control log category verbosity.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `GetLogEntries` | Get log entries from current session | optional `category`, `pattern` (regex), `maxEntries` | Array of log entry strings |
| `GetLogCategories` | List registered log categories | optional `filter` (substring) | Array of category name strings |
| `GetVerbosity` | Get verbosity for a category | optional `category` | Verbose level string |
| `SetVerbosity` | Set verbosity for a category | `verbosity`, optional `category` | — |

**Notes:** Verbosity levels: `NoLogging`, `Fatal`, `Error`, `Warning`, `Display`, `Log`, `Verbose`, `VeryVerbose`. Default category is `LogsToolset`.

### 8.22 ConfigSettingsToolset

**Toolset:** `ConfigSettingsToolset.ConfigSettingsToolset`
**Purpose:** List, inspect, and edit Config Settings sections (Project Settings, Editor Settings).

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `ListContainers` | List all containers | — | Array of container names (e.g. `Project`, `Editor`) |
| `ListCategories` | List categories in a container | `containerName` | Array of category names |
| `ListSections` | List sections in a category | `containerName`, `categoryName` | Array of section names |
| `GetSectionSchema` | Get JSON Schema for a section | `containerName`, `categoryName`, `sectionName` | JSON Schema string |
| `GetSectionPropertyValues` | Read property values | `containerName`, `categoryName`, `sectionName`, `propertyNames` | JSON object of values |
| `SetSectionProperties` | Set property values + save | `containerName`, `categoryName`, `sectionName`, `propertiesJson` | `true`/`false` |
| `SaveSection` | Save a section to disk | `containerName`, `categoryName`, `sectionName` | `true`/`false` |
| `ResetSectionToDefaults` | Reset section to defaults | `containerName`, `categoryName`, `sectionName` | `true`/`false` |

**Notes:** Hierarchical navigation: Container → Category → Section → Properties. Common containers: `Project`, `Editor`.

### 8.23 DataRegistryTools

**Toolset:** `DataRegistryToolset.DataRegistryTools`
**Purpose:** Query and inspect Data Registries — structured data stores.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `ListRegistries` | List all registered Data Registries | optional `structFilter` | Array of registry name strings |
| `GetRegistryInfo` | Get detailed info about a registry | `registryName` | `{registryName, description, itemStruct, itemCount, availability, idFormat}` |
| `GetSchema` | Get item struct schema as JSON | `registryName` | JSON schema string |
| `ListItems` | List all item names in registry | `registryName` | Array of item name strings |
| `GetItems` | Get cached item data | `registryName`, `itemNames` | Map of item name → struct data |
| `ListDataSources` | Get editor-defined sources | `registryName` | Array of source summaries |
| `ListRuntimeSources` | Get runtime sources (incl. transient) | `registryName` | Array of source summaries |

**Notes:** Availability levels: `DoesNotExist`, `Unknown`, `Remote`, `OnDisk`, `LocalAsset`, `PreCached`.

### 8.24 GameplayTagsToolset

**Toolset:** `GameplayTagsToolset.GameplayTagsToolset`
**Purpose:** Read and manage gameplay tags in the GameplayTagsManager.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `ListTags` | List registered tags | optional `parentTag` (hierarchy filter) | Array of fully-qualified tag names |
| `GetTagInfo` | Get tag details | `tagName` | `{comment, source, children}` |
| `FindReferencersByTag` | Find assets referencing a tag | `tagName` | Array of package path strings |
| `AddTag` | Add new tag (user permission required) | `tagName`, optional `comment`, `tagSource` | — |
| `RemoveTag` | Remove tag (user permission required) | `tagName` | — |
| `RenameTag` | Rename tag + update refs (user permission required) | `oldTagName`, `newTagName` | — |

**Notes:** Tags are hierarchical (e.g. `Character.State.Dead`). Source is typically `DefaultGameplayTags.ini`.

### 8.25 AutomationTestToolset

**Toolset:** `AutomationTestToolset.AutomationTestToolset`
**Purpose:** Discover and run automation tests (same backend as Session Frontend).

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `DiscoverTests` | Initialize test discovery | optional `bForceRediscover` | JSON status (async) |
| `ListTests` | List available tests | optional `nameFilter`, `tagFilter`, `limit` | JSON `{tests, total, returned}` |
| `RunTests` | Run tests by name | `testNames` array | JSON summary (async) |
| `RunTestsByFilter` | Run tests by filter expression | `filterExpression` | JSON summary (async) |
| `GetTestStatus` | Get controller status snapshot | — | JSON status object |
| `GetTestResults` | Get detailed test results | — | JSON with per-test state, duration, errors |
| `StopTests` | Stop running tests | — | `true`/`false` |

**Notes:** Filter syntax: `StartsWith:Path`, `^Prefix`, `Suffix$`, `Substring`, `Group:Name`. Must call `DiscoverTests` first.

### 8.26 PluginToolset

**Toolset:** `PluginToolset.PluginToolset`
**Purpose:** Create, edit, enable, and query Unreal plugins.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `ListEnabledPlugins` | List enabled plugins | — | Array of plugin name strings |
| `ListDiscoveredPlugins` | List all plugins (enabled + disabled) | — | Array of plugin name strings |
| `IsEnabled` | Check if plugin is enabled | `pluginName` | `true`/`false` |
| `SetPluginEnabled` | Enable/disable plugin (restart required) | `pluginName`, `bEnabled` | — |
| `GetPluginInfo` | Get plugin metadata | `pluginName` | `{description, version, versionName, baseDir, contentDir, descriptorPath, mountedAssetPath}` |
| `GetPluginDescriptor` | Get editable descriptor fields | `pluginName` | Descriptor object |
| `UpdatePluginDescriptor` | Update descriptor fields | `pluginName`, `newDescriptor` | — |
| `GetPluginDependencies` | Get plugin's dependencies | `pluginName` | Array of `{name, bOptional, bEnabled}` |
| `GetPluginDependents` | Get plugins depending on this one | `pluginName` | Array of plugin name strings |
| `AddPluginDependency` | Add dependency | `pluginName`, `dependencyName`, `bOptional`, `bEnabled` | — |
| `RemovePluginDependency` | Remove dependency | `pluginName`, `dependencyName` | — |
| `GetPluginForAsset` | Find plugin owning an asset | `assetPath` | Plugin name string |
| `GetPluginTemplateDescriptions` | List available templates | — | Array of template descriptors |
| `CreatePlugin` | Create plugin from template | `pluginName`, `templateInfo`, `description`, etc. | Descriptor filename |
| `ValidateNewPluginNameAndLocation` | Validate plugin name/location | `pluginName`, `relativePluginLocation`, `bPlaceInEngine`, `templateInfo` | `true`/`false` |
| `IsPluginCreationAllowed` | Check if creation is permitted | — | `true`/`false` |
| `IsPluginModificationAllowed` | Check if modification is permitted | — | `true`/`false` |

**Notes:** `SetPluginEnabled` takes effect on next editor restart.

### 8.27 SemanticSearchToolset

**Toolset:** `SemanticSearchToolset.SemanticSearchToolset`
**Purpose:** Hybrid vector + BM25 asset search via the SemanticSearch plugin.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `Search` | Semantic search over indexed assets | `query`, optional `classFilter`, `pathRegexes`, `k` | Array of `{path, class, caption}` sorted by relevance |
| `FindSimilar` | Find assets similar to a reference asset | `assetPath`, optional `classFilter`, `pathRegexes`, `k` | Array of `{path, class, caption}` |

**Notes:** Indexed base classes: `Blueprint`, `StaticMesh`, `SkeletalMesh`, `Texture`, `Material`, `MaterialInstance`. Vector-only for `FindSimilar` (no BM25).

### 8.28 SlateInspectorToolset

**Toolset:** `SlateInspectorToolset.SlateInspectorToolset`
**Purpose:** Playwright-style Slate UI automation for driving the Unreal Editor UI programmatically.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `Snapshot` | Accessibility snapshot of widget tree | optional `ref`, `maxDepth`, `bIncludeSourceLocations` | JSON string of widget tree |
| `Screenshot` | Screenshot a Slate widget or active window | optional `ref` | `{mimeType, data}` (base64 PNG) |
| `Click` | Click a widget | `ref`, optional `button`, `doubleClick`, `modifiers` | `true`/`false` |
| `Type` | Type text into a textbox | `ref`, `text`, optional `submit` | `true`/`false` |
| `Hover` | Hover over a widget | `ref` | `true`/`false` |
| `PressKey` | Press a keyboard key | `key` (e.g. `Enter`, `Ctrl+A`) | `true`/`false` |
| `SelectOption` | Select combobox option | `ref`, `value` | `true`/`false` |
| `FillForm` | Fill multiple form fields | `fields` array of `{ref, value, fieldType}` | `true`/`false` |
| `Drag` | Drag from one widget to another | `startRef`, `endRef`, optional `modifiers` | `true`/`false` |
| `Observe` | Register observer for deep widget coverage | `ref`, optional `maxDepth` | Observer identifier string |
| `Unobserve` | Remove an observer | `identifier` | `true`/`false` |
| `ListObservers` | List active observers | — | JSON array of observer info |
| `Windows` | List/select/close editor windows | optional `action`, `index` | JSON string |
| `WaitFor` | Check if text is present/absent | `text`, `textGone` | `true`/`false` |

**Notes:** Shallow root observer (depth 0) covers top-level windows. Call `Observe()` on a specific window/panel for deep coverage. Input simulation uses direct Slate event APIs (not AutomationDriver which deadlocks on game thread).

### 8.29 GameplayCueToolset

**Toolset:** `GASToolsets.GameplayCueToolset`
**Purpose:** Inspect, execute, and manage gameplay cues and their notify assets.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `ListCues` | List registered gameplay cue tags | optional `parentTag` | Array of fully-qualified cue tag names |
| `GetCueInfo` | Get cue details | `cueTag` | `{tag, notifyAssetPath, notifyType}` |
| `FindCueNotifyAssets` | Find all GCN assets | optional `parentTag` | Array of `{cueTag, assetPath, assetName, notifyType}` |
| `FindCueTagsWithoutNotifies` | Find orphaned cue tags | — | Array of tag names with no notify |
| `ExecuteCueOnSelectedActor` | Preview cue on selected actor | `cueTag`, `normalizedMagnitude`, `location`, `normal` | `true`/`false` |
| `AddCueTag` | Add new cue tag (user permission required) | `cueTag`, optional `comment` | `true`/`false` |
| `RemoveCueTag` | Remove cue tag (user permission required) | `cueTag` | `true`/`false` |
| `CreateCueNotifyAsset` | Create GCN Blueprint (user permission required) | `cueTag`, `packagePath`, `assetName`, `bIsActor` | Asset path string |

**Notes:** Cue tags must begin with `GameplayCue.`. Notify types: `Static` (instant effect), `Actor` (spawned world actor). `ExecuteCueOnSelectedActor` requires PIE session or configured GameplayCueManager.

### 8.30 AttributeSetToolset

**Toolset:** `GASToolsets.AttributeSetToolset`
**Purpose:** Discover AttributeSet classes and their gameplay attributes (GAS).

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `FindAttributeSetClasses` | Find all AttributeSet subclasses | — | Array of `{className, assetPath, attributes[]}` |
| `ListAttributes` | List attributes on a specific class | `className` | Array of `{attributeName, fullName, setClassName}` |

**Notes:** Covers both native C++ and Blueprint AttributeSet subclasses. Full name format: `UMySetClass.AttributeName`.

### 8.31 AbilitySystemInspectorToolset

**Toolset:** `GASToolsets.AbilitySystemInspectorToolset`
**Purpose:** Inspect runtime state of an AbilitySystemComponent on an actor.

| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| `GetAttributeValues` | Get base + current attribute values | `actor` | Array of `{attributeName, fullName, setClassName, baseValue, currentValue}` |
| `GetGrantedAbilities` | Get all granted abilities | `actor` | Array of `{abilityName, level, bIsActive}` |
| `GetActiveEffects` | Get all active gameplay effects | `actor` | Array of `{effectName, stackCount, totalDuration, remainingDuration, grantedTags[]}` |
| `GetActiveTags` | Get all owned gameplay tags | `actor` | Array of tag name strings |

**Notes:** Each function takes a direct actor pointer. Raises script error if actor is null or has no ASC. Duration of `-1` means infinite.
