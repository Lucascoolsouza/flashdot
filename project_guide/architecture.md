# Architecture & Design Patterns

## System Architecture Overview

Flashdot follows a modular plugin-based architecture that integrates seamlessly with Godot's editor ecosystem.

```
┌─────────────────────────────────────────┐
│              Godot Editor               │
├─────────────────────────────────────────┤
│          Flashdot Plugin System         │
├─────────────────┬─────────────┬─────────┤
│   Timeline      │   Vector    │ Symbol  │
│   Editor        │   Tools     │ Library │
├─────────────────┼─────────────┼─────────┤
│        Core Data Model & Events        │
├─────────────────────────────────────────┤
│           Export & Rendering            │
└─────────────────────────────────────────┘
```

## Core Design Patterns

### 1. Plugin Architecture Pattern
**Purpose**: Modular tool system allowing community extensions

```gdscript
# Base tool interface
class_name FlashdotTool
extends RefCounted

signal tool_activated(tool: FlashdotTool)
signal tool_deactivated(tool: FlashdotTool)

func activate() -> void:
	pass

func deactivate() -> void:
	pass

func handle_input(event: InputEvent) -> bool:
	return false
```

### 2. Observer Pattern for Timeline Updates
**Purpose**: Decouple timeline changes from UI updates

```gdscript
# Timeline event bus
class_name TimelineEvents
extends RefCounted

signal frame_changed(frame_number: int)
signal layer_added(layer: TimelineLayer)
signal keyframe_created(frame: int, layer: int)

static var instance: TimelineEvents

static func get_instance() -> TimelineEvents:
	if not instance:
		instance = TimelineEvents.new()
	return instance
```

### 3. Command Pattern for Undo/Redo
**Purpose**: Reversible operations for user actions

```gdscript
class_name DrawCommand
extends RefCounted

var _stroke_data: Array[Vector2]
var _layer_id: int

func execute() -> void:
	# Add stroke to layer
	pass

func undo() -> void:
	# Remove stroke from layer
	pass
```

### 4. Factory Pattern for Vector Shapes
**Purpose**: Consistent creation of different shape types

```gdscript
class_name ShapeFactory
extends RefCounted

enum ShapeType { RECTANGLE, ELLIPSE, LINE, BEZIER }

static func create_shape(type: ShapeType) -> VectorShape:
	match type:
		ShapeType.RECTANGLE:
			return RectangleShape.new()
		ShapeType.ELLIPSE:
			return EllipseShape.new()
		# ... other shapes
```

### 5. State Pattern for Tool Behavior
**Purpose**: Clean tool state management

```gdscript
class_name DrawingTool
extends FlashdotTool

enum State { IDLE, DRAWING, SELECTING }
var current_state: State = State.IDLE

func handle_input(event: InputEvent) -> bool:
	match current_state:
		State.IDLE:
			return _handle_idle_input(event)
		State.DRAWING:
			return _handle_drawing_input(event)
		State.SELECTING:
			return _handle_selecting_input(event)
```

## Data Model Architecture

### Symbol System
```gdscript
class_name Symbol
extends Resource

@export var name: String
@export var layers: Array[SymbolLayer]
@export var metadata: Dictionary

class_name SymbolLayer
extends Resource

@export var visible: bool = true
@export var locked: bool = false
@export var shapes: Array[VectorShape]
```

### Timeline Structure
```gdscript
class_name Timeline
extends Resource

@export var frame_rate: float = 24.0
@export var layers: Array[TimelineLayer]
@export var total_frames: int

class_name TimelineLayer
extends Resource

@export var name: String
@export var keyframes: Dictionary  # frame_number -> Keyframe
```

## Component Communication

### 1. Signal-Based Events
- **Loose coupling** between UI components
- **Event-driven** updates for real-time feedback
- **Centralized** event handling through manager classes

### 2. Dependency Injection
- **Service locator** pattern for core systems
- **Interface-based** dependencies for testability
- **Plugin registration** system for extensions

### 3. Resource Management
- **Shared resources** for symbols and templates
- **Lazy loading** for large assets
- **Memory pooling** for frequent allocations

## Performance Considerations

### Rendering Pipeline
1. **Dirty flagging** for changed elements only
2. **Batched rendering** for multiple shapes
3. **Level-of-detail** for complex symbols
4. **Viewport culling** for off-screen content

### Memory Management
- **Object pooling** for temporary drawing objects
- **Weak references** to prevent circular dependencies
- **Resource caching** with LRU eviction
- **Streaming** for large animation files

## Extension Points

### Custom Tools
```gdscript
# Register new drawing tools
ToolManager.register_tool("custom_brush", CustomBrushTool.new())

# Add custom export formats
ExportManager.register_exporter("custom_format", CustomExporter.new())
```

### Event Hooks
```gdscript
# Listen to core events
TimelineEvents.get_instance().frame_changed.connect(_on_frame_changed)
SymbolEvents.get_instance().symbol_created.connect(_on_symbol_created)
```

## Testing Architecture

### Unit Testing
- **Component isolation** with dependency mocking
- **State verification** for data model changes
- **Signal testing** for event-driven behavior

### Integration Testing
- **Tool interaction** workflows
- **Export pipeline** validation
- **Performance benchmarking** for complex scenes

---
_Last updated: July 3, 2025_
