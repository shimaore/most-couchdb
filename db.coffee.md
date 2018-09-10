This is a minimalist CouchDB (HTTP) API
It provides exactly what this module needs, but no more.

    LRU = require 'lru-cache'
    {EventEmitter2} = require 'eventemitter2'

    http = require 'http'
    https = require 'https'

    lru_dispose = new EventEmitter2 newListener: false, verboseMemoryLeak: true

    options =
      max: 200
      dispose: (key) -> ev.emit key
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
          @limit = most.fromEvent @uri, lru_dispose
        else
          @cache = static_cache
          @limit = most.never()

        switch
          when uri.match /^http:/
            @agent = agent.agent http.globalAgent
          when uri.match /^https:/
            @agent = agent.agent https.globalAgent
          else
            @agent = agent

        return

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

      get: (_id,{rev} = {}) ->
        uri = new URL ec(_id), @uri
        uri.searchParams.set 'rev', rev if rev?
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

      query_changes: (map_function,{since}) ->
        changes_view map_function, @changes {live:true,include_docs:true,since}

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

The stream might end because the server disconnects or other errors.
In all cases we let it finish cleanly.

        s = ->
          uri.searchParams.set 'since', since
          fromEventSource new EventSource uri.toString()
          .continueWith ->
            console.error 'retry', uri.host, uri.pathname, since
            most.never()
          .recoverWith (error) ->
            console.error 'retry', (error.stack ? error), uri.host, uri.pathname, since
            most.never()

        stream = autoRestart(s)
          .map ({data}) -> data
          .map JSON.parse
          .tap ({seq}) -> since = seq if seq?
          .until @limit
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
    autoRestart = require './restart'
    view_stream = require './view-stream'
    changes_view = require './changes-view'
