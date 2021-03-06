# Seraph.js

A terse & familiar binding to the [Neo4j](http://neo4j.org/) REST API that is 
idiomatic to [node.js](http://nodejs.org/).

## Install

```
npm install seraph
```

## Quick Example

```javascript
var db = require("seraph")("http://localhost:7474");

db.save({ name: "Test-Man", age: 40 }, function(err, node) {
  if (err) throw err;
  console.log("Test-Man inserted.");

  db.delete(node, function(err) {
    if (err) throw err;
    console.log("Test-Man away!");
  });
});
```

## Documentation

<a name="seraph.db_list" />
### Initialization
* [seraph](#seraph) - initialize the seraph client

### Generic Operations

* [query](#query) - perform a cypher query and parse the results
* [rawQuery](#rawQuery) - perform a cypher query and return unparsed results

### API Communication Operations

* [operation](#operation) - create a representation of a REST API call
* [call](#call) - take an operation and call it
* [batch](#batch) - perform a series of atomic operations with one api call.

### Node Operations
* [save (node.save)](#node.save) - create or update a node
* [saveUnique (node.saveUnique)](#node.saveUnique) - save a node using an index 
  to enforce uniqueness
* [read (node.read)](#node.read) - read a node
* [find (node.find)](#node.find) - find a node using a predicate
* [delete (node.delete)](#node.delete) - delete a node
* [relate (node.relate)](#node.relate) - relate two nodes
* [relationships (node.relationships)](#node.relationships) - read the 
  relationships of a node
* [index (node.index)](#node.index) - add a node to an index
* [label (node.label)](#node.label) - add a label to a node
* [removeLabel (node.removeLabel)](#node.removeLabel) - remove a label from a 
  node
* [nodesWithLabel (node.nodesWithLabel)](#node.nodesWithLabel) - fetch all nodes
  with a label
* [readLabels (node.readLabels)](#node.readLabels) - read the labels of a node
  or all available labels.

### Relationship Operations
* [rel.create](#rel.create) - create a relationship
* [rel.createUnique](#rel.createUnique) - create a relationship using an index
  to enforce uniqueness
* [rel.update](#rel.update) - update the properties of a relationship
* [rel.read](#rel.read) - read a relationship
* [rel.delete](#rel.delete) - delete a relationship

### Index Operations
* [index.create](#index.create) - create an index
* [index.add](#index.add) - add a nodes/rels to an index
* [index.read](#index.read) - read nodes/rels from an index
* [index.remove](#index.remove) - remove nodes/rels from an index
* [index.delete](#index.delete) - delete an index
* [index.getOrSaveUnique](#index.getOrSaveUnique) - get or save a node using an 
  index for uniqueness
* [index.saveUniqueOrFail](#index.saveUniqueOrFail) - save a node using an index
  to enforce uniqueness

## Compatibility

Seraph 0.8.0 only works with Neo4j-2.0.0-RC1 and later.

## Testing

You can test Seraph simply by running `npm test`. It will spin up its own neo4j 
instance for testing. **Note** that the first time you run your tests (or change 
neo4j version), a new version of neo4j will need to be downloaded. That can,
of course, take a little time.

## Initialization
<a name="seraph" />
### seraph([server|options])

Creates and returns the Seraph instance.  If no parameters are given,
assumes the Neo4J REST API is running locally at the default location
`http://localhost:7474/db/data`.

__Arguments__

* `options` (default=`{ server: "http://localhost:7474", endpoint: "/db/data" }` - `server` is protocol and authority part of Neo4J REST API URI, and `endpoint` should be the path segment of the URI.
* `server` (string) - Short form to specify server parameter only. `"http://localhorse:4747"` is equivalent to `{ server: "http://localhorse:4747" }`.

__Example__

```javascript
// To http://localhost:7474/db/data
var dbLocal = require("seraph")();

// To http://example.com:53280/neo
var dbRemote = require("seraph")({ server: "http://example.com:53280",
                                   endpoint: "/neo" });

// Copy node#13 from remote server
dbRemote.read({ id: 13 }, function(err, node) {
  if (err) throw err;
  delete node.id; // copy instead of overwriting local node#13
  dbLocal.save(node, function(err, nodeL) {
    if (err) throw err;
    console.log("Copied remote node#13 to " +
                "local node#" + nodeL.id.toString() + ".");
  });
});
```

## Generic Operations

<a name="query" /><a name="rawQuery"/>
### query(query, [params,] callback), rawQuery(query, [params,] callback)

`rawQuery` performs a cypher query and returns the results directly from the
REST API.  
`query` performs a cypher query and map the columns and results together.

__Note__: if you're performing large queries it may be advantageous to use
`queryRaw`, since `query` attempts to infer whole nodes and relationships that
are returned (in order to transform them into a nicer format).

__Arguments__

* `query` - Cypher query as a format string.
* `params` (optional, default=`{}`). Replace `{key}` parts in query string.  See 
  cypher documentation for details. **note** that if you want to send a list of
  ids as a parameter, you should send them as an array, rather than a string
  representing them (`[2,3]` rather than `"2,3"`).
* `callback` - (err, result).  Result is an array of objects.

__Example__

Given database:

```javascript
{ name: 'Jon', age: 23, id: 1 }
{ name: 'Neil', age: 60, id: 2 }
{ name: 'Katie', age: 29, id: 3 }
// 1 --knows--> 2
// 1 --knows--> 3
```

Return all people Jon knows:

```javascript
var cypher = "START x = node({id}) "
           + "MATCH x -[r]-> n "
           + "RETURN n "
           + "ORDER BY n.name";

db.query(cypher, {id: 1}, function(err, result) {
  if (err) throw err;
  assert.deepEqual(result, [
    { name: 'Katie', age: 29, id: 3 },
    { name: 'Neil', age: 60, id: 2 }
  ]);
};

db.rawQuery(cypher, {id: 3}, function(err, result) {
  if (err) throw err;
  // result contains the raw response from neo4j's rest API. See
  // http://docs.neo4j.org/chunked/milestone/rest-api-cypher.html
  // for more info
})
```

---------------------------------------

<a name="operation" />
### operation(path, [method='get/post'], [data])

Create an operation object that will be passed to [call](#call). 

__Arguments__

* `path` - the path fragment of the request URL with no leading slash. 
* `method` (optional, default=`'GET'`|`'POST'`) - the HTTP method to use. When 
  `data` is an  object, `method` defaults to 'POST'. Otherwise, `method` 
  defaults to `GET`.
* `data` (optional) - an object to send to the server with the request.

__Example__

```javascript
var operation = db.operation('node/4285/properties', 'PUT', { name: 'Jon' });
db.call(operation, function(err) {
  if (!err) console.log('Set `name` to `Jon` on node 4285!')
});
```

---------------------------------------

<a name="call" />
### call(operation, callback)

Perform an HTTP request to the server.

If the body is some JSON, it is parsed and passed to the callback.If the status
code is not in the 200's, an error is passed to the callback. 

__Arguments__

* `operation` - an operation created by [operation](#operation) that specifies
  what to request from the server
* `callback` - function(err, result, response). `result` is the JSON parsed body
  from the server (otherwise empty). `response` is the response object from the
  request.

__Example__

```javascript
var operation = db.operation('node/4285/properties');
db.call(operation, function(err, properties) {
  if (err) throw err;

  // `properties` is an object containing the properties from node 4285
});
```

---------------------------------------

<a name="batch" />
### Batching/transactions - `batch([operations, callback])`

Batching provides a method of performing a series of operations atomically. You
could also call it a transaction. It has the added benefit of being performed
all in a single call to the neo4j api, which theoretically should result in 
improved performance when performing more than one operation at the same time.

When you create a batch, you're given a new `seraph` object to use. All calls to
this object will be added to the batch. Note that once a batch is committed, you
should no longer use this object.

* [How do I use it?](#how-do-i-use-it)
* [What happens to my callbacks?](#what-happens-to-my-callbacks)
* [Can I reference newly created nodes?](#can-i-reference-newly-created-nodes)
* [I didn't use any callbacks. How can I find my results when the batch is done?](#i-didnt-use-any-callbacks-how-can-i-find-my-results-when-the-batch-is-done)
* [What happens if one of the operations fails?](#what-happens-if-one-of-the-operations-fails)
* [Can I nest batches?](#can-i-nest-batches)
* [How can I tell if this `db` object is a batch operation?](#how-can-i-tell-if-this-db-object-is-a-batch-operation)

#### How do I use it?

There's two ways. You can do the whole thing asynchronously, and commit the 
transaction whenever you want, or you can do it synchronously, and have the 
transaction committed for you as soon as your function is finished running.
Here's a couple of examples of performing the same operations with batch 
synchronously and asynchronously:

##### Asynchronously

```javascript
var txn = db.batch();

txn.save({ title: 'Kaikki Askeleet' });
txn.save({ title: 'Sinä Nukut Siinä' });
txn.save({ title: 'Pohjanmaa' });

txn.commit(function(err, results) {
  /* results -> [{ id: 1, title: 'Kaikki Askeleet' },
                 { id: 2, title: 'Sinä Nukut Siinä' },
                 { id: 3, title: 'Pohjanmaa' }] */
});
```

##### Synchronously

**Note** - it's only the creation of operations that is synchronous. The actual
API call is asynchronous still, of course.

```javascript
db.batch(function(txn) {
  txn.save({ title: 'Kaikki Askeleet' });
  txn.save({ title: 'Sinä Nukut Siinä' });
  txn.save({ title: 'Pohjanmaa' });
}, function(err, results) {
  /* results -> [{ id: 1, title: 'Kaikki Askeleet' },
                 { id: 2, title: 'Sinä Nukut Siinä' },
                 { id: 3, title: 'Pohjanmaa' }] */
});
```

#### What happens to my callbacks?

You can still pass callbacks to operations on a batch transaction. They will
perform as you expect, but they will not be called until after the batch has
been committed. Here's an example of using callbacks as normal:

```javascript
var txn = db.batch();

txn.save({ title: 'Marmoritaivas' }, function(err, node) {
  // this code is not reached until `txn.commit` is called
  // node -> { id: 1, title: 'Marmoritaivas' }
});

txn.commit();
```

#### Can I reference newly created nodes?

Yes! Calling, for example, `node.save` will synchronously return a special object
which you can use to refer to that newly created node within the batch.

For example, this is perfectly valid in the context of a batch transaction:

```javascript
var txn = db.batch();

var singer = txn.save({name: 'Johanna Kurkela'});
var album = txn.save({title: 'Kauriinsilmät', year: 2008});
var performance = txn.relate(singer, 'performs_on', album, {role: 'Primary Artist'});
txn.rel.index('performances', performance, 'year', '2008');

txn.commit(function(err, results) {});
```

#### I didn't use any callbacks. How can I find my results when the batch is done?

Each function you call on the batch object will return a special object that you
can use to refer to that call's results once that batch is finished (in 
addition to the intra-batch referencing feature mentioned above). The best
way to demonstrate this is by example:

```javascript
var txn = db.batch();

var album = txn.save({title: 'Hetki Hiljaa'});
var songs = txn.save([
  { title: 'Olen Sinussa', length: 248 },
  { title: 'Juurrun Tähän Ikävään', length: 271 }
]);
// note we can also use `songs` to reference the node that will be created
txn.relate(album, 'has_song', songs[0], { trackNumber: 1 });
txn.relate(album, 'has_song', songs[1], { trackNumber: 3 });

txn.commit(function(err, results) {
  var album = results[album]; // album -> { title: 'Hetki Hiljaa', id: 1 }
  var tracks = results[songs];
  /* tracks -> [{ title: 'Olen Sinussa', length: 248, id: 2 },
                { title: 'Juurrun Tähän Ikävään', length: 271, id: 3}] */
});
```

#### What happens if one of the operations fails?

Then no changes are made. Neo4j's batch transactions are atomic, so if one
operation fails, then no changes to the database are made. Neo4j's own
documentation has the following to say: 
> This service is transactional. If any of the operations performed fails 
> (returns a non-2xx HTTP status code), the transaction will be rolled back and
> all changes will be undone.

#### Can I nest batches?

No, as of now we don't support nesting batches as it tends to confuse the
intra-batch referencing functionality. To enforce this, you'll find that the
seraph-like object returned by `db.batch()` has no `.batch` function itself.

#### How can I tell if this `db` object is a batch operation?

Like so:

```javascript
// db.isBatch -> undefined
var txn = db.batch();
// txn.isBatch -> true
if (txn.isBatch) // Woo! I'm in a batch.
```

-------------

## Node Operations

<a name="node.save" />
### save(object, [key, value,] callback)
*Aliases: __node.save__*

Create or update a node. If `object` has an id property, the node with that id
is updated. Otherwise, a new node is created. Returns the newly created/updated
node to the callback.

__Arguments__

* `node` - an object to create or update
* `key`, `value` (optional) - a property key and a value to update it with. This
  allows you to only update a single property of the node, without touching any
  others. If `key` is specified, `value` must also be. 
* `callback` - function(err, node). `node` is the newly saved or updated node. If
  a create was performed, `node` will now have an id property. The returned 
  object is not the same reference as the passed object (the passed object will
  never be altered).

__Example__

```javascript
// Create a node
db.save({ name: 'Jon', age: 22, likes: 'Beer' }, function(err, node) {
  console.log(node); // -> { name: 'Jon', age: 22, likes: 'Beer', id: 1 }
  
  // Update it
  delete node.likes;
  node.age++;
  db.save(node, function(err, node) {
    console.log(node); // -> { name: 'Jon', age: 23, id: 1 }
  })
})
```

---------------------------------------

<a name="node.saveUnique" />
### node.saveUnique(node, index, key, value, [returnExistingOnConflict = false,] callback)

Save a node, using an index to enforce uniqueness.

See also [node.index.saveUniqueOrFail](#index.saveUniqueOrFail) &
[node.index.getOrSaveUnique](#index.getOrSaveUnique).

__Arguments__

* `node` - an object to create or update
* `index` - the index in which `key` and `value` are relevant
* `key` - the key under which to index this node and enforce uniqueness
* `value` - the value under which to index this node and enforce uniqueness
* `returnExistingOnConflict` (optional, default=`false`) - what to do when there is
  a conflict (when the index you specified already refers to a node). If set to 
  `true`, the node that the index currently refers to is returned.  Otherwise, 
  an error is return indicating that there was a conflict (you can check this by 
  testing `err.statusCode == 409`.
* `callback` - function(err, node) - `node` is the newly created node or the node 
  that was in the specified index, depending on `returnExistingOnConflict`.

__Example__

```javascript
db.saveUnique({name: 'jon'}, 'people', 'name', 'jon', function(err, node) {
  //node = { name: 'jon', id: 1 }
  db.saveUnique({age: 24}, 'people', 'name', 'jon', function(err, node) {
    //err.statusCode == 409, because there was already a node indexed under
    //people(name="jon").
  });
  
  db.saveUnique({location:'Bergen'}, 'people', 'name', 'jon', true, function(err, node) {
    // node = {name: 'jon', id: 1}
    // because there was node already indexed under people(name="jon") and we 
    // specified that we wanted that node returned on the event of a conflict.
  });
});
```

---------------------------------------

<a name="node.read" />
### read(id|object, [property,] callback)
*Aliases: __node.read__*

Read a node.

**Note**: If the node doesn't exist, Neo4j will return an exception. You can 
check if this is indicating that your node doesn't exist because
`err.statusCode` will equal `404`. This is inconsistent with behaviour of
[node.index.read](#index.read), but it is justified because the Neo4j REST api
behaviour is inconsistent in this way as well. 

__Arguments__

* `id | object` - either the id of the node to read, or an object containing an id
property of the node to read.
* `property` (optional) - the name of the property to read. if this is specified,
  only the value of this property on the object is returned.
* `callback` - function(err, node). `node` is an object containing the properties
of the node with the given id.

__Example__

```javascript
db.save({ make: 'Citroen', model: 'DS4' }, function(err, node) {
  db.read(node.id, function(err, node) {
    console.log(node) // -> { make: 'Citroen', model: 'DS4', id: 1 }
  })
})
```

---------------------------------------

<a name="node.delete" />
### delete(id|object, [force | property], [callback])
*Aliases: __node.delete__*

Delete a node.

__Arguments__

* `id | object` - either the id of the node to delete, or an object containing an id
property of the node to delete.
* `force` (optional - default = false) -  if truthy, will delete all the node's 
  relations prior to deleting the node.
* `property` (optional) - if specified, delete only the property with this name
  on the object. **note that you can either specify `property` or `force`, not
  both, as force is meaningless when deleting a property**
* `callback` - function(err). if `err` is falsy, the node has been deleted.

__Example__

```
db.save({ name: 'Jon' }, function(err, node) {
  db.delete(node, function(err) {
    if (!err) console.log('Jon has been deleted!');
  })
})
```

---------------------------------------

<a name="node.find" />
### find(predicate, [any, [start,]] callback)
*Aliases: __node.find__*

Perform a query based on a predicate. The predicate is translated to a
cypher query.

__Arguments__

* `predicate` - Partially defined object.  Will return elements which match
  the defined attributes of predicate.
* `any` (optional, default=`false`) - If true, elements need only match on one 
  attribute. If false, elements must match on all attributes.
* `start` (optional, default=`'node(*)'`) - The scope of the search. For alternate
  values, check the [neo4j docs on the cypher START command](http://docs.neo4j.org/chunked/stable/query-start.html).
* `callback` - function(err, results) - `results` is an array of the resulting
  nodes.

__Example__

Given database content:

```javascript
{ name: 'Jon'    , age: 23, australian: true  }
{ name: 'Neil'   , age: 60, australian: true  }
{ name: 'Belinda', age: 26, australian: false }
{ name: 'Katie'  , age: 29, australian: true  }
```

Retrieve all australians:

```javascript
var predicate = { australian: true };
var people = db.find(predicate, function (err, objs) {
    if (err) throw err;
    assert.equals(3, people.length);
};
```

---------------------------------------

<a name="node.relationships" />
### relationships(id|object, [direction, [type,]] callback)
**Aliases: __node.relationships__*

Read the relationships involving the specified node.

__Arguments__

* `id | object` - either the id of a node, or an object containing an id property of
  a node.
* `direction` ('all'|'in'|'out') (optional unless `type` is passed, 
  default=`'all'`) - the direction of relationships to read. 
* `type` (optional, default=`''` (match all relationships)) - the relationship
  type to find
* `callback` - function(err, relationships) - `relationships` is an array of the
  matching relationships

__Example__

```javascript
db.relationships(452, 'out', 'knows', function(err, relationships) {
  // relationships = all outgoing `knows` relationships from node 452
})
```

---------------------------------------

<a name="node.label" />
### label(id|object(s), label(s), [replace,] callback)
*Aliases: __node.label__*

Add a label to a node.

__Arguments__

* `id|object(s)` - either the id of the node to label, or an object containing an
  id property of the node to label. can be an array of objects/ids.
* `label(s)` - the label(s) to apply. can be an array of labels.
* `replace` (optional) - if set to true, this label will replace any previous 
  labels.
* `callback` - function(err). if err is falsy, the operation succeeded.

__Example__

```javascript
db.save({ make: 'Citroen', model: 'DS4' }, function(err, node) {
  db.label(node, ['Car', 'Hatchback'], function(err) {
    // `node` is now labelled with "Car" and "Hatchback"!
  });
})
```

---------------------------------------

<a name="node.removeLabel" />
### removeLabel(id|object(s), label, callback)
*Aliases: __node.removeLabel__*

Remove a label from a node.

__Arguments__

* `id|object(s)` - either the id of the node to delabel, or an object containing 
  an id property of the node to delabel. can be an array of objects/ids.
* `label` - the label to remove. cannot be an array.
* `callback` - function(err). if err is falsy, the operation succeeded.

__Example__

```javascript
db.save({ make: 'Citroen', model: 'DS4' }, function(err, node) {
  db.label(node, ['Car', 'Hatchback'], function(err) {
    // `node` is now labelled with "Car" and "Hatchback"!
    db.removeLabel(node, 'Hatchback', function(err) {
      // `node` is now only labelled with "Car".
    });
  });
})
```

---------------------------------------

<a name="node.nodesWithLabel" />
### nodesWithLabel(label, callback)
*Aliases: __node.nodesWithLabel__*

Fetch all of the nodes that are labelled with a specific label.

__Arguments__

* `label` - the label.
* `callback` - function(err, results). results is always an array (assuming no
  error), containing the nodes that were labelled with `label`. if no nodes were
  labelled with `label`, `results` is an empty array.

__Example__

```javascript
db.save({ make: 'Citroen', model: 'DS4' }, function(err, node) {
  db.label(node, ['Car', 'Hatchback'], function(err) {
    db.nodesWithLabel('Car', function(err, results) {
      results[0].model // -> 'DS4'
    });
  });
})
```

---------------------------------------

<a name="node.readLabels" />
### readLabels([node(s),] callback)
*Aliases: __node.readLabels__*

Read the labels of a node, or all labels in the database.

__Arguments__

* `node(s)` (optional) - the node to return the labels of. if not specified, every
  label in the database is returned. can be an array of nodes.
* `callback` - function(err, labels). labels is an array of labels.

__Example__

```javascript
db.save({ make: 'Citroen', model: 'DS4' }, function(err, node) {
  db.label(node, ['Car', 'Hatchback'], function(err) {
    db.readLabels(node, function(err, labels) {
      //labels -> ['Car', 'Hatchback']
    });
  });
})
```

---------------------------------------

## Relationship Operations

<a name="rel.create" />
<a name="node.relate" />
### rel.create(firstId|firstObj, type, secondId|secondobj, [properties], callback)
*Aliases: __relate__, __node.relate__*

Create a relationship between two nodes.

__Arguments__

* `firstId | firstObject` - id of the start node or an object with an id property
  for the start node
* `type` - the name of the relationship
* `secondId | secondObject` - id of the end node or an object with an id property
  for the end node
* `properties` (optional, default=`{}`) - properties of the relationship
* `callback` - function(err, relationship) - `relationship` is the newly created
  relationship

__Example__

```javascript
db.relate(1, 'knows', 2, { for: '2 months' }, function(err, relationship) {
  assert.deepEqual(relationship, {
    start: 1,
    end: 2,
    type: 'knows',
    properties: { for: '2 months' },
    id: 1
  });
});
```

---------------------------------------

<a name="rel.createUnique" />
### rel.createUnique(firstId|firstObj, type, secondId|secondObj, [properties,] index, key, value, [returnExistingOnConflict = false,] callback)

Create a relationship between two nodes, using an index to enforce uniqueness.

See also [rel.index.saveUniqueOrFail](#index.saveUniqueOrFail) &
[rel.index.getOrSaveUnique](#index.getOrSaveUnique).

__Arguments__

* `firstId | firstObject` - id of the start node or an object with an id property
  for the start node
* `type` - the name of the relationship
* `secondId | secondObject` - id of the end node or an object with an id property
  for the end node
* `properties` (optional, default=`{}`) - properties of the relationship
* `index` - the index in which `key` and `value` are relevant
* `key` - the key under which to index this relationship and enforce uniqueness
* `value` - the value under which to index this relationship and enforce uniqueness
* `returnExistingOnConflict` (optional, default=`false`) - what to do when there is
  a conflict (when the index you specified already refers to a relationship). If
  set to `true`, the relationship that the index currently refers to is returned.
  Otherwise, an error is return indicating that there was a conflict (you can 
  check this by testing `err.statusCode == 409`.
* `callback` - function(err, relationship) - `relationship` is the newly created
  relationship or the relationship that was in the specified index, depending
  on `returnExistingOnConflict`

__Example__

```javascript
db.rel.createUnique(1, 'knows', 2, 'friendships', 'type', 'super', function(err, rel) {
  // rel = {start: 1, end: 2, type: 'knows', properties: {}, id = 1}
  db.rel.createUnique(1, 'knows', 2, 'friendships', 'type', 'super', function(err, node) {
    //err.statusCode == 409, because there was already a relationship indexed under
    //friendships(type="super").
  });
  
  db.rel.createUnique(1, 'knows', 2, 'friendships', 'type', 'super', true, function(err, node) {
    // rel = {start: 1, end: 2, type: 'knows', properties: {}, id = 1}
    // no new index was created, the first one was returned - because there was
    // already a relationship indexed under friendships(type="super") and we
    // specified that on the event of a conflict we wanted the indexed rel to 
    // be returned
  });
});
```

---------------------------------------

<a name="rel.update" />
### rel.update(relationship, [key, value,] callback)

Update the properties of a relationship. __Note__ that you cannot use this
method to update the base properties of the relationship (start, end, type) -
in order to do that you'll need to delete the old relationship and create a new
one.

__Arguments__

* `relationship` - the relationship object with some changed properties
* `key`, `value` (optional) - if a key and value is specified, only the property with
  that key will be updated. the rest of the object will not be touched.
* `callback` - function(err). if err is falsy, the update succeeded.

__Example__

```javascript
var props = { for: '2 months', location: 'Bergen' };
db.rel.create(1, 'knows', 2, props, function(err, relationship) {
  delete relationship.properties.location;
  relationship.properties.for = '3 months';
  db.rel.update(relationship, function(err) {
    // properties on this relationship in the database are now equal to
    // { for: '3 months' }
  });
});
```

---------------------------------------

<a name="rel.read" />
### rel.read(object|id, callback)

Read a relationship.

__Arguments__

* `object | id` - the id of the relationship to read or an object with an id
  property of the relationship to read.
* `callback` - function(err, relationship). `relationship` is an object
  representing the read relationship.

__Example__

```javascript
db.rel.create(1, 'knows', 2, { for: '2 months' }, function(err, newRelationship) {
  db.rel.read(newRelationship.id, function(err, readRelationship) {
    assert.deepEqual(newRelationship, readRelationship);
    assert.deepEqual(readRelationship, {
      start: 1,
      end: 2,
      type: 'knows',
      id: 1,
      properties: { for: '2 months' }
    });
  });
});
```

---------------------------------------

<a name="rel.delete" />
### rel.delete(object|id, [callback])

Delete a relationship.

__Arguments__

* `object | id` - the id of the relationship to delete or an object with an id
  property of the relationship to delete.
* `callback` - function(err). If `err` is falsy, the relationship has been
  deleted.

__Example__

```javascript
db.rel.create(1, 'knows', 2, { for: '2 months' }, function(err, rel) {
  db.rel.delete(rel.id, function(err) {
    if (!err) console.log("Relationship was deleted");
  });
});
```

## Index Operations

<a name="index.create" />
### node.index.create(name, [config,] callback)
### rel.index.create(name, [config,] callback)

Create a new index. If you're using the default index configuration, this
method is not necessary - you can just start using the index with
[index.add](#index.add) as if it already existed.

__NOTE for index functions:__ there are two different types on index in neo4j - 
__node__ indexes and __relationship__ indexes. When you're working with __node__
indexes, you use the functions on `node.index`. Similarly, when you're working
on __relationship__ indexes you use the functions on `rel.index`. Most of the
functions on both of these are identical (excluding the uniqueness functions),
but one acts upon node indexes, and the other upon relationship indexes.

__Arguments__

* `name` - the name of the index that is being created
* `config` (optional, default=`{}`) - the configuration of the index. See the [neo4j docs](http://docs.neo4j.org/chunked/milestone/rest-api-indexes.html#rest-api-create-node-index-with-configuration)
  for more information.
* `callback` - function(err). If `err` is falsy, the index has been created.

__Example__

```javascript
var indexConfig = { type: 'fulltext', provider: 'lucene' };
db.node.index.create('a_fulltext_index', indexConfig, function(err) {
  if (!err) console.log('a fulltext index has been created!');
});
```

---------------------------------------

<a name="index.add" />
<a name="node.index" />
### node.index.add(indexName, id|object, key, value, callback);
### rel.index.add(indexName, id|object, key, value, callback);
*`node.index.add` is aliased as __node.index__ & __index__*

Add a node/relationship to an index.

__NOTE for index functions:__ there are two different types on index in neo4j - 
__node__ indexes and __relationship__ indexes. When you're working with __node__
indexes, you use the functions on `node.index`. Similarly, when you're working
on __relationship__ indexes you use the functions on `rel.index`. Most of the
functions on both of these are identical (excluding the uniqueness functions),
but one acts upon node indexes, and the other upon relationship indexes.

__Arguments__

* `indexName` - the name of the index to add the node/relationship to.
* `id | object` - the id of the node/relationship to add to the index or an object 
  with an id property of the node/relationship to add to the index.
* `key` - the key to index the node/relationship with
* `value` - the value to index the node/relationship with
* `callback` - function(err). If `err` is falsy, the node/relationship has 
  been indexed.

__Example__

```javascript
db.save({ name: 'Jon', }, function(err, node) {
  db.index('people', node, 'name', node.name, function(err) {
    if (!err) console.log('Jon has been indexed!');
  });
});
```

---------------------------------------

<a name="index.read" />
### node.index.read(indexName, key, value, callback);
### rel.index.read(indexName, key, value, callback);

Read the object(s) from an index that match a key-value pair. See also
[index.readAsList](#index.readAsList).

__NOTE for index functions:__ there are two different types on index in neo4j - 
__node__ indexes and __relationship__ indexes. When you're working with __node__
indexes, you use the functions on `node.index`. Similarly, when you're working
on __relationship__ indexes you use the functions on `rel.index`. Most of the
functions on both of these are identical (excluding the uniqueness functions),
but one acts upon node indexes, and the other upon relationship indexes.

__Arguments__

* `indexName` - the index to read from
* `key` - the key to match
* `value` - the value to match
* `callback` - function(err, results). `results` is a node or relationship object
  (or an array of them if there was more than one) that matched the given 
  key-value pair in the given index. If nothing matched, `results === false`.
  [index.readAsList](#index.readAsList) is similar, but always gives `results` as
  an array, with zero, one or more elements.

__Example__

```javascript
db.rel.index.read('friendships', 'location', 'Norway', function(err, rels) {
  // `rels` is an array of all relationships indexed in the `friendships`
  // index, with a value `Norway` for the key `location`.
});
```

---------------------------------------

<a name="index.readAsList" />
### node.index.readAsList(indexName, key, value, callback);
### rel.index.readAsList(indexName, key, value, callback);

Read the object(s) from an index that match a key-value pair. See also
[index.read](#index.read).

__NOTE for index functions:__ there are two different types on index in neo4j - 
__node__ indexes and __relationship__ indexes. When you're working with __node__
indexes, you use the functions on `node.index`. Similarly, when you're working
on __relationship__ indexes you use the functions on `rel.index`. Most of the
functions on both of these are identical (excluding the uniqueness functions),
but one acts upon node indexes, and the other upon relationship indexes.

__Arguments__

* `indexName` - the index to read from
* `key` - the key to match
* `value` - the value to match
* `callback` - function(err, results). `results` is an array of node or
  relationship objects that matched the given key-value pair in the given index.
  [index.read](#index.read) is similar, but gives `results` as `false`, an object
  or an array of objects depending on the number of hits.

__Example__

```javascript
db.rel.index.readAsList('friendships', 'location', 'Norway', function(err, rels) {
  // `rels` is an array of all relationships indexed in the `friendships`
  // index, with a value `Norway` for the key `location`.
});
```

---------------------------------------

<a name="index.remove" />
### node.index.remove(indexName, id|object, [key, [value,]] callback);
### rel.index.remove(indexName, id|object, [key, [value,]] callback);

Remove a node/relationship from an index. 

__NOTE for index functions:__ there are two different types on index in neo4j - 
__node__ indexes and __relationship__ indexes. When you're working with __node__
indexes, you use the functions on `node.index`. Similarly, when you're working
on __relationship__ indexes you use the functions on `rel.index`. Most of the
functions on both of these are identical (excluding the uniqueness functions),
but one acts upon node indexes, and the other upon relationship indexes.

__Arguments__

* `indexName` - the index to remove the node/relationship from.
* `id | object` - the id of the node/relationship to remove from the index or an 
  object with an id property of the node/relationship to remove from the index.
* `key` (optional) - the key from which to remove the node/relationship. If none
  is specified, every reference to the node/relationship is deleted from the
  index.
* `value` (optional) - the value from which to remove the node/relationship. If
  none is specified, every reference to the node/relationship is deleted for the
  given key.
* `callback` - function(err). If `err` is falsy, the specified references have
  been removed.

__Example__

```javascript
db.node.index.remove('people', 6821, function(err) {
  if (!err) console.log("Every reference of node 6821 has been removed from the people index");
});

db.rel.index.remove('friendships', 351, 'in', 'Australia', function(err) {
  if (!err) console.log("Relationship 351 is no longer indexed as a friendship in Australia");
})
```

---------------------------------------

<a name="index.delete" />
### node.index.delete(name, callback);
### rel.index.delete(name, callback);

Delete an index.

__NOTE for index functions:__ there are two different types on index in neo4j - 
__node__ indexes and __relationship__ indexes. When you're working with __node__
indexes, you use the functions on `node.index`. Similarly, when you're working
on __relationship__ indexes you use the functions on `rel.index`. Most of the
functions on both of these are identical (excluding the uniqueness functions),
but one acts upon node indexes, and the other upon relationship indexes.

__Arguments__

* `name` - the name of the index to delete
* `callback` - function(err). if `err` is falsy, the index has been deleted.

__Example__

```javascript
db.rel.index.delete('friendships', function(err) {
  if (!err) console.log('The `friendships` index has been deleted');
})
```

---------------------------------------

<a name="index.getOrSaveUnique" />
### node.index.getOrSaveUnique(node, index, key, value, callback);
### rel.index.getOrSaveUnique(startNode, relName, endNode, [properties,] index, key, value, callback);

Save a node or relationship, using an index to enforce uniqueness. If there is
already a node or relationship saved under the specified `key` and `value` in 
the specified `index`, that node or relationship will be returned.

Note that you cannot use this function to update nodes.

__NOTE for index functions:__ there are two different types on index in neo4j - 
__node__ indexes and __relationship__ indexes. When you're working with __node__
indexes, you use the functions on `node.index`. Similarly, when you're working
on __relationship__ indexes you use the functions on `rel.index`. Most of the
functions on both of these are identical (excluding the uniqueness functions),
but one acts upon node indexes, and the other upon relationship indexes.

__Arguments (node)__

* `node` - the node to save
* `index` - the name of the index in which `key` and `value` are relevant
* `key` - the key to check or store under
* `value` - the value to check or store under
* `callback` - function(err, node) - returns your saved node, or the node that
  was referenced by the specified `key` and `value` if one already existed.

__Arguments (relationship)__ 
* `startNode` - the start point of the relationship (object containing id or id)
* `relName` - the name of the relationship to create
* `endNode` - the end point of the relationship (object containing id or id)
* `properties` (optional) - an object containing properties to store on the
  created relationship.
* `index` - the name of the index in which `key` and `value` are relevant
* `key` - the key to check or store under
* `value` - the value to check or store under
* `callback` - function(err, rel) - returns your created relationship, or the 
  relationship that was referenced by the specified `key` and `value` if one 
  already existed.

__Example__

```javascript
var tag = { name: 'finnish' };
db.node.index.getOrSaveUnique(tag, 'tags', 'name', tag.name, function(err, tag) {
  // tag == { id: 1, name: 'finnish' }

  // save another new object with the same properties
  db.node.index.getOrSaveUnique({name: 'finnish'}, 'tags', 'name', 'finnish', function(err, newTag) {
    // newTag == { id: 1, name: 'finnish' }
    // no save was performed because there was already an object at that index
  });
});
```

---------------------------------------

<a name="index.saveUniqueOrFail" />
### node.index.saveUniqueOrFail(node, index, key, value, callback);
### rel.index.saveUniqueOrFail(startNode, relName, endNode, [properties,] index, key, value, callback);

Save a node or relationship, using an index to enforce uniqueness. If there is
already a node or relationship saved under the specified `key` and `value` in 
the specified `index`, an error is returned indicating that there as a conflict.
You can check if the result was a conflict by checking if 
`err.statusCode == 409`.

__NOTE for index functions:__ there are two different types on index in neo4j - 
__node__ indexes and __relationship__ indexes. When you're working with __node__
indexes, you use the functions on `node.index`. Similarly, when you're working
on __relationship__ indexes you use the functions on `rel.index`. Most of the
functions on both of these are identical (excluding the uniqueness functions),
but one acts upon node indexes, and the other upon relationship indexes.

__Arguments (node)__

* `node` - the node to save
* `index` - the name of the index in which `key` and `value` are relevant
* `key` - the key to check or store under
* `value` - the value to check or store under
* `callback` - function(err, node) - returns your created node, or an err with 
  `statusCode == 409` if a node already existed at that index

__Arguments (relationship)__ 
* `startNode` - the start point of the relationship (object containing id or id)
* `relName` - the name of the relationship to create
* `endNode` - the end point of the relationship (object containing id or id)
* `properties` (optional) - an object containing properties to store on the
  created relationship.
* `index` - the name of the index in which `key` and `value` are relevant
* `key` - the key to check or store under
* `value` - the value to check or store under
* `callback` - function(err, rel) - returns your created relationship, or an 
  err with `statusCode == 409` if a relationship already existed at that index

__Example__

```javascript
var tag = { name: 'finnish' };
db.node.index.saveUniqueOrFail(tag, 'tags', 'name', tag.name, function(err, tag) {
  // tag == { id: 1, name: 'finnish' }

  // save another new object with the same properties
  db.node.index.saveUniqueOrFail({name: 'finnish'}, 'tags', 'name', 'finnish', function(err, newTag) {
    // newTag == undefined
    // err.statusCode == 409 (conflict)
    // an error was thrown because there was already a node at that index.
  });
});
```

---------------------------------------

Development of Seraph is lovingly sponsored by 
[BRIK Tekonologier AS](http://www.github.com/brikteknologier) in Bergen, Norway.

<img src="http://i.imgur.com/9JjcBcx.jpg" width="800"/>
