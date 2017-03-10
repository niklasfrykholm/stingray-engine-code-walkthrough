## Part 24: Multiplayer

* Implements synchronization of game state between multiple players over (some) network
	* Different games can have different ideas about what needs to be synced
	* Syncing "everything" is usually much too expensive
	* We require user to be in charge of sync instead of doing it "automatically"
* A complicated system with several layers and multiple backends
	* Hard to follow the code paths
	* Hard to reason about (Lots of state, async events, failures)
	* Unhealthy couplings between different layers
	* Frankly it's messy (cleanup, anyone?)
* Why is this system bad?
	* Hard problem to begin with -- external events, failure, what is the state space?
	* Abstracts over "backends" that work differently
	* Different configurations: peer-to-peer vs host-client
	* Did not fully understand problem at first (handshake, etc)
	* Scope creep over time (QoS, drop-in play, migration, priorities,  ...)
	* Added by multiple people
	* Hard to debug, hard to think about, hard to reproduce problems
	* Fear of refactor (at least it is working now)



## Networking concepts overview

* Matchmaking
	* Find other players to play with and establish a network connection to them
	* LAN, Steam, PSN, ...
* PeerID
	* 64-bit identifier used to identify other players
* Transport layer
	* Can send/receive messages (data packets) to peers
	* Matchmaking opens "channels" in the transport layer
* Connection layer
	* Transport layer is generally unreliable, unordered (UDP)
	* Implements reliable messaging on top of transport layer
* RPC message
	* Lua call sent as message between peers
	* Encoded and decoded at both ends
* Game object
	* Abstract object with "properties" that are automatically synched between clients
	* "Owned" by some peer, who holds the authorative state
	* Easier to reason about in most cases than using explicit messages
* Game session
	* "Scope" within which game objects are synchronized
	* Entered by peers
	* Either peer-to-peer or client-server
	* In client-server -- server owns *most* objects, all messages go through server
* Drop-in
	* A new client joins an already ongoing game session
	* Must be synchronized up to the current state
* Drop-out
	* A client leaves the game session (or disconnects, etc)
	* Ownership of game-objects must be transferred to other clients
* Encoding/packing
	* Encode rpc messages and synchronization messages as efficient as possible
	* Minimize network use
* Quality-of-Service
	* We have a limited bandwidth to each peer
	* Make sure we use it as well as possible
	* Don't send more data than it can handle
	* But send as much as it can handle
	* Prioritize the more important data
	* Quite tricky problem in general



## Example: Using network from Lua

* (Simple example from Stingray *testbed* project)
* `global.network_config` -- Network configuration
* `network_scene.lua` -- Pick network system
* `lan_scene.lua` -- LAN matchmaking implementation
* `steam_scene.lua` -- Steam matchmaking implementation
* `game_scene.lua` -- Inside game session



## Network

* Highest level network interface
* `network.h`
	* Uses a "backend" to implement lobby system (matchmaking) and transport protocol
	* Setup in Lua with `Network.init_lan_client()` (and similar for other backends)
	* `if_lan.cpp`
	* Keeps track of lobbies
	* Start/stops the game session
* `lan_client.cpp`
* `steam_client.cpp`



## Transport layer

* Send and receive messages (to a specific PeerID)
	* Simulate bad networks
* `transport.h`
* LAN
	* PeerID is a random number
	* Transport layer stores a map `PeerID -> IP`
	* This map is set up by matchmaking layer
* Steam
	* PeerID is SteamID
	* Sends data using `SendP2PPacket(peer_id, ...)`
	* Also sends steam authentication and VOIP packets
	* `Connection` layer calls `create_authenticator` on `ITransport`
	* Breaks the connection on bad authentication
		* Ugly



## Matchmaking / Lobby system

* Find games to join / players to start games with
* Open communication paths in transport layers (on backends where necessary)
* Messy: Needs separate communication path
	* Because `Connection` layer depends on communication already being set up in transport layer
* LAN
	* Uses a separate *lobby port*
	* Lobby finder broadcasts on lobby port (using raw sockets)
	* When a lobby replies, talks to it using commands on the lobby port
	* Member list is synchronized between lobby members
	* Lobby members get added to the transport layer (`PeerID -> IP` mapping)
	* Lobby server sets a flag to start a game, `Network.create_game_session()`
