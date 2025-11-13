# Phase 1 Implementation Guide: Command Buffer and Basic Batching

## Overview
This guide provides step-by-step instructions for implementing the Command Buffer and Basic Batching system as the foundation of the enhanced rendering pipeline.

## Files to Create

### 1. Header Files
- `include/ui_framework/rendering/command_buffer.h`
- `include/ui_framework/rendering/draw_command.h`
- `include/ui_framework/rendering/render_state.h`

### 2. Source Files
- `src/rendering/command_buffer.c`
- `src/rendering/draw_command.c`
- `src/rendering/render_state.c`

### 3. Modified Files
- `include/ui_framework/pal/pal_renderer.h` - Add command buffer support
- `src/pal/sdl/pal_sdl_renderer.c` - Integrate command buffer

## Implementation Steps

### Step 1: Create Basic Data Structures

**File: `include/ui_framework/rendering/draw_command.h`**
```c
#ifndef DRAW_COMMAND_H
#define DRAW_COMMAND_H

#include "../pal/pal_renderer.h"
#include "../drawing/color.h"

typedef enum {
    DRAW_CMD_TRIANGLES,
    DRAW_CMD_TEXTURED_QUAD,
    DRAW_CMD_SET_SCISSOR,
    DRAW_CMD_RESET_SCISSOR
} DrawCommandType;

typedef struct DrawCommand {
    DrawCommandType type;
    union {
        struct {
            PAL_TextureHandle texture;
            PAL_Vertex* vertices;
            size_t vertex_count;
        } triangles;
        
        struct {
            PAL_TextureHandle texture;
            float x, y, w, h;
            float u0, v0, u1, v1;
            Color color;
        } quad;
        
        struct {
            int x, y, width, height;
        } scissor;
    } data;
} DrawCommand;

#endif // DRAW_COMMAND_H
```

### Step 2: Implement Command Buffer

**File: `include/ui_framework/rendering/command_buffer.h`**
```c
#ifndef COMMAND_BUFFER_H
#define COMMAND_BUFFER_H

#include "draw_command.h"
#include <stddef.h>

typedef struct CommandBuffer CommandBuffer;

// Lifecycle
CommandBuffer* command_buffer_create(size_t initial_capacity);
void command_buffer_destroy(CommandBuffer* buffer);

// Frame operations
void command_buffer_clear(CommandBuffer* buffer);

// Command submission
void command_buffer_push_triangles(CommandBuffer* buffer,
                                  PAL_TextureHandle texture,
                                  const PAL_Vertex* vertices,
                                  size_t vertex_count);

void command_buffer_push_quad(CommandBuffer* buffer,
                             PAL_TextureHandle texture,
                             float x, float y, float w, float h,
                             float u0, float v0, float u1, float v1,
                             Color color);

void command_buffer_push_scissor(CommandBuffer* buffer,
                                int x, int y, int width, int height);

void command_buffer_push_reset_scissor(CommandBuffer* buffer);

// Command access
size_t command_buffer_get_count(const CommandBuffer* buffer);
const DrawCommand* command_buffer_get_commands(const CommandBuffer* buffer);

#endif // COMMAND_BUFFER_H
```

### Step 3: Modify PAL Renderer Structure

**In `src/pal/sdl/pal_sdl_renderer.c`**, add to the PAL_Renderer struct:
```c
struct PAL_Renderer {
    // ... existing fields ...
    
    // Command buffer support
    CommandBuffer* command_buffer;
    bool use_command_buffer;
    
    // Vertex buffer for batching
    PAL_Vertex* batch_vertices;
    size_t batch_vertex_count;
    size_t batch_vertex_capacity;
};
```

### Step 4: Implement Basic Batching Logic

