# Source Engine Deep Study Notes

> A beginner-friendly guide that gets progressively technical — designed for aspiring game programmers targeting Valve's Source Engine codebase.
>
> Repository: [nillerusr/source-engine](https://github.com/nillerusr/source-engine) (TF2 2018 leak–based, community-maintained)
> ~10,771 source files (`.cpp` + `.h`) across 83 top-level directories.

---

## How to Use This Guide

Each section is labeled **[BEGINNER]**, **[INTERMEDIATE]**, or **[EXPERT]** so you can skip around. File paths are relative to the repo root. Read the beginner sections linearly, then pick intermediate/expert topics based on what interests you.

The best way to learn: open the file paths listed, read the actual source, and experiment with modifications.

---

# BEGINNER NOTES

---

## 1. Codebase at a Glance

```
source-engine/
├── tier0/          — Lowest layer: memory, threading, CPU detection, debugging
├── tier1/          — Common: bitbuf, KeyValues, string tools, ConVars, containers
├── tier2/          — Mid-level: file utils, render utils, sound utils
├── tier3/          — High-level: choreo, MDL utils
├── vstdlib/        — "Standard lib": coroutines, jobs, KeyValues system, random
├── appframework/   — App system lifecycle (IAppSystem, IAppSystemGroup)
├── engine/         — Core engine DLL (~540 files). Client/server engine, renderer,
│                     networking, demo, filesystem, tools. Contains subsystems like
│                     audio/ and voice_codecs/.
├── game/
│   ├── server/     — Server-side game DLL (~602 files). Entities, AI, weapons,
│   │                  physics, HL2/TF2/CS:S/DOD/Portal code per subdirectory.
│   ├── client/     — Client-side game DLL (~561 files). Rendering, prediction,
│   │                  effects, HUD, view, mirroring server entity hierarchy.
│   └── shared/     — Code shared between client and server (~264 files).
├── public/         — ~1,146 shared header files. THE most important directory for
│                   understanding interfaces. Mirrors the tier structure with
│                   public/tier0, public/tier1, etc.
├── materialsystem/ — Material/shader system (~363 files). DirectX 9 + OpenGL paths.
├── vgui2/          — VGUI 2 UI framework (~189 files).
├── vguimatsurface/ — Material-system-based VGUI surface rendering.
├── mathlib/        — Vector/matrix math library. SSE-optimized paths.
├── vphysics/       — Physics system wrapper (around IVP/Havana).
├── ivp/            — IVP physics engine + Havok integration.
├── networksystem/  — UDP networking, channels.
├── soundsystem/    — Audio playback and mixing.
├── filesystem/     — File I/O abstraction (Stdio, Steam).
├── inputsystem/    — Keyboard, mouse, joystick, touch, Steam Controller.
├── particles/      — Particle system library.
├── datamodel/      — Attribute/element data model for tools.
├── replay/         — Demo recording/playback system.
├── hammer/         — Hammer world editor (~461 files, MFC-based).
├── launcher/       — Game executable launcher + CSourceAppSystemGroup.
├── common/         — Cross-module shared code (netmessages, steam, lua, protobuf).
├── utils/          — Standalone tools (vbsp, vrad, vvis, studiomdl, vtex, vpk, etc.).
├── thirdparty/     — SDL2, curl, freetype, libpng, zlib, protobuf, OpenAL, stb.
└── devtools/ + vpc_scripts/ — Build configuration and tooling.
```

**Naming conventions:**
- `c_` prefix = client-side entity (e.g., `c_baseentity.cpp`)
- `cl_` prefix = client engine code (e.g., `cl_main.cpp`)
- `sv_` prefix = server engine code (e.g., `sv_main.cpp`)
- `r_` prefix = renderer code (e.g., `r_rmain.cpp`)
- `gl_` prefix = OpenGL/D3D abstraction layer
- `te_` prefix = temp entity effects (e.g., `te_explosion.cpp`)
- `dt_` prefix = datatable/send proxy code
- `hud_` prefix = HUD elements

---

## 2. C++ Class Basics in Source

### The `abstract_class` Macro

In `public/tier0/platform.h`, Valve defines:

```cpp
#define abstract_class class
```

This is purely a documentation hint — it has no compiler effect. When you see:

```cpp
abstract_class IAppSystem { ... };
```

It means "this is an interface with pure virtual methods." All interfaces in the engine follow this convention.

### `DECLARE_CLASS` Pattern

Every game entity uses this macro to set up the inheritance chain:

```cpp
// game/server/baseentity.h:347
class CBaseEntity : public IServerEntity
{
    DECLARE_CLASS_NOBASE( CBaseEntity );
    DECLARE_PREDICTABLE();
    DECLARE_SERVERCLASS();
    DECLARE_DATADESC();
    ...
};
```

For derived classes:

```cpp
class CBasePlayer : public CBaseEntity
{
    DECLARE_CLASS( CBasePlayer, CBaseEntity );
    ...
};
```

This macro creates:
- A typedef `BaseClass1` pointing to the parent
- Function `GetClassname()` for runtime type identification
- Support for `dynamic_cast`-like casting via `MyEntityPointer->CastTo<CBasePlayer>()`

### The `abstract_class` + interface pattern

Interfaces are pure virtual classes with no data members:

```cpp
// public/vphysics_interface.h
abstract_class IPhysicsEnvironment {
    virtual void SetGravity(const Vector& gravity) = 0;
    virtual void GetGravity(Vector* pGravity) const = 0;
    virtual IPhysicsObject* CreatePolyObject(...) = 0;
    // ...more pure virtuals...
};
```

The implementation is hidden inside the DLL — you only get the interface pointer. This is how the engine stays modular.

---

## 3. The Interface System (How DLLs Talk to Each Other)

Source Engine doesn't use COM, but it follows a similar pattern. Every DLL exports a single function:

```cpp
// public/tier1/interface.h:68
typedef void* (*CreateInterfaceFn)(const char *pName, int *pReturnCode);
```

### Registration

Interfaces register themselves automatically:

```cpp
// public/tier1/interface.h:72-83
class InterfaceReg {
    InstantiateInterfaceFn m_CreateFn;
    const char *m_pName;
    InterfaceReg *m_pNext;
    static InterfaceReg *s_pInterfaceRegs;  // linked list head
};
```

### Export Macros

```cpp
// Creates a new instance each time
EXPOSE_INTERFACE( className, interfaceName, versionName )

// Exposes a global singleton
EXPOSE_SINGLE_INTERFACE_GLOBALVAR( className, interfaceName, versionName, globalVar )
```

### Loading a Module

In `tier1/interface.cpp`:

1. `Sys_LoadModule("engine" DLL_EXT_STRING)` → calls `LoadLibraryEx()` / `dlopen()`
2. `Sys_GetFactory(module)` → gets `CreateInterface` function pointer via `GetProcAddress`
3. Call factory to get the interface: `g_pEngine = (IEngine*)factory(VENGINE_CLIENT_INTERFACE_VERSION, NULL)`

This is how the launcher loads the engine, the engine loads the game DLLs, and everyone finds everyone else.

---

## 4. The Host / Frame Loop

### Startup Chain

```
WinMain() → LauncherMain()
  → CSourceAppSystemGroup().Run()
    → Create() — loads engine.dll, inputsystem.dll, materialsystem.dll, etc.
    → ConnectSystems() — calls Connect() on each IAppSystem
    → InitSystems() — calls Init() on each IAppSystem
      → Host_Init() — initializes all engine subsystems
    → Main() → g_pEngineAPI->Run()
      → Sys_InitGame() → Host_InitGame()
```

### The Frame Loop

In `engine/host.cpp`, `Host_Frame()` runs every tick:

```
Host_Frame(float time):
  1. Host_State() — state machine (loading, running, changing level, quitting)
  2. Host_FrameTime() — computes frametime, clamps it
  3. NET_UpdateNetChan() — process network traffic
  4. Host_UpdateScreen() — renders the frame
  5. Sound update, audio mix
```

### `gpGlobals`

Defined in `public/game/shared/shareddefs.h`:

```cpp
struct CGlobalVars {
    float     curtime;       // current time in seconds
    float     frametime;     // time for this frame
    int       maxClients;
    int       tickcount;     // current tick number
    float     interval_per_tick;
    float     framecount;
    ...
};
extern CGlobalVars *gpGlobals;
```

Used everywhere in game code for timing, simulation, and frame counting.

---

## 5. ConVars and ConCommands

### Creating a ConVar

```cpp
// game/shared/gamemovement.cpp:46
ConVar xc_uncrouch_on_jump( "xc_uncrouch_on_jump", "0", FCVAR_REPLICATED );
```

The constructor: `ConVar( name, default_value, help_string, flags )`.

### Key Flags

Defined in `public/tier1/convar.h`:

| Flag | Value | Purpose |
|------|-------|---------|
| `FCVAR_CHEAT` | (1<<7) | Requires `sv_cheats 1` |
| `FCVAR_REPLICATED` | (1<<10) | Sent from server to all clients |
| `FCVAR_ARCHIVE` | (1<<5) | Saved to `config.cfg` |
| `FCVAR_NOT_CONNECTED` | (1<<3) | Can be changed when not in a server |
| `FCVAR_DEVELOPMENTONLY` | (1<<12) | Debug/cheat flag, hidden in release |
| `FCVAR_GAMEDLL` | (1<<11) | Only exists in the game DLL |

### ConCommand (for actions / functions)

```cpp
ConCommand cmd_test( "mycommand", MyCallback, "Does something", FCVAR_CHEAT );
```

The callback: `void MyCallback( const CCommand &args )`.

### Where the Console Lives

- `public/tier1/convar.h` — ConVar and ConCommand class definitions
- `vstdlib/cvar.cpp` — `CCvar` implementation (the actual ICvar interface)
- `engine/cvar.cpp` — Engine-side cvar queries (like `CVar_GetFloat()`)
- `vstdlib/KeyValuesSystem.cpp` — backing for cvar auto-complete

---

## 6. The Entity System

### `CBaseEntity`

The root of all game objects, defined in `game/server/baseentity.h:344` (2717 lines).

**Lifecycle:**
1. `CreateEntityByName("my_entity")` — static factory, allocates via `new`
2. `DispatchSpawn()` — calls `Spawn()`, sets up model, collision, network state
3. `Activate()` — called after all entities exist, resolves pointers
4. `Think()` — called each frame (if `SetNextThink()` was called)
5. `Event_Killed()` / `Death()` — cleanup
6. `UTIL_Remove()` — schedules for removal

**Key members:**

```cpp
// Entity identity and networking
CServerNetworkProperty  m_Network;           // links to engine's edict
string_t                m_iClassname;        // "npc_combine", "prop_dynamic", etc.
CNetworkVar( short,     m_nModelIndex );     // model to render
CNetworkVar( Vector,    m_vecOrigin );       // position
CNetworkVar( QAngle,    m_angRotation );     // rotation

// Think system
BASEPTR m_pfnThink;                          // function pointer for Think()
void SetNextThink( float nextThink );        // schedule next think

// Physics
CNetworkHandle( CBaseEntity, m_hMoveParent ); // parent in hierarchy
int     m_iEFlags;                           // engine flags (EFL_DIRTY, etc.)
int     m_fFlags;                            // FL_ONGROUND, FL_INWATER, etc.
```

### Edicts

In `public/edict.h`, `edict_t` is the engine's slot for an entity:

```cpp
struct edict_t : public CBaseEdict {
    IServerNetworkable  *m_pNetworkable;     // points to CServerNetworkProperty
    IServerEntity       *m_pEntity;          // points to CBaseEntity
};
```

Every entity has a unique edict index (`entindex()`). The engine uses edicts to manage network state, not the game code.

---

## 7. Vectors and Math

### Core Types (in `public/mathlib/`)

```cpp
typedef float vec_t;                    // base type
class Vector  { vec_t x, y, z; };      // 3D position/direction
class QAngle  { vec_t x, y, z; };      // pitch, yaw, roll (in degrees)
class Vector2D { vec_t x, y; };
class Vector4D { vec_t x, y, z, w; };
class Quaternion { vec_t x, y, z, w; }; // rotation as quaternion
```

### Common Operations

```cpp
Vector pos  = GetAbsOrigin();
Vector fwd;
AngleVectors( viewAngles, &fwd );       // QAngle → direction vectors

Vector delta = target - pos;
float dist  = delta.Length();           // vector magnitude
float dist2 = delta.LengthSqr();        // cheaper, no sqrt
float dot   = fwd.Dot( target );        // dot product

// Lerp between two values
float lerped = Lerp( 0.5f, start, end );  // 0.5 = halfway
```

### Function Reference

All in `public/mathlib/mathlib.h`:
- `AngleVectors()`, `VectorAngles()` — angle ↔ direction conversions
- `Matrix3x4` operations — bone transforms
- `SmoothCurve()`, `SimpleSpline()`, `Hermite_Spline()` — interpolation curves
- `IntersectRayWithTriangle()`, `IntersectRayWithPlane()` — collision

---

## 8. Basic Containers (Why No STL)

Valve doesn't use STL containers. They wrote their own, found in `public/tier1/`:

### `CUtlVector<T>` (utlvector.h)

A dynamic array. Like `std::vector` but with a different growth strategy.

```cpp
CUtlVector<Vector> points;
points.AddToTail( Vector(0,0,0) );      // append
points.InsertBefore( 2, myVec );        // insert at index, shifts elements
points.FastRemove( 3 );                 // remove by swapping with last (O(1), no order)
points.Remove( 3 );                     // remove with shift (O(n), preserves order)
points.Purge();                         // free all memory
```

**Growth:** When capacity is exceeded, it doubles (or grows by `m_nGrowSize` if set). No geometric growth control like `std::vector::reserve()` — but there IS `EnsureCapacity()`.

### `CUtlString` (utlstring.h)

A wrapper around `char*`. No small-string optimization (always heap-allocated):

```cpp
CUtlString name = "hello";
name += " world";
const char* cstr = name.Get();           // get raw pointer
```

### `CUtlBuffer` (utlbuffer.h)

A byte buffer with both read (`m_Get`) and write (`m_Put`) cursors:

```cpp
CUtlBuffer buf;
buf.PutString( "hello" );
buf.PutInt( 42 );
buf.SeekGet( SEEK_HEAD, 0 );
const char* s = buf.GetString();
int i = buf.GetInt();
```

Flags: `TEXT_BUFFER`, `EXTERNAL_GROWABLE`, `READ_ONLY`.

### `CUtlSymbol` (utlsymbol.h)

String interning — "hello" maps to a small integer ID. Used everywhere for classnames, model names, etc. `UtlSymId_t` is `unsigned short`.

### Why Not STL?

- Deterministic allocation patterns (critical for a real-time game)
- Better debugging support (the `memdbgon.h` system can track every allocation)
- Smaller code size / faster compile times (2001-era compilers had slow template support)
- Control over memory: custom allocators per-container (`CUtlMemory<T>` template parameter)

---

## 9. Entity I/O (Inputs & Outputs)

Entities communicate through a named input/output system.

### Defining I/O

In a class's `DATADESC` block:

```cpp
// Input: a function that receives a variant_t
void InputBreak( inputdata_t &data );

// Output: a named event other entities can listen to
COutputEvent m_OnDamaged;
```

### AcceptInput

When an input fires, the engine calls `CBaseEntity::AcceptInput()` (`game/server/baseentity.cpp`). It looks up the input name in the class's DataDesc map, then calls the associated function.

### `variant_t`

Input parameters come as `variant_t` — a tagged union:

```cpp
class variant_t {
    fieldtype_t m_fieldType;   // FIELD_INTEGER, FIELD_FLOAT, FIELD_STRING, FIELD_VECTOR, etc.
    union {
        int     iVal;
        float   fVal;
        string_t iszVal;
        Vector  vecVal;
        color32 rgbVal;
        ...
    };
};
```

### Linking

In Hammer, you connect outputs to inputs:
```
[OnDamaged] → [Break] of prop_door
```

This creates an `CEventAction` that the engine follows at runtime.

---

# INTERMEDIATE NOTES

---

## 10. The Client/Server Split

### Mirror Architecture

The `game/` directory has three subdirectories:

| Directory | Purpose | Key Class |
|-----------|---------|-----------|
| `game/server/` | Server-side logic | `CBaseEntity` |
| `game/client/` | Client-side rendering/prediction | `C_BaseEntity` |
| `game/shared/` | Both | Included from both DLLs |

### `C_` Prefix Convention

For every entity class on the server, there's a client counterpart:

| Server | Client |
|--------|--------|
| `CBaseEntity` | `C_BaseEntity` |
| `CBasePlayer` | `C_BasePlayer` |
| `CBaseCombatWeapon` | `C_BaseCombatWeapon` |

The client version handles rendering, interpolation, and prediction. The server version handles authority, physics, AI.

### Shared Code

Files in `game/shared/` are compiled into BOTH the client and server DLLs. The common pattern:

```cpp
// game/shared/baseplayer_shared.h:57-62
#ifdef CLIENT_DLL
    #define CBasePlayer C_BasePlayer
#endif
```

This macro swaps the class name depending on which DLL is compiling, so shared code can refer to `CBasePlayer` regardless.

### Data That Crosses the Boundary

- **Networked variables** (`CNetworkVar`) — automatically synchronized
- **Temp entities** (`TE_*`) — server fires, client renders
- **Usercmds** (`CUserCmd`) — client sends button/input state
- **Game events** — server fires named events, client handles them
- **Prediction** — client simulates movement locally, server corrects

---

## 11. Network Variables (SendTable / RecvTable)

### How Data Gets from Server to Client

**On the server side:**

```cpp
// Each entity declares what it sends
IMPLEMENT_SERVERCLASS_ST( CBaseEntity, DT_BaseEntity )
    SendPropInt( SENDINFO(m_nModelIndex), 10, SPROP_UNSIGNED ),
    SendPropVector( SENDINFO(m_vecOrigin) ),
    SendPropQAngles( SENDINFO(m_angRotation) ),
    SendPropEHandle( SENDINFO(m_hMoveParent) ),
    // ...
END_SEND_TABLE()
```

**On the client side:**

```cpp
IMPLEMENT_RECVCLASS_ST( C_BaseEntity, DT_BaseEntity )
    RecvPropInt( RECVINFO(m_nModelIndex) ),
    RecvPropVector( RECVINFO(m_vecOrigin) ),
    RecvPropQAngles( RECVINFO(m_angRotation) ),
    RecvPropEHandle( RECVINFO(m_hMoveParent) ),
    // ...
END_RECV_TABLE()
```

### `CNetworkVar`

The magic that triggers dirty tracking:

```cpp
// public/networkvar.h
CNetworkVar( Vector, m_vecOrigin );   // Client receives Vector, change detected
CNetworkVar( int, m_iClip1 );         // ammo count, networked
CNetworkArray( float, m_flBonePos, 16 );  // array of 16 floats
```

When you assign to a `CNetworkVar`, it calls `NetworkStateChanged()` on the owning entity, which marks the edict as dirty, which tells the engine to include this property in the next delta update.

### SendProxies and RecvProxies

Proxies transform data between game types and network types:

```cpp
// Server sends entity handles as integers
SendProxy_EHandleToInt

// Client receives integers as entity handles
RecvProxy_IntToEHandle
```

Other common proxies:
- `SendProxy_Color32ToInt` / `RecvProxy_IntToColor32`
- `SendProxy_AnimTime` — encodes as tick delta (8 bits)
- `SendProxy_Origin` — handles step simulation

Defined in `game/server/sendproxy.h` and `game/client/recvproxy.h`.

### Delta Compression

The engine tracks which fields changed since the last update. Each tick, it sends:
- A bitmask of changed fields (per-entity)
- Only the values that changed
- This is why `StateChanged()` takes an optional offset — per-field granularity

### Datatable Types

From `public/dt_common.h`:

```cpp
enum SendPropType {
    DPT_Int = 0,
    DPT_Float,
    DPT_Vector,
    DPT_VectorXY,      // only XY components
    DPT_String,
    DPT_Array,         // fixed-count array
    DPT_DataTable,     // nested datatable
};
```

---

## 12. The Rendering Pipeline

### Frame Entry Point

```
Host_UpdateScreen()
  → V_RenderView()                    (engine/view.cpp)
    → g_ClientDLL->View_Render()      (game/client/view.cpp)
      → CViewRender::Render()         (game/client/view.cpp:1062)
```

### `CViewRender::Render()` Flow

```
1. Set up views (camera origin, angles, FOV)
2. For each eye (stereo support via viewsetup)
3. RenderView():
   a. SetupMain3DView() — push render target, set up projection
   b. Render skybox (if 3D skybox exists)
   c. ViewDrawScene() — the core draw
   d. DrawPortals() — engine portal rendering
   e. DrawViewModels() — weapon arms
   f. DrawUnderwaterOverlay()
   g. DoEnginePostProcessing() — bloom, HDR tonemapping
   h. Screen space effects
   i. Paint VGUI (HUD)
4. Swap buffers
```

### `ViewDrawScene()` Draw Order (`game/client/viewrender.cpp:1312`)

```
1. Shadow texture prep
2. SetupVis() — compute visible leaves
3. DrawWorldAndEntities():
   a. DrawWorld() — opaque BSP surfaces
   b. DrawOpaqueRenderables() — opaque entities + static props
   c. DrawTranslucentRenderables() — sorted back-to-front
   d. DrawNoZBufferTranslucentRenderables() — overlays
4. CGlowOverlay::DrawOverlays()
5. DrawPrecipitation() — rain/snow
6. Client effects
7. Post-render game system hooks
```

### Water Rendering

`DrawWorldAndEntities()` dispatches to:
- `CAboveWaterView` — normal view with reflective/refractive water
- `CUnderWaterView` — underwater fog + distortion
- `CSimpleWorldView` — cheap/no water

Defined in `game/client/viewrender.cpp` around line 2533.

### Renderable Group Buckets

Opaque entities are drawn by size bucket (largest first):

```
RENDER_GROUP_OPAQUE_ENTITY_HUGE → LARGE → MEDIUM → SMALL
RENDER_GROUP_OPAQUE_STATIC_HUGE → LARGE → MEDIUM → SMALL
```

NPCs are drawn last within their bucket. Ropes and particles are drawn after all opaque entities.

---

## 13. VGUI Panel System

### Three-Layer Architecture

| Layer | File | Purpose |
|-------|------|---------|
| `IPanel` | `public/vgui/IPanel.h` | Pure virtual interface. VPANEL handle, positioned, parenting, PaintTraverse |
| `VPanel` | `vgui2/src/VPanel.h` | Private implementation. Stores positions, children, visibility, z-order |
| `Panel` | `public/vgui_controls/Panel.h` | User-facing base class. `Paint()`, input handlers, layout |

### Paint Traversal

Each frame, the surface system walks the panel tree:

```
CMatSystemSurface::PaintTraverseEx()
  → StartDrawing() — orthographic projection
  → PaintTraverse(rootPanel) — draw main tree
  → For each popup (back-to-front):
      Set stencil reference → PaintTraverse(popup)
  → FinishDrawing()
```

At the Panel level:
```
Panel::PaintTraverse():
  PushMakeCurrent() — set up translate + scissor rect
  Draw background
  Call Paint()       — user's drawing code
  Recurse children
  PopMakeCurrent()
```

### Key Controls (in `vgui2/vgui_controls/`)

- `Button`, `Label`, `Frame`, `ComboBox`, `ListPanel`, `TextEntry`
- `Menu`, `RichText`, `Slider`, `CheckButton`, `RadioButton`
- `ProgressBar`, `TreeView`, `PropertySheet`, `ImagePanel`

### The Surface

Two implementations:
- `CWin32Surface` — GDI-based (for standalone tools like Hammer)
- `CMatSystemSurface` — material-system-based (for the game). All drawing goes through `IMesh` + `CMeshBuilder`.

The material system surface (`vguimatsurface/MatSystemSurface.cpp`) draws quads with `MATERIAL_QUADS` primitives, using `m_pWhite` (an `UnlitGeneric` material with vertex color) for untextured fills.

---

## 14. Player Movement

### Interface

`IGameMovement` (`game/shared/igamemovement.h:110`):

```cpp
class IGameMovement {
    virtual void ProcessMovement( CBasePlayer *pPlayer, CMoveData *pMove ) = 0;
    virtual void GetPlayerMins( CBasePlayer *pPlayer, Vector *pMins, ... ) = 0;
    ...
};
```

### `CGameMovement::ProcessMovement()` (`game/shared/gamemovement.cpp:1133`)

```
1. Store frametime, apply lag compensation
2. PlayerMove():
   a. CheckParameters() — validate input
   b. ReduceTimers() — decrement duck/jump timers
   c. CheckStuck() — unstick if in solid
   d. CategorizePosition() — determine ground/water status
   e. Duck() — crouch logic
   f. LadderMove()
   g. Switch on movetype:
      - MOVETYPE_WALK: FullWalkMove()
      - MOVETYPE_NOCLIP: FullNoClipMove()
      - MOVETYPE_FLY: FullTossMove()
      - MOVETYPE_LADDER: FullLadderMove()
3. FinishMove()
```

### `FullWalkMove()` (`game/shared/gamemovement.cpp:2025`)

```
1. CheckWater() — is player in water?
2. Water jump check
3. Swimming: WaterMove()
4. Ground movement:
   - CheckJumpButton() — pressed space? apply jump velocity
   - Friction() — ground friction
   - WalkMove() — walk on ground
   - AirMove() / AirAccelerate() — air control
5. CategorizePosition() — final ground/water
6. FinishGravity() — residual gravity
```

### `CMoveData` — The Movement Data Packet

```cpp
struct CMoveData {
    // Input
    int     m_nButtons;           // IN_FORWARD, IN_JUMP, etc.
    float   m_flForwardMove;
    float   m_flSideMove;
    float   m_flUpMove;
    QAngle  m_vecViewAngles;

    // State
    Vector  m_vecVelocity;
    Vector  m_vecAbsOrigin;
    QAngle  m_vecAngles;

    // Output
    float   m_outStepHeight;
    float   m_outWishVel;
    float   m_outJumpVel;
};
```

---

## 15. Client Prediction

### Why Prediction

Without prediction, there would be a ~100ms delay between pressing W and the player moving. The client simulates movement locally while waiting for the server's authoritative response.

### `CPrediction::RunCommand()` (`game/client/prediction.cpp:825`)

```
1. StartCommand() — save state, set up prediction environment
2. Set gpGlobals->curtime and frametime to match the command
3. RunPreThink() — pre-think all entities
4. RunThink() — think functions
5. SetupMove() — copy player state → CMoveData
6. ProcessMovement() — run game movement locally
7. FinishMove() — copy CMoveData → player state
8. RunPostThink() — weapon post-frame (firing, reloading)
9. FinishTrackPredictionErrors()
```

### Prediction Error Correction

After the server responds:
1. Compare predicted position with server position
2. If error exceeds threshold, snap to server position (with smoothing)
3. `cl_showerror 1` to visualize prediction errors

### Predictable Entities

Entities that are created and simulated on the client (like projectiles):
- Declared with `DECLARE_PREDICTABLE()`
- Fields copied during prediction defined in `BEGIN_PREDICTION_DATA()` blocks
- Uses `IPredictionSystem` to filter/create/destroy predictable entities

---

## 16. Physics System

### Interface Layer

The physics system lives in `public/vphysics_interface.h` as a set of interfaces:

| Interface | Purpose |
|-----------|---------|
| `IPhysics` | Top-level: create environments, surface props |
| `IPhysicsEnvironment` | A simulation world: set gravity, step simulation |
| `IPhysicsObject` | A simulated rigid body: apply force/torque, get position |
| `IPhysicsCollisionSet` | Collision model management |
| `IPhysicsConstraint` | Join two objects (hinge, ballsocket, ragdoll) |
| `IPhysicsShadowController` | Follow a position (for player physics) |
| `IPhysicsFluidController` | Buoyancy simulation |
| `IPhysicsSurfaceProps` | Get surface properties (friction, bounce, sound) |

### Collision Queries

```cpp
// Trace a ray through the physics world
Ray_t ray;
ray.Init( startPos, endPos );

trace_t tr;
IPhysicsEnvironment::TraceRay( ray, MASK_SOLID, &tr );

// Sweep a box
Ray_t boxRay;
boxRay.Init( startPos, endPos, boxMins, boxMaxs );
g_pPhysics->SweepBox( &boxRay, ... );
```

### The `Ray_t` Structure

Defined in `public/cmodel.h`:

```cpp
struct Ray_t {
    VectorAligned m_Start;    // ray start
    VectorAligned m_Delta;    // end - start
    VectorAligned m_StartOffset;  // for swept shapes
    VectorAligned m_Extents;  // half-widths for box sweeps
    bool m_IsRay;            // true = point ray
    bool m_IsSwept;          // false = stationary shape
};
```

### Surface Properties

Physics → gameplay feedback:
- `IPhysicsSurfaceProps::GetSurfaceData( surfaceIndex )` → `surfacedata_t`
- Game code uses `gamesoundtype`, `impacteffect`, `friction`, `bounce`, etc.
- Material type drives footstep sounds, bullet impacts, particle effects

### Constraints

Types: ragdoll (ragdoll constraint), hinge (door), ballsocket (chains), pulley, spring, fixed.
Defined in `public/constraint.h`.

---

## 17. BSP & Level Loading

### Map Loading

`engine/sys_dll.cpp` → `Host_InitGame()` → calls into the BSP loader. Key files:
- `engine/mod_*.cpp` — model loaders (mod_brush.cpp for world)
- `engine/com_model.cpp` — `CModelLoader`
- `engine/world.cpp` — world rendering

### BSP Tree Structure

A `.bsp` file contains (among other things):

| Lump | Contents |
|------|----------|
| Leaves | Convex polyhedral regions of empty space |
| Nodes | Split planes that form the tree |
| Faces | Polygons on leaf boundaries |
| Edges | Shared edges between faces |
| Visibility | PVS data (which leaves can see which) |
| Entities | All map entities in keyvalue text |

### Visleaf / PVS

- The BSP tree partitions the world into **visleafs** (visible leafs).
- PVS (Potentially Visible Set) is a precomputed bitfield: for each leaf, which other leaves might be visible.
- The renderer uses PVS to skip rendering geometry in non-visible leafs.
- `engine/vis.cpp` — PVS decompression

### Loading Flow

```
Host_InitGame()
  → Host_LoadGame()
    → COM_LoadMap( mapName )
      → LoadBSPFile() → parse all lumps
      → Mod_LoadModel() → create model_t
      → PostLoad() → set up leaf lists, vis data
```

---

## 18. Network System

### NetChan (Network Channel)

Defined in `engine/net_chan.h` / `engine/net_chan.cpp`. Each connected client has one `NET_Chan` for the server → client channel.

**Packet types:**
- Sign-on (full state) — sent when first connecting
- Delta (changed state only) — sent every tick during gameplay
- Voice — compressed audio data
- File transfer — for custom files

### Rate Management

CVars that control bandwidth:
- `sv_maxrate` / `cl_cmdrate` / `cl_updaterate`
- `sv_minrate` — minimum acceptable rate
- The server adjusts update frequency based on client rate

### Packet Structure

Each network packet has:
- Sequence number (for ordering and loss detection)
- Acknowledgment of received packets
- Changed entity data (delta against last acknowledged state)
- Unreliable data (sound events, temp entities)
- Reliable data (signals, important events)

### Connection State Machine

Defined in `engine/server.h` / `engine/client.h`:
```
DISCONNECTED → CONNECTING → SPAWNING → ACTIVE
                                 ↓
                            Fully connected, receiving updates
```

---

## 19. Sound System

### Interface

`IEngineSound` (`public/engine/IEngineSound.h`):

```cpp
// Emit a sound from an entity
EmitSound( entChannel, entityIndex, soundName, volume, pitch, origin, ... );

// Precache a sound (must be done before first use)
PrecacheSound( soundName );
```

### Sound Emitter System

`soundemittersystem/` — higher-level sound system that reads `soundscapes.txt` and `soundemitters.txt`:
- Maps game events to actual `.wav` filenames
- Sound stacks and ambiance
- 3D position vs. 2D (UI) sounds

### Attenuation Models

How sound volume decreases with distance:
- `ATTN_NONE` — full volume everywhere (UI)
- `ATTN_NORMAL` — standard distance falloff
- `ATTN_IDLE` — idle NPC voice attenuation
- `ATTN_GUNFIRE` — weapon fire (loud, long-range)

---

## 20. Game Events

### Event System

`IGameEventManager2` (`public/igameeventmanager2.h`):

```cpp
// Fire an event from server → all clients
IGameEvent *event = gameeventmanager->CreateEvent( "player_death" );
event->SetInt( "userid", player->GetUserID() );
event->SetString( "weapon", "shotgun" );
gameeventmanager->FireEvent( event );
```

### Handling Events on Client

```cpp
class CMyEventListener : public IGameEventListener2 {
    void FireGameEvent( IGameEvent *event ) {
        if ( !Q_strcmp( event->GetName(), "player_death" ) ) {
            int userId = event->GetInt( "userid" );
            // Show death notice, play sound, etc.
        }
    }
};
```

### Uses
- Kill feed, round start/end, bomb events
- Achievements
- Replay/demo markers
- Sound event triggers

---

## 21. Design Patterns Used in Source

| Pattern | Where | Description |
|---------|-------|-------------|
| **Factory** | `CreateInterface`, `InterfaceReg` | Each DLL exports a factory that creates interfaces by name |
| **RAII** | `AUTO_LOCK( mutex )`, `CAutoLock` | Constructor locks, destructor unlocks |
| **Self-registration** | `InterfaceReg`, `ConCommandBase` | Static linked list — objects register themselves before `main()` |
| **Observer** | Entity I/O, Game Events | Objects fire events, listeners respond |
| **Strategy** | `IGameMovement` | Movement is swappable per game mod |
| **Singleton** | `g_pGameRules`, `g_pCVar` | Per-process or per-module global instance |
| **State Machine** | `CBaseEntity` weapon states, `Host_State()` | Entities track state via `m_iState`, host uses `HostState_t` |
| **Pimpl** | `IPanel`/`VPanel` | Interface hides implementation behind a pointer |
| **Template Method** | `CBaseEntity::Spawn()` → subclass overrides | Skeleton algorithm with overridable steps |

---

## 22. Memory Management

### The Tier0 Allocator

All memory goes through `g_pMemAlloc` (`IMemAlloc*`, `public/tier0/memalloc.h:134`).

```cpp
void* p = malloc( 100 );     // → g_pMemAlloc->Alloc( 100 )
void* p = new MyClass;       // → operator new → g_pMemAlloc->Alloc(...)
```

### Stack Allocator (`CMemoryStack`)

Defined in `public/tier1/memstack.h`. A linear allocator backed by `VirtualAlloc`:

```cpp
CMemoryStack stack;
stack.Init( 1024*1024 );      // 1MB reservation

int mark = stack.GetCurrentAllocPoint();
int* p = (int*)stack.Alloc( sizeof(int) * 100 );
// ... use p ...
stack.FreeToAllocPoint( mark );  // rewind — O(1), no destructors called
```

Used for per-frame allocations that all reset at once (like rendering data).

### Pool Allocator (`CUtlMemoryPool`)

Fixed-size block allocator. No per-allocation overhead:

```cpp
CUtlMemoryPool pool( sizeof(MyStruct), 100, 0, "MyStruct pool" );
MyStruct* p = (MyStruct*)pool.Alloc();   // O(1), pop from free list
pool.Free( p );                          // O(1), push to free list
```

### Fixed-Size Allocator Pattern

Classes that are allocated frequently use this pattern:

```cpp
// In the header
DECLARE_FIXEDSIZE_ALLOCATOR( MyFrequentClass );

// In the .cpp
DEFINE_FIXEDSIZE_ALLOCATOR( MyFrequentClass, 256, CUtlMemoryPool::GROW_FAST );
```

This gives every instance its own pool — no heap fragmentation.

### Scratch Allocator

`MemAllocScratch()` / `MemFreeScratch()` — a small, fast allocator for temporary buffers.

---

## 23. The Tier System

The dependency chain is strictly one-way: `tier0 → tier1 → tier2 → tier3`.

| Tier | What It Provides | Key Dependency |
|------|-----------------|----------------|
| **tier0** | Memory allocation, threading, CPU detection, debugging, command line, platform abstraction | Nothing |
| **tier1** | ConVars, KeyValues, string tools, bit buffers, checksums, containers, interfaces | tier0 |
| **tier2** | File utils, render utils, sound utils, camera utils | tier1 |
| **tier3** | Choreo (facial animation), MDL utils, scene token processing | tier2 |

Why tiers? DLL loading order: tier0 → tier1 → engine → game DLLs. Each tier is linked as a static library into the DLL that needs it (or a shared DLL on some platforms). The tier system prevents circular dependencies.

---

# EXPERT NOTES

---

## 24. SIMD Math

### `fltx4` — The SSE Primitive

```cpp
// public/mathlib/ssemath.h
typedef __m128 fltx4;   // 4 floats in one XMM register
```

Source uses this for batch operations on vectors, quaternions, and bone transforms.

### `FourVectors`

```cpp
class FourVectors {
    fltx4 x, y, z;      // 4 vectors processed in parallel

    // Batch length computation
    fltx4 LengthSqr() const;    // (x^2 + y^2 + z^2) for all 4

    // Batch normalize
    void NormalizeFast();
};
```

Used for AI sensor queries (checking distance to 4 entities at once), culling, and particle systems.

### SSE Quaternions

In `public/mathlib/ssequaternion.h`:

```cpp
// Batch quaternion operations
void QuaternionAlignS( ... );
void QuaternionBlendS( ... );  // blend 2 quaternions
void QuaternionSlerp( ... );   // spherical interpolation
```

Note: `QuaternionMult` often uses scalar paths on PC because the x87/SSE pipeline flushes introduced too much latency compared to scalar code.

### SIMD Trig

```cpp
// batch sine/cosine using SSE
fltx4 SinEst01( fltx4 x );
fltx4 CosEst01( fltx4 x );
```

### When SIMD Is Used

- Bone transformation (skinning 4 vertices at once)
- Particle simulation
- Bounding volume transformations
- Physics collision detection (OBB tests)
- AI distance checks in `CUtlVector` loops
- View frustum plane tests

---

## 25. Lock-Free Data Structures

### `CTSListBase` (Lock-Free Stack)

Defined in `public/tier0/tslist.h`. A lock-free singly-linked list (Treiber stack).

**Problem:** ABA — if a thread reads head pointer A, gets preempted, another thread pops A and pushes A back, the first thread's CAS succeeds but the list structure has changed.

**Solution:** Include a 32-bit sequence counter alongside the pointer. Pack them into a 64-bit (or 128-bit) CAS-able word:

```cpp
struct TSLHead_t {
#ifdef _WIN64
    int64   pNext;     // pointer + sequence packed
#else
    void*   pNext;
    int     sequence;  // prevents ABA
#endif
};
```

**Push:**

```cpp
void Push( TSLLink_t* pNode ) {
    TSLHead_t head;
    do {
        head = m_Head;
        pNode->Next = head.pNext;
    } while ( !ThreadInterlockedAssignIf64x128( &m_Head, newHead, oldHead ) );
}
```

**Pop:**

```cpp
void* Pop() {
    TSLHead_t head;
    do {
        head = m_Head;
        if ( !head.pNext ) return NULL;
    } while ( !CAS( &m_Head, newHead, oldHead ) );
    return head.pNext;
}
```

### `CTSQueue` (Lock-Free Queue)

Michael-Scott-style FIFO using the same ABA-proofed nodes:
- `Push()` — CAS on `Tail->Next`, then CAS on `Tail`
- `Pop()` — CAS on `Head`, with help logic for concurrent pushes

### Where Used

- `CJobQueue` in the thread pool (vstdlib/jobthread.cpp)
- `CSmallBlockHeap` free lists
- `CUtlMemoryPool` free lists
- Event system message dispatch

---

## 26. The Small Block Heap (SBH)

### Problem

Frequent small allocations (`new Entity`, `new CUtlVector` elements) fragment the CRT heap and kill cache performance.

### Solution

`CStdMemAlloc` (tier0/memstd.cpp) includes a Small Block Heap for allocations ≤ 2048 bytes.

**Pool sizes:**

```cpp
// 42 pools covering:
// 8-byte increments: 8-128   (16 pools)
// 16-byte increments: 128-256 (8 pools)
// 32-byte increments: 256-512 (8 pools)
// 64-byte increments: 512-768 (4 pools)
// 128-byte increments: 768-1024 (2 pools)
// 256-byte increments: 1024-2048 (4 pools)
```

**O(1) pool lookup:**

```cpp
int FindPool( size_t nBytes ) {
    return m_PoolLookup[(nBytes - 1) >> 2];
}
```

A precomputed lookup table maps size → pool index.

**Allocation:**

```cpp
void* CSmallBlockPool::Alloc() {
    if ( m_FreeList ) {          // pop from lock-free free list
        return m_FreeList.Pop();
    }
    // Commit more virtual memory
    // Advance m_pNextAlloc through reserved VM region
}
```

**Backing:** Pre-reserved via `VirtualAlloc(NULL, NUM_POOLS * MAX_POOL_REGION, MEM_RESERVE, ...)`. Pages committed on demand.

---

## 27. `memdbgon.h` — The `#define` Trick

### How It Works

`public/tier0/memdbgon.h` is included as the **last** header in a `.cpp` file:

```cpp
#include "tier0/memdbgon.h"     // must be last include
```

It redefines `malloc`, `free`, `new`, `delete`:

```cpp
#undef malloc
#define malloc(s)    g_pMemAlloc->Alloc(s)

#undef free
#define free(p)      g_pMemAlloc->Free(p)

// Debug builds get file/line info:
#define malloc(s)    g_pMemAlloc->Alloc(s, __FILE__, __LINE__)
#define new          new(_NORMAL_BLOCK, __FILE__, __LINE__)
```

### The Guard

`memdbgon.h` has NO include guard — it can be included multiple times in one translation unit to toggle the override on and off:

```cpp
#include "tier0/memdbgoff.h"    // undefines the overrides
// ... STL code or other code that must use system allocator ...
#include "tier0/memdbgon.h"    // re-enable overrides
```

### Why

Allows Valve to:
- Track every allocation with file/line info in debug builds
- Redirect all memory through their custom allocator (SBH)
- Implement OOM handlers, leak detection
- This works on ALLOCATIONS made by third-party code that compiles with their headers

---

## 28. Coroutines

### Overview

Source Engine has a stack-swapping coroutine system for cooperative multitasking within a single thread. Found in `vstdlib/coroutine.h` + `coroutine.cpp`.

### API

```cpp
HCoroutine hCo = Coroutine_Create( MyFunc, param );

while ( Coroutine_Continue( hCo ) ) {
    // Coroutine yielded — do other work, then resume
}

void MyFunc( void* param ) {
    // Do some work...
    Coroutine_YieldToMain();
    // Do more work...
    Coroutine_Finish();
}
```

### Implementation

Uses `setjmp`/`longjmp` with stack saving:

```cpp
void Internal_Coroutine_Continue( HCoroutine hCo ) {
    CCoroutine* pCo = ...;

    setjmp( pCo->m_Registers );            // save current register state

    if ( !pCo->m_pSavedStack ) {
        // First launch — allocate guard gap, call function
        Coroutine_Launch( pCo );
    } else {
        // Resume — restore saved stack, jump to coroutine
        RestoreStack( pCo );
        longjmp( pCo->m_Registers, 1 );
    }
}
```

When `Coroutine_YieldToMain()` is called:
1. `setjmp` saves coroutine registers
2. `SaveStack()` copies the coroutine's stack to a heap buffer
3. Stack pointer restored to the caller's stack
4. `longjmp` back to the caller

### Stack Guard

A 64KB gap is reserved between the main stack and the coroutine stack to prevent accidental collision. Nested coroutines use a 64-byte gap.

### Uses

- Loading screens (load a map incrementally while rendering)
- AI behaviors that need to pause mid-execution

---

## 29. Threading & Memory Barriers

### `CThread` Hierarchy

From `public/tier0/threadtools.h`:
- `CThread` — base: `Start()`, `Join()`, `Run()` (pure virtual)
- `CWorkerThread` — bidirectional rendezvous via `CallWorker()` / `WaitForCall()` / `Reply()`
- `CJobThread` — `CWorkerThread` that pops from `CJobQueue`

### Spinlock Backoff Strategy

In `CThreadFastMutex::Lock()` (`tier0/threadtools.cpp`):

```
Phase 1: Spin 1024 iterations with ThreadPause() (PAUSE instruction)
Phase 2: Spin 1024 iterations, yield every 64
Phase 3: Spin 1024 iterations, ThreadSleep(0) every 1024
Phase N: ThreadSleep(1) — millisecond sleep, last resort
```

This avoids wasting CPU when contention is low, but still yields to the OS scheduler under high contention.

### `CThreadSpinRWLock`

A 32-bit word encodes the lock state:

```cpp
union LockInfo_t {
    uint32  m_i32;
    struct {
        uint16 m_nReaders;   // count of active readers
        uint16 m_fWriting;   // 0x0001 when writer holds lock
    };
};
```

- Write lock: CAS `0` → `0x00010000` (fail if readers exist)
- Read lock: atomically increment `m_nReaders` (fail if writing bit set)
- Both spinning.

### `ThreadMemoryBarrier()`

```cpp
// Win32: compiler barrier (prevents compiler reordering)
_ReadWriteBarrier();

// x86/x64: hardware is strongly ordered for normal loads/stores
// (no CPU barrier needed, but compiler barries required)

// On weakly-ordered CPUs (ARM, PowerPC), this becomes a full memory barrier:
__sync_synchronize();  // GCC barrier
```

### Where Barriers Matter

- Lock-free queue push/pop
- Thread-local storage access after write
- Signal/event flag checking

---

## 30. ConVar Internals

### `CCvar` Implementation

Located in `vstdlib/cvar.cpp`. The global `CCvar` object:

```cpp
class CCvar : public ICvar {
    CUtlVector<ConCommandBase*> m_Commands;  // command lookup
    CThreadFastMutex m_mutex;                // thread safety
};
```

### Lookup

`FindCommandBase(name)`:
1. Lock mutex
2. Linear search through `m_Commands` (yes, linear — not many cvars in practice)
3. Compare with `Q_strcasecmp()`

### Registration

When a `ConVar` is constructed:
1. The `ConCommandBase` constructor calls `CCvar::RegisterConCommand()`
2. Which adds it to `m_Commands`
3. If the cvar is `FCVAR_ARCHIVE`, saves to `config.cfg`

### ConCommandBase Linked List

All ConVars and ConCommands form a global linked list via `s_pConCommandBases` (defined in `tier1/convar.cpp:38`). This list is populated by static initialization before `main()`.

### Engine ↔ DLL Interface

The engine's `ICvar` is passed to game DLLs during `Connect()`. The game DLL registers its cvars through this interface. The engine owns the authoritative copy of `FCVAR_REPLICATED` cvars and broadcasts changes to all clients.

---

## 31. Physics Internals (IVP Pipeline)

### The Simulation Step

```
IPhysicsEnvironment::Simulate( deltaTime )
  → IVP_Core_Physics::do_physics()     // ivp_physics/
    → Apply forces (gravity, user forces)
    → Collision detection:
        → Broad phase: spatial hash/grid
        → Narrow phase: GJK (Gilbert-Johnson-Keerthi) for convex shapes
    → Generate contact points
    → Resolve constraints:
        → LCP solver (iterative, projected Gauss-Seidel)
        → 8-20 iterations per step
        → Resolve penetration, apply impulses
    → Integrate positions (semi-implicit Euler)
    → Update sleeping state
```

### Collision Pipeline in Source Terms

```
Game code calls → IPhysicsObject::ApplyForceCenter( vec )
                → Internal IVP object accumulates force
                → Next Simulate() step processes it
```

### Constraints (Ragdolls)

Each constraint joint type maps to an IVP constraint:
- `Hinge` → 1 rotational DOF free
- `Ballsocket` → 3 rotational DOFs free
- `Ragdoll` → configurable per-axis limits

### Surface Properties

When a collision occurs:
1. Both objects have surface material indices
2. `IPhysicsSurfaceProps` returns combined properties: friction, restitution, sound
3. Game code uses this to generate impact effects, sounds, damage

### Sleeping

Objects that haven't moved significantly for a while get put to sleep (no simulation) to save CPU. A small wake-up force (or touching a non-sleeping object) wakes them.

---

## 32. The Host State Machine

The engine's state machine in `engine/host_state.h` / `engine/host.cpp`:

```cpp
enum HostState_t {
    HS_NEW_GAME,           // Starting a new game
    HS_LOAD_GAME,          // Loading a saved game
    HS_CHANGE_LEVEL_SP,    // Singleplayer level transition
    HS_CHANGE_LEVEL_MP,    // Multiplayer level transition
    HS_RUN,                // Normal gameplay
    HS_GAME_SHUTDOWN,      // Shutting down
    HS_RESTART,            // Restarting
    HS_SHUTDOWN,           // Engine shutting down
};
```

`Host_State()` transitions between these states. Each state has a specific processing path in `Host_Frame()`:

- `HS_RUN` → simulate entities, process network, render
- `HS_CHANGE_LEVEL_MP` → unload current map, load new map, signal clients
- `HS_LOAD_GAME` → read save file, restore entity state, set up rendering

---

## 33. Shader System

### Two-Phase Rendering

Source shaders work in two phases, defined in `materialsystem/shadersystem.cpp`:

**Phase 1: Snapshot (`TakeSnapshot`)**

The shader computes its required render state:
- Which textures to bind
- What blending/alpha test modes
- Vertex format requirements
- All constant parameters

This produces a `ShaderRenderState_t` containing a list of `RenderPass_t` snapshots.

**Phase 2: Draw (`DrawSnapshot`)**

The snapshot is applied to the hardware:
- Bind textures set in the snapshot
- Set render states
- Set vertex and pixel shader constants
- Draw the mesh

### Why Two Phases

Allows the material system to:
- Sort draw calls by state to minimize state changes
- Cache snapshots across frames
- Precompute derived values once per snapshot, not per-draw

### IShaderShadow / IShaderDynamicAPI

The shader API is split into two interfaces:
- `IShaderShadow` — called during snapshot phase to set semi-permanent state
- `IShaderDynamicAPI` — called during draw phase for per-frame dynamic state

### Standard Shaders

Defined in `materialsystem/stdshaders/`:
- `UnlitGeneric` — no lighting, vertex color
- `VertexLitGeneric` — per-vertex lighting
- `LightmappedGeneric` — world geometry with lightmaps
- `DecalModulate` — decal blending

---

## 34. NetChan Packet Details

### Packet Format

Each UDP packet has:

```
SequenceNumber (4 bytes)       — Outgoing sequence #
Ack (4 bytes)                  — Last received sequence from peer
BReassembly (variable bits)    — Bitmask of received fragments

[Then, for each entity with changes:]
  EntityIndex (variable bits)
  ChangedFieldMask (variable bits)
  FieldData (1-32 bytes per changed field)

Reliable Data:
  - Signal messages (spawn, kill, weapon change)
  - Filenames for downloads

Unreliable Data:
  - Temp entities (effects)
  - Sound events
  - Voice data
```

### Sign-On vs. Delta

**Sign-on:** When connecting, the server sends the full state of every entity. This is a large packet.

**Delta:** After sign-on, only changes are sent. The server tracks the last acknowledged state for each client and sends only what changed.

### Loss Recovery

- **Reliable data:** Retransmitted until acknowledged. A lost reliable packet blocks all subsequent reliable data.
- **Unreliable data:** Lost packets are not retransmitted. The client interpolates.
- **Sequence numbers:** Both sides track outgoing and incoming sequences. Gap detection triggers loss compensation.

---

## 35. Demo Format

### Structure

A demo file (`.dem`) has:

```
Header:
  DemHeader_t: magic ("HL2DEMO"), protocol, tickcount, map name, etc.

Frames:
  DemFrame_t:
    - Tick number
    - Frame type (sign-on, delta, console command, data table, stop)
    - Compressed data:
        * Full sign-on state (first frame)
        * Delta from last frame (subsequent frames)
        * Console commands
        * Custom data

Sign-on (first frame):
  - All entity class names and SendTables
  - Full entity state
  - All models, sounds, decals, etc.

Delta (subsequent frames):
  - Changed entity properties only
  - Same format as network delta encoding
```

### Replay Playback

The demo system in `engine/demo.cpp` and `replay/` reads frames sequentially, applies entity state, and renders. The engine doesn't distinguish between live and demo playback for most purposes — same rendering code path.

---

## 36. Entity I/O Chain

### Event Queue

When an output fires, the I/O system doesn't process it immediately. Instead, it's queued:

```cpp
// Internal flow:
FireOutput( "OnBreak" )
  → CEventAction is created with delay and target
  → Added to CBaseEntity::m_EventQueue
  → During server frame, ProcessEventQueue() is called
    → For each event whose time has come:
        → AcceptInput() on the target entity
```

### Delayed Outputs

```cpp
m_OnDamaged.FireOutput( pActivator, this, 1.0f );  // fires 1 second later
```

The delay is stored in `CEventAction::m_flDelay`. Useful for:
- "Open door → close after 3 seconds"
- "Kill NPC after death animation"

### AI Response System

`CAI_BaseNPC` uses entity I/O extensively:
- `InputScriptSchedules` — run a predefined behavior
- `InputSetRelationship` — change faction relationships
- `OutputOnHalfHealth` — triggered at 50% HP

The I/O system is what makes scripted sequences and complex maps possible without programming.

---

## 37. AI & NextBot

### Overview

The AI system lives in `game/server/NextBot/`. NextBot is a behavior-tree-based AI system used in TF2 for bots and some NPCs.

### Behavior Trees

```cpp
class Behavior : public INextBotComponent {
    Behavior* m_pCurrentBehavior;     // current sub-behavior
    // Run the behavior tree each frame
    virtual BehaviorState Update( INextBot* bot ) = 0;
};
```

Behaviors can be:
- **Selector** — try children in order, run first one that succeeds
- **Sequence** — run children in order, fail if any fail
- **Decorator** — modify child behavior (invert, repeat, timeout)
- **Leaf** — specific action (move to, attack, wait)

### Pathfinding

The A* pathfinder operates on a nav mesh:
- `INextBotPath` — defines the path (waypoints + connections)
- `NextBotPathFollow` — drives movement along the path
- Updates path when blocked or when goal changes

Components:
- `ILocomotion` — movement interface (run, jump, crouch)
- `IBody` — body state (aim, turn rate)
- `IIntention` — top-level decision making
- `IVision` — line-of-sight queries, enemy tracking

### Nav Mesh

The `.nav` file defines the walkable surface as a graph of `CNavArea` nodes connected by bidirectional edges. A* finds the shortest path through this graph.

---

## 38. BSP/PVS Deep Dive

### PVS Encoding

PVS (Potentially Visible Set) is encoded as run-length-encoded bit vectors:

```cpp
// For each leaf:
//   A bit for every other leaf: 1 = possibly visible
//   RLE compressed to save space
//   Typical compression: 10:1 to 50:1
```

### Usage during Rendering

```
For each frame:
  1. Determine which leaf the camera is in
  2. Decompress PVS for that leaf (or use the cluster's PVS)
  3. Walk the BSP tree front-to-back
  4. For each leaf: if its bit is set in PVS, render its faces
  5. Skip all faces in non-visible leaves
```

### Area Clusters

Clusters are groups of adjacent leaves for coarser visibility queries. The BSP compiler (`utils/vvis/`) computes which clusters can see which other clusters.

---

## 39. Hammer Editor Architecture

### Overview

Hammer is a Windows MFC SDI/MDI application (~461 files, ~12k-line main doc file) that also links against Source's tier system (tier0-3, material system, filesystem). It's both a standalone tool and an `IAppSystem`.

### Application Entry

```cpp
// hammer/hammer.cpp
class CHammer : public CWinApp, public CTier3AppSystem<IHammer> {
    BOOL InitInstance();      // MFC init → loads game config → shows main window
    BOOL Run();               // Main loop (calls CWinApp::Run)
};
```

### Tool System

Hammer has 20 editing tools, all deriving from `CBaseTool` (`hammer/toolinterface.h`):

| Tool | Class | Purpose |
|------|-------|---------|
| Pointer | `Selection3D` | Select, move, scale objects |
| Block | `CToolBlock` | Create block brushes |
| Entity | `CToolEntity` | Place point entities |
| Morph | `Morph3D` | Vertex/edge editing |
| Clipper | `Clipper3D` | Clip brushes with a plane |
| Displace | `CToolDisplace` | Edit displacement surfaces |
| Decal | `CToolDecal` | Place decals |
| Camera | `Camera3D` | Place/edit camera positions |

`CToolManager` (`hammer/ToolManager.cpp`) manages the tool stack per-document. Tools receive mouse/keyboard events for 2D, 3D, and Logical views through virtual functions like `OnLMouseDown2D()`, `OnMouseMove3D()`, `OnKeyDown2D()`.

### Map Object Hierarchy

```
CMapClass (base)
├── CMapSolid        — brush with faces
├── CMapEntity       — point or solid-based entity
├── CMapGroup        — object grouping
├── CMapDisp         — displacement surface
├── CMapOverlay      — material overlay on faces
├── CMapInstance     — func_instance (linked map)
└── CMapWorld        — root of the object tree
```

### VMF File Format

Valve Map Format is a hierarchical chunk-based format parsed by `CChunkFile` (`public/chunkfile.h`):

```
versioninfo
{
    "editorversion" "400"
}
visgroups { ... }
world
{
    id = "1"
    solid
    {
        side { "plane" "(0 0 0) (0 0 0) (0 0 0)" "material" "CONCRETE/..."; }
    }
}
entity
{
    "classname" "info_player_start"
    "origin" "0 0 0"
}
```

### FGD System

FGD (Game Definition) files describe entity classes for the editor. Located in `public/fgdlib/`:

```
@SolidClass base(Targetname, Parent, Origin) = worldspawn : "World entity"
[
    lightmapscale(int) : "Lightmap Scale" : 16
    skyname(string) : "Skybox Name" : "sky_day01_01"
]
```

Hammer loads FGD files at startup and parses them into `GDclass` objects. Each entity in the editor has a `GDclass* m_pClass` pointer that defines its keyvalues, inputs, outputs, and visual helpers.

### 3D Rendering

Hammer renders the 3D view using Source's material system directly (NOT the game engine). `CRender3D` (`hammer/render3dms.h`):
- Traverses the `CCullTreeNode` spatial tree
- Calls `RenderMapClass()` for each visible object
- Draws using `IMesh` / `CMeshBuilder`
- Supports wireframe, flat, textured, and lightmap grid modes

### Undo System

`CHistory` (`hammer/history.h`) stores per-operation tracks containing per-object entries:
- `ttCopy` — stores a snapshot before modification
- `ttDelete` — stores the deleted object
- `ttCreate` — stores the created object

---

## 40. VPC Build System

### What VPC Was

Valve's Project Creator (VPC) was a custom build configuration tool that generated Visual Studio `.vcproj`/`.vcxproj` files from Valve-specific script files. Scripts used `$Configuration { }` blocks with platform conditionals:

```
$Configuration
{
    [$WIN32]
    {
        $TargetName  "server"
        $PreprocessorDefinitions  "WIN32;_WINDOWS"
    }
    [$POSIX]
    {
        $TargetName  "server.so"
        $PreprocessorDefinitions  "POSIX;LINUX"
    }
}
```

VPC supported:
- `$Project`, `$Folder`, `$File` — project structure
- `$Include` — file inclusion for sharing configs
- `$Macro` — variable substitution
- Conditional operators: `[$WIN32]`, `[$POSIX]`, `[$X360]`, with `&&`, `||`, `!`

### The Waf Bridge

This repository uses Waf (Python build system) instead of VPC, but reuses `.vpc` files through a Python parser bridge:

```python
# scripts/waifulib/vpc_parser.py
def parse_vpcs( name ):
    # Reads .vpc files
    # Evaluates conditionals
    # Extracts: 'defines', 'includes', 'sources'
    # Returns dict consumed by Waf
```

Game DLLs (client, server) use this bridge — their `wscript` files call `vpc_parser.parse_vpcs()` to get source file lists and defines from the `.vpc` files.

### Why This Matters

If you want to add a new `.cpp` file to the game:
1. Add it to the `.vpc` file in the appropriate `$File` block
2. The Waf bridge automatically picks it up

If you want to add a new build configuration:
1. Edit `wscript` to add options
2. Or edit the relevant `.vpc` files and the bridge will handle it

---

# Quick Reference: Important Files by Topic

| Topic | Key Files |
|-------|-----------|
| Entity system | `game/server/baseentity.h/.cpp`, `public/edict.h` |
| Network vars | `public/networkvar.h`, `public/dt_send.h`, `public/dt_recv.h` |
| Movement | `game/shared/gamemovement.h/.cpp`, `game/shared/igamemovement.h` |
| Prediction | `game/client/prediction.h/.cpp`, `game/shared/ipredictionsystem.h` |
| Rendering | `game/client/view.cpp`, `game/client/viewrender.cpp`, `game/client/glow_overlay.cpp` |
| VGUI | `public/vgui/IPanel.h`, `vgui2/src/VPanel.h`, `vguimatsurface/MatSystemSurface.cpp` |
| Materials | `public/materialsystem/imaterial.h`, `materialsystem/cmaterial.cpp` |
| Physics | `public/vphysics_interface.h`, `vphysics/`, `ivp/` |
| ConVars | `public/tier1/convar.h`, `vstdlib/cvar.cpp`, `public/icvar.h` |
| Interfaces | `public/tier1/interface.h`, `tier1/interface.cpp` |
| Memory | `public/tier0/memalloc.h`, `tier0/memstd.cpp`, `public/tier0/memdbgon.h` |
| Containers | `public/tier1/utlvector.h`, `public/tier1/utllinkedlist.h`, `public/tier1/utlbuffer.h` |
| Threading | `public/tier0/threadtools.h`, `public/tier0/tslist.h`, `vstdlib/jobthread.cpp` |
| Coroutines | `vstdlib/coroutine.h/.cpp` |
| Math | `public/mathlib/mathlib.h`, `public/mathlib/vector.h`, `public/mathlib/ssemath.h` |
| SIMD | `public/mathlib/ssemath.h`, `public/mathlib/ssequaternion.h` |
| BSP | `engine/mod_*.cpp`, `public/bspfile.h` |
| Network | `engine/net_chan.h/.cpp`, `networksystem/`, `public/engine/net_*.h` |
| Game Events | `public/igameeventmanager2.h`, `engine/GameEventManager.cpp` |
| Hammer | `hammer/hammer.cpp`, `hammer/mapdoc.cpp`, `hammer/`, `public/fgdlib/` |
| Build system | `wscript`, `scripts/waifulib/vpc_parser.py`, `vpc_scripts/` |
