# Source Engine Deep Dive — Volume 2

> Everything the first guide missed: filesystem internals, model/MDL system, materials and VMTs, input system, tool framework, particles and effects, lighting and shadows, water and fog, save/load, profiling, map compilers, and more.
>
> Same structure: **[BEGINNER]** → **[INTERMEDIATE]** → **[EXPERT]**.

---

# BEGINNER NOTES

---

## 1. The Filesystem — How Source Loads Files [BEGINNER]

### IFileSystem

The entire file I/O abstraction lives in `public/filesystem.h`. The interface version is `"VFileSystem022"`.

**Opening a file:**
```cpp
IFileSystem* g_pFullFileSystem;
FileHandle_t f = g_pFullFileSystem->Open( "maps/de_dust2.bsp", "rb", "GAME" );
int bytesRead = g_pFullFileSystem->Read( buffer, size, f );
g_pFullFileSystem->Close( f );
```

The third parameter to `Open()` is the **path ID** — it tells the filesystem which search path group to look in.

### Search Paths

The filesystem maintains an ordered list of search paths:
```cpp
g_pFullFileSystem->AddSearchPath( "c:/hl2/hl2", "GAME", PATH_ADD_TO_TAIL );
g_pFullFileSystem->AddSearchPath( "c:/hl2/cstrike", "MOD", PATH_ADD_TO_TAIL );
```

**Standard path IDs:** `"GAME"` (content), `"MOD"` (mod override), `"GAMECONFIG"`, `"GAMEBIN"` (DLLs), `"DEFAULT_WRITE_PATH"`, `"LOGDIR"`, `"BSP"` (map pack only).

Paths added with `PATH_ADD_TO_HEAD` are searched first. The first match wins.

### File Open Flow

`OpenForRead()` in `filesystem/basefilesystem.cpp`:
1. Check inline memory file cache
2. If absolute path: check for embedded pack syntax (`path.zip\file`)
3. Iterate search paths in priority order — try pack file (`.zip`), try VPK, try disk file
4. First match returns

### VPK Pack Files

Naming: `pak01_dir.vpk` (directory) + `pak01_000.vpk`, `pak01_001.vpk` (data chunks).

