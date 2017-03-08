# Part 16: Physics



## Physics overview

* Uses PhysX physics simulator (from Nvidia)
    * Rigid body simulation
    * Joints
    * Raycasts
    * Shape queries
    * Triggers
    * Character controllers, movers
    * Vehicles
    * Cloth
* Split between `physics_world.h`, `physics_world_internal.cpp`
    * Idea of supporting multiple backends
    * Never implemented -- abstraction is now leaky
    * Should really be a plugin
* Will not go too much into "general physics engine" issues in this talk
    * Shape types and performance
    * Stacking stability
    * Etc
    * Just a few common issues



## Setting up PhysX objects

* In Unit Editor:
    * Select meshes to have physics
    * Shape type (sphere, capsule, box, mesh, convex) fit to mesh shape
    * Simulation type (kinematic, dynamic, static)
    * Properties (referencing `global.physics_properties`)
    * `.physics` file
* In PhysX plugin for Maya:
    * Allows setup of joints/constraints
    * Cannot connect to properties in `global.physics_properties`
    * Map in Unit Editor
    * `.physx` file
* Manual from script:
    `PhysicsWorld.spawn_sphere(pw, Vector3(0,0,0), 1.0)`
* Cloth editor:
    * `.apx`



## Physics properties

* Specified in `global.physics_properties`:

```
materials = {
	default = {density = 1000, dynamic_friction = 0.1, static_friction = 0.1, restitution = 0.1, restitution_combine_mode = "max"}
}

collision_types = [
	"default"
	"character"
]

collision_filters = {
	default = {is = ["default"] collides_with_all_except = ["character"]}
	character = {is = ["character"]}
	character_trigger = {collides_with = ["character"]}
	non_collider = {is = [] collides_with = []}
	walkthrough = {}
}

shapes = {
	default = {}
	trigger = {trigger = true}
	sweeper = {sweep = true}
	character = {collision_filter = "character"}
	character_trigger = {trigger = true collision_filter = "character_trigger"}
	ragdoll = {}
	walkthrough = {disable_collision = true disable_raycasting = true collision_filter = "walkthrough"}
}

actors = {
	static = {dynamic = false}
	dynamic = {dynamic = true linear_damping = 0.1 angular_damping = 0.1}
	keyframed = {dynamic = true  kinematic = true linear_damping = 0.1 angular_damping = 0.1}
}
```

* Why do we have a centralized place to specify properties
    * Easy to tune behavior
    * Only limited number of collision types supported
* Disadvantages
    * Not self-contained
    * Copying units between projects breaks properties
* For collision, globally agreed on names need to be somewhere
    * `character.collision_type`



## Collision filters

```
collision_types = [
	"default"
	"character"
]

collision_filters = {
	default = {is = ["default"] collides_with_all_except = ["character"]}
	character = {is = ["character"]}
	character_trigger = {collides_with = ["character"]}
	non_collider = {is = [] collides_with = []}
	walkthrough = {}
}
```

* Limited to 64 different collision types (used as binary masks)
* Used for object-to-object collision, but also raycasts and shape queries



## PhysicsWorld

* One per World `physics_world.h`
    * Create objects
    * Raycasts, sweeps, overlaps
    * Update



## Connecting physics objects to world objects

* `actor_connector.h`
    * Corresponds to one physics actor
* `PhysicsWorld` keeps track of awake keyframed actors
    * Before `update()` --> read positions from engine
* `World` keeps track of moving simulated actors
    * After updating physics -- reads back physics positions into units
    * `world.cpp` `update_physics()`
    * Note how we make sure not to update *all* objects
* Some additional complexity to handle complicated situations
    * Keyframed linked to dynamic jointed to keyframed, etc
    * At some point a frame delay will be introduced (single update)
    * But we want to behave as nice as possible



## Physics time stepping

* Fixed time step 60 Hz -- no interpolation
* Take as many step as needed to keep up
* Causes jitter -- sometimes one step, sometimes two
* Alternative -- interpolate graphics position between timesteps
    * But this means physics no longer matches visual
    * Raycasts will miss!
        * Two cars running next to each other
        * Bullet time
    * Character will be able to penetrate graphics
    * Hmm...
* What if engine timestep is really long?
    * Have a limit on the number of physics steps
    * Otherwise: risk spiral of death
    * This means: In low-framerate, physics will run in slow-motion compared to rest of game
* What if engine timestep is really short?
    * Suppose engine uses bullet time/slow motion
    * If we stepped at 60 Hz, we would see object movement "snapping"
    * In this case: take a smaller timestep in physics
    * (Might cause instability)
* Not necessarily the right choice for every game
    * Configurable in `settings.ini`
    * Except interpolation: not supported yet



## Mover (character controller)

* `mover.h` `sweep_test.h`
* Used for controlling how a character moves in the physics world
* Extracts a cache of shapes around the character and does capsule sweeps on those shapes
* Typically sweeps up, forward, then down -- to clear steps and small bumps
* Lots of fine tuning to implement sliding, non-shaky collisions, etc



## Raycasts

* `raycast.h`
* API for synchronous and asynchronous raycasts
    * But asynchronous raycast isn't actually implemented (still runs synchronously)
    * Doh!
* Should really look into it
    * Raycasts can run on background thread in PhysX
    * Would offload main thread



## Cloth & vehicles

* Interface to PhysX cloth (apex) and vehicle systems
* Cloth implementation is complicated (needs render integration)



## Physics event processing

* `PhysicsWorld` registers event callbacks with PhysX
* When events occur they are put on a queue
    * `types.h`
* `World` polls the physics event queue and posts suitable flow events
    * `world.cpp` -- process_physics_events()



## Vector fields

* Push physics, particles, etc around with winds, explosion, etc
* We represent these things as *vector fields*
* Typically only one field: *wind velocity*
    * Can imagine others: gravity, force field, ...
* On a field we play "effects"
    * Base wind, helicopter airflow, explosion, ...
    * ASDR envelope + expression
    * `10 * normalize(pos - origin) / length(pos - origin) * 2`
* Expressions are implemented in vector language
    * Each frame we merge the expressions of all current effects into a single bytecode
    * And fix the variables that are constant for the frame into constants -- constant folding
    * Find position of all physics objects affected by wind
    * Evaluate wind strength at all those positions in parallel
    * Apply wind forces to objects
    * `physics_world_internal.cpp` apply_wind_to_dynamic_connectors()
    * Same for cloth and particles



## Some common physics problems

* Performance
    * Too many objects awake? Make sure they fall to sleep
        * Too unstable
        * Falling out of the world
    * Naive shapes? Use specific physics geometry
    * Lots of moving objects
        * Limbs should be query-only
        * Aggregates / multi-box pruning
        * Your own detail raycasting
    * Be raycast/overlap/sweep frugal
        * Consider caching
* Bullet-through-paper
    * Use sweep-shapes where necessary
* Instability / shaking constraints
    * Masses should be similar
    * Add extra constraints for long chains
    * More damping
* Ragdolls spawning inside geometry
    * Don't do that!
    * Easier said than done...
* Exploding things
    * Don't spawn things in other things!
    * Lower depenetration velocity
* <http://www.codercorner.com/blog>
