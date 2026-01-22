#NO EXIT

# THE BACKROOMS - Pygame Engine

## What this is

A pure Python/Pygame engine for an infinite, walkable backrooms-like space with a focus on:
- **Infinite procedural architecture** - World exists beyond camera, with mega-scale zone variety
- **Persistence** - Progressive wall destruction, drawing/graffiti system, save/load
- **Spatial audio** - Acoustic raycasting, ceiling light buzz with occlusion
- **Physical simulation** - Pixel-sized debris physics, collision-bound entities
- **VHS aesthetics** - Camcorder overlay, scanlines, recording battery life

**What makes it different:** Everything is built from mathematical first principles using only Pygame and NumPy. No game engines, no 3D libraries - just perspective projection, raycasting, and procedural generation. The world genuinely exists as data structures that persist when you walk away.

---

## Quick start

```bash
pip install -r requirements.txt
python3 main.py
```

**Controls:**
- **WASD / Arrow Keys:** Move
- **Mouse (when locked):** Look around
- **M:** Toggle mouse look
- **J/L:** Turn left/right (keyboard)
- **Shift:** Run
- **C:** Toggle crouch
- **Space:** Jump
- **Left Click:** Destroy wall (progressive damage)
- **Right Click + Drag:** Draw on walls
- **R:** Toggle performance mode
- **H:** Toggle help overlay
- **ESC:** Pause / Menu
- **N:** Enable voice echo (if acoustic system available)

**Known issues:**
- Acoustic system requires `sounddevice` or `pyaudio` for microphone input
- Performance varies with debris count (use R to toggle low-res mode)
- Very large zone transitions may cause brief FPS drops

---

## Architecture Map

### Core Systems

| File | Purpose | Owns | Talks to | Key constraints |
|------|---------|------|----------|----------------|
| `main.py` | Entry point + game loop | Game state, UI, input | All systems | Do NOT put game logic here |
| `engine.py` | Orchestrator / glue | Subsystem instances | All subsystems | No implementation - delegates only |
| `config.py` | Configuration constants | All tunables | Nothing | Pure data - no logic |
| `world.py` | Procedural world + streaming | Zone cache, destruction state, debris | Procedural, ceiling_heights, events | Deterministic generation only |
| `player.py` | Player state + movement | Position, rotation, jump state | Collision | No rendering logic |
| `camera.py` | View transform + projection | Camera pose, smoothing | Player | No game logic |
| `collision.py` | Circle-vs-segment physics | Collision resolution | World | Algorithm only - no side effects |
| `renderer.py` | 3D rendering pipeline | Textures, render queue, effects | Camera, world, drawing_system | Pure rendering - no state modification |

### Specialized Systems

| File | Purpose | Owns | Talks to | Key constraints |
|------|---------|------|----------|----------------|
| `procedural.py` | Zone type definitions | Zone properties | Nothing | Pure lookup tables |
| `ceiling_heights.py` | Variable ceiling logic | Height calculations | World, procedural | Deterministic per-position |
| `debris.py` | Particle physics | Debris instances, cracks | Renderer | No world modification |
| `drawing_system.py` | Wall graffiti | Stroke data per wall/pillar | Raycasting | UV-space only |
| `camcorder_overlay.py` | VHS UI + battery | Overlay state, timer | Renderer | Always-on, no toggle |
| `events.py` | Event bus | Subscribers, queue | All systems | Fire-and-forget, no blocking |
| `textures.py` | Procedural texture gen | Surface generation | Nothing | Pure NumPy synthesis |

### Audio

| File | Purpose | Owns | Talks to | Key constraints |
|------|---------|------|----------|----------------|
| `audio.py` | Procedural sound generation | NumPy waveforms | Nothing | Pure synthesis |
| `audio_backends.py` | Microphone input | Audio capture thread | Acoustic integration | Graceful degradation |
| `acoustic_integration.py` | Voice echo system | Raycaster, reverb | Audio backends, world | Optional feature |
| `light_audio_sources.py` | Ceiling light buzz | Light instances, raycasting | Acoustic integration | Spatial audio only |
| `simple_loopback.py` | Mic passthrough | Playback thread, audio queue | Audio backends | Independent 1000Hz update |

### Utilities

| File | Purpose | Owns | Talks to | Key constraints |
|------|---------|------|----------|----------------|
| `raycasting.py` | Möller-Trumbore algorithm | Ray-triangle intersection | Nothing | Pure math |
| `targeting.py` | Crosshair targeting | Ray queries | Camera, world, raycasting | Read-only world queries |
| `save_system.py` | Save/load | JSON serialization | Engine, world, camcorder | File I/O only |
| `smile_entity.py` | "Smile" NPC system | Entity instances, AI state | Collision, renderer | Collision-bound movement |

---

## File Details

### `main.py`
- **Purpose:** Game loop, state machine (MENU, PLAYING, PAUSED), input routing
- **Owns:** `GameState` enum, UI overlays, sound effect instances, auto-save timer
- **Frame order:**
  1. Event polling (keyboard, mouse, quit)
  2. State-specific updates (menu vs playing vs paused)
  3. Engine update (physics, audio, rendering)
  4. Overlay rendering (camcorder, help, game over)