**Add to `src/pal/sdl/pal_sdl_renderer.c`**:
```c
// Initialize command buffer in pal_renderer_create()
renderer->command_buffer = command_buffer_create(1024);
renderer->use_command_buffer = true; // Can be made configurable
renderer->batch_vertex_capacity = 4096;
renderer->batch_vertices = malloc(sizeof(PAL_Vertex) * renderer->batch_vertex_capacity);

// Modify pal_renderer_begin_frame()
void pal_renderer_begin_frame(PAL_Renderer* renderer, Color clear_color) {
    // ... existing clear code ...
    
    if (renderer->use_command_buffer) {
        command_buffer_clear(renderer->command_buffer);
        renderer->batch_vertex_count = 0;
    }
}

// Modify pal_renderer_render_triangles()
void pal_renderer_render_triangles(PAL_Renderer* renderer, 
                                  PAL_TextureHandle texture,
                                  const PAL_Vertex* vertices, 
                                  size_t vertex_count) {
    if (renderer->use_command_buffer) {
        // Add to command buffer instead of immediate render
        command_buffer_push_triangles(renderer->command_buffer,
                                     texture, vertices, vertex_count);
    } else {
        // Existing immediate rendering
        pal_renderer_render_triangles_immediate(renderer, texture, 
                                               vertices, vertex_count);
    }
}

// Add new function to flush commands
static void pal_renderer_flush_commands(PAL_Renderer* renderer) {
    const DrawCommand* commands = command_buffer_get_commands(renderer->command_buffer);
    size_t count = command_buffer_get_count(renderer->command_buffer);
    
    // Simple batching: group consecutive triangles with same texture
    PAL_TextureHandle current_texture = NULL;
    renderer->batch_vertex_count = 0;
    
    for (size_t i = 0; i < count; i++) {
        const DrawCommand* cmd = &commands[i];
        
        switch (cmd->type) {
            case DRAW_CMD_TRIANGLES: {
                // Check if we can batch with previous
                if (cmd->data.triangles.texture != current_texture ||
                    renderer->batch_vertex_count + cmd->data.triangles.vertex_count > 
                    renderer->batch_vertex_capacity) {
                    // Flush current batch
                    if (renderer->batch_vertex_count > 0) {
                        pal_renderer_render_triangles_immediate(
                            renderer, current_texture,
                            renderer->batch_vertices,
                            renderer->batch_vertex_count
                        );
                        renderer->batch_vertex_count = 0;
                    }
                    current_texture = cmd->data.triangles.texture;
                }
                
                // Add to batch
                memcpy(&renderer->batch_vertices[renderer->batch_vertex_count],
                       cmd->data.triangles.vertices,
                       cmd->data.triangles.vertex_count * sizeof(PAL_Vertex));
                renderer->batch_vertex_count += cmd->data.triangles.vertex_count;
                break;
            }
            
            case DRAW_CMD_SET_SCISSOR:
                // Flush batch before state change
                if (renderer->batch_vertex_count > 0) {
                    pal_renderer_render_triangles_immediate(
                        renderer, current_texture,
                        renderer->batch_vertices,
                        renderer->batch_vertex_count
                    );
                    renderer->batch_vertex_count = 0;
                }
                pal_renderer_set_scissor(renderer,
                    cmd->data.scissor.x, cmd->data.scissor.y,
                    cmd->data.scissor.width, cmd->data.scissor.height);
                break;
                
            // Handle other command types...
        }
    }
    
    // Flush remaining batch
    if (renderer->batch_vertex_count > 0) {
        pal_renderer_render_triangles_immediate(
            renderer, current_texture,
            renderer->batch_vertices,
            renderer->batch_vertex_count
        );
    }
}

// Modify pal_renderer_end_frame()
void pal_renderer_end_frame(PAL_Renderer* renderer) {
    if (renderer->use_command_buffer) {
        pal_renderer_flush_commands(renderer);
    }
    
    // ... existing swap buffer code ...
}
```

## Testing Strategy

### Test 1: Basic Command Buffer
```c
void test_command_buffer_basic() {
    CommandBuffer* buffer = command_buffer_create(10);
    
    // Test triangle command
    PAL_Vertex vertices[3] = {
        {0, 0, 0, 0, 0xFFFFFFFF},
        {1, 0, 1, 0, 0xFFFFFFFF},
        {0, 1, 0, 1, 0xFFFFFFFF}
    };
    command_buffer_push_triangles(buffer, NULL, vertices, 3);
    
    assert(command_buffer_get_count(buffer) == 1);
    
    // Test clear
    command_buffer_clear(buffer);
    assert(command_buffer_get_count(buffer) == 0);
    
    command_buffer_destroy(buffer);
}
```

### Test 2: Batch Merging
```c
void test_batch_merging() {
    // Create renderer with command buffer
    PAL_Renderer* renderer = create_test_renderer();
    
    // Submit multiple draws with same texture
    PAL_TextureHandle tex = create_test_texture();
    
    for (int i = 0; i < 10; i++) {
        PAL_Vertex vertices[6]; // Two triangles
        create_test_quad_vertices(vertices, i * 50, 0, 40, 40);
        pal_renderer_render_triangles(renderer, tex, vertices, 6);
    }
    
    // Should result in single draw call when flushed
    assert(get_draw_call_count(renderer) == 1);
}
```

## Performance Metrics to Track

1. **Draw Call Reduction**
   - Measure draw calls before/after batching
   - Track batch efficiency (vertices per batch)

2. **Frame Time**
   - Compare frame time with/without batching
   - Measure overhead of command buffer

3. **Memory Usage**
   - Track command buffer memory
   - Monitor vertex buffer usage

## Integration Checklist

- [ ] Create command buffer header and source files
- [ ] Implement basic DrawCommand structure
- [ ] Add command buffer to PAL_Renderer
- [ ] Modify pal_renderer_begin_frame() to clear commands
- [ ] Modify pal_renderer_render_triangles() to use command buffer
- [ ] Implement pal_renderer_flush_commands()
- [ ] Modify pal_renderer_end_frame() to flush commands
- [ ] Add configuration flag to enable/disable batching
- [ ] Create unit tests for command buffer
- [ ] Create integration tests for batching
- [ ] Benchmark performance improvements
- [ ] Update documentation

## Next Steps (Phase 2 Preview)

Once Phase 1 is complete and tested, Phase 2 will add:
- State sorting for better batching efficiency
- RenderState comparison functions
- Dynamic vertex buffer management with growing strategy
- Statistics collection and reporting

## Common Issues and Solutions

### Issue 1: Memory Allocation Overhead
**Solution**: Use memory pools for command allocation

### Issue 2: Texture Changes Breaking Batches
**Solution**: Implement texture sorting in Phase 2

### Issue 3: Scissor Rectangle Changes
**Solution**: Track scissor state and only flush when changed

### Issue 4: Buffer Overflow
**Solution**: Implement dynamic buffer growth or flush when full