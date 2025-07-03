# API Documentation

## Core Classes

### FlashdotPlugin
Main plugin entry point and coordinator.

```gdscript
class_name FlashdotPlugin
extends EditorPlugin

# Initialize all subsystems
func _enter_tree() -> void

# Cleanup on plugin disable
func _exit_tree() -> void

# Get reference to timeline editor
func get_timeline_editor() -> TimelineEditor

# Get reference to vector tools
func get_vector_tools() -> VectorToolManager
```

### TimelineEditor
Central timeline management and UI.

```gdscript
class_name TimelineEditor
extends Control

signal frame_changed(frame_number: int)
signal layer_selected(layer_index: int)
signal keyframe_added(frame: int, layer: int)

# Current playhead position
@export var current_frame: int

# Timeline playback controls
func play() -> void
func pause() -> void
func stop() -> void
func goto_frame(frame: int) -> void

# Layer management
func add_layer(name: String) -> TimelineLayer
func remove_layer(index: int) -> void
func get_layer(index: int) -> TimelineLayer
```

### VectorShape
Base class for all drawable vector elements.

```gdscript
class_name VectorShape
extends Resource

@export var stroke_color: Color = Color.BLACK
@export var stroke_width: float = 1.0
@export var fill_color: Color = Color.TRANSPARENT
@export var transform: Transform2D

# Render the shape to a canvas
func draw(canvas: CanvasItem) -> void

# Get bounding rectangle
func get_bounds() -> Rect2

# Hit testing for selection
func contains_point(point: Vector2) -> bool
```

### Symbol
Reusable animation asset with multiple layers.

```gdscript
class_name Symbol
extends Resource

@export var name: String
@export var layers: Array[SymbolLayer]

# Create instance in scene
func instantiate() -> SymbolInstance

# Get all shapes across layers
func get_all_shapes() -> Array[VectorShape]

# Timeline duration
func get_duration() -> int
```

## Tool System

### FlashdotTool
Base interface for all drawing and editing tools.

```gdscript
class_name FlashdotTool
extends RefCounted

signal tool_activated
signal tool_deactivated

# Tool metadata
@export var tool_name: String
@export var tool_icon: Texture2D
@export var keyboard_shortcut: String

# Tool lifecycle
func activate() -> void
func deactivate() -> void

# Input handling
func handle_input(event: InputEvent) -> bool
func handle_mouse_input(event: InputEventMouse) -> bool
func handle_key_input(event: InputEventKey) -> bool
```

### Drawing Tools

#### PenTool
```gdscript
class_name PenTool
extends FlashdotTool

# Current stroke being drawn
var current_stroke: VectorPath

# Drawing state
enum DrawState { IDLE, DRAWING, EDITING_POINTS }
var state: DrawState = DrawState.IDLE

func start_stroke(position: Vector2) -> void
func add_point(position: Vector2) -> void
func finish_stroke() -> void
```

#### SelectionTool
```gdscript
class_name SelectionTool
extends FlashdotTool

# Currently selected shapes
var selected_shapes: Array[VectorShape]

# Selection operations
func select_shape(shape: VectorShape) -> void
func deselect_shape(shape: VectorShape) -> void
func clear_selection() -> void
func get_selection_bounds() -> Rect2
```

## Event System

### TimelineEvents
Global event bus for timeline-related changes.

```gdscript
class_name TimelineEvents
extends RefCounted

# Frame navigation
signal frame_changed(frame_number: int)
signal playback_started()
signal playback_stopped()

# Layer events
signal layer_added(layer: TimelineLayer)
signal layer_removed(layer_index: int)
signal layer_visibility_changed(layer_index: int, visible: bool)

# Keyframe events
signal keyframe_created(frame: int, layer: int)
signal keyframe_deleted(frame: int, layer: int)
signal keyframe_moved(old_frame: int, new_frame: int, layer: int)

# Get singleton instance
static func get_instance() -> TimelineEvents
```

### DrawingEvents
Events for vector drawing operations.

```gdscript
class_name DrawingEvents
extends RefCounted

# Shape operations
signal shape_created(shape: VectorShape)
signal shape_deleted(shape: VectorShape)
signal shape_modified(shape: VectorShape)

# Selection events
signal selection_changed(shapes: Array[VectorShape])
signal shapes_transformed(shapes: Array[VectorShape], transform: Transform2D)

static func get_instance() -> DrawingEvents
```

## Export System

### ExportManager
Handles export to various formats.

```gdscript
class_name ExportManager
extends RefCounted

# Export to Godot scene
func export_to_scene(timeline: Timeline, path: String) -> Error

# Export to HTML5
func export_to_html5(timeline: Timeline, path: String) -> Error

# Export animation frames
func export_frames(timeline: Timeline, format: String, path: String) -> Error

# Register custom exporter
func register_exporter(format: String, exporter: FlashdotExporter) -> void
```

### FlashdotExporter
Base class for custom export formats.

```gdscript
class_name FlashdotExporter
extends RefCounted

# Export implementation
func export_timeline(timeline: Timeline, path: String) -> Error

# Format metadata
func get_format_name() -> String
func get_file_extension() -> String
func get_export_options() -> Dictionary
```

## Plugin Extension API

### ToolRegistry
Register custom tools with the editor.

```gdscript
class_name ToolRegistry
extends RefCounted

# Register new tool
static func register_tool(tool: FlashdotTool) -> void

# Get all registered tools
static func get_tools() -> Array[FlashdotTool]

# Get tool by name
static func get_tool(name: String) -> FlashdotTool
```

### MenuExtension
Add custom menu items to Flashdot.

```gdscript
class_name MenuExtension
extends RefCounted

# Add menu item
static func add_menu_item(path: String, callback: Callable) -> void

# Add separator
static func add_separator(menu_path: String) -> void

# Remove menu item
static func remove_menu_item(path: String) -> void
```

## Example Usage

### Creating a Custom Tool
```gdscript
extends FlashdotTool

func _init():
	tool_name = "Circle Tool"
	keyboard_shortcut = "C"

func handle_mouse_input(event: InputEventMouse) -> bool:
	if event is InputEventMouseButton and event.pressed:
		var circle = CircleShape.new()
		circle.center = event.position
		DrawingEvents.get_instance().shape_created.emit(circle)
		return true
	return false
```

### Listening to Timeline Events
```gdscript
extends Control

func _ready():
	TimelineEvents.get_instance().frame_changed.connect(_on_frame_changed)

func _on_frame_changed(frame_number: int):
	print("Timeline moved to frame: ", frame_number)
```

### Custom Export Format
```gdscript
extends FlashdotExporter

func get_format_name() -> String:
	return "Custom Animation Format"

func export_timeline(timeline: Timeline, path: String) -> Error:
	# Implementation here
	return OK
```

---
_Last updated: July 3, 2025_
