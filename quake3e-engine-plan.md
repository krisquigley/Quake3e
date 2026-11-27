# Quake3e → Continually Upgradable Engine: Context & Plan

Handoff from planning conversations. Read this before working on the Quake3e codebase.

## Goal

Turn Quake3e (https://github.com/ec-/Quake3e) into a **continually upgradable, maintainable game engine** with migration paths, so each subsystem can be modernised in turn **without rebuilding from scratch**. Performance must be preserved.

Non-goal: reaching Unreal Engine 5 parity directly. UE5-class features (Nanite, Lumen and the like) need changes to interfaces and data formats, not just swapped implementations. This plan builds the foundation that makes those changes possible over time.

## Codebase facts (measured at commit dec793d, Nov 2026)

Counted with a cloc-style script (code / comment / blank lines):

| Part | Code | Total incl. comments & blanks |
|---|---|---|
| Engine's own code | 220,794 | 312,635 |
| — C | 177,915 | |
| — Headers | 39,626 | |
| — GLSL | 2,040 | |
| Bundled third-party (`code/libcurl`, `libjpeg`, `libogg`, `libsdl`, `libvorbis`) | 99,931 | 141,267 |
| **Repo total** | | **453,902** |

Source folders under `code/`: asm, botlib, cgame, client, game, qcommon, renderer, renderer2, renderercommon, renderervk, sdl, server, ui, unix, win32 (plus the bundled libraries above).

### Verified against the code

**The game code is not in this repo.** `code/game`, `code/cgame` and `code/ui` contain only the API headers (`g_public.h`, `bg_public.h`, `cg_public.h`, `ui_public.h`). All gameplay comes from compiled QVMs in the game data (retail `pak8a.pk3` or a mod). Gameplay can't be changed or tested here until we own game code (see phase 0).

**Every boundary is already versioned.**

| Interface | Version | Size |
|---|---|---|
| Game (server ↔ qagame) | `GAME_API_VERSION 8` | ~192 system calls |
| Client game (client ↔ cgame) | `CGAME_IMPORT_API_VERSION 4` | ~115 system calls |
| UI | `UI_API_VERSION 6` | ~106 system calls |
| Renderer (`renderercommon/tr_public.h`) | `REF_API_VERSION 8` | 39 exports + 57 imports |
| Bot library (`botlib/botlib.h`) | via `GetBotLibAPI` | 22 imports |

Quake3e also has a feature-discovery call, `trap_GetValue`, handled at `server/sv_game.c:354` (`SV_GetValue`) and `client/cl_cgame.c:428` (`CL_GetValue`). A module asks by name whether an engine feature exists. This is the natural mechanism for opt-in extensions: old mods never ask, so they keep working.

**The BSP coupling is through the file format, not shared memory.**
- Each renderer parses the BSP itself (`renderer*/tr_bsp.c`) and has its own visibility (PVS) code (`renderer*/tr_world.c`).
- Collision parses it separately (`qcommon/cm_load.c`).
- The one in-memory link is network relevance: `server/sv_snapshot.c` uses the collision code's PVS (`CM_ClusterPVS`) to decide what each client is sent.
- **Collision runs on the client too.** The cgame calls `CM_BoxTrace` and related functions (`client/cl_cgame.c:554-578`) to predict player movement. Client and server must give bit-identical collision results or prediction breaks, so any replacement collision system must be shared by both.
- **Bots are a fourth BSP consumer.** They navigate with AAS files compiled offline from the BSP (the bspc tool); `botlib/be_aas_bspq3.c` also reads BSP entities. A world-format change breaks bots unless AAS generation moves into the asset pipeline.

**The renderers are heavily duplicated.**

| Renderer | Lines |
|---|---|
| `renderer` (OpenGL, default) | 32.5k |
| `renderervk` (Vulkan) | 39.9k |
| `renderer2` (OpenGL2, from ioquake3) | 50.5k |
| `renderercommon` (shared) | 4.4k |

Comparing `renderer` with `renderervk` file by file: `tr_marks.c`, `tr_curve.c` and `iqm.h` are identical; `tr_model.c`, `tr_world.c`, `tr_mesh.c`, `tr_animation.c` and `tr_shade_calc.c` differ by 8 lines or fewer; `tr_bsp.c` differs in 92 of 2,408 lines. Real differences sit in `tr_shader`, `tr_shade`, `tr_backend`, `tr_init` and `tr_image`. The natural seam is a **shared frontend** (world and model loading, visibility, scene building) with **separate drawing backends**, not separate whole renderers.

**Global state is concentrated.** About 750 `extern` declarations in the engine's own code (an earlier count of 564 used a different method). About 450 are in the three renderers, mostly `r_*` cvar pointers, which are trivial to move. The structural work is about ten large global structs: `cl`, `clc`, `cls` (client), `sv`, `svs` (server), `cm` (collision), `tr`, `backEnd` (renderer) and the sound mixer state.

**Determinism blockers are few and located.**
- `G_MILLISECONDS` (`server/sv_game.c:397`) returns the real wall clock to game code.
- The bot library is given the wall clock (`server/sv_bot.c:584`).
- The random seed comes from the clock: `srand( Com_Milliseconds() )` (`server/sv_init.c:528`).
- A fixed frame time already exists: the `fixedtime` cheat cvar (`qcommon/common.c:4336`).
- Native game code and QVMs may give slightly different floating-point results, so determinism tests should run QVMs.

**Build systems.** A 1,555-line `Makefile` and a 233-line `CMakeLists.txt` coexist. The CMake build's gaps haven't been checked yet.

### Verified by running it

- `make -j` builds the client, dedicated server and OpenGL/Vulkan renderer libraries with no changes.
- Headless runs work: with `SDL_VIDEODRIVER=offscreen` the engine still renders on the GPU through EGL and opens no window, so CI needs no display server. Use `+set nextdemo quit` to exit after a demo.
- Timedemo of `four.dm_68` at 1280×720, five runs each: OpenGL 2423–2466 fps, Vulkan 2456–2540 fps (1260 frames).
- `four.dm_68` runs in about 0.5 s and the engine reports whole milliseconds, so run-to-run noise is around 2–3%. Benchmarks need longer demos and the median of several runs to catch ~1% regressions.
- A demo replay exercises the client and renderer only. Demos store server snapshots, so server-side game logic, physics and collision never run during playback.

## Target game

The first game on the engine: a **medieval, physics-based melee game, 1v1 (2v2 at most)**, multiplayer.

### What the engine lacks for it

- **Skeletons on the server.** Renderers can draw IQM skeletal models (`renderer/tr_model_iqm.c`), but the server has no notion of bones: there is no bone system call in `g_public.h`, and the only tag lookup (`CG_R_LERPTAG`) is client-side. Server-side hit detection uses box or capsule traces (`qcommon/cm_local.h:198`). Melee needs weapon positions from bone poses on the server, so a skeleton and animation module (pose evaluation, blending, later IK) must be shared by client and server and exposed as opt-in API extensions.
- **A physics engine.** There are no rigid bodies or ragdolls; movement is scripted (`bg_pmove`) plus brush traces. Embed **Jolt Physics** (MIT licensed, fast, has a cross-platform deterministic mode) behind the collision interface, controlled from game code through batched system calls. Because the client also runs collision for prediction, client and server must use the same physics.
- **Network protocol extensions.** Entities send a fixed list of fields (`entityStateFields`, `qcommon/msg.c:680`) and inputs are a small fixed struct (`usercmd_t`, `qcommon/q_shared.h:1284`). Melee inputs (swing direction, guard, feint) need opt-in protocol extensions; old mods keep the current protocol.
- **Terrain.** BSP suits castle interiors and courtyards; open terrain and foliage would need the world-model and asset-pipeline phases first. Start with enclosed arenas.

### What carries over

Authoritative server with client prediction, file system, network transport, console and cvars, input, audio, QVM sandboxing for mods, and the Vulkan renderer once its frontend is shared (PBR materials and dynamic shadows matter for the look).

### Why the small player count matters

- Entity and player limits (`MAX_CLIENTS` 64, `MAX_GENTITIES` 1024, `MAX_SNAPSHOT_ENTITIES` 256) are irrelevant, and there's enough bandwidth to send full bone poses if needed.
- `sv_fps` already allows up to 125 Hz (`server/sv_init.c:790`).
- Two to four physics-driven characters are cheap enough to re-simulate several frames per frame.
- This makes **fully simulated combat** (active ragdolls, weapon momentum decides damage) realistic, not just animation-driven combat with cosmetic physics.

### Networking: rollback with a server referee

Every machine runs the same deterministic simulation and only inputs are sent. When a late input arrives, rewind and re-simulate. A dedicated server runs the same simulation as a referee to catch cheating clients. Rationale: the opponent's actions show almost immediately, which timing-dependent parries and reads need; server authority with lag compensation shows them about one round trip late.

Existing footholds:
- Inputs already travel as a small per-tick command (`usercmd_t`, ~20 bytes).
- The client already rewinds and replays its own player through the shared movement code (`bg_pmove`). Rollback applies that idea to the whole match.
- QVM game state is one contiguous block (`dataBase` / `dataLength` in `qcommon/vm_local.h`) with offset-based pointers, so saving and restoring it is a single `memcpy` of a few MB, well under a millisecond. Natively compiled game code can't do this, so **keep game code in QVMs**.
- QVM trig and square roots are system calls (`server/sv_game.c:945` calls libm `sin`), so deterministic maths is a change in one place.
- The other nondeterminism sources (wall clock, random seed) are listed under "Determinism blockers" above.

New work:
1. Clients run the full game module as well as the cgame, much like a listen server.
2. Save and restore engine-side state alongside the QVM: Jolt (has state save/restore), the server's entity links (`server/sv_world.c`) and animation state.
3. Identical results on every machine: Jolt's deterministic mode, fixed compiler floating-point settings (no FMA contraction), deterministic maths functions. Verified by the harness: two machines, same inputs, same hash every frame.
4. Hide corrections: blend toward corrected state rather than snapping, plus a small input delay to reduce rollbacks.
5. Combat systems: shared skeletons, IK, active ragdolls that try to hold poses.

### Biggest risk: prototype first

Before committing, build a stripped-down prototype: two capsules with Jolt-driven sword bodies, rolled back over a simulated 100 ms connection. The question is whether corrections during close contact feel acceptable. That's the risk most likely to sink the design, and it's cheap to test early.

## Audio track: 3D sound with height and reflections

Runs before the target game. It only touches the client (no effect on simulation, networking or mods), it's one of the "easy seams", and it carries straight into the melee game.

**Target: headphones only.** No speaker-specific mode.

### Why height is inaudible today

All positioning happens in `S_SpatializeOrigin` (`code/client/snd_dma.c:393`). It rotates the sound direction into the listener's frame and keeps **only the left/right component** (`dot = -vec[1]`) for the stereo volumes. Forward/back and up/down are discarded, so a sound above, below, in front or behind all sound the same. The only other cue is distance falloff; there is no filtering, delay, reverb or reflection anywhere in the ~4.8k lines of sound code. Output is mono or stereo only (`s_sdlChannels`, `code/sdl/sdl_snd.c:214`). Quake's vertical axis is **Z** (`[2]`), not Y.

### How height becomes audible

1. **HRTF (head-related transfer function) filtering** reproduces how the ear shape filters sound, mostly in the 5–16 kHz range. Works on headphones. A personal HRTF profile (SOFA file) can be loaded later.
2. **Reflections** from floor and ceiling near the listener and the source.

Content limit: 572 of the 575 sound files in retail `pak0.pk3` are 22,050 Hz and 3 are 11,025 Hz, so nothing above 11 kHz exists in the source. HRTF and reflections still help, but the melee game should author sounds at 48 kHz.

### Approach: a new sound backend using Steam Audio

- **The seam exists.** `soundInterface_t` (`code/client/snd_local.h`) is a 21-function interface (start sound, update listener, looping sounds, ...). The new backend implements it next to the current one, selected by a cvar, so the old mixer stays as a fallback and for A/B comparison. The cgame-facing sound API doesn't change, so mods get the improvement for free.
- **Steam Audio** (Apache 2.0, C API, Linux/Windows/macOS): per-sound HRTF, distance and air absorption, occlusion, and reflections computed from the actual level geometry. Quake level geometry is static, so reflections can later be precomputed per map in the asset pipeline (core principle 2).
- Rejected alternative: OpenAL Soft. Good HRTF and quicker to wire up, but its reverb is preset-based and doesn't follow level geometry.

### Stages

1. **Audio test harness.** Render audio offline to WAV deterministically: a test sound at known positions around the listener (above, below, level, behind) plus demo playback. Golden files compare backends and guard regressions. Test data: retail files in `~/Downloads/q3a/baseq3`.
2. **New mixer backend.** Floating point, 48 kHz, SDL output, implementing `soundInterface_t`. Old mixer kept as fallback.
3. **HRTF on every sound** (Steam Audio binaural effect). This is the step that adds height.
4. **Occlusion** via collision traces; the client already has the collision map loaded (`CL_CM_LoadMap`, `code/client/cl_cgame.c:367`).
5. **Geometry-based reflections.** Feed BSP surfaces to Steam Audio, with materials from surface flags (metal, stone, ...). Later, precompute per map in the asset pipeline.

Rough effort: stages 1–3 a few weeks, reflections a few more (estimates).

### Status (Nov 2026)

- **Stages 1–3 done** on branch `audio-spatial`. Test runner: `tests/audio/run_audio_tests.py` (see its docstring). New backend: `code/client/snd_spatial.c`; Steam Audio wrapper: `code/client/snd_steamaudio.c` (loaded at runtime via `s_steamAudioLib`).
- **Full-sphere HRTF.** Steam Audio's built-in HRTF has no data below about −45°, so directly below sounded like 45° down in front. `s_hrtfFile` loads a SOFA file instead. Recommended: SADIE II KU100 (`D1_48K_24bit_256tap_FIR_SOFA.sofa`, University of York, Apache 2.0, https://zenodo.org/records/12092466), 8,802 directions from −90° to +90°. Adds ~0.4 s to sound start-up. The HRTF gain is measured at start-up (pink noise from straight ahead) so loudness matches the classic mixer whatever dataset is loaded.
- Measured cost of HRTF: about 3.5% of one core during the `four` demo, no underruns.
- Next: stage 4 (occlusion), then stage 5 (reflections).

## Core principles

1. **The versioned game API is the contract.** Document the system-call ABI and support versions N and N-1 side by side through compatibility shims. New features are opt-in extensions discovered through `trap_GetValue`. Engine upgrades must never force game or mod rewrites, only opt-in migrations.
2. **Keep source assets separate from runtime formats.** Store content in open source formats (glTF or USD plus our own level description) and "cook" it into runtime formats, including bot navigation data. A renderer upgrade then means a new cooker and a re-cook, not rebuilt content.
3. **No subsystem owns the world model.** Use a neutral scene description that separate providers consume: a render world, a physics/collision world (shared by client and server) and network relevance, each replaceable independently.
4. **Every swap is guarded by tests.** Nothing gets replaced without the harness proving behaviour and performance stay the same.
5. **Performance:** keep indirection off the per-frame hot paths. Prefer data-oriented designs and resolve abstractions at load time, not per call.

## Phased plan

0. **Own the game code.** Bring in the GPL Quake III game, cgame and ui source (as carried by ioquake3), build it as QVMs, and confirm it plays identically to the retail QVMs. It stays a separate module that talks to the engine only through the API. Phases 1 and 2 depend on this.
1. **Test harness.**
   - Deterministic test mode: route `G_MILLISECONDS` and the bot library's clock through a test clock, fix the random seed, and run with `fixedtime`.
   - Server determinism: run the dedicated server with bots in that mode and hash every snapshot; repeated runs must match.
   - Client and renderer: demo playback with screenshots at fixed frames, compared against golden images with a small per-pixel tolerance, for each renderer.
   - Timedemo benchmarks (longer demos, median of several runs) in CI as a performance guard.
   - Open decision: where CI runs. Retail `pak0.pk3` is copyrighted and can't go into GitHub-hosted CI. Options are a self-hosted runner with retail data, or freely licensed content such as OpenArena's assets.
2. **Formalise the game API.** Document every system call, write down the N/N-1 compatibility policy, and add new features as `trap_GetValue` extensions. Version numbers and discovery already exist, so this is mostly documentation and policy.
3. **Share the renderer frontend.** Move the duplicated frontend code (BSP and model loading, world traversal, marks, curves, scene building) into `renderercommon`, leaving the OpenGL and Vulkan backends separate. Estimated 10–15k duplicated lines removed. Guarded by screenshot tests. Decide whether to retire `renderer2` and make Vulkan the forward path.
4. **Asset pipeline.** Source formats in, cooked formats out, including AAS bot navigation data. First reproduce today's BSP/MD3/AAS behaviour exactly (verified by the harness).
5. **Separate the world model.** Split collision and network relevance away from BSP behind their own interfaces, with BSP as the first provider of each. Rendering is already independent at load time, so the hard parts are collision (client and server must stay bit-identical for prediction) and bots.
6. **Upgrade subsystems one at a time** behind those interfaces: renderer features (physically based materials, dynamic shadows and lighting), physics, audio and so on.

**Order for the target game.** The deterministic test mode from phase 1 becomes the foundation the netcode runs on, not just a test feature. Critical path:
1. Phase 0: own the game code.
2. Phase 1: determinism and the harness.
3. Rollback prototype (see "Biggest risk" above).
4. Jolt behind the collision interface (pulls phase 5 forward for collision; network relevance can wait).
5. Shared client/server skeleton and animation module.
6. Rollback netcode with server referee.
7. Combat game code.

Renderer work (phase 3, then PBR and dynamic shadows from phase 6) runs in parallel and doesn't block the game.

Rough effort: phases 0–4 take about a year for a small team and deliver most of the long-term benefit. A full clean-architecture refactor would take roughly 2–4 person-years. Both are estimates, not measurements.

Refactoring approach throughout: strangler-fig. Add tests around one subsystem, pull it behind an interface, then remove the global state it uses. Global-state removal means turning the ~10 large global structs listed above into explicitly passed context objects. Start with the easy seams (file system, network transport, audio, input) before the renderer and world.

Housekeeping, any time: standardise on one build system (probably CMake, so tests can use `ctest`), and replace the bundled third-party libraries with system or package-manager versions.

## Prior art worth studying

- **ioquake3**: the baseline community port (CMake, SDL2) and the source of the GPL game code for phase 0.
- **Daemon engine** (Unvanquished): a descendant of ioq3 and XreaL, an earlier attempt at modernising the renderer. Useful lessons on where incremental upgrades hit limits.

## Suggested next tasks for Claude Code

1. Decide where CI runs (self-hosted with retail data, or OpenArena content).
2. Prototype the phase 1 runner locally: a `tests/` folder with a Python script that runs timedemos on both renderers (headless, several runs, JSON output, baseline comparison) and screenshot comparison against golden images.
3. Phase 0 and the deterministic test mode, then server snapshot hashing.
4. The rollback prototype for the target game.
