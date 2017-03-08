# Part 18: Entities



## Goals of entity system

* Units work reasonably well for most cases, but:
    * Not possible to create really light-weight units
    * Extensibility is limited
        * Script-data (key-value store)
        * Increasingly abused
    * No inheritance/templating/prefab system
        * Make one of these, but red
        * Make one of these, but with different windows
* Eventually: completely replace unit system



## Entities and components

* `entity.h`
* An entity is represented by a unique 30 bit ID
* Just as with units, this is a weak reference with 22 index bits and 8 generation bits
* Nothing is stored *in* the entity, no list of components, etc
* `entity_manager.h`
* Components are handled by `ComponentManagers` -- knows all the components of a certain type
    * Maintains an `entity → component+` mapping
    * Loose coupling, entity has no explicit list of components
    * Components are destroyed either by callback in entity manager or garbage collection
    * Entities can have an owner, destroyed when owner is destroyed
    * Component managers can layout their data internally and process it as they want
    * You can create a ComponentManager anywhere -- even in Lua

```
-- Associates Lua data with an entity
LuaDataComponent = class(LuaDataComponent)

function LuaDataComponent:init()
    self._lookup = []
    self._gc_iter = nil
end

function LuaDataComponent:set_data(e, data)
    self._lookup[e] = data
end

function LuaDataComponent:get_data(e)
    return self._lookup[e]
end

function LuaDataComponent:garbage_collect()
    self._gc_iter = next(self._lookup, self._gc_iter)
    if self._gc_iter then
        if not EntityManager.alive(self._gc_iter) then
            self._lookup[self._gc_iter] = nil
        end
    end
end
```

* There is no way of enumerating all components of an entity
    * You have to ask all ComponentManagers (which may be in Lua, on a server, etc)



## Entity resources

* A basic entity resource is a list of components with data in an `.entity` file:

```
components = {
	"36e5fb10-79d6-47ad-a1c5-bfedf5337116" = {
		name = "Change me!"
		$type = "debug_name"
	}
    "efe341c4-8f87-499c-9e2d-41537866c0a8" = {
		pos = [0 0 0]
		$type = "transform"
		name = "transform"
	}
}
```

* Identified by GUID (for merging, and data referencing)
* `$type` maps to the name of a component manager
    * Compiler for the component parses the rest of the JSON data
    * Compilers are registered `register_component_compiler("debug_name", f)`
    * Components are compiled and made part of the entity (more on this later)



## Child entities

* An entity can have an arbitrary number of child entities
    * Spawned together with the entity when it is spawned
    * Child entities are kept track of by the `TransformComponent` (that handles entity positions)

```
components = {
	...
}
children = {
    "abddb409-fd2d-4e29-82a2-f4033086263b" = {
        components = {
        	"36e5fb10-79d6-47ad-a1c5-bfedf5337116" = {
        		name = "I am a child!"
        		$type = "debug_name"
        	}
            "efe341c4-8f87-499c-9e2d-41537866c0a8" = {
        		pos = [0 0 3]
        		$type = "transform"
        		name = "transform"
        	}
        }
        children = {
            ...
        }
    }
}
```

* The child entities can have child entities of their own
    * Etc ad inifinitum
* When the entity system is complete it will replace not only units, but also levels
    * A level will just be an entity with a lot of child entities (level objects)



## Entity reuse (aka inheritance, aka prefabs)

* An entity can *inherit* another entity
    * It will get all the components and children of that entity
    * Can add its own components and its own children in addition to those inherited

```
children = {
    "234234234234" = {
        inherit = {
            $resource_name = "entities/tree"
    	    $resource_type = "entity"
        }
    }
    "dfdfd" = {
        inherit = {
            $resource_name = "entities/tree"
    	    $resource_type = "entity"
        }
    }

}
```

```
inherit = {
	$resource_name = "entities/box"
	$resource_type = "entity"
}

components = {
	"36e5fb10-79d6-47ad-a1c5-bfedf5337116" = {
		name = "My own component!"
		$type = "debug_name"
	}
}
```



## Inheritance with modification

* We might want to inherit, but modify some properties of the inherited entity:

```
modified_components = {
	"36e5fb10-79d6-47ad-a1c5-bfedf5337116" = {
		name = "I changed the name from the parent component!"
	}
}
deleted_components = [
	"5f1fbf2e-1ed6-4554-bfef-ef595bcb7dc1"
]
modified_children = {
	"f5a0dd51-2d35-47ab-a341-bd67d01dc828" = {
		deleted_components = [
			"70b2a4e6-9756-460e-ade8-4ed3efe258cb"
		]
	}
}
deleted_children = [
    "f5a0dd51-2d35-47ab-a341-bd67d01dc828"
]
```

