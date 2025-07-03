# Best Practices for Godot 4.x

A condensed guide to maintainable, performant, and scalable projects in Godot Engine 4.x.

## 1. Project Structure
- Organize content into folders:  
  - `scenes/`  
  - `scripts/`  
  - `assets/` (textures, audio, fonts)  
  - `addons/` (custom plugins)  
- Use consistent naming and avoid deeply nested paths.

## 2. Scene & Node Organization
- One scene = one responsibility (e.g., Player, UI, Enemy).  
- Use meaningful root node names.  
- Leverage instancing for reuse and memory efficiency.

## 3. Naming Conventions
- Scripts & classes: `PascalCase` with `class_name`.  
- File names & resources: `snake_case`.  
- Node names: descriptive (e.g., `PlayerController`, `HealthBar`).

## 4. GDScript Tips
- Enable static typing.  
- Use `@export` for inspector variables.  
- Favor `preload()` for critical resources and `load()` for on-demand.

## 5. Signals & Events
- Define custom signals in scripts.  
- Emit and connect signals in code or via the editor.  
- Use signals for loose coupling between nodes.

## 6. Performance Optimization
- Use `_physics_process()` vs `_process()` appropriately.  
- Avoid per-frame allocations; reuse objects.  
- Batch draw calls with `MultiMeshInstance3D` or `MultiMeshInstance2D`.

## 7. UI & Theming
- Use `Control` nodes with anchors and margins for responsive layouts.  
- Centralize style with `Theme` resources.  
- Follow the core palette: `#2EDBF3`, `#4565FF`, `#B94AF1`.

## 8. Version Control
- Add `.godot/` and `.import/` to `.gitignore`.  
- Commit scenes, scripts, and assets only.  
- Use `.tscn` over binary `.scn` format for diff-friendly merges.

## 9. Plugins & Add-ons
- Place custom tools in `addons/` and register in `project.godot`.  
- Document entry points and settings in `plugin.cfg`.

## 10. Testing & Debugging
- Use the built-in debugger and output profiler.  
- Write simple automated tests with Godot’s `UnitTest` add-on or custom scripts.

## 11. Documentation & Comments
- Establish doc-string conventions for GDScript (e.g. use triple-quotes and tags like `@param`/`@return`)
- Keep scene and script headers documenting their purpose and version
- Comment complex logic and signal connections
- Generate external docs with a tool like DocTools or custom scripts

_Last updated: July 3, 2025_