- **Important variables:**
  - `state`: Current game state
  - `drawing_mode`: Whether right-click drawing is active
  - `auto_save_timer`: Countdown to next auto-save (5min intervals)
- **Don't put:** Game logic, physics, rendering algorithms

### `engine.py`
- **Purpose:** Orchestrator that owns and coordinates all subsystems
- **Owns:** `World`, `Player`, `Camera`, `Renderer`, `CollisionSystem`, `DrawingSystem`, `SmileManager`
- **Calls:** Updates each subsystem in order, handles inter-system communication via events
- **Important variables:**
  - Properties for backward compatibility (`x`, `y`, `z`, `pitch`, `yaw` forward to player/camera)
  - `screen_shake_intensity`: Visual feedback for destruction events
  - `sound_timer`, `next_footstep`, `next_buzz`: Ambient sound scheduling
- **Frame order:**
  1. Player update (with collision callback)
  2. Camera smoothing
  3. Screen shake application
  4. Debris physics
  5. Smile entity AI
  6. Event bus processing
- **Don't put:** Direct rendering, input handling, save/load logic

### `player.py`
- **Purpose:** Player physics and state management
- **Owns:** Position `(x, y, z)`, rotation `(pitch, yaw)`, movement flags, jump state
- **Knows about:** `CollisionSystem` (passed from engine), movement speeds from `config`
- **Doesn't know about:** World geometry, rendering, camera
- **Public API:**
  - `update(dt, keys, mouse_rel, collision_checker, current_time)`
  - `toggle_mouse()`: Switch mouse look mode
  - `get_state_for_save()` / `load_state(data)`: Serialization
- **Important:** Collision callback fires on state transitions (not every frame stuck)

### `camera.py`
- **Purpose:** View transformation and perspective projection
- **Owns:** Smoothed camera position `(x_s, y_s, z_s)`, rotation `(pitch_s, yaw_s)`, head bob timers
- **Knows about:** Player state, config constants for smoothing/FOV
- **Doesn't know about:** World geometry, input handling
- **Public API:**
  - `update(dt, player)`: Smooth follow with head bob
  - `world_to_camera(x, y, z)`: Transform world → camera space
  - `project(cam_point)`: Transform camera space → screen coordinates
  - `clip_poly_near(poly)`: Near-plane clipping for rendering
  - `get_ray_direction()`: Ray from screen center for targeting
- **Important:** All projection math uses focal length calculation, not matrix transforms

### `collision.py`
- **Purpose:** Precise circle-vs-line-segment collision with sliding response
- **Owns:** Player radius, skin width, segment cache
- **Knows about:** World (via passed reference), pillar/wall geometry from `config`
- **Doesn't know about:** Player input, rendering, sound effects
- **Public API:**
  - `resolve_collision(from_pos, to_pos)` → `(final_x, final_z, collided)`
- **Algorithm:** Multi-pass depenetration + segment projection + sliding response
- **Important:** Returns corrected position, not just boolean - enables sliding along walls

### `config.py`
- **Purpose:** Single source of truth for all tunables
- **Categories:**
  - Display: `WIDTH`, `HEIGHT`, `FULLSCREEN`, `FPS`
  - Movement: `WALK_SPEED`, `RUN_SPEED`, `CROUCH_SPEED`, `JUMP_STRENGTH`
  - World: `PILLAR_SPACING`, `WALL_HEIGHT`, `ZONE_SIZE`, `PILLAR_MODE`
  - Camera: `CAMERA_SMOOTHING`, `HEAD_BOB_AMOUNT`, `FOV`
  - Audio: `SAMPLE_RATE`, `FOOTSTEP_INTERVAL`
- **Helper functions:**
  - `get_scaled_wall_height()`: Ceiling height with multiplier
  - `get_scaled_floor_y()`: Floor Y position
  - `get_scaled_head_bob_*()`: Head bob with ceiling multiplier
- **Important:** Change `PILLAR_MODE` to control internal pillar density: `"none"`, `"sparse"`, `"normal"`, `"dense"`, `"all"`

### `procedural.py`
- **Purpose:** Zone type definitions and property lookup
- **Owns:** `ZONE_TYPES` dictionary with zone configurations
- **Public API:**
  - `get_zone_type(zone_x, zone_z, seed)`: Deterministic zone selection
  - `get_zone_properties(zone_x, zone_z, seed)`: Returns property dict
- **Zone types:**
  - **Human-scale:** `normal`, `dense`, `sparse`, `maze`, `open`
  - **Mega-scale:** `atrium` (6x), `coliseum` (10x), `courtyard` (8x), `cathedral` (9x), `warehouse` (4x)
- **Properties:**
  - `pillar_density`: 0.0-1.0 probability of pillar at grid point
  - `wall_chance`: 0.0-1.0 probability of wall between pillars
  - `scale_multiplier`: Room size multiplier (1.0 = 400 units)
  - `room_size`: Actual grid spacing in world units
  - `min_ceiling` / `max_ceiling`: Ceiling height range
  - `color_tint`: RGB multiplier for zone color variation
- **Important:** Deterministic - same coords always return same zone type

### `ceiling_heights.py`
- **Purpose:** Per-position ceiling height calculations (zone-aware)
- **Owns:** Nothing (pure functions)
- **Public API:**
  - `get_ceiling_height_at_position(x, z, world)` → height in world units
  - `get_room_size_at_position(x, z, world)` → grid spacing
  - `snap_to_zone_grid(x, z, world)` → `(grid_x, grid_z)`
  - `get_zone_info_string(x, z, world)` → debug string
