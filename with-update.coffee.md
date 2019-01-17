    class CouchDBWithUpdate extends require './db'

      constructor: (args...) ->
        super args...
        @count =
          new: `0n`
          changed: `0n`
          unchanged: `0n`

      __update: (id,data,map) ->
        doc = await @get(id).catch -> {}
        if doc._rev?
          new_doc = Object.assign (map doc), data
          if not isDeepStrictEqual new_doc, doc
            await @put new_doc
            @count.changed++
          else
            @count.unchanged++
        else
          @count.new++
          await @put data
        return

`merge`: update the fields indicated by `data` in an existing document indicated by `id`
The update is only processed if the document would change.

      merge: (id,data) ->
        @__update id, data, (doc) -> Object.assign {}, doc

`update`: update a document
The update is only processed if the document would change.

      update: (data) ->
        @__update data._id, data, (doc) -> {_rev:doc._rev}

    module.exports = CouchDBWithUpdate
    {isDeepStrictEqual} = require 'util'
