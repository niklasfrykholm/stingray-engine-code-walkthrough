# Part 19: Flow

* Visual scripting language
    * Nodes, events, variables

```
+--------------+                   +--------------------------+
| Level loaded |                   | Spawn unit               |
+--------------+                   +--------------------------+
|        Out o +-------------------> o Spawn      Spawned   o +--------->
+--------------+                   | o Unspawn    Unspawned o |
                                   |                          |
                          +--------> o Unit type       Unit o +--------->
                          |        | o Position               |
                          |        | o Rotation               |
+------------------+      |        +--------------------------+
| String           |      |
+------------------+      |
| trees/larch/03 o +------+
+------------------+
```

* Design goals
    * No "update" cost -- only cost when events are propagating
    * Data-oriented: Single memory block (for resource and runtime)
    * Interface with Lua
    * Extensible (came later)



## Flow compilation

* Two binary blobs:
    * Resource data: `[ node_1 | node_2 | ... | node_n ]`
    * Runtime data:  `[ node_1 | node_2 | ... | node_n ]`
* Example -- compiling a node `(unit_type, pos, rot) -> SPAWN_UNIT -> (Unit reference)`
* Resource data:
    ```
    SPAWN_UNIT  (node_type_identifier)
    unit_type   (offset)
    pos         (offset)
    rot         (offset)
    unitref     (offset)
    [lookup table for output events]
    ```
* Variables (wires) are stored as offsets into the runtime data
    * I.e., they specify where in the runtime data to find that particular data
* Runtime data for spawn unit:
```
unitref   (UnitReference)
```
* As we compile a node, it will allocate memory for its *output variables* in the runtime buffer
    * And store the current offset in the buffer
* A node does not allocate memory for its *input variables*
    * Instead it finds where the input variables is stored, and directly references that location
    * In example above, when `String` is compiled, it's data is stored at some offset
    * When we compile `Spawn unit` we lookup the offset of the connected variable and store it in `unitref`
    * Unconnected variables (`position`) are stored as null-offsets (`0xffffffff`)
    * Note: No need to copy variables between nodes, they will link to the same data
    * Two compile passes are needed
        * First pass: Reserve memory for output vars and store their offsets
        * Second pass: Lookup offsets of input vars
* Output events are stored with offsets into *resource data* where connected nodes are found



## Flow evaluation

* We have an offset into the resource data where the current node is stored
* Read node type, lookup node evaluation function to call based on node type
* Call node evaluation function (will fetch input data, perform the "action" of the node, write outputs)
* Trigger output events (i.e continue flow evaluation at output event node)
    * Possibily different output events depending on what happens in the node



## Defining flow nodes

* `.flow_node_definitions` file:
    ```
    /*
        @adoc flow
        @node Unit > Unspawn Unit
        @des Unspawns the unit from the current level.
        @in Unit        The unit to unspawn.
        @in Unspawn     The input event triggered to unspawn the unit.
        @out Unspawned  The output event triggered when the unit is unspawned.
    */
    {
      name = "unspawn_unit"
      ui_legacy_class = "Stingray.Foundation.Flow.UnspawnUnit"
      ui_category ="Unit"
      ui_brief ="Unspawns the unit from the current level."
      inputs = [
        {
          name = "unit"
          type = "unit"
        }
        {
          name = "unspawn"
          type = "event"
        }
      ]
      outputs = [
        {
          name = "unspawned"
          type = "event"
        }
      ]
    }
    ```
* Read by both data compiler and editor (to show UI)
* Standard nodes are defined in `core/` and implemented in main source code
    * At runtime, evaluation function is registered on name `("unspawn_unit", unspawn_f)`
* Plugins can extend the system (their own `.flow_node_definitions` and registered evaluation functions)
* Design question -- multiple ways of doing gameplay:
    * Flow nodes
    * Lua
    * C API
    * Plugin API
    * Maybe they all should be merged into one single system? Avoid duplicate work?



## Query nodes

* Query nodes are nodes that fetch data, not perform actions: i.e. `GetUnitPosition`
* Query nodes to not have an explicit `trigger` event
    * Triggered automatically when their data is needed
    * A node stores a list of the query nodes connected to its input variables
    * Before node function runs, it triggers all the query nodes to refresh their data



## Level and unit flows, subflows

* Flows can exist for levels, but also for units, and entities
* With units, each unit instance has its own copy of flow runtime data (separate contexts)
* Level flow can call into unit flow
    * Unit flow exposes external events and variables (input and output)
    * Can be connected to things in the level flow
    * Flow nodes representing units have these inputs and outputs
```
    ---------------
    | Level unit  |
    ---------------
    | o Open door |
    |             |
--> | o Speed     o |------->
    ---------------
```

    * There is also a general subflow mechanism
* Data in a different flow cannot be "shared" (like data on the wires)
    * Explicit copy of data when entering/exiting subflow
* Setting up the external variables/events is a bit messy -- individual nodes



## Trigger context

* How does `SpawnUnit` know which world to spawn the unit in? (not an input parameter)
    * Flow nodes run in a `TriggerContext`
    * `flow.h`
    * Pointer to resource data
    * Pointer to runtime data
    * Unit/Level/World/Entity that the flow is running in
* Care must be taken in keeping the `TriggerContext` right when calling subflows
    * When calling into a unit flow
    * When returning back from the unit flow into the containing flow



## Events in flow

* Flow is our main mechanism for dispatching events (from physics, animation, ...)
    * We don't allow registering callbacks through Lua or C APIs
* Example: physics collision
    * When `.unit` is compiled
        * Check for each actor if it has an `OnCollision` flow node
        * If it does, store offset of flow node in actor resource
    * When physics world detects collision
        * If either actor has an `OnCollision` offset
            * Push event into output event stream
    * When processing physics events (in `World`)
        * Lookup flow event for the involved actors
        * And trigger it



## Delay node

* The `Delay` flow node will trigger its output at some later time
    * `World` keeps track of delayed triggers
    * Make sure to delete them if flow owner (`Unit`) is deleted
    * `world.cpp`



## Script flow nodes

* In addition to defining new flow nodes in plugins, we can also define new nodes in Lua
* `.script_flow_nodes` file
    ```
    /*
        @adoc flow
        @node HumanIK > Setup and Debug > HumanIK Switch On
        @des Switches on HumanIK, for all characters and creatures. This is the very first node to be called in a flow graph, usually on a Level Loaded event.
            By default, HumanIK is turned off to spare memory and CPU.
    */
    {
        name = "HumanIK Switch On"
        args = {
        }
        function = "HumanIKFlowCallbacks.humanik_switch_on"
        category = "HumanIK/Setup and Debug"
        brief = "Switches on HumanIK for all characters and creatures."
    }
    ```
* Inputs and outputs are setup in a Lua table (indexed by name of variable)
* Calls a Lua function with that table as argument
* Can also write the code directly in `.script_flow_nodes` instead of referencing a Lua function