- **Important:** Uses deterministic random based on position hash - not truly random

### `debris.py`
- **Purpose:** Particle physics for wall destruction
- **Classes:**
  - `DamageState`: Enum for wall damage progression
  - `Crack`: Growing crack on wall surface (visual guide)
  - `Debris`: Pixel-sized particle with gravity physics
  - `RubbleChunk`: Heavy debris that settles permanently
  - `DamagedWall`: Structural failure controller (cracks → leans → falls → rubble)
- **Important:** Debris instances are self-contained - they don't modify world state

### `drawing_system.py`
- **Purpose:** Persistent wall/pillar graffiti system
- **Owns:** `wall_drawings` dict, `pillar_drawings` dict, current stroke state
- **Storage format:**
  - Walls: `{wall_key: [stroke1, stroke2, ...]}`
  - Pillars: `{pillar_key: {face: [stroke1, stroke2, ...]}}`
  - Each stroke: `[(u, v), (u, v), ...]` in 0-1 UV space
- **Public API:**
  - `start_stroke(wall_key, uv_pos)` / `start_pillar_stroke(pillar_key, face, uv_pos)`
  - `add_to_stroke(uv_pos)`: Continuous drawing
  - `end_stroke()`: Finalize and save stroke
  - `get_drawings_for_wall(wall_key)` / `get_drawings_for_pillar(pillar_key, face)`
- **Helper functions:**
  - `world_to_wall_uv(hit_point, wall_key, world)` → `(u, v)`
  - `pillar_world_to_uv(hit_point, pillar_key, face)` → `(u, v)`
  - `get_wall_hit_point(camera, world, target_info)` → world coords
  - `get_pillar_hit_point_and_face(camera, world, target_info)` → `(world_coords, face_name)`
- **Important:** UV space is 0-1 normalized - independent of world scale

### `camcorder_overlay.py`
- **Purpose:** VHS camcorder UI and battery life system
- **Owns:** Battery state, recording time, VHS effect state, timestamp
- **Features:**
  - Blinking REC indicator
  - Real-time date/timestamp (starts at session begin)
  - Battery indicator with percentage and time remaining
  - VHS scanlines, tracking glitches, vignette
  - Camera info display (SP MODE, AUTO FOCUS, WHITE BAL)
  - Tape counter (H:MM:SS format)
  - Viewfinder guides (center crosshair corners)
- **Public API:**
  - `update(dt)`: Depletes battery, updates effects
  - `render(surface)`: Draws all overlay elements
  - `is_battery_dead()` → bool
  - `recharge_battery(percent)`: For testing/gameplay mechanics
- **Important:** Always enabled (no toggle), battery life configurable at init

### `events.py`
- **Purpose:** Decoupled event bus for system communication
- **Classes:**
  - `EventType`: Enum of all event types
  - `Event`: Data container with attribute access
  - `EventBus`: Pub/sub dispatcher
- **Event types:**
  - Wall: `WALL_HIT`, `WALL_CRACKED`, `WALL_FRACTURED`, `WALL_BREAKING`, `WALL_DESTROYED`
  - Pillar: `PILLAR_HIT`, `PILLAR_DESTROYED`
  - Debris: `DEBRIS_IMPACT`, `DEBRIS_SETTLED`
  - Player: `PLAYER_STEP`, `PLAYER_LAND`
  - Atmosphere: `FLICKER`
- **Usage pattern:**
  ```python
  from events import event_bus, EventType
  
  # Subscribe
  event_bus.subscribe(EventType.WALL_DESTROYED, my_handler)
  
  # Emit
  event_bus.emit(EventType.WALL_DESTROYED, 
                 wall_key=key, position=(x, y, z))
  
  # Handler
  def my_handler(event):
      print(event.wall_key, event.position)
  ```
- **Important:** Use `queue()` + `process_queue()` during update loops to avoid mutation-during-iteration

### `audio.py`
- **Purpose:** Runtime procedural audio generation using NumPy
- **Sounds generated:**
  - `generate_backrooms_hum()`: Low-frequency drone with modulation
  - `generate_footstep_sound()`: Distant ambient footsteps
  - `generate_player_footstep_sound(turn_factor)`: Carpet step with directional stereo
  - `generate_crouch_footstep_sound(turn_factor)`: Softer, slower step
  - `generate_electrical_buzz()`: 120Hz buzz for lights
  - `generate_destroy_sound()`: Wall destruction impact + crumble
  - `generate_crack_sound()`: Sharp crack + grit
  - `generate_fracture_sound()`: Layered cracks + debris
  - `generate_collision_boink(intensity)`: Hollow thud for collisions
  - `generate_soft_bump()`: Gentle collision sound
- **Important:** All sounds are generated at runtime - no audio files needed

### `audio_backends.py`
- **Purpose:** Flexible microphone input with auto-detection
- **Backends (priority order):**
  1. `sounddevice` (recommended, easiest install)
  2. `pyaudio` (traditional, harder install)
  3. `WaveFileBackend` (test tone generator fallback)
