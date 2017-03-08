# Part 9: Diagnostics



## Diagnostics

* Finding out why something is going wrong



## Most basic: logging

* `logging.h`, `logging.cpp`
	* Log a string
	* To console, stdout, file (if so desired)
	* Add call stack for warning and above

```
namespace {stingray::logging::System REFRESH = {"Refresh"};}

logging::warning(REFRESH, eprintf("Cannot refresh resource " ID64_FORMAT "." ID64_FORMAT
	", don't know how to refresh type '" ID64_FORMAT "'",
	item.name.id(), item.type.id(), item.type.id()));
```

* eprintf()?
	* "Error print"
	* Print without having to allocate and release memory for it
	* `error.cpp`
	* Kind of hacky, but it works



## Asserts

*  `assert.h`, `error.h`
* Write log message with callstack + lua callstack + ...
* End the program



## Profiler

* Used to measure engine performance
* Through explicit profiling scopes put into the code

```
	void fun()
	{
		Profile p("fun");
		...
	}
```

* Accesses the `ThreadProfiler` for the thread (thread local variable)
* Starts a new profiling scope
* Ends the scope when it goes out of scope
* Records start time, stop time
* `profiler.h`
* At root scope, the `ThreadProfiler` sends its data to the global `Profiler`
* The profiler sends the data to connected web socket clients at `/profiler`
	* They can analyze the data and present it in a useful way



## Strings in the profiler

* We don't want to record a string for every profiler scope
		* We will have a lot of them
* Record just the string "pointer"
* Send a table on the web socket to match string pointer to string



## Statistics and graphs

* `grapher.cpp`
