# Plugin Development Guide

## Overview
Flashdot provides a robust plugin system that allows developers to extend the editor with custom tools, exporters, and UI components. This guide covers everything needed to create powerful extensions for the Flashdot ecosystem.

## Plugin Architecture

### Plugin Structure
```
your_plugin/
├── plugin.cfg           # Plugin metadata
├── plugin.gd           # Main plugin entry point
├── tools/              # Custom drawing tools
├── exporters/          # Custom export formats
├── ui/                 # Custom UI components
├── assets/             # Plugin-specific assets
└── README.md           # Plugin documentation
```

### Plugin Configuration (`plugin.cfg`)
```ini
[plugin]

name="Custom Brush Tools"
description="Advanced brush tools for vector drawing"
author="Your Name"
version="1.0"
script="plugin.gd"
```

### Main Plugin Class (`plugin.gd`)
```gdscript
@tool
extends EditorPlugin

const CustomBrushTool = preload("res://addons/custom_brushes/tools/brush_tool.gd")

func _enter_tree():
	# Register custom tools
	FlashdotAPI.register_tool(CustomBrushTool.new())
	
	# Add custom UI dock
	add_control_to_dock(DOCK_SLOT_LEFT_UL, preload("res://addons/custom_brushes/ui/brush_panel.tscn").instantiate())

func _exit_tree():
	# Cleanup when plugin disabled
	FlashdotAPI.unregister_tool("custom_brush")
	remove_control_from_docks(brush_panel)
```

## Creating Custom Tools

### Basic Tool Template
```gdscript
@tool
extends FlashdotTool
class_name CustomBrushTool

var brush_size: float = 10.0
var brush_opacity: float = 1.0
var current_stroke: VectorPath

func _init():
	tool_name = "Custom Brush"
	tool_icon = preload("res://addons/custom_brushes/icons/brush.svg")
	keyboard_shortcut = "B"

func activate():
	super.activate()
	# Tool-specific initialization
	_setup_brush_cursor()

func deactivate():
	super.deactivate()
	# Cleanup
	_finish_current_stroke()

func handle_mouse_input(event: InputEventMouse) -> bool:
	if event is InputEventMouseButton:
		if event.pressed and event.button_index == MOUSE_BUTTON_LEFT:
			_start_stroke(event.position)
			return true
		elif not event.pressed and event.button_index == MOUSE_BUTTON_LEFT:
			_finish_stroke()
			return true
			
	elif event is InputEventMouseMotion and current_stroke:
		_add_stroke_point(event.position)
		return true
		
	return false

func _start_stroke(position: Vector2):
	current_stroke = VectorPath.new()
	current_stroke.stroke_width = brush_size
	current_stroke.stroke_color = get_current_color()
	current_stroke.add_point(position)

func _add_stroke_point(position: Vector2):
	if current_stroke:
		current_stroke.add_point(position)
		# Emit real-time drawing events
		FlashdotEvents.stroke_updated.emit(current_stroke)

func _finish_stroke():
	if current_stroke and current_stroke.points.size() > 1:
		# Add to current layer
		var layer = FlashdotAPI.get_current_layer()
		layer.add_shape(current_stroke)
		
		# Emit completion event
		FlashdotEvents.shape_created.emit(current_stroke)
		
		# Create undo/redo command
		var command = AddShapeCommand.new(layer, current_stroke)
		FlashdotAPI.add_undo_command(command)
		
	current_stroke = null
```

### Advanced Tool Features

#### Tool Properties Panel
```gdscript
# Custom tool properties UI
@tool
extends Control
class_name BrushPropertiesPanel

@onready var size_slider: HSlider = $VBox/SizeSlider
@onready var opacity_slider: HSlider = $VBox/OpacitySlider

var target_tool: CustomBrushTool

func _ready():
	size_slider.value_changed.connect(_on_size_changed)
	opacity_slider.value_changed.connect(_on_opacity_changed)

func set_tool(tool: CustomBrushTool):
	target_tool = tool
	size_slider.value = tool.brush_size
	opacity_slider.value = tool.brush_opacity

func _on_size_changed(value: float):
	if target_tool:
		target_tool.brush_size = value

func _on_opacity_changed(value: float):
	if target_tool:
		target_tool.brush_opacity = value
```

