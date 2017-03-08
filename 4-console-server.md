# Part 4: The console server



## Websocket server

* Used for all external communication
	* Tools, file serving, logging, profiler data, etc
* Websocket used for compatiblity
* `web_server.h`
	* Data is received on socket connections
	* Parsed into web socket and HTTP events
	* Tagged with request string `/profiler`, `/fileserver`
	* Recycling pool of memory pages for storing the data
* Higher level systems poll the web server and posts replies
* Possible future optimization: Reduce copying



## Console server

* Owns and pumps the main web server (HTTP and web sockets)
* Singleton object `console_server::get()`
	* So we can send logging messages from anywhere
* Other systems can hook into specific requests
	* `console_server::get().hook_websocket(this, IdString64("/profiler"), callback)`
	* Will receive all `/profiler` messages
* Other systems can also `detach()` the websocket
	* Take over the socket from the web server
	* Systems can manage it on their own for maximum efficiency
	* Processed on a custom thread
* Similary you can `hook_http()` to serve a web page at a specific request



## Console server messages

* The console server also manages the default connection (request_string = `""`)
* Used by tools and the editor console (hence the name)
* Messages are JSON objects with a type field
	`{"type": "script", "script": "print(math.pi)"}`
* You hook a receiver for a message type:
	`console_server::get().hook_receiver("script", userdata, callback)`
* In the callback function -- process the message and optionally send a reply:
	`console_server::get().send(reply)`


## Console server message types

* Lots! Some examples:
	* `script` -- Lua code to run
	* `command` -- Console command line commands (more on that later)
	* `lua_debugger` -- Set breakpoints, evaluate variables, etc
	* `compile` -- Compile project (for `--asset-server`)
* Logging:
	* `{"type": "message", "level" : "warning", "system": "Physics", "message": "..."}`
	* Received and displayed by console tool



## Console commands

* Provide a command-line like interface to the engine
	`> physics debug raycasts`
* JSON
	`{"type": "command", "command": "physics", "arg" : ["debug", "raycasts"] }`
* Converted to Lua call by engine
	`Console.physics("debug", "raycasts")`



## Why all the options?

* To implement something (`set-breakpoint`) you have multiple options:
	* Custom web socket address: `hook_websocket("/lua_debugger")`
	* Custom JSON command type: `hook_receiver("lua_debugger")`
	* Command: `Console.set_breakpoint(file, line)`
	* Regular Lua API function: `LuaDebugger.set_breakpoint(file, line)`
* Wouldn't it be better to just use Lua API?
