# Part 14: Animation & AI



## Animation goals

* Animation curve (time and value)
    * `| t_1 v_1 | t_2 v_2 | ... | t_n v_n |`
    * Sampled (typically at 30 Hz) when exporting
    * Evaluate at time `t` -- linear interpolation
* Memory use
    * Animation data can be big: 100 bones * 30 fps * ...
* Performance -- efficient playback
    * Avoid binary searches for keys
    * Minimize pointer chasing and cache misses
    * Reduce memory use
* Streaming
    * Consume data linearly (no seeking)
    * Also good for performance



## Low level animation -- packing

* Vector3 and quaternion (position, rotation)
* Compressed values for less memory use
    * Quaternion: 10 bits per component + 2 for largest one -- (x, *y*, z, w)
    * Vector3: 16 bit fixed point in range [-10, 10]
    * Can also store uncompressed (for high quality or out of range)
* Hermite curve interpolation
    * Fewer keys needed than with linear interpolation
    * Any polynomial representation could be used
    * Hermite parameters are values -- can use same encoding
* Build hermite curve by adding keys until we are below error tolerance
* We cannot represent really fast rotating animations
    * A quaternion cannot represent multiple "laps" of orientation
    * We sample at 30 Hz on export
    * Increased sample rate or
        * Different representation (beware gimbal!)



## Low level animation -- streaming

* Most natural would be to store all keys for one curve after each other
    * `| t_1 v_1 | t_2 v_2 | ... | t_n v_n |`
* But then playback would require reading from multiple memory positions
    * One per bone
* Instead, multiplex the bone data and sort it by time
    * `| item_type (2) | bone id (10) | time code (20) | data (32/48)`
* Evaluator stores the four keys needed for Hermite evaluation for each bone
    * `| t_i v_i | t_i+1 v_i+1 | t_i+2 v_i+2 | t_i+3 v_i+3 |`
    * Advance time: read new data items and shift into evaluator
    * Evaluate value: Just need evaluator data
* Subtlety: Keys are sorted by when they are needed -- not the time code in the key
    * At time=0 we need the first four keys: t_0, t_1, t_2, t_3, so t_3 is sorted at 0
    * At time=t_1 we need t_1, t_2, t_3 and t_4, so t_4 is sorted at t_1
* Consequence: we cannot efficiently jump in animations or play them backwards
    * Double data for playing backwards
    * Sync frames?
    * Links between paired keys?
    * Two way streaming format?



## Low level animation -- multithreading

* Animation evaluation is multithreaded by job system
* `AnimationPlayer` keeps track of all running animations
    * Creates jobs for advancing the time for all of them
    * Run through job manager
    * `animation_player.cpp`
* Header structure is a bit weird
    * `interleaved_animation.h`, `interleaved_animation_job.h`
    * Because of PS3 SPUs
    * Could be refactored



## Animation merging

* Optimization used for "army animations"
* If the same animation is played multiple times with similar speeds and start times
    * Merged to a single animation playback
* Split into separate evaluations again if speed or time changes



## Animation curves

* Allows animations to contain other data than bone transforms
* Arbitrary float curves
    * Could be light values, etc
* Why do we treat position/quaternion data differently?
    * More efficient packing



## Animation compiling

* Animations are exported in `.fbx` files
* A `.skeleton` file lists the bones and compression settings for each bone
* In the animation file, bones are identified by index in skeleton
* Typically animations contain local space transforms
    * Better for compression
    * Better for blending
    * But this means error is multiplied along bone chains
    * Foot sliding
* Solutions
    * Constraints
    * Skeleton reparenting



## Animation blender

* `animation_blender.h`
* Responsible for blending individual animations into a composed animation
* Fixed blend graph
* Layers
    * Separate animations playing in each layer
    * Higher layers will play "over" lower ones with a per-bone opacity
* Crossfading
    * Transitioning between animations in a layer crossfades them
* Mix
    * Layers can also play a mix of animations (for basic movement)
    * Mix strength is controlled by expression language
* Blend tree animation
* Constraints
    * Callback function applied to pose
    * Applied to each individual animation before blending with others
    * Correct but expensive
    * Might be useful to have a "simple constraint" concept
* Quite a lot of math to make sure all blends are correct
    * Custom blend tree instead?
    * Would probably need features similar to this anyway
    * And harder to control blending
    * Probably better to keep current structure but add blend tree



## Constraints

* Callback function -- take a pose, return a modified pose
* Has access to scene graph so can apply constraint in world space
    * But needs to convert to world space and then back to local space
* Prefer simple constraints rather than complicated solvers
    * Turn first bone 30 % towards target, next bone 50 %, etc
    * Because complicated solvers can "pop"
* Example: aim constraint



## Animation lodding

* Bones are sorted in the order of importance
* Animation lodding is implemented by only evaluating the first *n* bones
* The rest of the bones will be "frozen" at their current pose



## State machine

* `animation_state_machine.h`
* Compiles a "state machine" animation graph -- created in the editor
* Controls the blender
* A *state* for each layer, *events* cause *transitions* between states
    * *Events* are strings
    * *States* have playback speed, etc -- controlled by variables
    * *Transitions* have crossfade time, beat synchronization, etc



## Triggers

* Animation tracks can also contain "triggers" -- events that occur at specific timestamps
* A trigger is just an `IdString32`
* Triggers are fed up from the player to the blender to the state machine
* The state machine has a queue of trigger that is polled by the `World`
* The `World` will trigger a flow event in the `Unit`'s flow corresponding to the trigger
    * No direct connection between low level systems
    * Polling is preferred over callbacks



## AI

* AI support in-engine is limited, because AI is very game dependent
* `navigation_mesh.h`
    * Superceeded by navigation plugin
* `broadphase.h`
    * System for finding "nearby" objects
    * Based on grid hashing
* Really -- all of this belongs in plugins
* There are a lot of interesting things to do in the AI/animation intersection
