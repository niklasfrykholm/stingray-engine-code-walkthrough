# Part 22: Particle system

* Simulate "particles"
    * Lots of small things
    * Controlled by a single set of rules
* Currently: fixed set of controllers
    * Vision of a more flexible future



## Particle effects in Stingray

* A particle effect consist of multiple "clouds"
* A cloud simulates a number of particles with a single ruleset
* A cloud is represented by a number of *float* and *Vector3* channels
    * Pre-sized to maximum number of particles in the cloud
    * Position, lifetime, velocity, mass, sweetness, cuteness, ...
    * The channels are not fixed and do not have any "meaning"
    * They get their meaning from the controllers, renderers, etc attached to them
* Controller types:
    * Initializers (sets the initial value of a channel when a particle is spawned)
    * Simulators (updates the particles with movement, spawns and kills particles, etc)
    * Visualizers (renders the particles on screen)



## Particle system resources

* Specifies for each cloud the channels and the controllers that operate on them
* Controller
    * Type (maps to a C function for performing the function)
    * Channels to operate on (sometimes implicit, i.e. `position_integrate`)
    * Parameters

```
life_time = 10000000000
clouds = [
	{
		capacity = 1000
		casts_shadows = false
		disable_culling = false
        max_radius = 0.0506776988506317

		float_channels = [
			"age"
			"life"
			"size"
			"collision_plane_offset"
		]

        vector3_channels = [
			"position"
			"velocity"
			"collision_plane_normal"
			"wind_velocity"
		]

		initializers = [
			{
                radius = [0 5]
				type = "position_sphere"
			}
			{
				channel = "size"
				range = [0.1 0.1]
				type = "random_float"
			}
			{
				channel = "collision_plane_offset"
				type = "float"
				value = -10000
			}
			{
				channel = "collision_plane_normal"
				type = "vector"
				value = [0 0 1]
			}
			{
				channel = "life"
				range = [10 10]
				type = "random_float"
			}
			{
                channel = "velocity"
				type = "zero"
			}
			{
				channel = "age"
				type = "zero"
			}
			{
				channel = "wind_velocity"
				type = "zero"
			}
		]

		simulators = [
			{
				type = "age_age"
			}

			{
				rate = 100
				scale = [ [0 1] [1 1] ]
				type = "rate_emitter"
			}
			{
				acceleration = [0 0 -9.82]
				type = "velocity_accelerate"
			}
			{
				output_channel = "wind_velocity"
				type = "query_vector_field"
				vector_field = "wind"
			}
			{
				noise_amplitude = 2
				type = "air_resistance"
				wind_coefficient = 5
				wind_velocity_channel = "wind_velocity"
			}
			{
				queries_per_frame = 1
				type = "query_collision"
			}
			{
				friction = 0.2
				restitution = 0.5
				rotation_mode = "none"
				type = "collision_resolution"
			}
			{
				type = "position_integrate"
			}
		]

		visualizers = [
			{
				channels = [
					{
						component = "position"
						name = "position"
						set = 0
						type = "float3"
					}
					{
						component = "color"
						name = "color"
						set = 0
						type = "ubyte4"
					}
					{
						component = "texcoord"
						name = "size"
						set = 7
						type = "float2"
					}
				]
				material = "Cloud 1"
				sort = false
				type = "billboard"
				vertex_writers = [
					{
						over_system_lifetime = false
						scale = [
							[0 0]
							[0.06942149 1.013554]
							[0.8712396 0.9927272]
							[1 0.01909091]
						]
						type = "size"
					}
					{
						gradient = [ [0.5 [255 255 255]] ]
						opacity = [ [0 1] [1 1] ]
						type = "color"
					}
					{
						dest = "position"
						source = "position"
						type = "copy_vector3"
					}
				]
			}
		]
	}
]
```

## Initializers

* Initializes particles to a default value
* `initializers.cpp`
    * Random float, position sphere, set float, set value, ...
* Have access to global context: particle effect position, system time, etc



## Simulators

* Updates particles
* `simulators.cpp`
    * Position integration, aging, wind, acceleration, collision, ...



## Visualizers

* Billboard
    * Draw billboard sprites (position, size, color) using a specific material
* Light
    * Create lights (color, intensity)
* Mesh
    * Draw full meshes (position, rotation, scale)



## Creation and destruction of particles

* Any simulator can emit or kill particles
* Emitting
    * Simulators have an *event stream* that they can post events to for later processing
    * Generates emit events (cloud index, position, velocity) to a stream when updated
    * When event stream is processed, new particles are emitted for each emit event
* Killing
    * Simulator swap-erases particle in particle buffer
    * Ok, because simulators run serial



## Particle simulation runs on the render thread

* Particle systems have a lot of state (position of every particle)
* Our thread design requires state changes to be reflected from main thread to render thread
    * Would be very expensive for particles
* To avoid this, we run the particle simulations on the *render* thread
* This creates *A LOT* of headache
    * If Lua wants to stop a particle effect it can't just do it
    * Needs to post an event to be consumed later by the render thread
    * `particle_world.cpp`
    * Similarly for *any* communication between particles and the rest of the world state
    * Resource management (loading and unloading) can be tricky
    * Seen *A LOT* of thread bugs in this system (hopefully all fixed now :)
* Maybe we should have used some other technique (double-buffering?) instead



## Particle collisions and wind effects

* Particles can interact with the world through collision and wind effects
    * Note: physics collision, not screen-space GPU collision
* But particles cannot directly query these systems (runs on separate thread)
* So how does it work?
    * Simulator posts an event to the event stream that it wants to query physics
    * When particle events are processed (still on render thread) this gets posted to an *external* event stream
    * External event stream is processed by the `World` update (on the main thread)
    * World performs the collision query and calls `collision_reply` on the `ParticleWorld`
    * Collision reply can't be processed immediately (because we are on the main thread)
    * Post an event with the collision reply to be processed by the render thread
    * When processed the result gets routed back to the simulator
    * Phew
* Particle collision
    * Particles raycast along projected path, stores hit as "collision plane"
    * Settings control how often raycasts are updated (how expensive collision is)
    * If "too few" raycasts -- particles will fall through the ground
* Particle wind effects
    * `query_vector_field` simulator creates event with all current particle positions
    * Processed by `World` in same way as collision queries
    * Stores wind strength for each particle in a channel (typically `wind`)
    * Another simulator (`air_resistance`) applies wind to update particle velocities



## Vector language controllers

* Controllers can also be written in the vector language (parallel bytecode mentioned earlier)
* This allows completely custom controllers with handwritten code



## Future visions for particle system

* Drawbacks of current system:
    * Limited by a fixed set of controllers
    * Does not use SIMD to full efficiency (up to each controller, inefficient storage)
    * Lot of things would run more efficiently on the GPU
    * You can do a lot cooler things with particles than just drawing billboards
* Vision
    * A generic "buffer processing system"
    * Takes blobs of data and operates on it using vector language (HLSL)
    * Can be used for other things to (e.g. audio mixing)
    * A single graph interface for simulation and feeding the GPU (connecting with shader graph)
    * Part of the evaluation runs on CPU (SIMD, MT), parts on the GPU
