# Part 2: Memory



## Allocator

* We use an explicit allocator model
* No: `new`, `delete`, `malloc`
* Instead you always allocate through a specific allocator object
* Reason: We want to track all memory usage
  * We assert on memory leaks when exiting the application
* `allocator.h`
* `override_new.cpp`
  * Libs do not behave well



## Using explicit allocators

```
  void *p = a.allocate(sizeof(T));
  T *obj = new (p) T(arg1, arg2);
```

Or use macro:

```
    T *obj = MAKE_NEW(a, T, arg1, arg2);
```


## Specific allocators

* Allocators used for specific purposes
* `page_allocator.cpp` - allocates virtual memory, OS interface
* `heap_allocator.h` - dlmalloc based heap
* `slot_allocator.cpp` - small memory allocations
* `generic_allocator.cpp` - most used allocator



## Allocator boot-strapping

* All memory is allocated through allocators
* How do we allocate the first allocator?
* Static heap
  * `memory_globals.cpp`



## Temporary memory allocations

* Used when you need a little bit of memory in a function
* `temp_allocator.h`
* Warning: Cannot be passed between threads/subtle bugs



## Tracking memory allocations

* TraceAllocator is used to provide a scope for allocations
* Counts allocated memory
* ~TraceAllocator() asserts that all memory is released
* `trace_allocator.h`
* We can print a tree of allocation scopes and memory usage



## Discovering memory leaks

* In `trace_allocator.cpp`: TRACE_ALLOCATOR_ENABLE_TRACING
* Will save a callstack for every allocation
* If there are leaks at the end, dump the callstacks
* You can find out where the leak was



## Using memory after free, using uninitialized memory

* We explicitly clear memory (with 0x66) at dealloc
* There is a flag you can set to fill memory randomly on allocation
  * `allocator.h`


## Discovering memory overwrites

* Switch out a suspicious allocator for a debug one
* `canary_allocator.cpp`
* `end_of_page_allocator.cpp`



## Allocator issues

* Too much locking
* No easy way of investigating fragmentation
* Memory usage reports are not super helpful
* Jakob is working on this