* Parameters set for modified components will override those in the inherited entity
    * Component GUID is used to match
* Deleted components are components that exist in the inherited entity, but that will be removed for this one
* Same applies to `modified_children`, `deleted_children`
* Note in `modified_children` you can use `components`, `modified_components`, `deleted_components`
* And `children`, `modified_children`, `deleted_children`
    * And in the `modified_children` of the child you can do that again (modify a grandchild)
    * Etc, ad inifinitum
    * The rabbit hole goes deep



## Component interface

* Concepts
    * `id` -- identifies one specific component instance (permanently)
    * `instance` -- "pointer" to instance in memory (may change frame to frame)
* Basic component interface
    ```
        create(entity) -> (id, instance)
        lookup(entity, id) -> instance

            set_transform(instance, new_transform)...

        all_instances(entity) -> [instance]
        all_ids(entity) -> [id]
        destroy(entity)
        destroy(instance)
    ```
* Considering changing this to: `create(entity, id) -> instance`
* Not all components support multiple instances (i.e. `TransformComponent`)
    * Note: Currently separate APIs for SingleInstance/MultiInstance -- will be merged



## Typical component implementation

* Structure-of-array in single memory block
* TransformComponent:
```
    [ local_1 | local_2 | ... | local_n |
      world_1 | world_2 | ... | world_n |
      ...
      id_1    | id_2    | ... | id_n    |
      prev_1  | prev_2  | ... | prev_n  |
      next_1  | next_2  | ... | next_n    ]
```
* Basic interface
    * `instance` is index into these arrays, `id` unique number
    * `create()`: grow array if necessary, default initialize data
    * `lookup(entity, id)`: `HashMap<entity, first_instance>`, follow `next` to find `id`
    * `all_*()`: hash map lookup, follow `next` to find all
    * `destroy()`: Swap-erase, update `next` and `prev` pointers
* Component specific interface
    * `set_local(instance, local)`, `world()`, ...



## Compiling entities

* `compile()`: Merged compile data -> binary buffer
    * `actor_component.cpp`
* `spawn()`: Gets pointer to the compiled data + list of entities -> create components
    * Create multiple components at the same time
    * `actor_component.cpp`
* EntitySpawner: `entity_resource.cpp`
    * Create all entities
    * For each component manager (in order)
        * Create all component instances for that manager
    * Link entities to parents
    * Perform spawned callbacks



## Property system

* Allows editor to configure entities in a general / data-driven way
    * `set_property(instance, key, value)`
    * `get_property(instance, key) -> value`
* Value can be `bool`, `float`, `char *` or `float *`
* Key is an unsigned hash of a sequence of key values
    * `transform.position` = (1,0,0)
    * `combine(hash(transform), hash(position))`
* `transform_component.cpp`
* How does the editor know which properties exist?
    * Defined in type descriptor files in the editor
    * No introspection (good/bad?)



## Component types

* `debug_name_component.cpp`
* `tag_component.cpp`
* `data_component.cpp`
* `flow_component.cpp`
* `render_data_component.cpp`
* `script_component.cpp`
* `transform_component.cpp`
* `unit_component.cpp`

### Not yet in editor

* `actor_component.cpp`
* `animation_blender_component.cpp`
* `animation_state_machine_component.cpp`
* `mesh_component.cpp`
* `scene_graph_component.cpp`



## TransformComponent && SceneGraphComponent

* Stores local pose, world pose, parent entity, parent node
* `set_local()` transforms all children
    * World position is never out-of-date
    * Long chains where all nodes transform is rare
    * In those situations we can have custom solutions for updating them


## Unresolved issues: Editor-engine interaction

* Engine and editor has different views
    * Editor keeps track of "inheritance" and "overrides"
    * In runtime engine everything is merged
* Synching between editor and engine is tricky
    * Compile and reload is (maybe?) to slow
    * If a "prefab" is changed, editor must compute all resulting changes and update the objects
    * Lots of handwritten synch code that is hard to maintain
* Solution (probably): use the delta-JSON interface with notifications
    * Engine viewport is responsible for understanding delta-JSON changes
    * Default to a full compile and reload
    * Add fast paths: is this a change that I understand, use fast path
    * Engine needs to keep inheritance graph information at runtime
