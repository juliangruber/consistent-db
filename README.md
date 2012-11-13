
# consistency

Distributed master-only set store with strong consistency guarantees.

There is only one central component in the architecture: A service registry, so
nodes know all other nodes all the time

Open questions:

* What happens if the connection is lost? It might also be on purpose when you
  remove a node and keep using set without replication
* How to persist `set` on disk

## Scenarios

* 2 nodes, connection breaks => halt updates
* 3 nodes, connection breaks => 2 nodes replicate, 3rd halts
* 4 nodes, connection breaks => one half replicates, other halts

=> If the node graph is split, only one half can continue working

## usage

```js
var ConsistentSet = require('consistent-set')

var set = new ConsistentSet()

// set the expected size of the cluster
// updates are queued when the cluster's size < clusterSize / 2
set.clusterSize(6)

// replicate
var setStream = set.createStream()
setStream.pipe(net.connect(3000)).pipe(setStream)

set.add('name', function (err) {
  if (err) throw new Exception('key already present')
  // it worked
})

set.rem('name', function (err) {
  if (err) throw new Exception('a strange error occured')
  // it worked
})

```

## API

### ConsistentSet()

Create a new consistent set

### set#add(str, cb)

Add `str` to `set`. `cb` gets an error if `str` is already present or another
error occured.

### set#rem(str, cb)

Remove `str` from `set`. `cb` gets an error if something unexpected bad happens

### set#createStream()

Returns a new Duplex stream for replication with other `sets`s.

## How it works

Every `add` operations is checked with every known node, to see if `str` has
already been added but that hasn't been replicated yet.

Here node(0) is connected with node(1), node(2) and node(3)

When node(0) wants to store 'str' at t0, it adds it to it's own set and sends
the following packages to node(1-3)

```json
{
  "nodes-checked" : [0, 1, 2, 3], // don't ask nodes twice
  "ts" : t0,                      // for deciding who wins an a conflict
  "str" : "my-string"
}
```

node(1-3) then

* add 'str' with `ts = t0` to their set
* check the `nodes-checked` field to see if they're connected with other nodes
  that node(0) doesn't know about
  * If so, tell node(0) about that node and repeat the package sending to those
    new ones, with an updated `nodes-checked` field.
  * If not so, send an `accept:ts` message to all nodes

## License

(MIT)

Copyright (c) 2012 &lt;julian@juliangruber.com&gt;

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
