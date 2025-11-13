# UI Framework Architecture

## Overview

This UI framework is designed as a lightweight, immediate-mode GUI library written in C. The architecture emphasizes simplicity, performance, and portability through a clean separation of concerns.

## Core Design Principles

1. **Immediate Mode Paradigm**: No complex widget state management; UI is rebuilt every frame
2. **Zero Dependencies**: Core framework has no external dependencies; platform-specific code isolated in PAL
3. **Single Header Interface**: Simple integration through unified header includes
4. **Performance First**: Minimal allocations, efficient batched rendering
5. **Platform Agnostic**: Core logic independent of OS/graphics API

## System Layers

### Layer 1: Platform Abstraction Layer (PAL)
- **Purpose**: Isolates all platform-specific code
- **Components**:
  - Window Management (creation, events, lifecycle)
  - Input Handling (keyboard, mouse, text input)
  - Rendering Backend (OpenGL, future: Vulkan, DirectX)
  - Timer/Clock Services
- **Current Implementation**: SDL2 + OpenGL 3.3

### Layer 2: Core Framework
- **Purpose**: Platform-independent UI logic
- **Components**:
  - UI Context (global state, frame management)
  - Command Buffer (deferred drawing commands)
  - Layout Engine (automatic positioning, constraints)
  - ID Stack (widget identification system)
  - Style System (theming, customization)

### Layer 3: Rendering Pipeline
- **Purpose**: Efficient batched rendering
- **Components**:
  - Vertex Buffer Management
  - Draw List Optimizer (merges draw calls)
  - Texture Atlas Manager
  - Font Rendering System
  - Clipping/Scissor Management

### Layer 4: Widget Library
- **Purpose**: High-level UI components
- **Components**:
  - Basic Widgets (button, text, checkbox, slider)
  - Containers (window, panel, scrollable area)
  - Complex Widgets (table, tree, menu, combo box)
  - Custom Widget API

## Data Flow

1. **Input Phase**: PAL captures input events -> Updates UI Context
2. **Logic Phase**: Application calls widget functions -> Commands added to buffer
3. **Layout Phase**: Automatic positioning calculated -> Draw commands generated
4. **Render Phase**: Draw commands optimized -> Batched rendering to GPU

## Memory Management

- **Per-Frame Allocations**: Temporary arena allocator, reset each frame
- **Persistent State**: Hash table for widget states, grows as needed
- **String Storage**: Interned strings for efficiency
- **Vertex Buffers**: Pre-allocated, growing buffers

## Key Structures

### UIContext
- Current input state
- Draw command buffer
- Widget state storage
- Layout stack
- Style configuration

### DrawCommand
- Primitive type (rect, text, image)
- Vertex data
- Texture/font reference
- Clipping rectangle

### WidgetState
- Widget ID (hashed string/pointer)
- Interaction state (hover, active, focused)
- User data storage