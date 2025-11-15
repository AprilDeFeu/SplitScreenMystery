# Copilot Instructions for UE_SplitScreen

## Project Vision

A split-screen cyberpunk detective game where a detective interviews suspects whose conscious mind is severed from the metaverse, but whose subconscious remains connected and can be spied on by police.

**Left Screen:** Detective interview room (dialogue tree with multiple choice options)  
**Right Screen:** First-person POV of suspect's subconscious in the metaverse (reacts to dialogue choices)

**Art Direction:** PSX-style low-poly geometry with 1-bit dithered shading (Obra Dinn/Who's Lila? inspired), neo-noir cyberpunk aesthetic with switchable palettes.

## Technical Foundation

- **Engine:** Unreal Engine 5.6
- **Target Platforms:** PC/Desktop only (Windows, Linux, macOS)
  - **Not targeting mobile devices** - This game requires keyboard input for typing and is optimized for desktop performance
  - Mobile-specific optimizations are disabled
- **Language:** C++ (minimal Blueprint usage preferred)
- **Module:** `UE_SplitScreen` (see `Source/UE_SplitScreen/`)
- **Build System:** UnrealBuildTool with custom `.Target.cs` and `.Build.cs` files
- **Primary Map:** `/Game/Levels/MAP_Default.umap`

## Project Structure

```
Source/UE_SplitScreen/
  ├── UE_SplitScreen.cpp/h         # Main game module
  ├── UE_SplitScreen.Build.cs      # Module dependencies
  └── [Future folders]
      ├── Characters/              # Detective, suspect characters
      ├── Dialogue/                # Dialogue tree system
      ├── SplitScreen/             # Split-screen viewport management
      ├── Metaverse/               # Metaverse environment/mechanics
      └── Rendering/               # Custom shader/material systems

Config/
  ├── DefaultEngine.ini            # Engine settings, startup map
  ├── DefaultGame.ini              # Game-specific settings
  └── DefaultInput.ini             # Input mappings (gamepad + KB/M)

Content/
  ├── Levels/                      # Level files
  ├── Materials/                   # Custom shaders (PSX + 1-bit dithering)
  ├── Models/                      # Low-poly PSX-style meshes
  └── Audio/                       # Dialogue, ambient sound
```

## Core Architectural Patterns

### 1. Split-Screen System
- Use `UGameViewportClient` with custom viewport layout for persistent dual-screen rendering
- Left viewport: Static camera for interview room
- Right viewport: First-person camera following suspect's metaverse avatar
- Both viewports must render independently with separate post-process chains

### 2. Dialogue System
- Implement a data-driven dialogue tree using `UDataTable` or custom C++ graph structure
- Each dialogue node should trigger:
  - Detective/suspect animation states
  - Metaverse behavior changes (environment reactions, avatar actions)
  - Narrative state tracking for branching storylines
- Store dialogue in JSON or CSV for easy iteration by writers

### 3. Custom Rendering Pipeline
- **PSX-Style Geometry:**
  - Use custom `MaterialParameterCollection` to control vertex snapping/jitter
  - Implement vertex shader that snaps positions to low-resolution grid
  - Disable texture filtering, use point sampling
  - Low-poly models (< 1000 tris per asset)

- **1-Bit Dithered Shading:**
  - Custom post-process material using ordered dithering patterns
  - Palette swapping system via material parameters
  - Reference: Obra Dinn's 1-bit dithering technique
  - Store palettes as texture atlases or material parameter collections
  - Apply dithering in screen space, not world space

- **Performance Target:** 60+ FPS on mid-range hardware (avoid expensive features like Lumen/Nanite for this style)

### 4. Input System
- Use Enhanced Input System (already in dependencies)
- **Desktop input only:**
  - Mouse + Keyboard (primary)
  - PlayStation DualShock/DualSense
  - Xbox controller
  - Generic PC gamepads
- **Keyboard typing required** - Players will need to type during gameplay, confirming this is a desktop-only experience
- Map all actions to context-aware bindings (e.g., dialogue selection via d-pad or WASD)
- See `Config/DefaultInput.ini` for axis/action definitions

### 5. Testing & Quality Framework
- **Unit Tests:** Use Unreal's Automation Framework (`FAutomationTestBase`)
  - Place tests in `Source/UE_SplitScreen/Tests/`
  - Run via `Editor > Tools > Test Automation`
- **Code Coverage:** Integrate OpenCppCoverage or Visual Studio's native coverage tools
- **Security:** 
  - Avoid Blueprint-exposed functions with arbitrary code execution
  - Validate all user input (dialogue choices, save files)
  - Use `UE_LOG` with sanitized strings, never raw user input
- **Performance:** Profile with Unreal Insights and session frontend regularly

## Build & Run Workflows

### Building the Project
Use VS Code tasks (see `.vscode/tasks.json`):
- **Editor Build:** `UE_SplitScreenEditor Win64 Development Build`
- **Game Build:** `UE_SplitScreen Win64 Development Build`
- **Clean Build:** Use corresponding `Clean` tasks before `Build`

Or manually via command line:
```powershell
# Replace <UE5_ROOT> with your Unreal Engine 5.6 installation path
# Replace <PROJECT_ROOT> with your project directory path
cd <UE5_ROOT>
Engine\Build\BatchFiles\Build.bat UE_SplitScreenEditor Win64 Development "<PROJECT_ROOT>\UE_SplitScreen.uproject" -waitmutex
```

### Running Tests
```powershell
# From Unreal Editor: Tools > Test Automation > Run All Tests
# Or via command line:
UnrealEditor-Cmd.exe "<PROJECT_ROOT>\UE_SplitScreen.uproject" -ExecCmds="Automation RunTests Now <TestName>" -unattended -nopause -testexit="Automation Test Queue Empty"
```

### Debugging
- Use Visual Studio or VS Code with C++ debugger attached to `UnrealEditor.exe`
- Set breakpoints in `Source/UE_SplitScreen/` files
- Use `UE_LOG(LogTemp, Warning, TEXT("Debug message"));` for runtime logging

## Code Conventions

### Module Dependencies
Declared in `Source/UE_SplitScreen/UE_SplitScreen.Build.cs`:
```csharp
PublicDependencyModuleNames.AddRange(new string[] { 
    "Core", "CoreUObject", "Engine", "InputCore", "EnhancedInput" 
});
```

**When adding new dependencies:**
- Keep dependencies minimal (avoid large plugin bloat)
- Prefer `PrivateDependencyModuleNames` for internal-only modules
- Document why each dependency is needed

### Naming Conventions
- **Classes:** `UMyClass` (UObject-derived), `AMyActor` (AActor-derived), `FMyStruct` (plain structs)
- **Files:** Match class names (`MyClass.h`, `MyClass.cpp`)
- **Blueprints:** `BP_MyBlueprint` (use sparingly)
- **Materials:** `M_MaterialName`, `MI_MaterialInstance`
- **Textures:** `T_TextureName`

### Gameplay Code Style
- Prefer C++ over Blueprints for core gameplay logic
- Use `TObjectPtr<>` for UObject pointers (UE5 standard)
- Avoid `new`/`delete`—use `NewObject<>()`, `CreateDefaultSubobject<>()`
- Mark functions `const` when they don't modify state
- Use `UFUNCTION(BlueprintCallable)` only when Blueprint access is explicitly needed

## Custom Shader Development

### Material Setup
1. Create master material in `Content/Materials/Master/`
2. Use material functions for reusable logic (vertex snapping, dithering)
3. Instance materials for each palette variation (`MI_Palette_Noir`, `MI_Palette_Neon`)

### Vertex Snapping (PSX Effect)
```hlsl
// In custom vertex shader or material function:
float3 SnappedPosition = floor(WorldPosition / SnapGridSize) * SnapGridSize;
```

### 1-Bit Dithering (Post-Process)
```hlsl
// In post-process material:
float Luminance = dot(SceneColor.rgb, float3(0.299, 0.587, 0.114));
float DitherPattern = GetDitherPattern(ScreenUV); // 8x8 Bayer matrix
float DitheredValue = Luminance > DitherPattern ? 1.0 : 0.0;
float3 FinalColor = DitheredValue * PaletteColor;
```

Reference materials in `Content/Materials/PostProcess/`

## Split-Screen Implementation Example

```cpp
// In UGameViewportClient subclass:
void UMySplitScreenViewport::LayoutPlayers()
{
    // Left viewport: Detective camera
    FViewportLayout LeftView;
    LeftView.Offset = FVector2D(0.0f, 0.0f);
    LeftView.Size = FVector2D(0.5f, 1.0f); // 50% width, 100% height

    // Right viewport: Metaverse camera
    FViewportLayout RightView;
    RightView.Offset = FVector2D(0.5f, 0.0f);
    RightView.Size = FVector2D(0.5f, 1.0f);

    // Apply separate post-process settings per viewport
}
```

## Performance Guidelines

- **Target:** 60 FPS minimum on GTX 1660 / RX 580 equivalent
- Disable expensive UE5 features:
  - Lumen (use baked lighting)
  - Nanite (use LODs with low-poly meshes)
  - Virtual Shadow Maps (use traditional shadow maps)
- Set in `Config/DefaultEngine.ini`:
  ```ini
  r.DynamicGlobalIlluminationMethod=0
  r.ReflectionMethod=0
  r.Lumen.HardwareRayTracing=False
  ```

## Common Pitfalls to Avoid

- **Don't** use heavy Blueprint logic in tick events (prefer C++ timers)
- **Don't** enable Nanite/Lumen for this art style (defeats the PSX aesthetic)
- **Don't** add plugins unless absolutely necessary (keep project lean)
- **Don't** hardcode dialogue text in C++ (use data tables)
- **Don't** use dynamic lighting everywhere (bake static lights for performance)

## Testing Checklist

- [ ] Dialogue tree branches correctly based on player choices
- [ ] Metaverse reactions sync with interview screen events
- [ ] Split-screen viewports render independently without visual glitches
- [ ] Custom shaders (PSX + dithering) apply correctly to all objects
- [ ] Controller input works for all supported devices
- [ ] Frame rate stays above 60 FPS in both viewports
- [ ] No memory leaks during long play sessions

## Key Files to Reference

- `Source/UE_SplitScreen/UE_SplitScreen.Build.cs` - Module dependencies
- `Config/DefaultEngine.ini` - Rendering/performance settings
- `Config/DefaultInput.ini` - Input mappings
- `Content/Materials/Master/` - Custom shader logic

## Next Steps for Development

1. **Set up split-screen viewport system** (custom `UGameViewportClient`)
2. **Implement dialogue tree framework** (data-driven with C++ backend)
3. **Create PSX vertex snapping shader** (material function + vertex shader)
4. **Build 1-bit dithering post-process chain** (ordered dithering + palette swapping)
5. **Develop metaverse reaction system** (event-driven behaviors triggered by dialogue)
6. **Set up automated testing pipeline** (unit tests + integration tests)

---

**Project Ethos:** Keep it lean, performant, and narratively focused. Every technical decision should serve the split-screen interrogation concept and the PSX neo-noir aesthetic. Avoid feature creep—this is a tightly scoped narrative experience, not an open-world RPG.
