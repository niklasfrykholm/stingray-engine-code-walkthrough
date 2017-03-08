# Part 3: Collection classes



## Philosophy

* No STL-classes
  * We want full control
  * Needs to work with our Allocator system
* STL-algorithms are ok though
  * std::find, std::sort, ...
* Minimalistic philosophy
  * We don't need a ton of different collection classes
  * Focus on the few that we need
* Don't abstract too much
  * It is ok to now that a string is `Array<char>`



## Array, Vector

* `array.h`
* Dynamically resizing array of objects
  * `T *`, `_size`, `_capacity`, `_allocator`
* Exponential growth -- linear amortized time
* `Array<T>` uses `memcpy()`, `Vector<T>` proper constructors
  * `Array<T>` is faster for POD
  * `Vector<T>` is required when constructors need to be called
* Important special case: `T(Allocator &)`
  * E.g. `Vector< Vector<int> >`
  * How does `Vector<T>` know that `T` needs to be called with an allocator?
  * Macro/template magic
  * Put the macro `ALLOCATOR_AWARE` in definition of `T`
  * `Vector` will check for it using template magic and call the right constructors
  * If you forget it, you get an error



## Mixing allocators

```
void foo(Vector< Vector<int> > &vv)
{
  TempAllocator ta;
  Vector<int> v(ta);
  v.push_back(3);
  vv.push_back(v);
}
```

* This is safe
* `push_back()` will create a `Vector<int>` using vv's allocator
* Then call `Vector<int>::operator=` to assign v
* This copies v's data into vv
* In short: Mixing allocators is safe -- will work as expected
* But in general, avoid vectors of vectors, etc
  * More copying than you expect behind the scene
  * Can be expensive



## Strings

* No string class -- string classes are stupid!
* A string is either static (`const char *`) or dynamic (`Array<char>`)
* Using the same class for these two leads to confusion and bad performance
* Strings are always UTF-8 -- it's the only sane thing
* `string.h`



## HashMap, HashSet

* `hash_map.h`
* O(1) lookup
* Open hashing, but linked list is stored in-place in spill area of buffer

  [ hash item buckets | collision spill area]

* Items are `[key | value | marker]`
* `marker` is used for links and to mark empty slots
* `hash_function.h` Murmur hash is used for hashing



## IdString64 / IdString32

* Most of the time we only want to compare/lookup strings by identity
* E.g., resource names: `trees/larch/03`
* Use a hash instead of the full string `murmur_hash(s)`
    * Faster comparison
    * Less memory use
    * Fixed size (can be stored in array easily)
* `id_string.h` -- Classes that wrap a hashed string



## Printing IdStrings

* Sometimes we want to reverse idstrings
    * Seeing a value in the debugger. What does a hash mean: `3c85f5eef13ac468`
    * Error messages: `Resource not found "trees/larch/03"`
* But we only have a hash and no way to reverse it
* We don't really want to use the memory to store the string data
* Solution: `strings.txt`
    * Generated during *compile time* to contain all unique hashed strings
    * `id_string.cpp`
    * Tools can lookup in this table
    * Printed IdStrings `#ID[3c85f5eef13ac468]` are automatically converted
    * Note: Only works for strings encountered during compile time
* Maybe this optimization no longer makes sense?
    * Modern consoles have a lot more memory
    * We could probably keep the string-table loaded "in-memory"
    * We would still use hashes for speed and struct layout



## Option<T> and Error<T>

* "Algebraic types" for cleaner error handling
* `Option = T | Nil`, `Error = T | char *`
* `option.h`, `error.h`

```
Error<int64_t> size_res = file_system.size("settings.ini");
if (size_res.is_error()) {
    logging::error(SETTINGS, "Could not read size of `settings.ini`");
    return;
}
int64_t size = size_res.value();
```

* Protection against "forgetting" to check error code
* `.value()` asserts on error.
