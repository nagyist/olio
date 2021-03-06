# olio

Synchronized collaborative state editing.  

Use as:

0. A client which syncs with a server.
0. A server which syncs with multiple clients.
0. A client which syncs directly with other clients in a peer-to-peer network.

Transport layer and synchronization timing are up to you. Use HTTP, Websockets,
WebRTC data channel, or whatever.

Peers generate sync requests and sync responses (a sync cycle). A sync request
contains changes to state made by peer A which peer B should apply. The response
from peer B contains changes since peer's A last sync.

```js
// A state is any generic data structure.
// State objects are synchronized between peers.
var myState = new State();

// Sync objects wrap a state and manage synchronization between an arbitrary
// number of peers.
var sync = new Sync(myState);

// Add a peer to synchronize states with. Can be a server peer, or any other.
// Add as many peers as you like.
// In a centralized server architecture, all clients add the server as a peer;
// and the server adds all clients as peers.
sync.addPeer("server");

// Modify the state:
myState.set("/name/first", "Donald");
myState.set("/name/last", "Duck");
myState.set("/nephews", ["Huey", "Dewey", "Louie"]);

// Start a synchronization cycle with a peer:
var patch = sync.patchPeer("server");

// Send the patch to your peer and wait for an answer...

// When the answer arrives, receive it:
sync.receive("server", answer);

// Now my state and peer's state are identical.
// Ready for a new sync cycle...
```

Based on [Differential Synchronization by Neil Fraser](https://neil.fraser.name/writing/sync/eng047-fraser.pdf).

[Collaborative drawing demo:](examples/collab_app/draw_client)

![Draw demo](demo_draw.gif)

[Collaborative text editing demo:](examples/collab_app/write_client)

![Text demo](demo_write.gif)

![Text demo](demo_write_words.gif)

## API

### State

A state is where your data is stored. It's comparable to a plain JS object
bundled with an event emiiter. So you can be nottified of any changes made to
the state.  
You can store any primitive JS type in your state. Setting arrays and plain
objects in the state will traverse them recursively and set the primitive values
in the proper paths in the state.

```Javascript
import State from "olio/state";
var State = require("olio/state");
require(["olio/state"], function(State){ /* ... */ });
```

`State` is a constructor which receives an optional JSON object of the initial
state.

`var s = new State()`  
`var s = new State(init)`

#### State modifiers

##### `s.set(keypath, value)`

0. `keypath {String}` - A [JSON pointer](http://jsonpatch.com/#json-pointer)
0. `value {JSON/Array/Primitive}`

Set a primitive value in a path in the state.

```js
s.set("/name", {
  first: "Donald",
  last: "Duck"
});

// equivalent to:
s.set("/name/first", "Donald");
s.set("/name/last", "Duck");
```

##### `s.clear()`

Clear the state by removing all keys.

##### `s.remove(keypath)`

0. `keypath {String}` - A [JSON pointer](http://jsonpatch.com/#json-pointer)

Remove value in the specified keypath.

##### `s.update(keypath, updater)`

0. `keypath {String}` - A [JSON pointer](http://jsonpatch.com/#json-pointer)
0. `updater {Function(oldVal)}`

```js
s.set("/a", [1, 2, 3]);
s.update("/a", a => a.map(v => v * v));
s.toJSON();
// { a: [1, 4, 9] }
```

#### State accessors

##### `s.get(keypath)`

0. `keypath {String}` - A [JSON pointer](http://jsonpatch.com/#json-pointer)

Get the value under the specified keypath.
Returns a primitive JS value, a plain object or an array.

```js
s.set("/a", [1, 2, 3]);
s.get("/a/0"); // 1
s.get("/a"); // [1, 2, 3]
```

##### `s.toJSON()`

Get the state as a JSON.

```js
var init = { foo: "bar" },
    s = new State(init);

var o1 = s.toJSON();
// { foo: "bar" }

assert(init !== o1);

s.set("/foo", "baz");
var o2 = s.toJSON();
// { foo: "baz" }

assert(o1 !== o2);
```

#### State events

##### Observing changes in state with `"change"` event

Any change in the state emits a `"change"` event.  
The event handler is called with three parameters:

0. `keypath {String}` - A [JSON pointer](http://jsonpatch.com/#json-pointer)
0. `newVal {JSON/Array/Primitive}` - The value in that path, after the change
0. `oldVal {JSON/Array/Primitive}` - The value in that path, before the change

```js
var s = new State();
s.on("change", (path, newVal, oldVal) => {
    console.log(path, oldVal, "-->", newVal);
});

s.set("/a", [1, 2, 3]);
// /a undefined --> [ 1, 2, 3 ]

s.update("/a", a => a.map(v => v * v));
// /a [ 1, 2, 3 ] --> [ 1, 4, 9 ]

s.set("/a/0", -1);
// /a/0 1 --> -1

s.set("/a", "foo");
// /a [ -1, 4, 9 ] --> foo

s.clear();
//  { a: 'foo' } --> {}
```

#### Immutability behind the scenes

Behind the scenes, state is manages with [immutable](https://facebook.github.io/immutable-js)
objects. It makes it easier to track and report changes in state, as well as
keeping history of states and the possibility to undo changes.

This means that state must be accessed and modified through one of the provided
accessors or modifiers.

```js
var a = [1, 2, 3],
    s = new State();

s.set("/a", a);

// a may be changes without affecting the state
a[0] = -1;
assert(a.get("/a/0") === 1);
```

### Sync

```Javascript
import Sync from 'olio';
var Sync = require('olio').Sync;
require(['olio/sync'], function(Sync){ /* ... */ });
```

`Sync` is a constructor which receives a state.

`var sy = new Sync(state)`

### Adding a peer

`sy.addPeer(id);`

0. `id {String}`

### Sync cycles

A sync cycle can be initiated by any peer. Each cycle is made of:

0. A patch is generated by peer A (the initiating peer).
0. The patch is sent to peer B (over the network for example).
0. Peer B applies the patch and generates an answer patch.
0. The answer patch is sent to peer A.
0. Peer A applies the answer patch.

At the end of the cycle both peer have a consistent state; if further changes
were made, a new cycle needs to be executed.

#### Patch peer

`sy.patchPeer(id)`

0. `id {String}`

Generate a patch for the specified peer.

#### Receive a patch

`sy.receive(id, patch, preferRemote)`

0. `id {String}`
0. `Patch {Array}`
0. `preferRemote {Boolean}`

Receive a patch either as an answer or as a cycle request. If a cycle request
this method will return the answer to be sent to the other peer.
