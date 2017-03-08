# Part 13: Asset server



## Currently: Disk-based workflows

* Editor edits a source file, saves to disk
* Data compiler compiles to binary format, saves to disk
* Runtime loads file from disk
* Problems:
    * Latency: Touching disk takes time
    * Not fast enough for dragging a slider
    * Editor wants to preview before the file is saved
    * State is unclear with multiple editors/runtimes
    * Writing tools is complex -- needs to sync state
    * No way to do collaborative editing



## Asset server: Move responsibility to the engine

* Editors don't talk directly to disk but to an "asset server"
    * Send new file content or delta-JSON changes
    * In-memory state with explicit save command
    * Compiles from this state to in-memory binary data
    * File server can serve directly from this state
* Clients can connect to asset server and listen to changes
    * Editor viewports are just another client
    * No explicit state synchronization
* Multiple editors, multiple viewports can connect to the same server



## Asset server implementation

* `IDatabase` -- represents disk source with memory overlay
    * Listens to disk changes (so still compatible with disk workflows)
    * Server thread accepts client messages and forwards to database
    * Read, write, list, notify, etc
    * Delta-JSON writes sends delta-JSON notifications
* `database.cpp`
* `FileSystemCache` -- data compile output (cached in memory)



## Left to do on asset server

* Rewrite tools to work against asset server
    * In progress
* Concept of "versions" of compiled data
    * To avoid race conditions
* Speed up file server
    * So we can run against it locally
* Stability checks
