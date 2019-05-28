    merge_map = (doc) -> Object.assign {}, doc
    update_map = (doc) -> {_rev:doc._rev}
    id_of = ({_id}) -> _id

    class CouchDBWithUpdate extends require './db'

      constructor: (args...) ->
        super args...
        @count =
          new: `0n`
          changed: `0n`
          unchanged: `0n`

Single-record merge / update
----------------------------

      __merge: (doc,data,map) ->
        if doc?._rev?
          new_doc = Object.assign (map doc), data
          if not isDeepStrictEqual new_doc, doc
            @count.changed++
            new_doc
          else
            @count.unchanged++
            null
        else
          @count.new++
          data

      __update: (id,data,map) ->
        doc = await @get(id).catch -> {}
        new_doc = @__merge doc,data,map
        if new_doc?
          await @put new_doc
        return

`merge`: update the fields indicated by `data` in an existing document indicated by `id`
The update is only processed if the document would change.

      merge: (id,data) ->
        @__update id, data, merge_map

`update`: replace a document with new content
The update is only processed if the document would change.

      update: (data) ->
        @__update (id_of data), data, update_map


Bulk update
-----------

      bulk_get: (docs) -> # docs is an Array of {id,rev}
        uri = new URL '_bulk_get', @uri+'/'
        {body} = await @agent
          .post uri.toString()
          .send {docs}
          .accept 'json'

        {results} = body
        docs = results.map ({docs}) -> docs[0].ok ? null

      bulk_docs: (docs) -> # docs is an Array of documents
        uri = new URL '_bulk_docs', @uri+'/'
        {body} = await @agent
          .post uri.toString()
          .send {docs}
          .accept 'json'
        body

      __bulk_update: (ids,new_docs,map) ->
        docs = await @bulk_get ids.map (id) -> {id}

        new_docs = docs.map (doc,i) =>
          data = new_docs[i]
          @__merge doc,data,map

        changed = new_docs.filter (doc) -> doc?

        if changed.length > 0
          results = await @bulk_docs changed
        else
          results = []

        results

      bulk_merge: (ids,changes) ->
        @__bulk_update ids, changes, merge_map

      bulk_update: (docs) ->
        @__bulk_update (docs.map id_of), docs, update_map

    module.exports = CouchDBWithUpdate
    {isDeepStrictEqual} = require 'util'
