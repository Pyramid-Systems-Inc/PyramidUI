# Enhanced Rendering Pipeline - Architecture Diagrams

## 1. System Component Interaction

```mermaid
graph LR
    subgraph "UI Layer"
        UI[UI Framework Code]
    end
    
    subgraph "Command Generation"
        CB[Command Buffer]
        CS[Command Sorter]
    end
    
    subgraph "Resource Management"
        AM[Atlas Manager]
        FM[Font Manager]
        TC[Texture Cache]
        GC[Glyph Cache]
    end
    
    subgraph "Batch Processing"
        BM[Batch Manager]
        BB[Batch Builder]
        VB[Vertex Buffer]
    end
    
    subgraph "GPU Backend"
        GL[OpenGL Renderer]
        SH[Shader Programs]
        TX[GPU Textures]
    end
    
    UI -->|Draw Commands| CB
    CB -->|Unsorted Commands| CS
    CS -->|Sorted Commands| BB
    
    FM -->|Glyphs| GC
    GC -->|Glyph Textures| AM
    TC -->|Texture Data| AM
    AM -->|Atlas Textures| TX
    
    BB -->|Batches| BM
    BM -->|Vertex Data| VB
    VB -->|Draw Calls| GL
    GL -->|Bind| TX
    GL -->|Use| SH
```

## 2. Frame Execution Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant PAL as PAL Renderer
    participant Cmd as Command Buffer
    participant Batch as Batch Manager
    participant Atlas as Atlas Manager
    participant GPU as GPU/OpenGL
    
    App->>PAL: begin_frame()
    PAL->>Cmd: clear()
    PAL->>Batch: reset()
    
    loop Draw Operations
        App->>PAL: render_triangles()
        PAL->>Cmd: add_command()
        Note over Cmd: Commands accumulated
    end
    
    App->>PAL: end_frame()
    PAL->>Cmd: get_commands()
    Cmd->>Batch: build_batches()
    
    Note over Batch: Sort by state
    Note over Batch: Group similar draws
    
    loop For each batch
        Batch->>Atlas: get_texture()
        Batch->>GPU: bind_texture()
        Batch->>GPU: upload_vertices()
        Batch->>GPU: draw_triangles()
    end
    
    PAL->>GPU: swap_buffers()
```

## 3. Texture Atlas Packing Process

```mermaid
flowchart TB
    Start([New Texture Request])
    
    Start --> CheckSpace{Space in<br/>existing atlas?}
    
    CheckSpace -->|Yes| FindNode[Find suitable node<br/>in atlas tree]
    CheckSpace -->|No| CreateAtlas[Create new atlas]
    
    CreateAtlas --> InitTree[Initialize binary tree<br/>for new atlas]
    InitTree --> FindNode
    
    FindNode --> Pack[Pack texture<br/>into node]
    Pack --> Split{Node needs<br/>splitting?}
    
    Split -->|Yes| SplitNode[Split node into<br/>left and right]
    Split -->|No| UpdateUV[Calculate UV<br/>coordinates]
    
    SplitNode --> PlaceTexture[Place texture in<br/>left child]
    PlaceTexture --> UpdateUV
    
    UpdateUV --> Upload[Upload texture data<br/>to GPU atlas]
    Upload --> Return([Return texture ID<br/>and UV coords])
```

## 4. Font Rendering Pipeline

```mermaid
flowchart LR
    subgraph "Text Input"
        Text[Text String]
        Font[Font Handle]
        Size[Font Size]
    end
    
    subgraph "Glyph Processing"
        Cache{In Cache?}
        Raster[Rasterize<br/>Glyph]
        Pack[Pack into<br/>Atlas]
    end
    
    subgraph "Layout"
        Measure[Measure<br/>Text]
        Position[Calculate<br/>Positions]
        Kern[Apply<br/>Kerning]
    end
    
    subgraph "Rendering"
        Quads[Generate<br/>Quads]
        Batch[Add to<br/>Batch]
    end
    
    Text --> Cache
    Font --> Cache
    Size --> Cache
    
    Cache -->|No| Raster
    Raster --> Pack
    Pack --> Cache
    
    Cache -->|Yes| Measure
    Measure --> Position
    Position --> Kern
    Kern --> Quads
    Quads --> Batch
```

## 5. Batch Building State Machine

```mermaid
stateDiagram-v2
    [*] --> Idle: Initialize
    
    Idle --> Collecting: begin_frame()
    
    Collecting --> Collecting: add_command()
    Collecting --> Sorting: end_frame()
    
    Sorting --> Building: commands_sorted()
    
    Building --> NewBatch: state_change
    Building --> AddToBatch: same_state
    
    NewBatch --> Building: batch_created
    AddToBatch --> Building: vertex_added
    AddToBatch --> FlushBatch: batch_full
    
    FlushBatch --> NewBatch: batch_flushed
    
    Building --> Rendering: all_commands_processed
    
    Rendering --> Idle: frame_complete
