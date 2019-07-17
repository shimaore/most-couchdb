most-couchdb: Streaming CouchDB
===============================

Synopsis
--------

```
const CouchDB = require('most-couchdb')
const db = new CouchDB(uri)

await db.get(id)
await db.put(doc)

// Provides async iterators for queries (Mango and legacy)
var S = db.queryAsyncIterable(design, view, {include_docs:true})

for await (rec of S) {
  {doc} = rec
  …
}
// also available as Object Stream and `most` stream.

// `most` (non-blocking) stream for changes
db
.changes({include_docs:true})
.observe( function ({doc}) {
  …
})
```
with `update` and `merge`:

```
const CouchDB = require('most-couchdb/with-update')
const db = new CouchDB(uri)

await db.update(doc)
await db.merge(id,values)
```


`db.info()`
-----------

Returns information about the database.

`db.create()`
-------------

Creates the database.

`db.destroy()`
--------------

Deletes the datbase.

`db.get(id)`
------------

Retrieves the document.

`db.get(id,{rev})`
------------------

Retrieves the document at the given revision.

`db.put(doc)`
-------------

Puts the document (will fail on conflict).

`db.has(id)`
------------

Returns true if the database contains the document, false otherwise.

`db.has(id,{rev})

`db.delete(doc)`
----------------

Removes the document. (Must at least provide `_id` and `_rev`.)

`db.findAsyncIterable(params)`
------------------------------

Runs a Mango query (CouchDB 2+), returns an async iterator.

`db.findStream(params)`
-----------------------

Runs a Mango query (CouchDB 2+), returns an object (blocking) Stream.

`db.find(params)`
-----------------

Runs a Mango query (CouchDB 2+), returns a `most` (non-blocking) stream.

`db.createIndex(params)`
------------------------

Create a Mango index.

`db.queryAsyncIterable(design,view,params)`
----------------------------------------

Runs a design-document's view, returns an async iterator.

`db.queryStream(design,view,params)`
----------------------------------------

Runs a design-document's view, returns an object (blocking) Stream.

`db.query(design,view,params)`
------------------------------

Runs a design-document's view, returns a `most` (non-blocking) stream.

`db.query_changes(map_function)`
--------------------------------

Monitors changes on the database, returns a `most` (non-blocking) stream of documents through the map function.

`db.changes()`
--------------

Monitor changes on the database, returns a `most` (non-blocking) stream of changes.

`db.changes(options)`
---------------------

Monitor changes on the database, returns a `most` (non-blocking) stream of changes.

`db.getAttachment(id,name)`
---------------------------

Returns a buffer containing the attachment.

`db.putAttachment(id,name,rev,buf,type)`
----------------------------------------

Save a buffer as attachment.

`db.deleteAttachment(id,name,rev)`
----------------------------------

Remove an attachment.

`db.merge(id,changes)`
---------------------

Update the top fields in the document with the provided values.

Avalaible in `most-couchb/with-update`.

`db.update(data)`
----------------

Replace the document, but only if changes occurred. (`_id` is required.)

Avalaible in `most-couchb/with-update`.

`db.bulk_get(docs)`
-------------------

Retrieves documents. `docs` is an Array of `{id,rev}`.

Avalaible in `most-couchb/with-update`.

`db.bulk_docs(docs)`
--------------------

Update documents. `docs` is an Array of documents.

Avalaible in `most-couchb/with-update`.

`db.bulk_merge(ids,changes)`
----------------------------

Update the top fields in the documents specified with `ids` using the values specified in the matching `changes`.

Avalaible in `most-couchb/with-update`.

`db.bulk_update(datas)`
----------------------------

Replaces the documents with the new content, but only if changes occurred.

Avalaible in `most-couchb/with-update`.
