# Endless Wandering: A Meditation on Striving

**Endless Wandering** is an interactive 3D experience built with Three.js. It serves as a digital meditation on the theme of endless striving inherent in the human condition. The player navigates an infinite, procedurally generated landscape, encountering scattered orbs. Collecting these orbs reveals philosophical quotes, prompting moments of reflection on effort, meaning, and the perpetual nature of goals. The environment also features ephemeral "anomalies" – fleeting visual oddities that add to the contemplative and sometimes surreal atmosphere of the journey.

## Core Features

* **Infinite Procedural Terrain**: Explore a never-ending landscape generated on the fly.
* **Collectible Orbs & Philosophical Quotes**: Discover orbs that trigger quotes related to striving, freedom, and the search for meaning.
* **Atmospheric Anomalies**: Witness temporary, glowing orbs that appear and fade, adding to the otherworldly feel.
* **First-Person Exploration**: Navigate the world from a first-person perspective.
* **Dynamic Starry Sky**: A procedurally generated starfield enhances the sense of an expansive, open world.

## Implementation Details

This project leverages several key Three.js features and custom logic to create the experience:

### 1. Terrain Generation

* **Simplex Noise**: The core of the terrain height generation relies on `simplex-noise.js`. A 2D Simplex noise function determines the elevation at any given (x, z) world coordinate.
* **Fractional Brownian Motion (fBm)**: To create more natural and detailed terrain, multiple "octaves" of noise are layered. Each octave uses a higher frequency and lower amplitude than the last.
    * `TERRAIN_OCTAVES`: Defines the number of noise layers.
    * `NOISE_SCALE`: Controls the overall scale of the base noise pattern.
    * `TERRAIN_AMPLITUDE`: Determines the maximum height difference of the terrain.
    * `TERRAIN_LACUNARITY`: Factor by which frequency increases for each subsequent octave.
    * `TERRAIN_PERSISTENCE`: Factor by which amplitude decreases for each subsequent octave.
* **Chunking System**: To manage an "infinite" world, the terrain is divided into `CHUNK_SIZE` x `CHUNK_SIZE` segments.
    * `activeChunks`: A `Map` stores currently loaded terrain chunks around the player.
    * `chunkLoadRadius`: Determines how many chunks in each direction from the player's current chunk are kept loaded.
    * As the player moves, new chunks are generated for the leading edge of the load radius, and chunks falling outside the trailing edge are disposed of to save resources. This includes disposing of their geometries, materials, and any decorative objects parented to them.
* **Vertex Colors**: Instead of a uniform texture, the terrain material uses `vertexColors`. The color of each vertex in the terrain mesh is calculated based on its normalized height. This creates a gradient effect, with lower areas appearing darker (e.g., deep blues/greys) and higher peaks lighter (e.g., light greys/whites), adding visual depth and variation.
* **Decorative Objects**: Small, randomly generated geometric shapes (spheres, boxes, dodecahedrons) with varied metallic materials are scattered across the terrain chunks to add visual interest. Their spawn is probabilistic.

### 2. Player Movement & Camera

* **PointerLockControls**: Three.js's `PointerLockControls` is used for standard first-person mouse-look and keyboard (WASD) input for movement direction.
* **Custom Physics**:
    * `playerVelocity`: A `Vector3` tracks the player's current speed and direction.
    * `PLAYER_ACCELERATION`: Controls how quickly the player reaches top speed.
    * `PLAYER_DAMPING`: Simulates friction, gradually slowing the player down when no movement keys are pressed.
    * `GRAVITY`: Constantly applied to the player's vertical velocity.
    * `PLAYER_JUMP_FORCE`: An initial upward velocity applied when the spacebar is pressed and `canJump` is true.
    * `PLAYER_EYES_OFFSET`: The camera's Y position is maintained at this offset above the player's conceptual "base" or "feet" position. The `controls.getObject()` (which is the camera) has its world position managed.

### 3. Collision Detection and Response

A multi-stage collision system is implemented in `handlePlayerMovement`:

* **Player Bounding Box (AABB)**: An Axis-Aligned Bounding Box (`THREE.Box3`) is defined for the player, centered around their conceptual body (`playerCamera.position.y - PLAYER_EYES_OFFSET + PLAYER_HEIGHT / 2`).
* **Object Bounding Boxes**: The AABB for each decorative object is calculated on-the-fly using `Box3.setFromObject()`.
* **Axis-Separated Collision for Objects**:
    1.  **Horizontal (XZ) Movement**:
        * Movement is attempted along the camera's local X-axis (strafe).
        * The player's AABB is updated.
        * Collisions with decorative objects are checked. If an intersection occurs, the penetration depth on the X-axis is calculated. The player is then pushed back out of the object by this depth plus a small `COLLISION_EPSILON`. `playerVelocity.x` is zeroed.
        * The same process is repeated for the camera's local Z-axis (forward/backward movement).
    2.  **Vertical (Y) Movement & Object Collision**:
        * Gravity and jump velocity are applied to the player's camera Y position.
        * The player's AABB is updated.
        * Collisions with decorative objects are checked:
            * **Landing on Top**: If the player is moving downwards (`playerVelocity.y <= 0`) and their AABB intersects an object's AABB, and their feet are aligned with the object's top surface, their camera Y position is set to `objectBox.max.y + PLAYER_EYES_OFFSET`. `playerVelocity.y` is zeroed, and `canJump` becomes true.
            * **Hitting Underside (Head)**: If jumping upwards and colliding with an object's bottom, the camera Y position is adjusted to be just below the object, and `playerVelocity.y` is zeroed.
* **Terrain Collision (Raycasting)**:
    * If the player is not considered to be standing on a decorative object (`!onObject`), a ray is cast downwards from the camera's current position.
    * The ray's length is `PLAYER_EYES_OFFSET + 0.2` (to reach just below the player's conceptual feet).
    * If an intersection with a terrain chunk occurs at `groundPoint.y`, the camera's Y position is set to `groundPoint.y + PLAYER_EYES_OFFSET`. `playerVelocity.y` is zeroed, and `canJump` becomes true.
    * A fallback check teleports the player up if they somehow fall significantly below the estimated terrain height.
* **Player Shadow**: A simple circular decal is projected onto the ground beneath the player using raycasting.

### 4. Collectible Orbs & Quotes

* **Spawning**: Glowing icosahedron orbs (`orbGeometry`, `orbMaterial`) are spawned probabilistically on terrain vertices within `createTerrainChunk`. They are added directly to the scene and tracked in the `collectibleOrbs` array.
* **Collection**: In `checkOrbCollisions`, the distance between the player's camera position and each orb is checked. If within `ORB_COLLECTION_RADIUS`, the `collectOrb` function is called.
* **Quote Display**:
    * `collectOrb` removes the orb from the scene and array.
    * A random quote is selected from the predefined `quotes` array.
    * The `quotePopupElement` (a `div`) is made visible, and its text content is updated with the selected quote and its source.
    * `controls.unlock()` is called to release the mouse pointer, allowing interaction with the popup.
    * Clicking the popup hides it and re-locks the controls (if the game wasn't paused by ESC).
* **Unloading**: Orbs are unloaded (removed from scene and array) in `updateTerrain` if their distance from the player exceeds `ORB_UNLOAD_DISTANCE`.

### 5. Anomalies (Floating Oddities)

* **Appearance**: Simple spherical meshes (`oddityGeometry`, `oddityMaterial`) that glow with an orange color.
* **Spawning**: `spawnOddity` creates an anomaly at a random position near the player, hovering above the terrain. Spawning occurs at random intervals (`ODDITY_SPAWN_INTERVAL_MIN` to `ODDITY_SPAWN_INTERVAL_MAX`).
* **Behavior**:
    * `updateOddities` manages their lifecycle.
    * They fade in over `ODDITY_FADE_TIME`, remain fully visible, and then fade out.
    * Their total lifespan is `ODDITY_DURATION`.
    * They exhibit a gentle bobbing motion.
    * Materials are cloned for each oddity to allow independent opacity animation and are disposed of upon removal.

### 6. Starry Background

* A `Points` object (`stars`) is created in `createStarfield`.
* Thousands of vertices are randomly distributed within a large sphere, far beyond the terrain and fog.
* `PointsMaterial` is used with `sizeAttenuation: false` so stars maintain a consistent size regardless of distance.
* `depthWrite: false` and `depthTest: true` are set on the material to ensure stars are occluded by terrain but don't interfere with its rendering.
* The starfield's position is updated in `handlePlayerMovement` to match the camera's position, creating the illusion of an infinitely distant, static starscape.

## Running the Project

1.  Ensure you have a modern web browser with WebGL support.
2.  Save the HTML file (containing all HTML, CSS, and JavaScript) to your local system.
3.  Open the HTML file directly in your web browser.
    * Due to browser security restrictions regarding `file:///` URLs and ES6 modules, you might need to serve the file through a local web server. Simple servers can be started with Python (`python -m http.server`) or Node.js (`npx http-server`).

## Controls

* **W, A, S, D**: Move forward, left, backward, and right respectively.
* **Mouse**: Look around.
* **Spacebar**: Jump (when on the ground or a solid object).
* **Click "Click to begin"**: Start the experience and lock mouse pointer.
* **ESC**: Release mouse pointer and pause the experience.
* **Click Quote Popup**: Close the popup and resume.

This project aims to be a simple yet evocative exploration of an endless, procedurally generated world, offering moments of quiet contemplation through discovered wisdom.