- **Classes:**
  - `AudioBackend`: Base class
  - `SoundDeviceBackend`, `PyAudioBackend`, `WaveFileBackend`: Implementations
  - `MicrophoneCapture`: Simplified API wrapper
- **Usage:**
  ```python
  mic = MicrophoneCapture(sample_rate=22050, chunk_size=1024)
  mic.start()
  
  while True:
      chunk = mic.get_audio_chunk()  # Returns np.int16 array or None
      if chunk:
          process_audio(chunk)
  
  mic.stop()
  ```
- **Important:** Graceful degradation - falls back to test tone if no mic available

### `acoustic_integration.py`
- **Purpose:** Voice echo system with raycasting
- **Classes:**
  - `AcousticRaycaster`: Casts rays to calculate reverb
  - `AcousticIntegration`: Main integration class
- **Features:**
  - Microphone input capture
  - Ray-based room size estimation
  - Dynamic reverb calculation
  - Echo playback with spatial attenuation
  - Visualization of acoustic rays
- **Integration points:**
  - `update(dt)`: Process audio, update reverb
  - `render_visualization(screen, camera)`: Draw acoustic rays
  - `toggle()`: Enable/disable system
- **Important:** Optional feature - main game works without it

### `light_audio_sources.py`
- **Purpose:** Ceiling lights as spatial audio emitters
- **Classes:**
  - `LightSource`: Single ceiling light instance
  - `LightAudioManager`: Manages light streaming around player
- **Integration:**
  - `integrate_light_audio(acoustic_integration, player_pos, dt)`: Main update
  - `render_light_audio_viz(screen, camera, light_audio_result, fade)`: Visualization
- **Features:**
  - Lights positioned at ceiling height (zone-aware)
  - Audio raytracing with occlusion
  - Distance falloff and volume mixing
  - Green rays = direct path, Red rays = occluded
- **Important:** Requires acoustic system to be enabled

### `raycasting.py`
- **Purpose:** Möller-Trumbore ray-triangle intersection
- **Public API:**
  - `ray_intersects_triangle(ray_origin, ray_dir, v0, v1, v2)` → `(distance, triangle)` or `None`
- **Algorithm:** Standard graphics algorithm, no modifications
- **Important:** Pure math - no side effects, no world knowledge

### `world.py`
- **Purpose:** Procedural world generation, zone management, destruction state
- **Owns:** 
  - Zone cache: `zone_cache`, `pillar_cache`, `wall_cache`
  - Destruction state: `destroyed_walls`, `destroyed_pillars`, `destroyed_lamps`
  - Progressive damage: `wall_states`, `wall_health`, `wall_cracks`
  - Debris: `debris_pieces`, `_spawned_rubble`
  - World seed: `world_seed`
- **Knows about:** `ProceduralZone`, `ceiling_heights`, `Debris`, `event_bus`
- **Doesn't know about:** Rendering, player input, camera
- **Public API:**
  - `get_zone_at(x, z)` → `(zone_x, zone_z)`
  - `get_zone_properties(zone_x, zone_z)` → property dict
  - `has_pillar_at(px, pz)` → bool (cached, deterministic)
  - `has_wall_between(x1, z1, x2, z2)` → bool (zone-aware, cached)
  - `is_wall_destroyed(wall_key)` / `is_pillar_destroyed(pillar_key)` → bool
  - `get_wall_damage(wall_key)` → float (1.0 = intact, 0.0 = rubble)
  - `get_doorway_type(x1, z1, x2, z2)` → "solid" / "doorway" / "hallway"
  - `hit_wall(wall_key, damage)` → bool (returns True if wall destroyed)
  - `destroy_wall(wall_key, destroy_sound)`: Instant destruction + debris spawn
  - `destroy_pillar(pillar_key, destroy_sound)`: Instant pillar destruction
  - `check_collision(x, z)` → bool (simple collision query, deprecated - use CollisionSystem instead)
  - `update_debris(dt, player_x, player_z)`: Update and cull debris
  - `get_state_for_save()` / `load_state(data)`: Serialization
- **Enums:**
  - `WallState`: `INTACT`, `CRACKED`, `FRACTURED`, `BREAKING`, `DESTROYED`
- **Important details:**
  - **Zone-aware grid:** Uses `get_room_size_at_position()` to query correct grid spacing for mega-scale zones
  - **Deterministic generation:** Same seed + coords always produces same result
  - **Pre-damaged walls:** Some walls spawn partially damaged based on zone `decay_chance`
  - **Progressive damage system:** Walls transition through damage states, emit events
  - **Debris management:** Hard cap at 12,000 particles, auto-culls distant/old debris
  - **Caching:** All queries are cached for performance (cleared on load)

### `renderer.py`
- **Purpose:** 3D rendering pipeline with effects, textures, and optimizations
- **Owns:**
  - Render surface: `render_surface`, `render_scale`, `target_render_scale`
  - Textures: `carpet_texture`, `ceiling_texture`, `wall_texture`, `pillar_texture`
  - Average colors: `carpet_avg`, `ceiling_avg`, `wall_avg`, `pillar_avg`
  - Flicker state: `flicker_timer`, `is_flickering`, `flicker_brightness`
