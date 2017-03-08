# Part 1: Stingray overview



## Teams

* core
* render
* tools

* Files, memory, network, threading
* Physics, animation, sound, scripting



## Code philosophy

* Simple (decoupled, linear)
  * C-like C++
* Data driven
* Do not compromise on performance
  * O(N) mostly


## Data pipeline overview

Resources: `trees/larch.unit` (name.type)

                 compile
    [source data] -----> [compiled data]

* Source data (JSON, textures, fbx, ...)
* Compiled data: Packed binary (can be platform specific)
* Separate steps: compile & run

* The windows executable is the data compiler (`--asset-server`)
* Cross-compiles to other platforms
* Data compile is incremental



## Data at runtime

* Managed by `resource_manager.h`
* Load into memory: `load("tree/larch", "unit")`
* Accessed by systems: `get("tree/larch", "unit")`

* Grouped in packages
  * Becomes bundles



## Engine and editor

* Editor is a HTML5 application (Chromium)
* There is also a C# editor backend (we want to get rid of it)
* The editor viewport is the regular engine

+==============================+
|        | Engine              |
|        |                     |
|        |                     |
|        |---------------------|
|                              |
|                              |
+------------------------------+

* The engine runs in a window on top of the editor window
* It is moved in sync with the editor windows
* Engine & editor communicates through a web socket
* Editor reads and writes source data --> triggers compiles & reloads

* We can use the same mechanism to connect to a remote viewport
   * and for example make an Android phone slave after the level editor



## Extending the engine

* Engine can load DLLs to extend its functionality (Navigation, Wwise, ...)
* A C based API is used for communicating between the engine and its plugins
* It's based upon querying for interfaces
* `get_engine_api(DATA_COMPILER_API_ID)` -> struct with C function pointers
* `plugin_api.h`
* Plugins expose APIs that engine can query for
* (Editor has an extension system too)



## Game dynamics

* Can be written as a plugin DLL
* Or scripted with Lua (LuaJIT)
  * Lua API based on top of the C API
* Flow (visual scripting language)
  * Gets compiled to a "bytecode" (of sorts)
  * Interpreted at runtime



## Build system

* CMAKE generates project files for the different platforms
* Ruby frontend to CMAKE: `make.rb --engine -p ps4 -c debug`
* Once generated, projects can be built in Visual Studio
* Libraries are installed with `spm.rb` (from S3)
  * `spm-packages.sjson`


## Platforms

* win32, xb1, android, macosx, ios, ps4, webgl, linux
* `make.rb -p PLATFORM`
* Defines: WINDOWSPC, XB1, ANDROID, ...
* Platform specific data folder
* Secret platforms: xb1, ps4
* Code for these platforms is in subrepositories
* Only the win32 can compile data



## Running on platforms

* Usually we run in "file client mode"
* Start a file server locally
  * `stingray.exe --asset-server -secret xHG7fHy`
* (File server is the same as the data compiler)
* Launch runtime
  * `stingray.exe --host 162.0.0.15 --secret xHG7fHy --data-dir C:\data_xb1`
* Data is streamed over TCP to the client
* Can also deploy data locally (less common)


## Runtime

* `application.cpp`
* Load `settings.ini` for project
* Init all systems (threading, files, resources, ...)
* Lua callback `init()`
* Update loop
  * Update timers
  * TCP/IP connection (console server)
  * System windows
  * Input
  * Lua `update()`
    * Calls update on game worlds
  * `render()` (Rendering happens on render thread)
  * Other various system updates
* Lua callback `shutdown()`
