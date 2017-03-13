# Q & A

## Rick Appleton:

* @niklasfrykholm regarding plugin "interception" points, is that not handled by your high level game loop in Lua? Is granularity too big?

* @niklasfrykholm allocators are C++, your plugin api C. I noticed ALLOCATOR_API_ID, to get allocators in plugin? Can you give details?

## Jeremie St-Amand:

* Concerning entiy-components, are there any ways to minimize entity lookups in a component when communicating between different components?

  What happens in a real game scenario is that there is a lot of inter-communication between components, and the only easy way to do that with Stingray's approach is to lookup a ton of entities (sometimes each frame).

  What would be the best practice to do communication between multiple components without looking up the instance by entity?

## Per Larsson

* The Entity hierarchy and the scene graphs both form a transform hierarchy.
  Did you ever consider having just a big scene graph or just using entities?



* The Entity system uses Component Instance IDs and Component Instances (to support garbage collection?).
  When we need to store a reference to a component instance we use the component instance id and when we want
  to interact with the component instance we need to do a lookup from id to instance.
  Can you talk a little bit about the design decisions about this?

## Laurie Hedge:

### Collection Classes

* You mentioned that all your strings are encoded in UTF-8. Do you find this causes any problems, such as not being able to index into the string in a meaningful way, making iteration difficult etc.?

* On a related note, it seems like one of the reasons for your use of char const * and Array<char> for strings is to avoid lots of dynamic string allocations. However by using UTF-8 as your internal representation, don't you find you're constantly having to convert these strings into other formats to use them? You mention that for calling into Windows functions, but I'd have thought most usages of strings would require a fixed character size (e.g. glyph rendering in freetype uses UTF-16/UTF-32 etc.).

* One disadvantage of using C's native strings is that you have to do with null terminated strings rather than buffer and length strings. Do you think this is a problem? I find having a cached length can be pretty useful and help avoid some buffer overrun mistakes.

* With the other containers, one thing that often frustrates me is the tight coupling between data layout and semantics. For example, I often find that I want the semantics of a hash map, but will only ever have a very few elements, and so an array of key-value pairs would be more efficient. Have you considered adding containers like boost's flat_map/flat_set etc.? Are there any down sides you can see to doing this?

### Threading

* You spoke a lot about the design decisions behind decomposing individual systems into tasks, whilst not overlapping the tasks of different systems very much (beyond the overlap between the tasks from the game-thread and the render-thread). This makes a lot of sense now, but do you see it as being a threat to the scalability of the engine long term? Many modern engines like the Destiny Engine, Nitrous engine etc. seem to be going for a completely task-based approach. Do you think this is something Stingray will eventually have to do?

### Testing

* Like you, I've found unit tests to be really valuable. However I always find I'm running into the problem that games systems, by their very nature, seem to have a lot of inter-dependencies. In a "normal" application, I would resolve this through dependency injection and mocking/spoofing all the dependencies of the SUT. This works well in languages like Java where everything is virtual, or in dynamic languages with duck typing like Python, but I tend to find that in games, I don't really want to pay the runtime cost of making everything a virtual interface just to support mocks, when there is only going to be one concrete implementation in the application that could be statically linked. Do you have any approaches in Stingray to avoiding this problem? I've seem some people discussing link-time dependency injection as a possibility. Does your build/test system allow for anything like this?

* Is your integration testing/automation system something that only the engine can use, or is it available for testing gameplay code as well? You mentioned that the system was under-used in the engine. If it's open to customers, do they tend to use it more? I know some game studios have invested heavily in automated testing.

* What led to the decision to write test scripts in Ruby? Do you think there is any down-side to introducing more languages into the engine? Does Ruby have any properties as a language that lend it particularly well to the job of testing? Is it possible to use other languages here to drive the tests?

### Input

* The engine seems to provide a fairly "raw" kind of access to the user input, with enumerating what buttons/axes are available, getting input events or measuring deltas from them etc. Other engines, such as Unreal, prefer to provide a higher level of abstraction, where various inputs from different controller types are mapped onto more conceptual actions (e.g. mapping <A> on the 360 controller and <Enter> on the keyboard onto an 'accept' action). Does Stingray offer anything like this? Should it? Is there any design philosophy for avoiding this kind of thing?

### Entities

* In the inheritance system for entities, you are allowed to add and remove components from the parent, as well as overriding them. You mention that this can lead to some confusion about where things are added/removed in the inheritance hierarchy, and this definitely matches my experience of working in similar systems. Does Stingray offer any tooling to help track down where components and their values are coming from?

* One problem I've often come across with entity systems is component coupling. Components often need to communicate between each other for one reason or another. What is Stingray's approach to this?

* Another issue I've had with component systems is that they often lead to either an additional level of indirection for every operation, or writing a lot of wrapper functions. For example, for moving a unit, you might want to say my_unit->move(x, y). Under an entity system I often find I either have to write my_unit->movement_component->move(x, y) in each place, or alternatively, for every method, I have to write a function like void Unit::move(float x, float y) { movement_component->move(x, y); }. Does Stingray have this problem, and do you consider it an issue? If so, do you have a preferred approach to solving it?

### Multiplayer

* It seems like the engine is using UDP for performance reasons, but has then built a reliable packet system on top of that. What is it that makes Stingray's custom reliable packet system faster than TCP? My understanding was that the reason TCP was slower than UDP was because it does something pretty similar to Stingray's own system, with having to buffer packets, resend etc.

* Does Stingray's communication or transport layer perform any compression/decompression, beyond packing multiple messages into a single packet? If so, what kind of compression/heuristics do you use?

* You mentioned that the engine offers a pre-built system for using Game Objects to synchronise units across the network. Will something similar be available/possible for the new entity based system? If so, how will it work? Will components be responsible for syncing their properties with Game Objects?

### Misc

* Does Stingray offer an approach to localisation? If so, is there anything you can say about that?

* You mentioned in the containers talk that Stingray makes use of Optional and Error types for some error handling. Could you say any more about the general philosophy behind error handling in Stingray? Are exceptions ever used? How about return codes?

* Similarly I remember you saying (I think in the testing talk) about moving away from a "crash on error" approach because of the impact this has on the rest of the team in the case of a mistake. Could you say any more about defensive programming in Stingray? How defensively is the code written? For example if you use an Error type when getting a pointer and there is no error, would you null check the pointer too?

* You mentioned in several talks the design decisions around backwards compatibility (particularly in regards to the plugin system). Do you have a philosophy about managing backwards compatibility overall? Do you believe you can ever drop support for a feature? If so, when? There were many systems you suggested could do with refactoring, or that had already been refactored. How do you avoid building up numerous legacy systems? For example, will you ever drop the old unit system do you think, once entities are fully matured?

* Finally, and I know this is a bit vague, but could you say any more about your general philosophy on developing a general-purpose game engine? You mentioned several times the idea of only paying for what you use. Are there any other principles you try to follow when designing the engine? It feels to me like designing a single framework to support things as varied as mobile tower defence games through to first-person multiplayer console RPGs and everything in-between must be incredibly difficult, especially when you don't want to burden the little mobile tower defence game with all the weight associated with a giant AAA console game.
