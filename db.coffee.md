This is a minimalist CouchDB (HTTP) API
It provides exactly what this module needs, but no more.

    LRU = require 'lru-cache'

    http = require 'http'
    https = require 'https'

    lru_options =
      max: 200
      dispose: (key,{source}) ->
        debug 'lru_cache: dispose', key
        source.close()
      maxAge: 20*60*1000

    lru_cache = new LRU lru_options
    lru_cache.delete = lru_cache.del

    static_cache = new Map()

    {HttpsAgent} = Agent = require 'agentkeepalive'
    http_agent = new Agent
      maxSockets: 100           # per host
      maxFreeSockets: 10        # per host
      timeout: 60000            # active socket keepalive
      freeSocketTimeout: 30000  # free socket keepalive
    https_agent = new HttpsAgent
      maxSockets: 100           # per host
      maxFreeSockets: 10        # per host
      timeout: 60000            # active socket keepalive
      freeSocketTimeout: 30000  # free socket keepalive

    class CouchDB

      constructor: (uri,use_lru) ->
        if uri.match /\/$/
          @uri = uri.slice 0, -1
        else
          @uri = uri

        if use_lru
          @cache = lru_cache
        else
          @cache = static_cache

        switch
          when uri.match /^http:/
            @agent = Request.agent http_agent
          when uri.match /^https:/
            @agent = Request.agent https_agent
          else
            @agent = Request
        return

      info: ->
        @agent
        .get @uri
        .accept 'json'
        .then ({body}) -> body

      create: ->
        @agent
        .put @uri
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
        uri = new URL ec(_id), @uri+'/'
        @agent
        .put uri.toString()
        .type 'json'
        .accept 'json'
        .send doc
        .then ({body}) -> body

Get a document, optionally at a given revision.

      get: (_id,options = {}) ->
        uri = new URL ec(_id), @uri+'/'
        for own k,v of options when v?
          uri.searchParams.set k, v
        @agent
        .get uri.toString()
        .accept 'json'
        .then ({body}) -> body

      has: (_id,options = {}) ->
        uri = new URL ec(_id), @uri+'/'
        for own k,v of options when v?
          uri.searchParams.set k, v
        @agent
        .get uri.toString()
        .accept 'json'
        .then -> true
        .catch (err) ->
          if err.status is 404
            false
          else
            Promise.reject err

Delete a document based on its `_id` and `_rev` fields.

      delete: ({_id,_rev}) ->
        uri = new URL ec(_id), @uri+'/'
        uri.searchParams.set 'rev', _rev if _rev?
        @agent
        .delete uri.toString()
        .accept 'json'
        .then ({body}) -> body

Basic support for Mango queries and indexes

Non-blocking (most.js)

      find: (params) ->
        S = @findStream params
        most
          .fromEvent 'data', S
          .until most.fromEvent 'end', S

Blocking (Stream)

      findStream: (params) ->
        streamify @findAsyncIterable params

      findAsyncIterable: (params) ->
        uri = new URL '_find', @uri+'/'

        agent = @agent

        do ->
          bookmark = null
          limit = 100
          done = false
          while not done
            {body} = await agent
              .post uri.toString()
              .send Object.assign {bookmark,limit}, params
              .accept 'json'

            {docs} = body
            for doc from docs
              yield doc

            {bookmark} = body

            done = docs.length < limit
          return

      createIndex: (params) ->
        uri = new URL '_index', @uri+'/'
        @agent
        .post uri.toString()
        .send params
        .accept 'json'
        .then ({body}) -> body

Uses a server-side view, returns a stream containing one event for each row.

Non-blocking (most.js)

      query: (app,view,params) ->
        S = @queryStream app, view, params
        most
          .fromEvent 'data', S
          .merge most.fromEvent('error', S).map most.throwError
          .until most.fromEvent 'end', S

Blocking (Stream)

      queryStream: (app,view,params) ->
        view_stream @uri, app, view, params

Uses a wrapped client-side map function, returns a stream containing one event for each new row.
Please provide `map_function(emit)`, wrapping the actual `map` function.

      query_changes: (map_function,options = {}) ->
        {since,filter,selector,view} = options
        source = @changes {live:true,include_docs:true,since,filter,selector,view}
        changes_view map_function, source, options

Build a continuous `most.js` stream for changes.

      changes: (options = {}) ->
        options = Object.assign {}, options
        options.live ?= true
        options.since ?= 'now'

        @__changes options
        .map ({data}) -> data
        .map JSON.parse
        .tap ({seq}) -> options.since = seq if seq?
        .recoverWith =>
          debug 'recoverWith'
          @__changes options
        .continueWith =>
          debug 'continueWith'
          if options.live
            @__changes options
          else
            most.empty()

      __changes: (options) ->

        uri = new URL '_changes', @uri+'/'
        content = {}

        uri.searchParams.set 'feed', 'eventsource'
        uri.searchParams.set 'heartbeat', 5*1000
        uri.searchParams.set 'include_docs', options.include_docs ? false
        uri.searchParams.set 'conflicts', true if options.conflicts
        uri.searchParams.set 'attachments', true if options.attachments

        if options.filter?
          uri.searchParams.set 'filter', options.filter

        if options.selector?
          uri.searchParams.set 'filter', '_selector'
          content = selector: options.selector

        if options.view?
          uri.searchParams.set 'filter', '_view'
          uri.searchParams.set 'view', options.view

        uri.searchParams.set 'since', options.since

        uri_string = uri.toString()

        content_string = JSON.stringify content

        key = [uri_string,content_string].join ' '

        cacheable = options.live and options.since is 'now'
        if cacheable and @cache.has key
          debug '__changes: cached', key
          return (@cache.get key).stream

        debug '__changes: new', key
        source = new EventSource uri_string,
          method: 'POST'
          headers: 'Content-Type': 'application/json'
          content: content_string

        source.on 'open', ->
          debug '__changes: open', key
        source.on 'close', ->
          debug '__changes: close', key
        source.on 'error', (error) ->
          debug '__changes: error', key, error

        dispose = =>
          debug '__changes: dispose', key
          @cache.delete key
          source.close()
          return

        stream = fromEventSource source,dispose

        if cacheable
          stream = stream.multicast()
          @cache.set key, {stream,source}

        stream

    module.exports = CouchDB
    ec = encodeURIComponent
    most = require 'most'
    EventSource = require '@shimaore/eventsource'
    {fromEventSource} = require 'most-w3msg'
    {URL} = require 'url'
    Request = require 'superagent'
    view_stream = require './view-stream'
    changes_view = require './changes-view'
    debug = (require 'debug') 'most-couchdb'
    streamify = require 'async-stream-generator'