#### Pressure Sensitivity Support
```gdscript
func handle_mouse_input(event: InputEventMouse) -> bool:
	# Check for tablet/stylus input
	if event is InputEventMouseMotion and has_tablet_data(event):
		var pressure = get_tablet_pressure(event)
		var dynamic_size = brush_size * pressure
		_add_stroke_point_with_pressure(event.position, pressure)
		return true
	
	return super.handle_mouse_input(event)

func _add_stroke_point_with_pressure(position: Vector2, pressure: float):
	if current_stroke:
		var point = VectorPoint.new()
		point.position = position
		point.pressure = pressure
		point.width = brush_size * pressure
		current_stroke.add_point_advanced(point)
```

## Custom Export Formats

### Exporter Template
```gdscript
@tool
extends FlashdotExporter
class_name GIFExporter

func get_format_name() -> String:
	return "Animated GIF"

func get_file_extension() -> String:
	return "gif"

func get_export_options() -> Dictionary:
	return {
		"frame_rate": 12,
		"loop_count": 0,
		"quality": "medium",
		"dither": true
	}

func export_timeline(timeline: Timeline, path: String, options: Dictionary) -> Error:
	var frames := []
	var frame_rate = options.get("frame_rate", 12)
	
	# Render each frame
	for frame_num in range(timeline.get_duration()):
		timeline.goto_frame(frame_num)
		var frame_image = _render_frame(timeline)
		frames.append(frame_image)
	
	# Create GIF from frames
	return _create_gif_file(frames, path, options)

func _render_frame(timeline: Timeline) -> Image:
	var viewport = SubViewport.new()
	viewport.size = Vector2i(800, 600)
	
	# Render all visible layers
	for layer in timeline.get_visible_layers():
		for shape in layer.shapes:
			shape.draw_to_viewport(viewport)
	
	var image = viewport.get_texture().get_image()
	viewport.queue_free()
	return image

func _create_gif_file(frames: Array[Image], path: String, options: Dictionary) -> Error:
	# Implementation using Godot's ImageWriter or external library
	# This would require additional GIF encoding logic
	return OK
```

### Registering Custom Exporters
```gdscript
# In plugin's _enter_tree()
func _enter_tree():
	FlashdotAPI.register_exporter(GIFExporter.new())
	FlashdotAPI.register_exporter(SpriteSheetExporter.new())
	FlashdotAPI.register_exporter(LottieExporter.new())
```

## UI Extension Points

### Custom Dock Panels
```gdscript
@tool
extends Control
class_name SymbolBrowserDock

@onready var symbol_tree: Tree = $VBox/SymbolTree
@onready var preview_rect: TextureRect = $VBox/PreviewRect

func _ready():
	_populate_symbol_tree()
	symbol_tree.item_selected.connect(_on_symbol_selected)

func _populate_symbol_tree():
	symbol_tree.clear()
	var root = symbol_tree.create_item()
	
	for symbol in FlashdotAPI.get_symbol_library().get_all_symbols():
		var item = symbol_tree.create_item(root)
		item.set_text(0, symbol.name)
		item.set_metadata(0, symbol)

func _on_symbol_selected():
	var selected = symbol_tree.get_selected()
	if selected:
		var symbol = selected.get_metadata(0) as Symbol
		preview_rect.texture = symbol.get_preview_texture()
```

### Custom Menu Items
```gdscript
# Add custom menu items
func _enter_tree():
	add_tool_menu_item("Generate Sprite Sheet", _generate_sprite_sheet)
	add_tool_menu_item("Import Flash SWF", _import_swf_file)

func _generate_sprite_sheet():
	var dialog = AcceptDialog.new()
	dialog.dialog_text = "Sprite sheet generation started..."
	get_editor_interface().get_base_control().add_child(dialog)
	dialog.popup_centered()

func _import_swf_file():
	var file_dialog = FileDialog.new()
	file_dialog.file_mode = FileDialog.FILE_MODE_OPEN_FILE
	file_dialog.add_filter("*.swf", "Flash SWF Files")
	file_dialog.file_selected.connect(_on_swf_selected)
	get_editor_interface().get_base_control().add_child(file_dialog)
	file_dialog.popup_centered_ratio(0.75)
```

