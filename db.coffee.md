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

    http_agent = new http.Agent
      keepAlive: true
      # keepAliveMsecs: 100
      maxSockets: 768           # per host
      maxFreeSockets: 256       # per host
      timeout: 30000            # active socket keepalive
    https_agent = new https.Agent
      keepAlive: true
      # keepAliveMsecs: 100
      maxSockets: 768           # per host
      maxFreeSockets: 256       # per host
      timeout: 30000            # active socket keepalive

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    class CouchDB

      constructor: (uri,use_lru,@limit = 100) ->
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
        mostify @findStream params

Blocking (Stream)

      findStream: (params) ->
        streamify @findAsyncIterable params

      findAsyncIterable: (params) ->
        uri = new URL '_find', @uri+'/'

        agent = @agent
        limit = @limit

        do ->
          bookmark = null
          done = false
          while not done
            body = null
            until body?
              {body} = await agent
                .post uri.toString()
                .send Object.assign {bookmark,limit}, params
                .accept 'json'
                .catch (error) ->
                  debug 'findAsyncIterable: error', app, view, params, error
                  if error.status is 404
                    body: docs: []
                  else
                    body: null
              unless body?
                await sleep 100
              else
                await sleep 1

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
        mostify @queryStream app, view, params

Blocking (Stream)

      queryStream: (app,view,params) ->
        streamify @queryAsyncIterable app, view, params

Async Iterable

      queryAsyncIterable: (app,view,params) ->
        if app?
          uri = new URL "_design/#{app}/_view/#{view}", @uri+'/'
        else
          uri = new URL view, @uri+'/'

        agent = @agent
        limit = @limit

        query = Object.assign {}, params

Normalize the request

        if query.startkey?
          query.start_key ?= query.startkey
          delete query.startkey

        if query.endkey?
          query.end_key ?= query.endkey
          delete query.endkey

        if query.key?
          query.keys = [query.key]
          delete query.key

Build the ranges

        switch
          when query.keys?
            {keys} = query
            ranges = ->
              for key in keys
                yield start_key:key,end_key:key,inclusive_end:true
              return
          else
            {start_key,end_key,inclusive_end} = query
            ranges = ->
              yield {start_key,end_key,inclusive_end}
              return

        delete query.keys
        delete query.start_key
        delete query.end_key
        delete query.inclusive_end

        do ->

          for range from ranges()
            query.startkey = range.start_key
            query.endkey = range.end_key
            query.inclusive_end = range.inclusive_end

            done = false
            while not done

              query.limit ?= limit
              query.sorted = true

              body = null
              until body?
                {body} = await agent
                  .get uri.toString()
                  .query stringify query
                  .accept 'json'
                  .catch (error) ->
                    debug 'queryAsyncIterable: error', app, view, params, error
                    if error.status is 404
                      body: rows: []
                    else
                      body: null
                unless body?
                  await sleep 100
                else
                  await sleep 1

              {rows} = body

              if rows.length is limit
                next_row = rows.pop()
              else
                next_row = null

              for row in rows
                yield row

              if next_row?
                query.startkey = next_row.key
                query.startkey_docid = next_row.id
              else
                done = true
                delete query.startkey_docid
          return

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
        key_options = {}

        key_options.include_docs = true if options.include_docs
        key_options.conflicts    = true if options.conflicts
        key_options.attachments  = true if options.attachments

        uri.searchParams.set 'feed', 'eventsource'
        uri.searchParams.set 'heartbeat', 5*1000
        uri.searchParams.set 'timeout', 30*1000
        uri.searchParams.set 'include_docs', true if key_options.include_docs
        uri.searchParams.set 'conflicts',    true if key_options.conflicts
        uri.searchParams.set 'attachments',  true if key_options.attachments

        if options.filter?
          uri.searchParams.set 'filter', options.filter
          key_options.filter = options.filter

        if options.selector?
          uri.searchParams.set 'filter', '_selector'
          content =selector: options.selector
          key_options.selector = options.selector

        if options.view?
          uri.searchParams.set 'filter', '_view'
          uri.searchParams.set 'view', options.view
          key_options.view = options.view

        uri.searchParams.set 'since', options.since

        uri_string = uri.toString()

        key_string = JSON.stringify key_options

        key = [uri_string,key_string].join ' '

        cacheable = options.live and options.since is 'now'
        if cacheable and @cache.has key
          debug '__changes: cached', key
          return (@cache.get key).stream

        debug '__changes: new', key
        source = new EventSource uri_string,
          method: 'POST'
          headers: 'Content-Type': 'application/json'
          content: JSON.stringify content

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

      getAttachment: (_id,file) ->
        uri = new URL ec(_id)+'/'+encodeURI(file), @uri+'/'
        @agent
        .get uri.toString()
        .then ({body}) -> body

      putAttachment: (_id,file,rev,buf,type) ->
        uri = new URL ec(_id)+'/'+encodeURI(file), @uri+'/'
        @agent
        .put uri.toString()
        .query {rev}
        .type type
        .accept 'json'
        .send buf
        .then ({body}) -> body

      deleteAttachment: (_id,file,rev) ->
        uri = new URL ec(_id)+'/'+encodeURI(file), @uri+'/'
        @agent
        .delete uri.toString()
        .query {rev}
        .accept 'json'
        .then ({body}) -> body

    module.exports = CouchDB
    ec = encodeURIComponent
    EventSource = require '@shimaore/eventsource'
    {fromEventSource} = require 'most-w3msg'
    {URL} = require 'url'
    Request = require 'superagent'
    changes_view = require './changes-view'
    debug = (require 'debug') 'most-couchdb'
    streamify = require 'async-stream-generator'

    stringify = (params) ->
      params = Object.assign {}, params ? {}
      ['endkey','end_key','key','keys','startkey','start_key'].forEach (field) ->
        if field of params
          params[field] = JSON.stringify params[field]
        return
      params

    most = require 'most'
    mostify = (S) ->
      data   = most.fromEvent 'data', S
      errors = most.fromEvent('error', S).map most.throwError
      end    = most.fromEvent 'end', S
      data.merge(errors).until end
