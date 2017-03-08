# Part 7: Data compiling



## Data compiling

* Take all the source data and compile it to binary data
* Only compile the minimum needed
    * Fast turn-around times
* Can run stand-alone `--compile --source-dir SD --data-dir DD --continue`
* Or as a server `--asset-server`



## Compile overview

* `main_datacompiler.cpp` (1118)
    * Init all systems needed by compiler
    * Set up database with file system mounts
    * Register compilers (267)
    * Trigger compile
    * Shut everything down



## The compile environment

* In `--asset-server` mode server serves multiple compiles
* We want interactive compile times (<100 ms)
* We don't want to re-index the disks, etc -- takes too long
* Solution: Cache everything (oh no!)
* `compile_environment.h`
* Look up objects based on directories etc
  * So that we can quickly switch between projects
  * Restart between projects would be simpler and perhaps good enough?



## Compiler components

* `DataCompiler` -- does the actual compile
* `SourceCache` -- a thin wrapper around the Database
    * File folders -- folders treaded as files `.s2d`
* `FileSystemCache` -- caches writes so we don't have to wait for disk flush
* `ExplodedDatabase` -- the file format of the `_data` folder
* `PackageIncludesDatabase` -- tracks package tagalongs
* `InformationDatabase` -- stores general meta information
* `ShaderCacheDatabase` -- keeps track of shader permutations (render stuff)



## Data compiler code

* `data_compiler.cpp` 553
* Get all sources (from SourceCache)
* Find out which files need to be compiled
    * Source file changed? (ExplodedDatabase tracks hash)
    * Dependency changed? (ExplodedDatabase)
    * Binary format `VERSION` changed
    * Compiled resource doesn't exist
* Call `compile()` callback function for each resource
    * In parallel
* Save changes to ExplodedDatabase, PackageDatabase, InformationDatabase
* Note: Files are compiled individually (no tree/chain of compiles)
    * Exception: `.package` files


## Writing a data compiler

`typedef DataCompileResult (*CompileFunction)(DataCompileParameters &params);`

* Produce a result from parameters
* `data_compile_result.h`
* `data_compile_parameters.h`
    * Source data
    * Destination platform
    * Check for/read additional files
* Example: `font.h` `font.cpp`



## ExplodedDatabase

* Class that represents the compiled data on disk
* `exploded_database.h`
    * Index of resource names
* ExplodedDatabaseBuilder -- knows how to build it
    * Keeps track of the dependencies of every compiled resource
    * Knows their hashes
    * Can answer if a resource needs recompile
    * `exploded_database.cpp`



## Tracking dependencies

* A dependency occurs when another file is read during compile
* Example: `tree.unit` reads `tree.physics`
* We track dependencies automatically
    * `data_compile_parameters.h`
* We also track `exists()` dependencies
    * If we during a compile ask if a resource exists, we record that
    * If existence changes (added or removed) -- we recompile



## Tracking package includes

* Very very important: Understand these concepts
    * *Dependencies*: Used during compile, will trigger a recompile
    * *Package includes*: Including one file in a package will bring along others
    * *References*: Links to other files, will need change if file is moved
* Similar but different concepts
    * Sometimes "dependencies" are used to refer to all three of them :(
    * Confusing them will lead to tears
* Added by calling `include_in_package()`
* Recorded in `package_includes_database.h`



## Compiling a package

```
bik = [
	"video/form"
]
config = [
	"core/performance_hud/performance_hud"
]
font = [
	"core/performance_hud/debug"
	"gui/font"
]
ivf = [
	"video/parkjoy"
]
level = [
	"levels/*"
]
lua = [
	"lua/*"
	"core/scripts/smoketest"
]
material = [
	"core/performance_hud/debug"
	"core/performance_hud/gui"
	"gui/font"
	"video/yuv"
	"video/bink"
	"units/procmesh"
]
network_config = [
	"global"
]
package = [
	"cloth"
	"localized_sounds"
]
physics_properties = [
	"global"
]
strings = [
	"gui/strings"
]
unit = [
	"core/units/camera"
	"core/editor_slave/units/skydome/skydome"
	"units/character/character"
	"core/editor_slave/units/sound_source_icon/sound_source_icon"
	"units/box/box"
]
vector_field = [
	"vector_fields/*"
]
```

* `resource_package.cpp`
    * Take the original resources
    * Add all package includes recursively from the database
    * Modifiers `IGNORE_INCLUDES`, `ADD_PACKAGES`, `SUBTRACT_PACKAGES`
* Globbing
    * A resource package can include globs `lua = ["*.lua"]`
    * Includes all matching files
    * We resolve it by calling `globs()` in `data_compile_parameters.h`
    * This creates a "glob dependency"
    * Whenever the glob match changes -- the file is recompiled
* Packages in packages
    * Source of some confusion
    * Just references the packages, does not include its resources
    * Use `ADD_PACKAGES` for that



## Overrides

* `settings.ini` can specify overrides for the data compiler

```
data_compiler = {
	resource_overrides = [
		{suffix = ".win32", platforms = ["win32"]}
		{suffix = ".android", platforms = ["android"]}
    {suffix = ".hires", flags = ["hires"]}
	]
}
```

* This means that on Android any resource with `.android` before the extension will
  replace the original resource
* `tree/larch.android.unit` will replace `tree/larch.unit`
* Used for localization, platform specific resources, and more
* If `hires` flag is set, `tree/larch.hires.unit` will replace `tree/larch.unit`
* Flags can be set statically (at compile time) or dynamically (at runtime)
* When compiling a package, we store all the overrides found for its resources
    * `tree/larch.android.unit` should override `tree/larch.unit`
    * `resource_package.h`
    * `resource_id.h`
    * The actual override is done at runtime, by the resource loader
* Note: We also need to take overrides into account when figuring out the package includes



## Writing additional information

* Some times we want to store extra metadata during the compile
    * For caching or for the editor to use (not a part of the resource)
* We have two ways of doing so
* `data_compile_parameters.h`
* Editor file system
    * `editor_write()`, `editor_read()`
    * Used to cache `.fbx` files
    * And to save meta-information about compiled shaders for the editor
* Information database (currently not in use)
    * `void record_information(const char *table_name, const DynamicConfigValue &dv);`
    * Arbitrary key-value store



## Bundling

* Bundles together resources for deployment (`.pak` file)
* Takes all the resources in a package and puts them in one big file
* All the stream data put into a single stream file
* Anatomy of a bundle
    * `bundle.h`
* Bundle is zlib compressed in chunks of 64 K
    * `segment_compressed_file_output_buffer.h`
* Loading a bundle walks linearly over bundle, loading all resources



## Data duplication

* If two packages include the same resource, it will be duplicated in the bundles
    * Example: Same unit used in two different levels
* Only loaded once in memory, but uses more disk space
* This makes sense for DVD distribution
    * Less so for other media
* Strategies:
    * Create packets with shared resources: `forest_units.package`
    * Load that for all forest levels
    * In the level packages, use `SUBTRACT_PACKAGES = ["forest_units.package"]`
* Should maybe be changed?
    * Digital distribution + SSDs