- **Knows about:** Camera, world, drawing_system, textures module, config
- **Doesn't know about:** Player input, game logic, physics
- **Public API:**
  - `render(surface, camera, world, player, drawing_system)`: Main render call
  - `toggle_render_scale()`: Switch between 1.0x and 0.5x resolution
  - `update_render_scale(dt)`: Smooth transition animation
  - `update_flicker(dt)`: Update light flicker effect
- **Internal methods:**
  - `draw_world_poly(...)`: Draw 3D polygon with all effects
  - `apply_fog(color, distance)` → fogged color
  - `apply_surface_noise(color, x, z)` → noisy color
  - `apply_zone_tint(color, world, zone_x, zone_z)` → tinted color
  - `_get_floor_tiles(camera, world)` → list of (depth, draw_func) tuples
  - `_get_ceiling_tiles(camera, world)` → list of (depth, draw_func) tuples
  - `_get_walls(camera, world, drawing_system)` → list of (depth, draw_func) tuples
  - `_get_pillars(camera, world, drawing_system)` → list of (depth, draw_func) tuples
  - `_render_debris(surface, camera, world)`: Draw all debris particles
  - `_draw_wall_drawings(...)`: Render graffiti on walls
  - `_draw_pillar_drawings(...)`: Render graffiti on pillars
- **Rendering pipeline:**
  1. Clear render surface to BLACK
  2. Build render queue (floor, ceiling, walls, pillars)
  3. Sort back-to-front by depth
  4. Execute all draw functions
  5. Render debris (sorted by depth)
  6. Scale up to target resolution if needed
  7. Draw crosshair overlay
- **Visual effects:**
  - **Fog:** Linear interpolation from `FOG_START` to `FOG_END`
  - **Flicker:** Random light dimming with configurable chance/duration
  - **Ambient occlusion:** Walls darker near floor/ceiling
  - **Zone tint:** RGB multiplier per zone type
  - **Surface noise:** Deterministic per-position variation
- **Optimizations:**
  - Frustum culling (margin: 500 pixels)
  - Near-plane clipping via camera
  - Render distance culling (configurable)
  - Dynamic resolution scaling (R key toggles)
  - Back-to-front sorting for painter's algorithm
  - Debris distance culling (600 unit radius)
- **Important:** 
  - Uses procedural textures (no image files)
  - Drawings rendered as 3D world-space strokes
  - Progressive damage states affect wall color
  - All rendering is stateless - doesn't modify world

### `targeting.py`
- **Purpose:** Find wall or pillar under crosshair using raycasting
- **Owns:** Nothing (pure query function)
- **Knows about:** Camera, world, raycasting, config
- **Doesn't know about:** Player input, rendering, game state
- **Public API:**
  - `find_targeted_wall_or_pillar(camera, world)` → `(type, key)` or `None`
    - `type`: "wall" or "pillar"
    - `key`: wall tuple `((x1, z1), (x2, z2))` or pillar tuple `(x, z)`
- **Algorithm:**
  1. Get ray from camera center (`camera.get_ray_direction()`)
  2. Check all walls/pillars in render range (200 units)
  3. Test ray against all triangles for each face
  4. Return closest hit within max distance (100 units)
- **Important:**
  - Respects destroyed state (skips destroyed walls/pillars)
  - Checks all 4 pillar faces
  - Checks both horizontal and vertical walls
  - Uses Möller-Trumbore for intersection tests
  - Read-only world queries

### `save_system.py`
- **Purpose:** Game state persistence to JSON files
- **Owns:** Nothing (static utility class)
- **Knows about:** Engine structure, file system, `SAVE_DIR` from config
- **Doesn't know about:** Rendering, physics, audio
- **Public API:**
  - `SaveSystem.ensure_save_dir()`: Create save directory
  - `SaveSystem.get_save_path(slot)` → filepath string
  - `SaveSystem.save_game(engine, camcorder, slot)` → bool
  - `SaveSystem.load_game(slot)` → save_data dict or None
  - `SaveSystem.list_saves()` → list of save metadata dicts
- **Save structure:**
  ```python
  {
    'version': '1.0',
    'timestamp': ISO datetime,
    'player': {x, y, z, pitch, yaw},
    'world': {
      'seed': int,
      'destroyed_walls': [wall_key_list],
      'wall_damage': {key_str: damage_value},
      'destroyed_pillars': [pillar_key_list]
    },
    'drawings': {wall_drawings dict, pillar_drawings dict},
    'smiles': {entity states},
    'camcorder': {battery_seconds, recording_time, ...},
    'stats': {play_time}
  }
  ```
- **Important:**
  - Saves to `backrooms_saves/save_slot_{N}.json`
  - Auto-saves every 5 minutes to slot 1
  - Saves on exit
  - Wall keys converted to string format for JSON
  - Engine must implement `load_from_save()` to consume

### `smile_entity.py`
- **Purpose:** Collision-bound "Smile" NPC that follows player
- **Classes:**
  - `SmileEntity`: Individual entity instance
  - `SmileManager`: Manages multiple entities and spawn zones
- **Philosophy:**
  - **Geometry is authority** - no teleporting, no cheating
  - **Collision-bound movement** - slides along walls like player
  - **Billboard smile** - face always points at camera, never tilts
  - **No pathfinding** - simple directional bias, collision decides outcome
  - **Quantum observer effect** - 85% chance to freeze when directly observed
