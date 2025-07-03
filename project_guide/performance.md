# Performance Guidelines

## Performance Targets

### Core Performance Metrics
- **Timeline Scrubbing**: 60fps with 500+ shapes per layer
- **Vector Rendering**: 1000+ simple shapes at 60fps
- **Real-time Drawing**: <16ms input-to-visual latency
- **Symbol Instantiation**: <50ms for complex nested symbols
- **Export Processing**: 10MB/second for scene exports
- **Memory Usage**: <200MB for typical projects

## Rendering Optimization

### Vector Shape Batching
```gdscript
# Batch similar shapes for efficient rendering
class_name ShapeBatcher
extends RefCounted

var batched_shapes: Dictionary = {}

func add_shape(shape: VectorShape):
	var batch_key = _get_batch_key(shape)
	if not batched_shapes.has(batch_key):
		batched_shapes[batch_key] = []
	batched_shapes[batch_key].append(shape)

func render_batches(canvas: CanvasItem):
	for batch_key in batched_shapes:
		var shapes = batched_shapes[batch_key]
		_render_shape_batch(canvas, shapes, batch_key)

func _get_batch_key(shape: VectorShape) -> String:
	# Group by stroke color, width, and fill
	return "%s_%f_%s" % [
		shape.stroke_color.to_html(),
		shape.stroke_width,
		shape.fill_color.to_html()
	]
```

### Level of Detail (LOD) System
```gdscript
class_name LODManager
extends RefCounted

enum LODLevel { HIGH, MEDIUM, LOW }

func get_lod_level(shape: VectorShape, viewport_bounds: Rect2, zoom: float) -> LODLevel:
	var shape_bounds = shape.get_bounds()
	var screen_size = shape_bounds.size * zoom
	
	# Determine appropriate detail level
	if screen_size.x < 5 or screen_size.y < 5:
		return LODLevel.LOW
	elif screen_size.x < 50 or screen_size.y < 50:
		return LODLevel.MEDIUM
	else:
		return LODLevel.HIGH

func render_shape_with_lod(shape: VectorShape, lod: LODLevel, canvas: CanvasItem):
	match lod:
		LODLevel.LOW:
			_render_simple_rect(shape.get_bounds(), canvas)
		LODLevel.MEDIUM:
			_render_simplified_shape(shape, canvas)
		LODLevel.HIGH:
			shape.render_full_detail(canvas)
```

### Viewport Culling
```gdscript
class_name ViewportCuller
extends RefCounted

static func cull_shapes(shapes: Array[VectorShape], viewport: Rect2) -> Array[VectorShape]:
	var visible_shapes: Array[VectorShape] = []
	
	for shape in shapes:
		if viewport.intersects(shape.get_bounds()):
			visible_shapes.append(shape)
	
	return visible_shapes

static func get_expanded_viewport(viewport: Rect2, margin: float = 100.0) -> Rect2:
	# Expand viewport for pre-loading nearby shapes
	return viewport.grow(margin)
```

## Memory Management

### Object Pooling for Frequent Allocations
```gdscript
class_name VectorPointPool
extends RefCounted

static var available_points: Array[VectorPoint] = []
static var max_pool_size: int = 1000

static func get_point() -> VectorPoint:
	if available_points.is_empty():
		return VectorPoint.new()
	else:
		return available_points.pop_back()

static func return_point(point: VectorPoint):
	if available_points.size() < max_pool_size:
		point.reset()  # Clear data but keep object
		available_points.append(point)

# Usage in drawing code
func add_stroke_point(position: Vector2):
	var point = VectorPointPool.get_point()
	point.position = position
	current_stroke.points.append(point)

func finish_stroke():
	# Return temporary points to pool
	for point in temp_points:
		VectorPointPool.return_point(point)
	temp_points.clear()
```

### Resource Caching Strategy
```gdscript
class_name ResourceCache
extends RefCounted

static var texture_cache: Dictionary = {}
static var shader_cache: Dictionary = {}
static var max_cache_size: int = 100

static func get_cached_texture(path: String) -> Texture2D:
	if texture_cache.has(path):
		return texture_cache[path]
	
	var texture = load(path) as Texture2D
	if texture_cache.size() >= max_cache_size:
		_evict_oldest_texture()
	
	texture_cache[path] = texture
	return texture

static func _evict_oldest_texture():
	# LRU eviction logic
	var oldest_key = texture_cache.keys()[0]
	texture_cache.erase(oldest_key)

static func clear_cache():
	texture_cache.clear()
	shader_cache.clear()
```

