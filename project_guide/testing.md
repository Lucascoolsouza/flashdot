# Testing Strategy

## Testing Philosophy
Flashdot uses a comprehensive testing approach to ensure reliability, performance, and maintainability across all animation and vector editing features.

## Test Categories

### 1. Unit Tests
**Purpose**: Test individual components in isolation

**Coverage Areas**:
- Vector shape mathematics (bounds, transformations, hit testing)
- Timeline data model operations
- Symbol management and nesting
- Export format serialization
- Tool state management

**Framework**: Godot's built-in testing with GUT (Godot Unit Test) addon

```gdscript
# Example unit test
extends GutTest

func test_vector_shape_bounds():
	var rect = RectangleShape.new()
	rect.size = Vector2(100, 50)
	rect.position = Vector2(10, 20)
	
	var bounds = rect.get_bounds()
	assert_eq(bounds.position, Vector2(10, 20))
	assert_eq(bounds.size, Vector2(100, 50))
```

### 2. Integration Tests
**Purpose**: Test component interactions and workflows

**Coverage Areas**:
- Timeline editor with vector tools
- Symbol library integration
- Export pipeline end-to-end
- Plugin system loading
- Event system communication

```gdscript
# Example integration test
extends GutTest

func test_drawing_workflow():
	var timeline = TimelineEditor.new()
	var pen_tool = PenTool.new()
	
	pen_tool.activate()
	pen_tool.start_stroke(Vector2(0, 0))
	pen_tool.add_point(Vector2(50, 50))
	pen_tool.finish_stroke()
	
	assert_eq(timeline.get_current_layer().shapes.size(), 1)
```

### 3. Performance Tests
**Purpose**: Ensure smooth operation under load

**Benchmarks**:
- **Timeline Scrubbing**: 60fps with 100+ keyframes
- **Vector Rendering**: 500+ shapes at 60fps
- **Symbol Instantiation**: <100ms for complex symbols
- **Export Speed**: 1MB/second for scene exports

```gdscript
# Example performance test
extends GutTest

func test_large_timeline_performance():
	var timeline = create_large_timeline(1000) # 1000 frames
	
	var start_time = Time.get_ticks_msec()
	timeline.goto_frame(500)
	var end_time = Time.get_ticks_msec()
	
	assert_lt(end_time - start_time, 16) # < 16ms (60fps budget)
```

### 4. UI Tests
**Purpose**: Validate user interface behavior and responsiveness

**Test Areas**:
- Tool palette interactions
- Timeline scrubbing and playback
- Symbol library drag-and-drop
- Keyboard shortcuts
- Context menus

## Test Structure

### Directory Organization
```
tests/
├── unit/
│   ├── vector/          # Shape and path tests
│   ├── timeline/        # Timeline model tests
│   ├── symbols/         # Symbol system tests
│   └── export/          # Export format tests
├── integration/
│   ├── workflows/       # End-to-end user workflows
│   ├── plugins/         # Plugin system tests
│   └── performance/     # Performance benchmarks
├── fixtures/            # Test data and assets
└── utils/               # Test utilities and helpers
```

### Test Data Management
```gdscript
# Test fixture loading
class_name TestFixtures
extends RefCounted

static func load_sample_timeline() -> Timeline:
	return load("res://tests/fixtures/sample_timeline.tres")

static func create_test_shapes() -> Array[VectorShape]:
	return [
		create_rectangle(Vector2(0, 0), Vector2(100, 100)),
		create_circle(Vector2(50, 50), 25),
		create_path([Vector2(0, 0), Vector2(100, 100)])
	]
```

## Testing Tools & Setup

### GUT (Godot Unit Test) Configuration
```gdscript
# .gutconfig.json
{
	"dirs": ["res://tests/unit", "res://tests/integration"],
	"include_subdirs": true,
	"log_level": 1,
	"yield_between_tests": true,
	"double_strategy": "script_only"
}
```

### Automated Test Runner
```gdscript
# test_runner.gd
extends SceneTree

func _init():
	var gut = load("res://addons/gut/gut.gd").new()
	gut.set_yield_between_tests(true)
	gut.run_tests()
	quit()
```

### Continuous Integration
```yaml
# .github/workflows/test.yml
name: Run Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Godot
        uses: lihop/setup-godot@v1
        with:
          godot-version: '4.4'
      - name: Run Tests
        run: |
          godot --headless --script test_runner.gd
```

## Test Coverage Goals

### Minimum Coverage Targets
- **Core Components**: 90%+ code coverage
- **Vector Operations**: 95%+ (critical for accuracy)
- **Timeline Logic**: 85%+ 
- **Export Systems**: 80%+
- **UI Components**: 70%+

### Coverage Measurement
```gdscript
# coverage_collector.gd
extends RefCounted

static var covered_lines := {}
static var total_lines := {}

static func record_line_execution(script_path: String, line: int):
	if not covered_lines.has(script_path):
		covered_lines[script_path] = {}
	covered_lines[script_path][line] = true

static func get_coverage_report() -> Dictionary:
	var report := {}
	for script_path in total_lines:
		var covered = covered_lines.get(script_path, {}).size()
		var total = total_lines[script_path]
		report[script_path] = float(covered) / float(total)
	return report
```

## Test-Driven Development Workflow

### 1. Red Phase - Write Failing Test
```gdscript
func test_shape_selection():
	var shape = RectangleShape.new()
	var tool = SelectionTool.new()
	
	tool.select_shape(shape)
	assert_true(tool.is_selected(shape)) # This will fail initially
```

### 2. Green Phase - Implement Feature
```gdscript
# In SelectionTool class
func select_shape(shape: VectorShape):
	selected_shapes.append(shape)

func is_selected(shape: VectorShape) -> bool:
	return shape in selected_shapes
```

### 3. Refactor Phase - Improve Code
```gdscript
# Optimized version with Set for O(1) lookup
var selected_shapes_set := {}

func select_shape(shape: VectorShape):
	selected_shapes_set[shape] = true

func is_selected(shape: VectorShape) -> bool:
	return selected_shapes_set.has(shape)
```

## Mock Objects & Test Doubles

### Canvas Mock for Drawing Tests
```gdscript
class_name MockCanvas
extends RefCounted

var draw_calls := []

func draw_line(from: Vector2, to: Vector2, color: Color, width: float):
	draw_calls.append({
		"method": "draw_line",
		"from": from,
		"to": to,
		"color": color,
		"width": width
	})

func verify_draw_call(expected_call: Dictionary) -> bool:
	return expected_call in draw_calls
```

### Event Bus Mock
```gdscript
class_name MockEventBus
extends RefCounted

var emitted_signals := []

func emit_signal(signal_name: String, args: Array):
	emitted_signals.append({
		"signal": signal_name,
		"args": args
	})

func verify_signal_emitted(signal_name: String) -> bool:
	return emitted_signals.any(func(s): return s.signal == signal_name)
```

## Debugging Test Failures

### Test Output Analysis
```gdscript
# Enhanced assertion with debug info
func assert_shape_bounds(shape: VectorShape, expected: Rect2):
	var actual = shape.get_bounds()
	if not actual.is_equal_approx(expected):
		print("Shape bounds mismatch:")
		print("  Expected: ", expected)
		print("  Actual: ", actual)
		print("  Shape type: ", shape.get_class())
		print("  Shape properties: ", shape.get_property_list())
	assert_eq(actual, expected)
```

### Test Isolation
```gdscript
func before_each():
	# Reset global state before each test
	TimelineEvents.get_instance().disconnect_all()
	ToolRegistry.clear_all_tools()
	cleanup_test_scenes()
```

---
_Last updated: July 3, 2025_