- **SmileEntity API:**
  - `__init__(spawn_x, spawn_z, radius, speed)`
  - `update(dt, player_pos, collision_system, camera_looking_at_me)`
  - `render(surface, camera, world)`: Draw sphere with smile face
  - `is_visible_to_camera(camera, world)` → bool (occlusion test)
  - `is_centered_in_view(camera)` → bool (within view cone)
  - `get_state_for_save()` / `load_state(data)`: Serialization
- **SmileManager API:**
  - `add_spawn_zone(x, z, count)`: Add entities at location
  - `update(dt, player_pos, collision_system, camera)`: Update all entities
  - `render(surface, camera, world)`: Render visible entities only
  - `get_state_for_save()` / `load_state(data)`: Serialization
- **SmileEntity state:**
  - Position: `x`, `z` (always on floor)
  - Velocity: `vx`, `vz` (current motion)
  - Visual rotation: `rotation_x`, `rotation_z` (for rolling effect)
  - Zone constraint: `spawn_x`, `spawn_z`, `max_roam_distance` (800 units)
  - Stuck detection: `last_moved_time`, `stuck_threshold`
- **Behavior:**
  - **Following:** Weak directional bias toward player (speed: 100 units/sec)
  - **Personal space:** Stops following when within 3× radius
  - **Zone constraint:** Slows near edge, drifts back if beyond boundary
  - **Observer effect:** 85% freeze when centered in view
  - **Collision:** Uses `CollisionSystem.resolve_collision()` like player
  - **Rolling:** Visual only - rotation based on actual movement distance
- **Rendering:**
  - Sphere wireframe (segmented circles)
  - Billboard smile face (always faces camera)
  - Occlusion testing (hidden behind walls)
  - Distance culling
- **Important:**
  - No navigation mesh, no pathfinding, no A*
  - Movement is pure "intent → collision → outcome"
  - Can get permanently stuck (intentional)
  - Smile never rotates with sphere (billboard effect)
  - Multiple entities managed by SmileManager

### `textures.py`
- **Purpose:** Runtime procedural texture generation
- **Owns:** Nothing (pure generation functions)
- **Knows about:** NumPy, pygame, `TEXTURE_SIZE` from config
- **Doesn't know about:** World, rendering pipeline, game state
- **Public API:**
  - `generate_carpet_texture(size)` → pygame.Surface
  - `generate_ceiling_tile_texture(size)` → pygame.Surface
  - `generate_wall_texture(size)` → pygame.Surface
  - `generate_pillar_texture(size)` → pygame.Surface
- **Texture details:**
  - **Carpet:** Blue base (30, 60, 140) with random noise (-15 to +15)
  - **Ceiling:** Light blue (200, 200, 240) with sine/cosine pattern + noise
  - **Wall:** Yellow (240, 220, 80) with vertical line pattern (every 8 pixels)
  - **Pillar:** Bright yellow (250, 230, 90) with random noise
- **Process:**
  1. Create NumPy array (size × size × 3)
  2. Fill with base color + pattern/noise
  3. Convert to pygame surface with `swapaxes(0, 1)` for correct orientation
- **Important:**
  - Generated once at startup (in `Renderer.__init__()`)
  - No image files required
  - Deterministic patterns (not random per-pixel)
  - Default size: 256×256 (from `TEXTURE_SIZE`)

### `simple_loopback.py`
- **Purpose:** Simple microphone-to-speaker passthrough for testing
- **Owns:** 
  - Microphone: `mic` (MicrophoneCapture instance)
  - Playback queue: `playback_queue` (queue.Queue)
  - Worker threads: `playback_thread`, `update_thread`
  - State: `enabled`, `should_stop`
- **Knows about:** `audio_backends`, pygame.mixer, threading
- **Doesn't know about:** Game state, acoustic system, raycasting
- **Public API:**
  - `SimpleAudioLoopback()`: Constructor
  - `start()` → bool: Start mic capture and playback
  - `stop()`: Stop all threads
  - `update()`: No-op (update thread runs independently)
- **Architecture:**
  - **Mic capture thread:** Runs in `MicrophoneCapture` backend
  - **Update thread:** 1000 Hz polling loop (`_update_worker()`)
  - **Playback thread:** Synchronized audio playback (`_playback_worker()`)
- **Audio processing:**
  - Convert int16 → float32
  - Gentle normalization (preserve dynamics)
  - 3x gain boost for audibility
  - Clip to prevent distortion
  - Convert back to int16 stereo
- **Update loop (1000 Hz):**
  1. Get audio chunk from mic
  2. Process chunk (gain, normalization)
  3. Manage queue (drop old if full)
  4. Queue processed audio
  5. Sleep 1ms
- **Playback loop:**
  1. Wait for audio chunk in queue
  2. Wait for previous chunk to finish
  3. Play new chunk
  4. Track current channel
- **Important:**
  - Independent of game FPS (runs at 1000 Hz)
  - Moderate queue size (8 chunks) balances latency/smoothness
  - No reverb, no effects - pure passthrough
  - Graceful degradation if mic unavailable
  - Used for acoustic system testing

---

## Frame Order (Per Tick)

**Main game loop in `main.py`:**
1. Clock tick (60 FPS target)
2. Event polling (pygame events)
3. State-specific logic (menu, paused, playing)