### Memory Leak Prevention
```gdscript
# Weak reference pattern for event connections
class_name WeakSignalManager
extends RefCounted

var weak_connections: Array[WeakRef] = []

func connect_weak(source: Object, signal_name: String, target: Object, method: String):
	var weak_target = weakref(target)
	weak_connections.append(weak_target)
	
	source.connect(signal_name, Callable(target, method))

func cleanup_dead_references():
	weak_connections = weak_connections.filter(func(ref): return ref.get_ref() != null)

func disconnect_all():
	for weak_ref in weak_connections:
		var target = weak_ref.get_ref()
		if target:
			# Safely disconnect signals
			pass
	weak_connections.clear()
```

## Timeline Performance

### Efficient Frame Caching
```gdscript
class_name TimelineCache
extends RefCounted

var frame_cache: Dictionary = {}  # frame_number -> FrameData
var cache_size_limit: int = 50

func get_frame_data(frame_number: int) -> FrameData:
	if frame_cache.has(frame_number):
		return frame_cache[frame_number]
	
	var frame_data = _generate_frame_data(frame_number)
	_add_to_cache(frame_number, frame_data)
	return frame_data

func _add_to_cache(frame_number: int, data: FrameData):
	if frame_cache.size() >= cache_size_limit:
		_evict_least_used_frame()
	
	frame_cache[frame_number] = data

func invalidate_frame(frame_number: int):
	frame_cache.erase(frame_number)

func invalidate_range(start_frame: int, end_frame: int):
	for frame in range(start_frame, end_frame + 1):
		frame_cache.erase(frame)
```

### Lazy Layer Evaluation
```gdscript
class_name TimelineLayer
extends Resource

var _shapes_dirty: bool = true
var _cached_shapes: Array[VectorShape]
var _cached_bounds: Rect2

func get_shapes() -> Array[VectorShape]:
	if _shapes_dirty:
		_update_shape_cache()
	return _cached_shapes

func get_bounds() -> Rect2:
	if _shapes_dirty:
		_update_bounds_cache()
	return _cached_bounds

func mark_dirty():
	_shapes_dirty = true

func _update_shape_cache():
	# Rebuild shape list from keyframes
	_cached_shapes = _generate_shapes_for_current_frame()
	_shapes_dirty = false

func _update_bounds_cache():
	var combined_bounds = Rect2()
	for shape in get_shapes():
		combined_bounds = combined_bounds.merge(shape.get_bounds())
	_cached_bounds = combined_bounds
```

## Drawing Performance

### Stroke Simplification
```gdscript
class_name StrokeSimplifier
extends RefCounted

# Douglas-Peucker algorithm for path simplification
static func simplify_path(points: Array[Vector2], epsilon: float = 1.0) -> Array[Vector2]:
	if points.size() <= 2:
		return points
	
	var dmax = 0.0
	var index = 0
	
	# Find the point with maximum distance from line
	for i in range(1, points.size() - 1):
		var distance = _point_to_line_distance(points[i], points[0], points[-1])
		if distance > dmax:
			index = i
			dmax = distance
	
	# If max distance is greater than epsilon, recursively simplify
	if dmax > epsilon:
		var left = simplify_path(points.slice(0, index + 1), epsilon)
		var right = simplify_path(points.slice(index), epsilon)
		
		# Combine results
		var result = left.slice(0, -1) + right
		return result
	else:
		return [points[0], points[-1]]

static func _point_to_line_distance(point: Vector2, line_start: Vector2, line_end: Vector2) -> float:
	var line_vec = line_end - line_start
	var point_vec = point - line_start
	var line_length = line_vec.length()
	
	if line_length == 0:
		return point_vec.length()
	
	var t = point_vec.dot(line_vec) / (line_length * line_length)
	t = clamp(t, 0.0, 1.0)
	
	var projection = line_start + t * line_vec
	return point.distance_to(projection)
```

