# Part 21: Lua

* Why a scripting language?
    * Doesn't require source access (fixed now with plugin)
    * Doesn't have to be compiled for every platform
    * Treat as part of project
    * Crash safe
    * Memory leak safe (with GC)
    * Abstraction sandboxing
    * Hot reloading
    * More expressive than C (maybe?)
    * Better for vague tasks than C (maybe?)
    * More "hackable" than C
        `stingray.World.spawn_unit = function (...) end`
* Why Lua?
    * Different enough from C to make it worth adding a new language
    * Popular language "style" (Python/Ruby/JavaScript)
    * Small, sane, minimalistic language
    * Designed for embedding, designed for extension
    * Hackable
    * LuaJIT is amazing (but lawyer-limited unfortunately)
* Drawbacks of Lua
    * No native multithreading
    * No static type checks (refactoring is hard)
    * Garbage collection is not ideal in game environment



## Lua type mapping

* How to represent Stingray types in Lua?
* Lua types: bool, number (double), string, table (array/hash), userdata
* Userdata:
    * `light userdata` is just a wrapped C pointer (`void *`)
    * Full userdata is a memory blob with garbage collection and metatable
    * Metatables represent "types" in Lua
    * Metatable is used for lookup of object methods (and other things)
    * `obj:do_something()`
* Full userdata is more convenient, but:
    * Increases garbage collection pressure
    * More expensive (conversion requires table lookup)
* We use light userdata as much as possible



## Making light userdata work

* Ownership
    * No automatic garbage collection
    * Explicit ownership on C side
    * Application owns worlds, world owns units, units owns meshes, ...
    * Lua explicitly tells owner to destroy object
    * Objects that die "automatically" (sounds, particle effects)
      are identified by IDs (weak references -- just numbers)
* Method calls `unit:do_something()`
    * Not supported (no metatable)
    * Must explicitly specify class of object when calling
    * `Unit.do_something(unit)`
    * You might want to do this anyway?
        * Clearer (no static typing)
        * Cache method lookup `local unit_do_something = Unit.do_something`
    * Wrap in table with metatable if you need methods
* Type checks
    * How can we check if user passed an object of the right type?
    * We store a type tag as first member of the object

```
class Level
{
    static const int LEVEL_MARKER = 0xa0db49ba;
    int _marker;

public:
    Level() : _marker(LEVEL_MARKER) {}
    ~Level() {_marker = 0;}

    ...
}
```

    * To check if a `void *p` from Lua is a Level:
    * `*(int *)p == LEVEL_MARKER`
* Checking for stale objects
    * Using a light userdata after it has been deleted
    * Most likely type marker will no longer match -- type error
    * Other possibilities: access violation, random match
* `lua_stack.h`
* `lua_stack.inl`



## Weak references

* For units and entities we want to be able to ask "still alive?"
* Weak reference
    * Lower two bits of light user data mark it as a reference
    * 00 - regular pointer `void *`
    * 01 - unit reference
    * 10 - entity reference
    * 11 - unused
* Rest of bits is used as an ID for the weak reference system
* `lua_stack.inl`



## Linear algebra

* How do we represent `Vector3` etc?
* Directly "on the stack" (as `double`)
    * Not possible in Lua -- only for native objects
    * Extending Lua with support for this is a big modification
* Light user data
    * Without metatable we can't do `local d = 2*a + b`
    * Must be: `local c = Vector3.add(Vector3.mul(2, a), b)`
    * No GC: Who destroys temporary objects? (`2*a`)
* Full userdata
    * Every temporary object will be heap allocated and generate garbage
    * Slow
* Table
    * Similar to full userdata (tables are heap allocated)



## Our approach

* Performance is most important -- have to use `light userdata`
* Destruction?
    * Objects are destroyed *automatically* at end of frame
    * No user destruction required
    * This means you can't save a `Vector3` and use it in the next frame
    * To save data you must wrap it in a heap allocated `Vector3Box`
    * A bit error prone...
* Implementation
    * Objects are allocated in reusable block buffers | STALE_TYPE x y z | STALE_TYPE x y z | ... |
    * Light user data points into these buffers
    * Each frame we clear type markers of previous frame's objects
    * Using an old `Vector3` will give a type error
    * (Could also cycle type markers)
* Operator overload
    * `local c = 2 * a + b` is kind of nice
    * Although we can't use individual metatables for lightuserdata...
    * ...we *can* use a single metatable for *all* lightuserdata
    * Check type: `Vector3`, `Vector4`, ... and call appropriate function
    * Adds extra type checks and branches, but not super expensive
* `lua_environment.inl`
* `lua_environment.cpp`
* If you use a lot of linear algebra, the buffers will grow very big
    * Managed by saving and restoring buffer size in a scope
        * Save buffer size
        * Perform a bunch of computations
        * Box every result you want to keep
        * Restore buffer size



## Lua bindings

* How to call C functions from Lua
* Lua uses an "abstract stack" for parameters and return values
    * `int function(lua_State *L)`
    * `lua_tonumber(L, i)` `lua_tostring(L, i)`
    * `lua_pushnumber(L, x)`
* Binding options:
    * Manual bindings (write functions)
        * `int world_spawn_unit(lua_State *L) { }`
    * Code generation (from `.h` file)
    * Template magic
    * FFI
* We use manual bindings
    * Detailed control over API
    * Error handling
    * Not much work
    * Since we now have a C API -- could use more automatic method
* `if_unit.cpp`
* Documentation is generated by parsing comments with custom markup (`adoc`)
* `lua_environment.h`, `lua_environment.cpp`
* `lua_stack.h`, `lua_stack.inl`



## Lua hot reloading

* Lua is hot reloaded through `refresh()` call
* Hot reload of Lua is handled by just running the code again
* `function Shape:draw(size) ... end` <=> `Shape.draw = function (self, size) ... end`
* Care must be taken with class definitions
    * `Shape = {}`
    * Would create a new shape class on reload
    * Old shape objects won't be updated with new functions
    * `Shape = Shape or {}`
* We have a standard `class()` implementation to take care of this
    * `Shape = class(Shape)`



## Lua garbage collection

* Garbage collection stalls is terrible for real-time
* Lua and LuaJIT supports incremental garbage collection
    * Garbage for *x* ms every frame
* Problem -- how much should *x* be?
    * Incremental mark and sweep
    * We don't know how much work we have left to do until it is done
    * Do too much -- wasting time
    * Do too little -- not keep up with generated garbage
* Control problem
    * Scale garbage collection time based on estimate of garbage
    * Exposed in Lua
    * `lua_garbage.cpp`
* Multithreading
    * Garbage collection can run in parallel with any code that doesn't touch Lua
    * Currently it runs serial



## Lua allocator

* LuaJIT on 64-bit can only use the low 2 GB of memory
* How can we ensure that without allocating the whole range?
    * Also the whole range may not be available
* `low_address_page_allocator.cpp`
    * Try to *reserve* but not *allocate* the whole low 2 GB address range
    * Reserve entire range -- if it fails split and reserve halves
    * Store the free memory in a "buddy allocator" structure
    * Use a heap with this allocator as backing allocator



## Lua debugger

* Debugging of Lua is exposed through a web socket interface
* Takes debugging commands `break`, `step`, `eval`, etc and returns results