```

## 6. Draw Command Batching Decision Tree

```mermaid
flowchart TD
    Cmd[New Draw Command]
    
    Cmd --> Check1{Same Texture?}
    Check1 -->|No| NewBatch1[Create New Batch]
    Check1 -->|Yes| Check2{Same Blend Mode?}
    
    Check2 -->|No| NewBatch2[Create New Batch]
    Check2 -->|Yes| Check3{Same Shader?}
    
    Check3 -->|No| NewBatch3[Create New Batch]
    Check3 -->|Yes| Check4{Same Scissor?}
    
    Check4 -->|No| NewBatch4[Create New Batch]
    Check4 -->|Yes| Check5{Batch Full?}
    
    Check5 -->|Yes| NewBatch5[Create New Batch]
    Check5 -->|No| AddBatch[Add to Current Batch]
    
    NewBatch1 --> SetState[Set New State]
    NewBatch2 --> SetState
    NewBatch3 --> SetState
    NewBatch4 --> SetState
    NewBatch5 --> SetState
    
    SetState --> AddBatch
    AddBatch --> End([Command Batched])
```

## 7. Memory Management Strategy

```mermaid
graph TB
    subgraph "Vertex Memory"
        VBO1[VBO Buffer 1<br/>Frame N]
        VBO2[VBO Buffer 2<br/>Frame N+1]
        VBO3[VBO Buffer 3<br/>Frame N+2]
    end
    
    subgraph "Command Memory"
        CmdPool[Command Pool<br/>Pre-allocated]
        CmdFree[Free List]
        CmdUsed[Used List]
    end
    
    subgraph "Atlas Memory"
        StaticAtlas[Static Atlas<br/>UI Elements]
        DynamicAtlas[Dynamic Atlas<br/>Runtime Textures]
        GlyphAtlas[Glyph Atlas<br/>Font Cache]
    end
    
    VBO1 -.->|Triple Buffer| VBO2
    VBO2 -.->|Rotation| VBO3
    VBO3 -.->|System| VBO1
    
    CmdPool -->|Allocate| CmdUsed
    CmdUsed -->|Free| CmdFree
    CmdFree -->|Reuse| CmdPool
    
    StaticAtlas -->|Persistent| GPU[GPU Memory]
    DynamicAtlas -->|Updated| GPU
    GlyphAtlas -->|On-demand| GPU
```

## 8. Performance Optimization Flow

```mermaid
flowchart LR
    subgraph "Input Stage"
        Draw[Draw Calls]
        Prof[Profile Data]
    end
    
    subgraph "Analysis"
        Sort[State Sorting]
        Group[Command Grouping]
        Pred[Prediction]
    end
    
    subgraph "Optimization"
        Batch[Batching]
        Atlas[Atlas Packing]
        Cache[Cache Management]
    end
    
    subgraph "Output"
        GPU[GPU Commands]
        Stats[Statistics]
    end
    
    Draw --> Sort
    Prof --> Pred
    
    Sort --> Group
    Group --> Batch
    Pred --> Cache
    
    Batch --> GPU
    Atlas --> GPU
    Cache --> Atlas
    
    GPU --> Stats
    Stats -->|Feedback| Prof
```

## 9. Integration Points with Existing PAL

```mermaid
graph TB
    subgraph "Existing PAL Functions"
        BF[pal_renderer_begin_frame]
        RT[pal_renderer_render_triangles]
        RQ[pal_renderer_render_textured_quad]
        EF[pal_renderer_end_frame]
    end
    
    subgraph "New Enhanced Layer"
        CB[Command Buffer]
        BM[Batch Manager]
        AM[Atlas Manager]
        FM[Font Manager]
    end
    
    subgraph "Integration Points"
        IF[Interceptor Functions]
        CFG[Configuration Flags]
        FB[Fallback Paths]
    end
    
    BF -->|Modified| IF
    RT -->|Modified| IF
    RQ -->|Modified| IF
    EF -->|Modified| IF
    
    IF -->|If Enabled| CB
    IF -->|If Enabled| BM
    IF -->|If Disabled| FB
    
    CB --> BM
    BM --> AM
    FM --> AM
    
    CFG -->|Control| IF
    FB -->|Direct Render| GPU[GPU]
    BM -->|Batched Render| GPU
```

## 10. Module Dependencies

```mermaid
graph BT
    subgraph "Core Dependencies"
        STB[stb_truetype.h]
        GL[OpenGL/GLAD]
        SDL[SDL2]
    end
    
    subgraph "Base Modules"
        VTX[Vertex Management]
        TEX[Texture Management]
        SHD[Shader Management]
    end
    
    subgraph "Enhanced Modules"
        CMD[Command Buffer]
        SORT[Command Sorter]
        BATCH[Batch Manager]
        ATLAS[Atlas Manager]
        FONT[Font Manager]
    end
    
    subgraph "PAL Integration"
        PAL[PAL Renderer]
    end
    
    STB --> FONT
    GL --> VTX
    GL --> TEX
    GL --> SHD
    SDL --> PAL
    
    VTX --> BATCH
    TEX --> ATLAS
    SHD --> PAL
    
    CMD --> SORT
    SORT --> BATCH
    BATCH --> PAL
    ATLAS --> PAL
    FONT --> ATLAS
    FONT --> PAL