# Part 18: Worlds, levels and units



## Worlds

* `world.h`
* A *world* is the context in which game objects live
    * Units
    * Animations
    * Physics
    * Particles
    * Entities
    * Vector fields
    * Decals
    * ...
* Multiple worlds can co-exist
    * Game world, inventory world, menu world, minimap world, loading screen world, ...
    * Worlds are updated and rendered from Lua
    * Lua is driving/has control over game loop



## Unit

* `unit.h`
* A game object
    * Meshes
    * Terrains
    * Cameras
    * Physics actors
    * Lights
    * Animation state machine
    * Unit data
    * ...
* Set up in Unit Editor (from `.fbx` file)
* Will be replaced by entities
    * More lightweight by default
    * Looser coupling, extensible
    * Inheritance, overrides



## SceneGraph

* `scene_graph.h`
* Represents the hierarchy of nodes *within* a unit
    * Note: not between units, that is done with unit linking
* Meshes, lights, etc are attached to a particular node in the scene graph
    * Note: They are not themselves nodes, the nodes are only transforms
* Nodes have local transform, world transform, parent
    * local → world update is lazily applied (once per frame)
    * So even if local position is changed many times, only one transform cost
    * On the other hand it means world pos is "out of date" until update
    * Some scenarios require explicitly transforming the scene graph
    * I'm thinking now this isn't such a good choice
* SceneGraph uses data-oriented design
    * Nodes in a flat array `[n_0 | n_1 | n_2 | ...]`
    *                        `l_0 w_0 p_0 | l_1 w_1 p_1 | ... `
    * Dirty flag for nodes with dirty local transforms
    * Ordered in update order (so we can just walk the array)
    * Export dirty flags for modified world transforms (for renderer)



## Handling moving objects

* Work should be *O(moving objects)* not *O(total objects)*
* Key -- keep list of moving units `Array<Unit *>`
    * Unit knows its own position in the array `Unit::_moving_keyframed_index`
    * O(1) add and remove (through swap)
* Script changes position, unit is animated
    * Added to `_moving_keyframed` array
* Object awakes in physics
    * Added to `_moving_dynamic` array
* We check some units in these arrays every frame
    * If the unit has stopped moving -- remove it from the arrays
* Only units in these arrays have scene graphs updated
    * In parallel
* Linked units
    * Compute link-depth of units (number of parents)
    * Sort unit by link_depth
    * Transform all moving units in parallel
    * Transform all level-1 linked units in parallel
    * Transform all level-2 linked units in parallel
    * Transform the rest of the linked units in-order



## World update order

* `Application::update()`
    * Tick application timer
    * Lua `update(dt, t)`
    * Lua `World.update(w, dt)`
    * `World::update(dt)`
        * Tick timer
        * Update animation state machines
        * Update playing animations
        * Update animation blends
        * Compute unit link depth
        * Update scenegraphs of moving keyframed units
        * Update scenegraphs of moving dynamic units
        * Update scenegraphs of linked units
        * Mirror colliding kinematics into physics
        * Apply wind and other continuous forces
        * Simulate physics
        * Separate overlapping movers
        * Get keyframed, dynamic and linked unit positions from physics
        * Reflect moved units to render thread
        * Check for sleeping units
        * Process events from animation, physics



## Unit Data

* Store small pieces of data in a unit
* `dynamic_data.h`
    * JSON-like structure (bools, strings, numbers, maps)
    * Map keys are either integers or hashed strings
    * Intended for small amounts of data (loop over keys)
    * Stored in a single memory block
* Used for storing arbitrary gameplay data in a unit `health = 100, score = 50`
    * Can be set in level editor and the unit editor
    * At runtime, we can also store arbitrary Lua objects
* Kind-of abused for storing more and more data in the units
* World and Level also have script-data blocks



## Unit references

* Most things in the engine use hard references
    * Explicit create, destruction
    * Clear ownership
    * Application owns Worlds that own Units that own Actors
* Exceptions
    * Playing sounds, animations, etc -- identified by id (weak reference)
    * `is_sound_playing(id)`
    * Unit reference -- weak reference to units
* Unit reference
    * 32 bits (to fit into Lua light userdata on 32-bit system)
    * `[index (22) | generation (8) | marker (2)]`
    * `index` -- index of `Unit *` lookup slot in array
    * `generation` -- increased every time a new `Unit *` is put in a slot
    * Index: `[ Unit*, gen | Unit*, gen | ... ]`
    * `marker` -- distinguishes unit reference from other light userdata
    * Max 2^22 units
    * Generation will eventually wrap-around
    * Should change this to 64-bit structure once 32-bit platforms are dropped



## Levels

* Levels are collection of units that can be spawned into worlds
    * Also: lights, flow, particles, navmeshes, stories, entities, ...
    * Edited in editor
    * Multiple levels can be spawned and unspawned into the same world
* Special level objects
    * Prototypes: Whiteboxes -- gets compiled into a single static unit
    * Splines: Draw splines and query them (weird to have as a general object)
    * Volumes: Draw volumes (convex 2D regions + height) that can be queried
    * SoundScape: Place sounds that play when player gets close `sound_scape.h`
    * Scatter system: Below
* Issue: Spawning a level can cause stall



## Scatter system

* `scatter_system.h`
* Used to populate a world with lots of small units (vegetation, trash, ...)
* Stores just `(pos, rot)` for each unit
* As user gets close -- will spawn a real unit and blend it in
* Sorted in buckets based on distance to player
    * Units this far away `[ 2*x  | 4*x  | 8*x  | ...]`
    * Checkd per frame:   `[ d/x  | d/2x | d/4x | ...]`
    * Cost of checking proximity proportional to distance moved by player
