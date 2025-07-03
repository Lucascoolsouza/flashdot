# Developer Setup Guide

## Prerequisites
- **Godot Engine 4.4+**: Download from [godotengine.org](https://godotengine.org)
- **Git**: For version control and contribution workflow
- **Text Editor/IDE**: VS Code with GDScript extension recommended

## Development Environment Setup

### 1. Clone the Repository
```bash
git clone https://github.com/your-username/flashdot.git
cd flashdot
```

### 2. Godot Editor Configuration
- Open Godot Engine
- Import the project by selecting `project.godot`
- Enable **Advanced Settings** in Project Settings
- Configure editor settings:
  - **Text Editor > Behavior > Auto Brace Completion**: Enabled
  - **Text Editor > Behavior > Auto Indent**: Enabled
  - **Network > Language Server > Enable Smart Resolve**: Enabled

### 3. Recommended VS Code Extensions
If using VS Code for external scripting:
- **godot-tools**: Official GDScript support
- **Godot Files**: .godot file syntax highlighting
- **GitLens**: Enhanced Git integration

### 4. Project Structure Setup
Ensure the following folder structure exists:
```
flashdot/
├── addons/          # Plugin system
├── scenes/          # Scene files (.tscn)
├── scripts/         # GDScript files
├── assets/          # Art, audio, fonts
│   ├── textures/
│   ├── audio/
│   └── fonts/
├── tests/           # Unit and integration tests
├── docs/            # Generated documentation
└── project_guide/   # Development documentation
```

## Build Configuration

### Debug Build
- Default configuration for development
- Full debugging symbols enabled
- Performance profiling available

### Release Build
- Optimized for distribution
- Minimal file size
- Debug symbols stripped

## Version Control Best Practices
- **Branch Strategy**: Use feature branches for new tools/components
- **Commit Messages**: Follow conventional commit format
- **Pre-commit**: Run tests and linting before committing

## Getting Started Checklist
- [ ] Godot project opens without errors
- [ ] All placeholder scenes load correctly
- [ ] Plugin system initializes (check debug output)
- [ ] Timeline editor dock appears
- [ ] Vector drawing tools respond to input
- [ ] Symbol library panel displays

## Common Setup Issues

### Plugin Not Loading
- Check `addons/flashdot/plugin.cfg` exists
- Verify plugin is enabled in Project Settings > Plugins

### Missing Dependencies
- Ensure all `.import` files are generated
- Re-import assets if needed: Project > Reimport Assets

### Performance Issues
- Enable **Monitor > General > FPS** for debugging
- Check **Profiler** tab for bottlenecks

---
_Last updated: July 3, 2025_
