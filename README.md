# UI Framework

A lightweight, immediate-mode GUI library written in pure C. Designed for simplicity, performance, and ease of integration.

## Features

- **Immediate Mode Design**: Simple, stateless API with no complex object hierarchies
- **Zero Dependencies**: Core framework requires no external libraries
- **Cross-Platform**: Runs on Windows, Linux, macOS through Platform Abstraction Layer
- **High Performance**: Batched rendering, minimal allocations, optimized for modern GPUs
- **Comprehensive Widget Set**: From basic controls to complex components
- **Flexible Styling**: Complete control over appearance through style system
- **Small Footprint**: Minimal memory usage, suitable for embedded applications

## Current Status

The framework is in active development. Currently implemented:
- Platform Abstraction Layer (SDL2/OpenGL backend)
- Basic rendering pipeline
- Input handling system
- Foundation for immediate mode architecture

See [ROADMAP_DETAILED.md](ROADMAP_DETAILED.md) for development plans.

## Building

### Prerequisites

- C11 compatible compiler (GCC, Clang, MSVC)
- CMake 3.10+
- SDL2 development libraries
- OpenGL 3.3+ support

### Build Instructions

#### Linux/macOS
```bash
mkdir build && cd build
cmake ..
make -j4
./ui_framework
```

#### Windows
```batch
mkdir build && cd build
cmake .. -G "Ninja"
ninja
ui_framework.exe
```

## Usage Example

```c
#include "ui_framework/ui_framework.h"

int main() {
    // Initialize
    UIContext* ctx = ui_create_context();
    PAL_Window* window = pal_window_create(&(PAL_WindowConfig){
        .title = "My Application",
        .width = 1280,
        .height = 720
    });
    
    bool show_demo = true;
    float value = 0.5f;
    
    // Main loop
    while (!pal_window_should_close(window)) {
        // Start frame
        ui_begin_frame(ctx);
        
        // Create UI
        if (ui_begin_window("Settings", &show_demo)) {
            ui_text("Hello, World!");
            
            if (ui_button("Click Me")) {
                printf("Button clicked!\n");
            }
            
            ui_slider_float("Value", &value, 0.0f, 1.0f);
            
            ui_end_window();
        }
        
        // Render frame
        ui_end_frame(ctx);
    }
    
    // Cleanup
    ui_destroy_context(ctx);
    pal_window_destroy(window);
    return 0;
}
```

## Architecture

The framework follows a layered architecture:

1. **Platform Abstraction Layer (PAL)**: Handles OS/graphics API specifics
2. **Core Framework**: Platform-independent immediate mode logic
3. **Rendering Pipeline**: Optimized batched rendering system
4. **Widget Library**: High-level UI components

See [ARCHITECTURE.md](ARCHITECTURE.md) for detailed information.

## Contributing

The project is under active development. Key areas needing work:

- Text rendering system implementation
- Widget library expansion
- Performance optimizations
- Additional platform backends
- Documentation and examples

## License

This software is released into the public domain (Unlicense). See [LICENSE](LICENSE) for details.