**When PLAYING, `engine.update()` does:**
1. Player input processing → movement intent
2. Collision resolution → final position
3. Camera smoothing to follow player
4. Screen shake application (decay)
5. Debris physics update (gravity, settlement)
6. Smile entity AI update
7. Event bus processing (queued events fire)

**Audio updates (separate from main update):**
1. Ambient sound timer checks (footsteps, buzz)
2. Player footstep sync to head bob phase
3. Acoustic system update (if enabled)
4. Light audio integration (if acoustic enabled)

**Rendering (`engine.render()`):**
1. World geometry render (walls, pillars, floor, ceiling)
2. Debris render (particles and chunks)
3. Smile entity render (with occlusion)
4. Drawing system render (strokes on walls/pillars)
5. Camcorder overlay render (always on top)
6. Acoustic visualization (if enabled)
7. UI overlays (help, save message, drawing mode)

---

## Shared State / Contracts

### State Keys (conceptual - implementation varies)

**Engine holds:**
- `world`: World instance
- `player`: Player instance
- `camera`: Camera instance
- `collision_system`: CollisionSystem instance
- `drawing_system`: DrawingSystem instance
- `smile_manager`: SmileManager instance

**Player state:**
- Position: `x`, `y`, `z` (world coordinates, units in config)
- Rotation: `pitch`, `yaw` (radians)
- Flags: `is_moving`, `is_rotating`, `is_running`, `is_crouching`, `is_jumping`

**Camera state:**
- Smoothed position: `x_s`, `y_s`, `z_s`
- Smoothed rotation: `pitch_s`, `yaw_s`
- Timers: `head_bob_time`, `camera_shake_time`

**World state (conceptual - missing `world.py`):**
- `world_seed`: int
- `destroyed_walls`: set of wall keys
- `destroyed_pillars`: set of pillar keys
- `wall_health`: dict of wall health (1.0 = healthy, 0.0 = destroyed)
- `debris_particles`: list of Debris instances
- `damaged_walls`: dict of DamagedWall controllers

### Coordinate Systems

**World space:**
- Origin: (200, 50, 200) - player spawn point
- Units: Config-defined (PILLAR_SPACING = 400)
- Y-axis: Up (floor at negative Y, ceiling at positive Y)
- Grid: Aligned to PILLAR_SPACING intervals

**Camera space:**
- Origin at camera position
- Z-axis points forward (into screen)
- Transformations: yaw rotation, then pitch rotation

**Screen space:**
- Origin at top-left
- X-axis: right
- Y-axis: down
- Units: pixels

**UV space (for drawings):**
- Origin at bottom-left of surface
- Range: 0.0 to 1.0 on both axes
- Independent of world scale

### Naming Rules

- **Angles:** Always radians
- **Distances:** World units (config-defined)
- **Time:** Always seconds (`dt` is delta time in seconds)
- **Keys:** Tuples of coordinates, e.g., `((x1, z1), (x2, z2))` for walls
- **Colors:** RGB tuples (0-255) or RGBA (0-255, 0-255, 0-255, 0-255)

---

## Save/Load System

**Files saved to:** `backrooms_saves/` directory

**Save slots:** Numbered 1-3 (slot 1 used for auto-save)

**What gets saved:**
- Player state (position, rotation)
- World state (destroyed walls/pillars, debris)
- Drawing system (all strokes on all walls/pillars)
- Camcorder state (battery, recording time)
- Smile entities (positions, states)
- Stats (play time)

**Format:** JSON with custom serialization for complex objects

**Important:** Auto-saves every 5 minutes to slot 1, saves on exit

---

## Performance Considerations

**Bottlenecks:**
1. Debris physics (quadratic with particle count)
2. Drawing system rendering (linear with stroke count)
3. Acoustic raycasting (configurable ray count)
4. World geometry generation (lazy, but spikes on zone transition)

**Optimization strategies:**
- Press `R` to toggle low-res mode (reduces render resolution)
- Debris auto-culls when old or far from player
- Drawings only render for visible walls
- Acoustic rays limited to configurable count (default: 32)
- World chunks generated/unloaded based on player position

**Memory usage:**
- Debris: ~50 bytes per particle (cleared regularly)
- Drawings: ~16 bytes per UV point (persistent)
- World: Sparse - only stores exceptions (destroyed walls, etc.)

---

## How To...

### Add a new zone type

1. Open `procedural.py`
2. Add entry to `ProceduralZone.ZONE_TYPES`:
   ```python
   'my_zone': {
       'pillar_density': 0.3,      # 0.0-1.0
       'wall_chance': 0.2,         # 0.0-1.0
       'ceiling_height_var': 10,   # variance in units
       'color_tint': (1.0, 1.0, 1.0),  # RGB multiplier
       'scale_multiplier': 1.0,    # room size multiplier
       'room_size': 400,           # grid spacing
       'min_ceiling': 100,         # min height
       'max_ceiling': 120,         # max height
   }
   ```
3. Add decay chance in `get_zone_properties()`:
   ```python
   decay_chances = {
       ...
       'my_zone': 0.15,
   }
   ```

### Add a new sound effect

