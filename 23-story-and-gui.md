## Part 23: Story & GUI



## Story

* Used for creating animation sequences inside the level editor (cutscenes)
* Note: Distinct from character animation (edited in DCC tool)
* Story
	* A number of curves and events
	* Curve
		* (Time, value) pairs
		* Drives a specific target (unit node, unit property, entity property)
		* Settings: Interpolation type, endpoint type
	* Event
		* Name
		* Time indexes when the event occurrs
		* Used to drive other actions
	* `types.h`
* `player.h`
* Relatively simple system on engine side -- pretty complicated editor UI



## Curve targets

* `<unit index>.Head.position.x`
	* Unit index gets mapped to actual unit when *playing* the story
	* (You pass in a list of units to apply this too)
	* Same for entities
* For units we expose
	* Position, scale, rotation for all unit bones
	* Camera fov, near range, far range
	* Light color, intensity, angle
	* Mesh material properties
	* `unit_properties.cpp`
* For entities
	* All components have a generic property interface
	* Can set any entity property



## GUI

* Simple system for application UI
* (Note: Can also use Scaleform through plugin)



## Gui fundamentals

* A GUI object is `(id, layer, material, vertices*)`
	* GUI draw functions create vertices for the object
	* `gui.h`
	* Image: vertices for quad, material is image picture
	* Text: vertices are quads for each character, material is font bitmap
* GUI can live in either screen space or world space (drawn in the world)
* A GUI can run either in *immediate* and *deferred* mode
	* In *immediate* mode, all GUI objects are deleted every frame
	* You need to repeatedly draw the elements for them to appear
	* In *deferred* mode, GUI objects persist until explicitly `destroy()`ed
	* You can update an object by drawing again with the same `id`



## Gui drawing

* GUI objects are managed on the render thread
* `create/update/destroy` post messages to be processed by the render thread
* Render thread batches data by `(layer, material)`
	* Each batch drawn with a single draw call
* Could use more aggressive batching
	* Automatic texture atlases (currently needs setup by user)
	* Do not batch on layer (instead sort batches by layer)