* Steam
	* PeerID is SteamID (no need to open ports in Transport layer)
	* Lobby sends messages using transport layer, but with a special prefix (`0x7fffffff`)
	* Installs hook into transport layer so these messages can be routed back to Lobby layer
	* Ugly



## Connection

* Implements reliable networking on top of unreliable (UDP) Transport
* `connection.h`
* Basic reliable transmission idea:
	* Assign a sequence number to each sent message `[0, 1, 2, 3, ...]`
	* On receiving message `i`, send a reply `ACK-i`
	* Keep sent messages in a buffer
	* "After some time", if no `ACK` has been received -- resend message
	* Receiver -- store incoming messages in a buffer
	* Only deliver next expected message `[0, 1, 2, 3, ...]`
* Complication: We don't want to send individual messages
	* Overhead
	* Add together as many messages as we can fit in an UDP packet
	* Do sequence and `ACK` for "entire packet" (less header data)
* Feature: We want to send unreliable messages too
	* To avoid complexities/delays of reliable
	* Fill a UDP packet with all reliable data we need to send
	* Append as much unreliable data that can fit
	* `UnreliableSource` abstraction to fill the data
* Feature: We want to detect disconnects even if we have no data to send
	* Send regular ping/pong messages (also determines ping speed)
* Complication: What if one endpoint crashes and restarts
	* Now the message numbers are out of synch `[0, 1, 2, 3, ...]`
	* Need to "reset" the connection (simultanously on both side)
	* But we don't necessarily "know" we are out of synch (could be just lost messages)
	* Make sides agree on a *session ID* unique for this connection session -- Handshake phase
	* Restart if the session ID doesn't match
* Feature: Quality of service
	* Measure data sent, ping time, packet loss, etc
	* Allow caller to set a bandwidth cap for how much data to send
	* Respect the bandwidth cap when transmitting data
* More issues
	* This must be reasonable fast and memory efficient
	* Shutting down -- making sure all reliable messages have been sent (and received?)
	* Priority
	* Starving: Make sure data of high priority doesn't completely starve lower priority
	* Tuning of buffer sizes, resend times, etc, etc
	* How long does a connection live? Forever? (Garbage collection)



## GameSession

* Peers (connections), host, RPCs, Game objects
* `game_session.h`
* `game_object.h`
	* RPC and game object formats defined in `network_config`: `game_session_config.h`
	* Defines bit packing of fields (implemented in `packing.h`)
* Basic game object synchronization
	* Each game object has an owner (authorative, only modifier)
	* On created - sends `CREATE` to all peers
	* Tracks *version* of each field in game object (modify updates version)
	* Remembers *version* seen by each other peer (through `ACK` messages)
	* Sends `UPDATE` (unreliable) to peers which have an old version
	* Sends `DESTROY` when game object is destroyed
* Feature: Game object migration
	* If an owner drops out, someone else must take over ownership
	* Make sure only a single peer takes over ownership
	* As messages are failing...
	* And one or more (but not all peers) may have received a `DESTROY`
* Feature: Host migration
	* If host drops out, someone else needs to take over
	* Even though peers may not agree on peer list or game object list at that point
* Complexity: Ordering
	* What if `DESTROY` message is reordered to appear before `CREATE`?
	* What if `UPDATE` comes before `CREATE`? Or after `DESTROY`?
* Complexity: What if data arrives before connection handshake (session ID) has completed
	* Re-ordering could put data before handshake messages
	* Buffer it until handshake has completed
* Complexity: Ensuring objects have unique IDs
	* Even if created simulatenously on different peers
	* And not using too many bits for the ID
	* (Assign each peer a bit range)
* Feature: Game object priority
	* Game objects get different update priority
	* More important game objects send updates more often
	* Important: No starvation
* Feature: Client-server model
	* Server owns (most) objects -- clients only own their own players
	* Server *relays* changes to client-owned objects (no peer-to-peer)
* Feature: Interpolation of game object fields
	* For smoother but delayed updates



## More (not covered in this talk)

* Dump
	* Dump all network messages to a file
	* Analyzed by tool, so you can investigate traffic afterwards
* Replay
	* Replays a network game -- based on dumping tools
* Unit synchronizer
	* Automatically synchronizes units (as game objects)
* VOIP
* Statistics tracking
* PSN, XboxLive



## Currently not handled

* Cheating
* Hostile peers
* More network backends (what about Android, iOS, custom server?)