1. Open `audio.py`
2. Create generation function:
   ```python
   def generate_my_sound():
       duration = 1.0
       samples = int(SAMPLE_RATE * duration)
       t = np.linspace(0, duration, samples, False)
       
       # Your waveform here
       sound = np.sin(2 * np.pi * 440 * t)
       
       # Normalize and convert
       sound = sound / np.max(np.abs(sound)) * 0.7
       audio = np.array(sound * 32767, dtype=np.int16)
       stereo_audio = np.column_stack((audio, audio))
       
       return pygame.sndarray.make_sound(stereo_audio)
   ```
3. Generate in `main.py`:
   ```python
   my_sound = generate_my_sound()
   sound_effects['my_sound'] = my_sound
   ```

### Add a new overlay element

1. Create new class in camcorder_overlay.py style
2. Initialize in `main.py` after engine creation
3. Call `overlay.update(dt)` in game loop
4. Call `overlay.render(screen)` after engine render

### Modify collision behavior

1. Open `collision.py`
2. Adjust `player_radius` or `skin_width` in `CollisionSystem.__init__()`
3. For sliding behavior, modify `resolve_collision()` algorithm
4. For segment detection range, adjust `check_range` in `_get_nearby_segments()`

---

## Architecture Principles

### Separation of Concerns

- **No rendering in game logic:** Player, collision, debris don't import pygame.draw
- **No game logic in rendering:** Renderer receives pre-computed geometry, doesn't modify state
- **No world knowledge in math:** Raycasting, camera projection are pure functions
- **Events for cross-system communication:** Avoid direct coupling via event bus

### Data Flow

```
Input → Player → Collision → World State → Camera → Renderer → Screen
                     ↓           ↑
                  Events ← Event Bus → Sound/FX
```

### Determinism

- **Procedural generation:** Always deterministic from seed + coordinates
- **Physics:** Deterministic given dt and initial conditions
- **Rendering:** Deterministic from camera + world state

### File Organization

```
Core loop:     main.py → engine.py
Game state:    player.py, camera.py, world.py (missing docs)
Physics:       collision.py, debris.py
Rendering:     renderer.py (missing docs), drawing_system.py
Procedural:    procedural.py, ceiling_heights.py
Audio:         audio.py, audio_backends.py, acoustic_integration.py, light_audio_sources.py
UI:            camcorder_overlay.py
Utils:         config.py, events.py, raycasting.py
Persistence:   save_system.py (missing docs), targeting.py (missing docs)
Entities:      smile_entity.py (missing docs)
```

---

## File Organization

```
Core loop:     main.py → engine.py
Game state:    player.py, camera.py, world.py
Physics:       collision.py, debris.py
Rendering:     renderer.py, textures.py, drawing_system.py
Procedural:    procedural.py, ceiling_heights.py
Audio:         audio.py, audio_backends.py, acoustic_integration.py, 
               light_audio_sources.py, simple_loopback.py
UI:            camcorder_overlay.py
Utils:         config.py, events.py, raycasting.py, targeting.py
Persistence:   save_system.py
Entities:      smile_entity.py
```

---

### Where new features go

- **New movement mechanics** → `player.py`
- **New physics** → New file in root (follow `collision.py` or `debris.py` pattern)
- **New rendering effects** → `renderer.py` or new overlay class
- **New procedural content** → `procedural.py` (zone types) or `ceiling_heights.py` (geometry)
- **New sounds** → `audio.py` (generation) + `main.py` (integration)
- **New UI elements** → New overlay class (follow `camcorder_overlay.py`)

### Adding a new entity type

1. Create new file `my_entity.py`
2. Define entity class with:
   - `__init__(position, ...)`
   - `update(dt, player_pos, collision_system, camera)`
   - `render(screen, camera, world)` (with occlusion)
   - `get_state_for_save()` / `load_state(data)`
3. Create manager class (follow `SmileManager` pattern)
4. Initialize in `engine.py`
5. Update/render in `engine.update()` and `engine.render()`

### Performance-sensitive areas

- **Debris update loop** - Keep particle count low, cull aggressively
- **Collision segment generation** - Cache where possible, limit range
- **Acoustic raycasting** - Limit ray count, use early-out
- **Drawing stroke rendering** - Only render visible walls, use dirty rect optimization

---

## Philosophy & Design Goals

### "Defeat bugs by cheating them"

Build systems that naturally want to behave correctly:
- Collision returns corrected position (can't be ignored)
- Debris particles self-cull (can't leak memory)
- Events queue during iteration (can't corrupt state)
- Procedural generation is deterministic (can't desync)

### "The world exists"

The Backrooms persist beyond the camera:
- Destroyed walls stay destroyed
- Drawings remain on walls
- Debris settles and stays settled
- Entity AI runs even off-screen

### "Build from first principles"

Everything mathematical is implemented directly:
- Perspective projection from FOV
- Ray-triangle intersection from Möller-Trumbore
- Audio synthesis from sine waves
- Collision from geometric primitives

No black boxes, no magic - just math and iteration.

---

## Credits & Philosophy

Built by Demon Lord (Gulp of Wealth LLC) as an exploration of:
- What you can achieve with pure Python + Pygame
- How procedural generation creates "place feeling"
- How physical simulation enhances immersion
- How constraints breed creativity

The engine is a love letter to:
- 1990s software rendering
- VHS aesthetics and liminal spaces
- Emergent complexity from simple rules
- The joy of building systems that surprise you

---

## License

[Add your license here]

---

## Changelog

[Track major updates here as you iterate]
