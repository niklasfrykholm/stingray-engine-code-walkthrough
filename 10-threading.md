# Part 10: Threading and timing



## Threading

* Goal of threading
    * Maximize throughput
    * Minimize latency
* Cores as busy as possible
    * Doing actual work



## Thread model

* Two pipelined + worker threads:

```
---- update #23 -----\ ---- update #24 -------
---- render #22 ----- \---- render #23 -----

--- worker -------------------------------
--- worker -------------------------------
--- worker -------------------------------
--- worker -------------------------------

--- async task ---------------------------
--- async task ----------------------------
```

* Both main and render use worker threads to process jobs
* Latency 2 x frame time (+ display lag)
* Both rendering and the main update have single threaded parts
    * Pipelining allows these "holes" to be filled



## Main & render thread synchronization

* Render thread has copy of the mutable state (worlds, objects, etc)
    * Only the stuff needed by the renderer
* When main object is created / modified / destroyed mirror commands are posted
    * CREATE(id, type, info)
    * MODIFY(id, changes)
    * DESTROY(id)
* Renderer uses this to update its state
* Since states are separate, deleting the main state is ok
* Immutable state (geometry buffers, etc) is not copied
    * Care is taken to synchronize creation/deletion



## Worker threads

* We spawn as many of these as we need to get one thread/core
* Main and render threads post jobs to a queue
* Worker threads pull jobs from queue
* Main and render can wait for a particular job to finish
* While they are waiting, they work on jobs from the queue
* Typically we branch and join quickly
* `animation_player.cpp`
* AnimationPlayer::update()
    * Gather all animations that need to updated
    * Post N animation jobs
    * Wait for jobs to finish
* Waiting later would let us utilize the threads better
    * But it is harder to reason about



## ThreadManager

* Handles creation/destruction of threads (abstracts OS)
* Knows about all threads we have created
* `thread_manager.h`



## JobManager

* `job_manager.h`
* Manages the pool of "worker threads"
* Post jobs together with a "slice setting" (specifies splitting)
* A job has constants, input streams, output streams and a callback function (kernel)
* Streams will be split into suitable sizes to run on each thread
* In the kernel you can query to get back the streams
* Example: `animation_blender.cpp`
* This system suffers from over-design
    * Maybe better: A job is just a 1024 byte opaque blob
    * It is up to the kernel code to cast it to correct type and offset pointers
* See `job_manager.cpp` and `thread_pool.cpp` for implementation of job algorithm



## Critical sections

* Used in most places to prevent race conditions
* Translates into a light-weight mutex
* `CriticalSection` represents the critical section
* `CriticalSectionHolder` holds it for the scope of the object
* `critical_section.h`
* Sometimes it is hard to keep track of what a critical section locks
    * `LockedVars` pattern
    * `resource_manager.h`
* Event class wraps OS events: `event.h`



## Thread safety

* The engine is not generally thread-safe
    * Making everything thread-safe is expensive
* Only thread safe in systems that "need it"
    * Memory allocators (except TempAllocator)
    * ResourceManager::get()
    * ...
    * Poorly documented and not well thought-through
* This could need a re-think



## Threading issues: Why do we have two "main" threads?

* Makes it easier to reason about data flows, life times, etc
* One thread -- easiest, two threads -- easier
* Might make sense to go for something more loosely structured
    * But harder to reason about
    * Make sure increased complexity doesn't eat up the gains
* Fibers?



## Threading issues: How should jobs be prioritized?

* Should we pick main thread or render thread jobs?
* Depends on what the critical path is
    * Which we don't know
    * Could be optimized for a specific game
* Dynamically guess the critical path and prioritize?
    * Complexity



## Threading issues: Multithreading gameplay code

* Single-threaded gameplay code is increasingly the bottleneck
* Lua has no multithread support
* We can multithread Lua by running multiple Lua states
    * Send tasks to Lua states -- get back results
    * Hard to break down problems into isolated tasks
    * Might as well write solution in C
* Gameplay code tend to be sprawling
    * That's the split between engine/gameplay
    * Hard to organize into efficient multithreading
* Multithreading model must be easy enough to use by gameplay programmers
    * Actor model?
* Another big problem: Systems are not thread-safe
    * Needs to be possible to call systems efficiently and safely
* First step: Think about thread safety model



## Timing

* Real-world wall time through `timer.h`
* Translated into in-game time by TimeStepPolicy
    * Slow-down/speed-up time for slow-motion effects, etc
    * Fixed or variable time step
    * Throttling (save battery)
    * Smoothing (improves some algorithms)
    * Minimum/maximum time step (prevent bugs)
    * Debt payback (to keep in sync with world clock)
* By default we use variable time stepping for world update
* But physics steps at a fixed rate
    * Take as many steps as needed