## Event System Integration

### Listening to Core Events
```gdscript
func _enter_tree():
	# Connect to timeline events
	FlashdotEvents.frame_changed.connect(_on_frame_changed)
	FlashdotEvents.layer_added.connect(_on_layer_added)
	FlashdotEvents.shape_created.connect(_on_shape_created)

func _on_frame_changed(frame_number: int):
	# Update plugin UI based on timeline position
	update_frame_dependent_ui(frame_number)

func _on_shape_created(shape: VectorShape):
	# Automatically apply plugin-specific properties
	if shape is VectorPath:
		apply_custom_stroke_style(shape)
```

### Creating Custom Events
```gdscript
# Define plugin-specific signals
signal custom_effect_applied(effect_name: String, target_shapes: Array)
signal batch_operation_completed(operation_type: String, affected_count: int)

# Emit custom events
func apply_glow_effect(shapes: Array[VectorShape]):
	for shape in shapes:
		shape.add_effect(GlowEffect.new())
	
	custom_effect_applied.emit("glow", shapes)
```

## Plugin Distribution

### Package Structure
```
your_plugin_v1.0.zip
├── addons/
│   └── your_plugin/
│       ├── plugin.cfg
│       ├── plugin.gd
│       └── ... (plugin files)
├── README.md
├── LICENSE
└── CHANGELOG.md
```

### Plugin Marketplace Submission
```gdscript
# Include in plugin.cfg for marketplace
[plugin]
name="Advanced Vector Tools"
description="Professional-grade vector editing tools for Flashdot"
author="Your Studio"
version="1.2.0"
script="plugin.gd"

# Marketplace metadata
[marketplace]
category="Tools"
tags=["vector", "drawing", "animation"]
min_flashdot_version="1.0"
website="https://your-site.com"
repository="https://github.com/user/flashdot-plugin"
```

### Version Compatibility
```gdscript
# Check Flashdot version compatibility
func _enter_tree():
	var flashdot_version = FlashdotAPI.get_version()
	if not _is_compatible_version(flashdot_version):
		push_error("Plugin requires Flashdot 1.2+ but found " + flashdot_version)
		return
	
	# Continue with plugin initialization
	_initialize_plugin()

func _is_compatible_version(version: String) -> bool:
	var parts = version.split(".")
	var major = int(parts[0])
	var minor = int(parts[1])
	return major >= 1 and minor >= 2
```

## Best Practices

### Performance Optimization
- **Lazy Loading**: Load heavy resources only when needed
- **Object Pooling**: Reuse temporary objects in drawing operations
- **Batch Operations**: Group multiple shape operations for efficiency
- **Event Throttling**: Limit high-frequency event emissions

### Error Handling
```gdscript
func safe_tool_operation():
	try:
		var result = risky_vector_operation()
		return result
	catch error:
		push_error("Tool operation failed: " + str(error))
		FlashdotAPI.show_error_dialog("Operation failed", str(error))
		return null
```

### Memory Management
```gdscript
func _exit_tree():
	# Properly cleanup plugin resources
	_disconnect_all_signals()
	_free_cached_resources()
	_remove_custom_ui_elements()
```

## Example Plugins

### 1. Advanced Gradient Tool
- Custom gradient types (radial, conic, mesh)
- Interactive gradient editor
- Gradient presets library

### 2. Animation Utilities
- Onion skinning with customizable transparency
- Motion blur effects
- Frame interpolation tools

### 3. Asset Pipeline Integration
- SVG import with layer preservation
- Photoshop PSD import
- Spine animation import

---
_Last updated: July 3, 2025_