### Adaptive Quality Rendering
```gdscript
class_name AdaptiveRenderer
extends RefCounted

var target_frame_time: float = 16.0  # 60fps = 16.67ms
var current_quality: float = 1.0
var performance_history: Array[float] = []

func render_frame(shapes: Array[VectorShape], canvas: CanvasItem):
	var start_time = Time.get_ticks_msec()
	
	# Adjust quality based on recent performance
	_adjust_quality()
	
	# Render with current quality setting
	for shape in shapes:
		_render_shape_adaptive(shape, canvas, current_quality)
	
	var frame_time = Time.get_ticks_msec() - start_time
	performance_history.append(frame_time)
	
	# Keep only recent history
	if performance_history.size() > 10:
		performance_history = performance_history.slice(-10)

func _adjust_quality():
	if performance_history.size() < 3:
		return
	
	var avg_frame_time = performance_history.reduce(func(a, b): return a + b) / performance_history.size()
	
	if avg_frame_time > target_frame_time * 1.2:  # 20% over target
		current_quality = max(0.5, current_quality - 0.1)
	elif avg_frame_time < target_frame_time * 0.8:  # 20% under target
		current_quality = min(1.0, current_quality + 0.05)

func _render_shape_adaptive(shape: VectorShape, canvas: CanvasItem, quality: float):
	if quality < 0.7:
		_render_low_quality(shape, canvas)
	elif quality < 0.9:
		_render_medium_quality(shape, canvas)
	else:
		shape.render_full_quality(canvas)
```

## Profiling and Monitoring

### Performance Profiler Integration
```gdscript
class_name FlashdotProfiler
extends RefCounted

static var enabled: bool = false
static var timing_data: Dictionary = {}

static func start_timing(label: String):
	if not enabled:
		return
	timing_data[label] = Time.get_ticks_usec()

static func end_timing(label: String):
	if not enabled or not timing_data.has(label):
		return
	
	var duration = Time.get_ticks_usec() - timing_data[label]
	print("PROFILE: %s took %d μs" % [label, duration])
	timing_data.erase(label)

# Usage in performance-critical code
func _draw():
	FlashdotProfiler.start_timing("vector_rendering")
	_render_all_shapes()
	FlashdotProfiler.end_timing("vector_rendering")
```

### Memory Usage Monitoring
```gdscript
class_name MemoryMonitor
extends RefCounted

static func get_current_usage() -> Dictionary:
	return {
		"static_memory": OS.get_static_memory_usage(false),
		"dynamic_memory": OS.get_static_memory_usage(true),
		"object_count": Performance.get_monitor(Performance.OBJECT_COUNT),
		"resource_count": Performance.get_monitor(Performance.OBJECT_RESOURCE_COUNT)
	}

static func log_memory_usage(context: String):
	var usage = get_current_usage()
	print("MEMORY [%s]: Static=%d, Dynamic=%d, Objects=%d" % [
		context,
		usage.static_memory,
		usage.dynamic_memory,
		usage.object_count
	])

static func check_memory_threshold():
	var usage = get_current_usage()
	var threshold = 500 * 1024 * 1024  # 500MB
	
	if usage.static_memory > threshold:
		push_warning("High memory usage detected: %d MB" % (usage.static_memory / (1024 * 1024)))
		return true
	return false
```

## Best Practices Summary

### Do's
- ✅ **Use object pooling** for frequently created/destroyed objects
- ✅ **Cache expensive calculations** (bounds, transformations)
- ✅ **Implement viewport culling** for large scenes
- ✅ **Batch similar rendering operations**
- ✅ **Use weak references** to prevent memory leaks
- ✅ **Profile performance regularly** during development

### Don'ts
- ❌ **Avoid per-frame allocations** in drawing loops
- ❌ **Don't render invisible shapes**
- ❌ **Don't hold strong references** in signal callbacks
- ❌ **Avoid complex calculations** in `_process()` methods
- ❌ **Don't ignore memory usage** during development
- ❌ **Avoid blocking operations** on the main thread

### Performance Testing Checklist
- [ ] Timeline scrubbing with 100+ shapes per layer
- [ ] Real-time drawing with minimal input lag
- [ ] Symbol instantiation stress test
- [ ] Large project export performance
- [ ] Memory usage over extended sessions
- [ ] Plugin performance impact measurement

---
_Last updated: July 3, 2025_
