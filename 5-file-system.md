# Part 5: File system and I/O



## Overview

* Simple: Interface to OS disk reading functions
* But:
	* Reading from memory (.pak/.zip) files
	* Reading from network (file client mode)
	* Asynchronous reading
	* Prevent disk seeks
	* ...
* This functionality has been added over time -- a bit messy



## FileSystem

* Abstracts over reading files at paths
* `file_system.h`:
	* FileSystem < IWritableFileSystem < IFileSystem
	* `file_system::make_tree()`, etc
* `file_system.cpp`
	* Delegates to `file_client`



## Special file systems: IDatabase, "virtual file system"

* `database.h`
	* Associates files to paths
	* Also knows the hash of every file
* Database types
	* Memory based
	* Disk based
	* Mount based
* This is the basis for the asset server



## Special file systems: FileSystemCache

* `file_system_cache.h`
* Buffers written files in memory before writing them to disk
* Used for data compiler output
* Shared with file_server
* Allows compile to finish before data has been written to disk
* Faster iteration times



## Reading and writing files

* `input_buffer.h`
	* Get chunks of data from file into a temporary memory buffer
	* We don't want to read byte-by-byte from the file
* `file_input_buffer.cpp`
	* If `async=false` -- stalls to fill buffer
	* If `async=true` -- attempts to fill buffer in background
* `memory_input_buffer.h` -- InputBuffer from memory data
* `input_archive.h`
	* Thin wrapper around an InputBuffer + (range, position)
	* Uses a shared pointer, so can be passed by value
	* (Pretty much only place in engine)
* Output system looks similar



## File Queue

* Purpose: prevent disk stalls
* Why do we need it:
	* `open()` can stall
	* OS can split and reorder file reads between threads
	* Thread A reads 1 MB, Thread B reads 1 MB
	* Disk seeks
* FileQueue queues all file operations on a background thread
* `file_queue.h`



## Streaming data without stalling

* Create a `FutureInputArchive` (to avoid stall on open)
	* Wait for archive to become `ready()`
	* Obtain a real `InputArchive`
* `archive->buffer()->set_read_chunk()`
	* Set chunk of asynchronous read-ahead
* Check `can_flush_without_stalling()` to see if data is available
	* Force flush if necessary
* This is much too complicated
	* Would be better to talk to file_queue directly
	* But file_queue doesn't redirect to file_client
	* Some refactoring here would be nice



## New serialization system

* Generate a blob of binary data
* Typically using `stream::pack()` -- `stream.h`
* Write it directly to disk
* Read it back directly into memory



## Old serialization system

```
template <class STREAM>	void serialize(STREAM &s) {
	s & item1 & item 2 & item3;
}
```

* Single function for serializing in/out
* Templated on `STREAM` which can be `InputArchive` or `OutputArchive`
* `&` operator is resolved to either write or read data
* Special code for doing things like pointer patching
* Example: `unit_resource.h`
* Whole system is BLARGH!
	* Too much magic!
	* Too many templates
	* Slow
* Should be replaced with just writing binary blobs everywhere
	* A bit of work in that refactor



### File Client / File Server

* File server listens to web socket  `/fileserver`
* On connection, detaches web socket and spawns a thread to manage connection
	* Binary protocol for efficiency
	* `CONNECT` to directory
	* `OPEN` file
	* `READ` data
* Security
	* `--secret` must match between client and server
	* Only serves `.stingray-asset-server-directory` directories
