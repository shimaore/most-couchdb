This is a minimalist CouchDB (HTTP) API
It provides exactly what this module needs, but no more.

    LRU = require 'lru-cache'
    {EventEmitter2} = require 'eventemitter2'

    http = require 'http'
    https = require 'https'

    dispose = new EventEmitter2 newListener: false, verboseMemoryLeak: true

    options =
      max: 200
      dispose: (key) -> dispose.emit key
      maxAge: 20*60*1000

    lru_cache = LRU options

    static_cache = new Map()

    class CouchDB

      constructor: (uri,use_lru) ->
        if uri.match /\/$/
          @uri = uri
        else
          @uri = "#{uri}/"

        if use_lru
          @cache = lru_cache
        else
          @cache = static_cache

        switch
          when uri.match /^http:/
            @agent = agent.agent http.globalAgent
          when uri.match /^https:/
            @agent = agent.agent https.globalAgent
          else
            @agent = agent

        return

      info: ->
        @agent
        .get @uri
        .accept 'json'
        .then ({body}) -> body

      destroy: ->
        @agent
        .delete @uri
        .accept 'json'
        .then ({body}) -> body

Insert a document in the database (document must have valid `_id` and `_rev` fields).

      put: (doc) ->
        {_id} = doc
        uri = new URL ec(_id), @uri
        @agent
        .put uri.toString()
        .type 'json'
        .accept 'json'
        .send doc
        .then ({body}) -> body

Get a document, optionally at a given revision.

      get: (_id,options = {}) ->
        uri = new URL ec(_id), @uri
        for own k,v of options when v?
          uri.searchParams.set k, v
        @agent
        .get uri.toString()
        .accept 'json'
        .then ({body}) -> body

Delete a document based on its `_id` and `_rev` fields.

      delete: ({_id,_rev}) ->
        uri = new URL ec(_id), @uri
        uri.searchParams.set 'rev', _rev if _rev?
        @agent
        .delete uri.toString()
        .accept 'json'
        .then ({body}) -> body

Uses a server-side view, returns a stream containing one event for each row.

      query: (app,view,params) ->
        view_stream @uri, app, view, params

Uses a wrapped client-side map function, returns a stream containing one event for each new row.
Please provide `map_function(emit)`, wrapping the actual `map` function.

      query_changes: (map_function,{since} = {}) ->
        options = {live:true,include_docs:true,since}
        source = @changes options
        changes_view map_function, source, options

Build a continuous `most.js` stream for changes.

      changes: (options = {}) ->
        {live,include_docs,since} = options
        live ?= true
        throw new Error 'Only live streaming is supported' unless live is true

        if @cache.has @uri
          return @cache.get @uri

        since ?= 'now'
        uri = new URL '_changes', @uri
        uri.searchParams.set 'feed', 'eventsource'
        uri.searchParams.set 'heartbeat', true
        uri.searchParams.set 'include_docs', include_docs ? false

        uri.searchParams.set 'since', since
        source = new EventSource uri.toString()
        at_end = =>
          debug 'at_end', @uri, since
          @cache.delete @uri
          return

EventSource will reconnect.

        stream = fromEventSource source, at_end
          .until most.fromEvent(@uri,dispose).take(1)
          .map ({data}) -> data
          .map JSON.parse
          .tap ({seq}) -> since = seq if seq?
          .multicast()

        @cache.set @uri, stream
        stream


    module.exports = CouchDB
    ec = encodeURIComponent
    most = require 'most'
    EventSource = require 'eventsource'
    {fromEventSource} = require 'most-w3msg'
    {URL} = require 'url'
    agent = require 'superagent'
    view_stream = require './view-stream'
    changes_view = require './changes-view'
    debug = (require 'debug') 'most-couchdb'