The directory file contains a serialized tree: extension → directory → filename → `[CRC][parts...][0xFFFF]`. Each part is a `CFilePartDescr` (file #, offset, size). `0x7FFF` means data embedded in `_dir.vpk` itself.

Hash table lookup: 43-entry extension hash, 43-entry directory hash per extension.

V1 = marker + version + directory size. V2 adds embedded chunks, MD5 hashes, RSA signature.

### Async File IO

`filesystem/filesystem_async.cpp`: up to 4 IO threads. `CFileAsyncReadJob` extends `CJob`. Custom fetcher support via `IAsyncFileFetch` (CDN/HTTP). Controlled by `async_mode` ConVar, disabled with `-noasync`.

### UNC-Style Path IDs

```cpp
// These are equivalent:
g_pFullFileSystem->Open( "cfg/config.cfg", "r", "GAME" );
g_pFullFileSystem->Open( "//GAME/cfg/config.cfg", "r" );
```

---

## 2. The KeyValues System [BEGINNER]

Valve's universal config format (VMT, game info, entity keyvalues, console auto-complete).

**File:** `public/tier1/KeyValues.h`
```cpp
KeyValues* kv = new KeyValues( "MyConfig" );
kv->LoadFromFile( g_pFullFileSystem, "scripts/myconfig.txt" );
const char* name = kv->GetString( "name", "default" );
int value = kv->GetInt( "value", 0 );
kv->SetString( "name", "hello" );
kv->SaveToFile( g_pFullFileSystem, "output.txt" );
```

### KeyValues Token System

`vstdlib/KeyValuesSystem.cpp` — strings are interned. Each unique string gets a symbolic ID (unsigned short). `CUtlRBTree<CStringPoolIndex>` for fast lookup. `KEYVALUES_TOKEN_SIZE` = 4096 bytes per token.

**Not for perf-critical paths.** Use cvars or direct struct members instead.

---

## 3. Save/Load System [BEGINNER]

### How Saving Works

`engine/save_restore.cpp`. `SAVERESTOREDATA` tracks file state. `CSave::WriteEntity()` serializes each entity's DATADESC fields. Global state (game rules) is also saved.

### DATADESC
```cpp
BEGIN_DATADESC( CBaseEntity )
    DEFINE_FIELD( m_vecOrigin, FIELD_VECTOR ),
    DEFINE_FIELD( m_iHealth, FIELD_INTEGER ),
    DEFINE_FIELD( m_flSpeed, FIELD_FLOAT ),
    DEFINE_INPUT( InputBreak, FIELD_INPUT, "Break" ),
    DEFINE_OUTPUT( m_OnBreak, "OnBreak" ),
END_DATADESC()
```

Field types: `FIELD_INTEGER`, `FIELD_FLOAT`, `FIELD_VECTOR`, `FIELD_STRING`, `FIELD_EHANDLE`, `FIELD_CLASSPTR`, `FIELD_MODELINDEX`, etc.

### Restore Flow

Load file → create entities by classname → `CSaveRestoreBuffer::ReadEntity()` → `PostRestore()`.

---

## 4. The Command Buffer (Cbuf) [BEGINNER]

`engine/cbuf.h`, `engine/cbuf.cpp`:

```cpp
Cbuf_AddText( "sv_cheats 1\n" );       // Queued
Cbuf_AddText( "restart\n" );           // After current
```

`Cbuf_Execute()` processes the buffer — iterates lines, parses commands, looks up ConCommand/ConVar, calls callback or sets value. Supports `;` as separator.

Buffers: text (user-entered), execute (immediate), unbuffered (command-line `-` args).

---

## 5. Localization System [BEGINNER]

Files in `resource/` as `.txt`:
```
"[english]" { "Tokens" { "SFUI_Continue" "Continue" } }
"[japanese]" { "Tokens" { "SFUI_Continue" "続行" } }
```

Usage: `const wchar_t* text = g_pVGuiLocalize->Find( "#SFUI_Continue" );`

Searches current language > English > returns raw token name.

---

## 6. Spew / Debug Output [BEGINNER]

```cpp
Msg( "Standard\n" );        // Always shown
DevMsg( "Dev\n" );          // Requires -dev
Warning( "Warning\n" );     // Always shown
Error( "Fatal!\n" );        // Triggers error dialog
```

`SpewOutputFunc` hooks all output. Set up in `LauncherMain()`. `con_logfile` ConVar writes to file.

---

# INTERMEDIATE NOTES

---

## 7. Model (MDL) System Deep Dive [INTERMEDIATE]

### The .mdl File Structure

Every compiled model starts with `studiohdr_t` (`public/studio.h:2151`, ~280 fields). The format is position-independent.

```
studiohdr_t
├── Bones (mstudiobone_t[])
├── Bone Controllers (mstudiobonecontroller_t[])
├── Hitbox Sets (mstudiohitboxset_t[])
│   └── Hitboxes (mstudiobbox_t[])
├── Animation Descriptions (mstudioanimdesc_t[])
│   └── Per-frame bone data (mstudioanim_t)
├── Sequence Descriptions (mstudioseqdesc_t[])
│   ├── Animation Events (mstudioevent_t[])
│   ├── Auto-layers (mstudioautolayer_t[])
│   └── IK Rules (mstudioikrule_t[])
├── Body Parts (mstudiobodyparts_t[])
│   └── Models (mstudiomodel_t[])
│       └── Meshes (mstudiomesh_t[])
│           └── Vertices (mstudiovertex_t[])
├── Textures (mstudiotexture_t[])
├── Skin Reference Tables (short[])
├── Flex Descriptions (mstudioflexdesc_t[])
├── Flex Controllers (mstudioflexcontroller_t[])
├── Flex Rules (mstudioflexrule_t[] + RPN ops)
├── Attachments (mstudioattachment_t[])
├── IK Chains (mstudioikchain_t[])
└── Key-Value Block
```

### Accompanying Files

| File | Ext | Content |
|------|-----|---------|
| MDL | `.mdl` | Skeleton, animation, mesh metadata |
| VVD | `.vvd` | Vertex data (pos, normal, UV, bone weights) |
| VTX | `.vtx` | Optimized triangle strips with LOD hierarchy |
| PHY | `.phy` | Physics collision mesh |

Checksums must match across files.

### Flex / Facial Animation

`mstudioflex_t` (line 1147): per-vertex deltas in fixed-point or float16. Flex rules use RPN (reverse Polish notation): CONST, FETCH1/2, ADD, SUB, MUL, DIV, NEG, EXP, MAX, MIN, 2WAY, NWAY, COMBO, DOMINATE.

### studiomdl — The Model Compiler

`utils/studiomdl/studiomdl.cpp` (~10,400 lines). QC script controls compilation:
```
$modelname "player/player"
$body mybody "models/player_body.smd"
$bodygroup "heads" { studio "models/head1.smd" studio "models/head2.smd" }
$sequence "idle" "anim_idle.smd"
$sequence "walk" "anim_walk.smd" loop
$attachment "eyes" "head" 0 2.0 4.0
$collisionmodel "models/player_physics.smd"
```

### The Model Renderer

`CStudioRender` (`studiorender/r_studiodraw.cpp`):
1. `DrawModel()` > `R_StudioSetupModel()` > `R_StudioRenderFinal()`
2. `GenerateMorphAccumulator()` for GPU flex morphs
3. `R_StudioDrawPoints()` — iterate meshes, resolve skins, handle eyeballs
4. `ComputeSkinMatrix()` — up to 3 bone influences per vertex

### Model Caching

`datacache/mdlcache.cpp` — `studiodata_t` with DataCache handle, hardware LOD data, virtual model for multi-part models, demand-loaded animation blocks.

---

## 8. VMT / Material System Deep Dive [INTERMEDIATE]

### VMT File Format

Parsed by KeyValues in `cmaterial.cpp:3526`:
```
VertexLitGeneric
{
    $basetexture    "metal/metalplate001"
    $bumpmap        "metal/metalplate001_normal"
    $envmap         "env_cubemap"
    $phong          "1"
    $phongexponent  "20"
    $color          "{255 200 100}"
}
```

### Conditional Vars

`lowfill?$basetexture "detail/lowres_metal"` — only applies when fill rate reduction is active.
`hdr?$envmap "env/hdr_cubemap"` — only in HDR.
`!hdr?$color "[1 1 1]"` — only when NOT in HDR.
Other prefixes: `ldr?`, `srgb?`, `360?`, `gameconsole?`, `dx90?`

### Patch Files
```
"patch" { "include" "base_material" "replace" { "$basetexture" "custom/path" } }
```

### IMaterialVar Types

`MaterialVarType_t` (`imaterialvar.h:34`): `FLOAT`, `INT`, `STRING`, `VECTOR`, `TEXTURE`, `MATERIAL`, `FOURCC`, `MATRIX`.

### Material Proxies

C++ callbacks that modify params per frame, defined inside VMT:
```
Proxies { AnimatedTexture { animatedTextureVar "$basetexture" frameRate 15 } }
```
Common: `AnimatedTexture`, `Sine`, `EntityInput`, `PlayerProximity`. Implement `IMaterialProxy`: `Init()` once, `OnBind()` per frame.

### VTF Texture Format

`public/vtf/vtf.h` (version 7.5). Contains: magic "VTF", version, dimensions, flags (`TEXTUREFLAGS_*`), image format, mip levels, frame count, cubemap flag, resource entries.

**Image Formats** (`bitmap/imageformat.h`):
- `IMAGE_FORMAT_DXT1` — 4:1, no alpha
- `IMAGE_FORMAT_DXT5` — 4:1, interpolated alpha
- `IMAGE_FORMAT_RGBA8888` — 32-bit
- `IMAGE_FORMAT_RGBA16161616F` — 64-bit HDR float
- `IMAGE_FORMAT_ATI2N` — normal map compression

**Cubemap face order:** RIGHT(+X), LEFT(-X), BACK(+Y), FRONT(-Y), UP(+Z), DOWN(-Z), SPHEREMAP.

### Shader Combinatorial System

**Static combos** (compiled variants): `"BUMPMAP" "0..1"` > separate compiled shader per combo.
**Dynamic combos** (per-draw uniforms): `"PIXELFOGTYPE" "0..1"` > set at draw time.
**SKIP rules** exclude invalid combinations.

C++: `SET_STATIC_PIXEL_SHADER_COMBO(BUMPMAP, 1)` / `SET_DYNAMIC_PIXEL_SHADER_COMBO(PIXELFOGTYPE, 0)`.

### Material Sorting

By lightmap page ID > enumeration ID > pass type (opaque before translucent, no-Z last). `materialsystem/cmatlightmaps.cpp`.

---

## 9. Input System Deep Dive [INTERMEDIATE]

### IInputSystem

`public/inputsystem/iinputsystem.h`, `inputsystem/inputsystem.cpp`:

```cpp
g_pInputSystem->PollInputState();
bool wDown = g_pInputSystem->IsButtonDown( KEY_W );
int mouseDelta = g_pInputSystem->GetAnalogDelta( MOUSE_X );
```

### Button Codes

Flat integer namespace (`public/inputsystem/ButtonCode.h`):
- `KEY_0..KEY_9`, `KEY_A..KEY_Z`, `KEY_F1..KEY_F12`
- `MOUSE_LEFT/MOUSE_RIGHT/MOUSE_MIDDLE/MOUSE_4/MOUSE_5/MOUSE_WHEEL_UP/DOWN`
- `JOYSTICK_BUTTON(joy, btn)` / `JOYSTICK_POV_BUTTON(joy, pov)` / `JOYSTICK_AXIS_BUTTON(joy, axis)`
- `STEAMCONTROLLER_BUTTON(joy, btn)` / `STEAMCONTROLLER_AXIS_BUTTON(joy, axis)`

### Analog Codes

`AnalogCode_t`: `MOUSE_X = 0`, `MOUSE_Y`, `MOUSE_XY`, `MOUSE_WHEEL`, `JOYSTICK_AXIS(joy, axis)`.

### Doubled-State Buffer

Two `InputState_t` buffers: QUEUED and CURRENT. `PollInputState()` copies QUEUED > CURRENT, processes events, copies back without events.

### Joystick / Gamepad

SDL2 (`inputsystem/joystick_sdl.cpp`): `SDL_CONTROLLERAXISMOTION`, `SDL_CONTROLLERBUTTONDOWN/UP`, `SDL_CONTROLLERDEVICEADDED/REMOVED`. Axis deadzone from `joy_axis_deadzone` (0.2). Rumble via `SDL_HapticRumblePlay()`.

### Steam Controller

`inputsystem/steamcontroller.cpp`: Up to 8 controllers. Action sets (MenuControls, FPSControls, InGameHUD, Spectator). Digital actions mapped to ButtonCode_t. Debounce on action set change.

---

## 10. The Tool Framework [INTERMEDIATE]

### IToolSystem

In-engine tools (Foundry, PET, VMT Editor) loaded at runtime:
```cpp
class IToolSystem : public IAppSystem {
    virtual void Think( bool finalTick ) = 0;
    virtual void ServerFrameUpdatePreEntityThink() = 0;
    virtual void ClientPreRender() = 0;
    virtual void SetupEngineView( Vector& origin, QAngle& angles, ... ) = 0;
    virtual bool TrapKey( ButtonCode_t key, bool down ) = 0;
};
```

`IToolFrameworkInternal` manages all loaded tools: `GetToolCount()`, `SwitchToTool()`, `InToolMode()`.

`BaseToolSystem` (`tools/toolutils/BaseToolSystem.cpp`): manages UI, key bindings, `TrapKey()` for ESC/tilde.

| Tool | Directory | Purpose |
|------|-----------|---------|
| Foundry | `tools/foundry/` | In-game level editor |
| PET | `tools/pet/` | Particle editing |
| VMT Editor | `tools/vmt/` | Material editing |

### Server Plugin System

`IServerPluginCallbacks` (`public/engine/iserverplugin.h`):
```cpp
class IServerPluginCallbacks {
    virtual bool Load( CreateInterfaceFn interfaceFactory, ... ) = 0;
    virtual void GameFrame( bool simulating ) = 0;
    virtual PLUGIN_RESULT ClientConnect( bool* bAllowConnect, edict_t*, ... ) = 0;
};
```
Return: `PLUGIN_CONTINUE` / `PLUGIN_OVERRIDE` / `PLUGIN_STOP`.

`IServerPluginHelpers`: `CreateMessage()` (dialog), `ClientCommand()`, `StartQueryCvarValue()`.

---

## 11. Particles and Effects [INTERMEDIATE]

### Two Particle Systems

**Old** (`game/client/particlemgr.h`): `CParticleMgr` singleton, `IParticleEffect`, manual 96-byte particles, bucket-sorted by Z.

**New** (`public/particles/particles.h`): `CParticleSystemMgr`, `CParticleSystemDefinition` from `.pcf` files, `CParticleCollection` with SIMD SOA storage, operators (initializers, emitters, renderers, forces, constraints).

### Simple Emitter Hierarchy
```
CSimpleEmitter > CEmberEffect, CFireSmokeEffect, CFireParticle, CExplosionParticle
```

### Beam Rendering

`game/client/beamdraw.cpp`: `DrawSegs()` (standard), `DrawTeslaSegs()` (lightning), `DrawSplineSegs()` (Catmull-Rom), `DrawHalo()`, `DrawSprite()`, `DrawDisk()`, `DrawCylinder()`, `DrawRing()`, `DrawBeamFollow()`, `DrawBeamQuadratic()`.

### Effect Dispatch

```cpp
DECLARE_CLIENT_EFFECT( "RagdollImpact", RagdollImpactCallback );
```
Server fires name > client looks up in string table > calls callback.

### Decal System

`C_TEDecal` (entity/static prop), `C_TEBSPDecal` (BSP geometry), `C_TEWorldDecal` (world by origin), `C_TEProjectedDecal` (with rotation).

### Rope System

`rope_physics.h`: `CRopePhysics<N>`, spring constraint solver (3 iterations), gravity 1500, wind forces, 0.98 damping.

---

## 12. Lighting and Shadows [INTERMEDIATE]

### Dynamic Lights

`dlight_t` in `engine/gl_lightmap.cpp`. Added by `engine->LightSprite()`. Inverse-square + linear falloff to radius. Capped at `r_maxdlights` (32). Accumulated into `blocklights[][]`. Bumped: 4 channels (flat + 3 bump).

### Lightcache

`engine/lightcache.cpp`: 200-entry LRU, spatial hashing (5/5/7-bit grid). `lightcache_t` with static + lightstyle + dynamic + env_cubemap. Up to 162 random sampling directions.

### Shadow Mapping

`engine/shadowmgr.cpp` (3774 lines): `ProjectShadow()` (sphere-culled entity shadows) and `ProjectFlashlight()` (frustum-culled). Sutherland-Hodgman 2D clipping. Vertex caching (SMALL=8, LARGE=32, TEMP=48).

### HDR Pipeline

Three modes: NONE / INTEGER / FLOAT. Tonemapping via exponential smoothing. Rate limited to (1/16)*0.25 per frame. Bloom as post-process.

---

## 13. Water Rendering [INTERMEDIATE]

```cpp
struct WaterRenderInfo_t {
    bool m_bCheapWater, m_bReflect, m_bRefract;
    bool m_bReflectEntities, m_bDrawWaterSurface, m_bOpaqueWater;
};
```

Cheap = no reflection/refraction. Expensive = reflection + refraction render passes. Default cheap distance: 500-1000 units.

Reflection: inverted camera to `_rt_WaterReflection`. Refraction: normal camera to `_rt_WaterRefraction`. Underwater: fog, distortion, optional overlay.

---

## 14. Fog System [INTERMEDIATE]

```cpp
struct fogparams_t { Vector dirPrimary; color32 colorPrimary, colorSecondary;
    float start, end, farz, maxdensity; bool enable, blend; };
```

`CFogController` entity with master controller per level. Inputs: `SetStartDist`, `SetColor`, `StartFogTransition`. Types: distance fog (linear), height fog (vertical), directional blend (dot product).

Skybox fog with scaled distances. Override ConVars: `fog_override`, `fog_start`, `fog_end`, `fog_color`.

---

## 15. VGUI Scheme and .res Files [INTERMEDIATE]

Scheme defines colors, fonts, borders. Applied in `ApplySchemeSettings(IScheme*)`. `.res` files laid out via `LoadControlSettings("resource/MyPanel.res")` with `xpos`, `ypos`, `wide`, `tall`, `ControlName`, `text`, `command`.

---

# EXPERT NOTES

---

## 16. VPK Format Internals [EXPERT]

Directory data: `[CRC32:4][MetaSize:2][PartDesc...][0xFFFF:2][MetaData]`. Each `CFilePartDescr`: `[FileNumber:2][Offset:4][Size:4]`. Terminated by `0xFFFF`. File number `0x7FFF` = embedded in `_dir.vpk`.

Hash tables: 43 ext buckets, 43 dir buckets per ext. O(1) lookup.

Read cache: 1 MB chunk-based. V2 adds: embedded chunks, MD5 chain (directory > hashes > total), RSA-SHA256 signature.

---

## 17. SIMD Particles [EXPERT]

SOA format: `float* m_X, *m_Y, *m_Z, *m_PrevX, *m_PrevY, *m_LifeDuration, *m_Radius, *m_Rotation, *m_TintR...` — all `fltx4` arrays for 4-at-once SSE.

Attribute iterators: `CM128AttributeIterator` (fltx4*), `C4VAttributeIterator` (FourVectors*).

Threaded: `r_threaded_particles`, `CParticleMgr::UpdateNewEffects()` distributes to worker threads via `ProcessPSystem()`.

---

## 18. Threaded Rendering (Queued Context) [EXPERT]

`CMatQueuedRenderContext` records render state changes as call commands. Replayed on render thread. `IMaterialVar` thread-safe clones via `EnableThreadedMaterialVarAccess()`. `s_pTempMaterialVar[]` temp slots. Proxies wrapped with thread-safe access guards.

---

## 19. Map Compilers (vbsp, vrad, vvis) [EXPERT]

**vbsp:** VMF > BSP. Pipeline: load VMF, process brushes, CSG carve, BSP tree, leaf creation, portals, textures, entities, cubemap assignment.

**vrad:** Radiosity lighting. Patches > shoot lights > gather > write lightmaps. HDR output. Hemispherical sampling.

**vvis:** PVS computation. Portal flow > flood fill > cluster merge > RLE compression.

**VMPI:** Network-distributed compilation (TCP job queue).

---

## 20. VProf Profiler [EXPERT]

`VPROF("DrawModel")` creates scoped `CVProfNode`. Records QPC timestamps. Overlay sorted by total/self time/call count.

Groups: `VPROF_BUDGETGROUP_WORLD_RENDERING`, `_ENTITIES`, `_PHYSICS`, `_PARTICLES`, `_AI`, `_NETWORKING`.

ConVars: `vprof`, `vprof_graph`, `vprof_verbose`, `vprof_dump_on_interval`, `vprof_dump_spikes`.

---

## 21. Crash Handling [EXPERT]

`SpewOutputFunc` global hook. `Error()` > `SetAllocFailHandler()` > debug break/exit. Minidump via `MiniDumpWriteDump()` (Windows) or custom stack trace (Linux) > `debug/` directory.

---

## 22. Cross-Platform Portability [EXPERT]

Platform abstraction table: `LoadLibraryEx`/`dlopen`, `CreateThread`/`pthread_create`, `TlsGetValue`/`pthread_getspecific`, `_InterlockedIncrement`/`__sync_fetch_and_add`, `__m128` (SSE) / NEON (Android).

Graphics: DirectX 9 (`shaderapidx9/`), OpenGL (`togl/` ToGL), OpenGL ES (`togles/`). Same `IShaderAPI` interface.

---

## Quick Reference

| Topic | Key Files |
|-------|-----------|
| Filesystem | `public/filesystem.h`, `filesystem/basefilesystem.cpp`, `vpklib/packedstore.cpp` |
| MDL/Model | `public/studio.h`, `studiorender/r_studiodraw.cpp`, `utils/studiomdl/studiomdl.cpp` |
| Materials/VMT | `public/materialsystem/imaterial.h`, `materialsystem/cmaterial.cpp`, `public/vtf/vtf.h` |
| Input | `public/inputsystem/iinputsystem.h`, `inputsystem/inputsystem.cpp`, `inputsystem/joystick_sdl.cpp` |
| Tools | `public/toolframework/itoolsystem.h`, `tools/toolutils/BaseToolSystem.cpp` |
| Particles | `game/client/particlemgr.h`, `public/particles/particles.h`, `game/client/fx_*.cpp` |
| Beams | `game/client/beamdraw.cpp`, `game/client/c_te_basebeam.cpp` |
| Lighting | `engine/gl_lightmap.cpp`, `engine/lightcache.cpp`, `engine/shadowmgr.cpp` |
| Water | `game/client/viewrender.cpp`, `materialsystem/stdshaders/water.cpp` |
| Fog | `game/server/fogcontroller.cpp`, `game/shared/playernet_vars.h` |
| Save/Load | `engine/save_restore.cpp` |
| KeyValues | `public/tier1/KeyValues.h`, `vstdlib/KeyValuesSystem.cpp` |
| Cbuf | `engine/cbuf.h`, `engine/cbuf.cpp` |
| Localization | `vgui2/src/vlocalize.cpp` |
| VGUI Scheme | `public/vgui_controls/Panel.h`, `vgui2/vgui_controls/Panel.cpp` |
| Profiler | `public/tier1/vprof.h`, `engine/vprof.cpp` |
| Map Compilers | `utils/vbsp/vbsp.cpp`, `utils/vrad/vrad.cpp`, `utils/vvis/vvis.cpp` |
