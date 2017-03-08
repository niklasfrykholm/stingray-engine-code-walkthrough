# Part 20: Plugins



## Goals

* Allow Stingray to be extended
	* Without access to source code
	* In a decoupled way
* Stable across versions of Stingray
	* Forward/backward compatible
* Move game-specific concepts out of core engine
	* Splines
	* AI
	* ...
* "Asset store"



## Plugin interface

* Based on C rather than C++
	* C uses a stable ABI
	* Compatible across compilers, versions, settings, etc
	* C++ has name mangling (different across compilers)
* Based on single function `void *get_plugin_api(unsigned api)`
	* Returns a `struct` with C pointers based on the API ID
	* Nothing else is shared between engine and plugin
	* Minimal contact surface -- decoupled



## Plugin API

* Plugins must implement the `PLUGIN_API_ID`
```
struct PluginApi
{
	const char *(*get_name)();

	void (*loaded)(GetApiFunction get_engine_api);
	void (*unloaded)();

	...

	/* Reserved for expansion of the API. */
	void *reserved[32];
};
```
* Defines callback functions that the engine will call at specific points (loading plugin, starting game, ...)
	* Use `NULL` for things that the plugin doesn't care about
* Calling back from plugin to the engine works the same way
	* Engine defines `void *get_engine_api(unsigned api)`
	* Passed to plugin when it is loaded (through `loaded` callback)
	* Multiple APIs: LUA_API, DATA_COMPILER_API, ALLOCATOR_API, ...
* `plugin_api.h`
* Implementing a simple plugin:
	* `plugin.c`



## API versioning

* Plugin API is intended to be backwards AND forwards compatible
	* A plugin DLL can be used with an _newer_ version of the engine
	* A plugin DLL can be used with an _older_ version of the engine
* Versioning
	* Plugin API is locked -- we don't change or remove functions in `struct`
	* Only add new functions at the end (in reserved area)
	* If we need to make big changes, create a new API (while keeping the old one)
		* `LUA_API`, `LUA_API_V2`
* Old dll, new engine
	* Old functions will still be at the same location in struct, with same parameters
	* Old dll can continue to use them
	* New functions (in `reserved` won't disturb old DLL)
* New dll, old engine
	* New functions in API will be in reserved area of old engine
	* Initialized to `NULL`
	* New DLL can check which functions are available by checking function pointers against NULL



## C API

* One of the APIs we support is the C API (`C_API_ID`)
	* Exposes the same functions as our Lua API
	* Allows you to write gameplay code in C in a DLL
* Just like Lua API implements "error handling"
	* If you call with bad arguments, exit instead of crash
	* Implemented through `longjmp`



## Static and dynamic plugins

* Plugins can be linked statically or dynamically
* For static linking we just put the `get_plugin_api` function in a list of statically loaded plugins
* Otherwise proceeds the same way
* Needs a different name though (to prevent linking name collision)
* `simple.c`
* Static linking is used on iOS, Android, PS4, XB1



## Cross-plugin calls

* In addition to requesting APIs from the engine, plugins can also get APIs from other plugins
* `void *(*get_next_plugin_api)(unsigned api_id, void *prev_api);`
	* Enumerate all plugins that implement a particular API
* `plugin.c`



## Plugin system implementation

* Scan for plugin DLLs in specific directories (matching patterns)
* `LoadLibrary(path)`
* `GetProcAddress(module, "get_plugin_api")`
* Call `loaded`
* At specific points (`start_game`, `stop_game`)
	* Call into loaded plugins
* Issue: What are good "interception" points for plugins?
	* How do we put a limit on that?



## Hot reloading

* Plugins can be hot-reloaded while game is running
* Especially useful when working on gameplay plugin
* Hot reload procedure - look for changed DLLs
	* Call `plugin->start_reload() -> state` -- returns serialized state
	* Unload old DLL, load new DLL
	* Call `plugin->finish_reload(state)`
* Working around windows file locking
	* Can't build DLL over old one (file locked)
	* Can't delete old DLL (file locked)
	* Same for PDB (except multiple versions can be locked)
	* Solution: Rename pre-build step:
		* `plugin.dll` -> `plugin.old.dll`
		* `plugin.pdb` -> `plugin.old.pdb`
		* Delete `plugin.pdb`
	* Build new `plugin.dll`



## Engine and editor plugins

* Engine and editor has separate plugin systems
* To make a plugin which extends both engine and editor, your plugin needs both parts
