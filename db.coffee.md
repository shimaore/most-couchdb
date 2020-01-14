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

      constructor: (uri,options,limit = 100) ->
        options ?= {}
        options = {use_lru: options} if typeof options is 'boolean' # legacy

Parse options

        @limit = options.limit ? limit

        {@poll_delay} = options

        if uri.match /\/$/
          @uri = uri.slice 0, -1
        else
          @uri = uri

        if options.use_lru
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

      create: (n) ->
        query = {}
        query.n = n if n?
        @agent
        .put @uri
        .query query
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
        fromAsyncIterable @findAsyncIterable params

Blocking (Stream)

      findStream: (params) ->
        streamify @findAsyncIterable params

      findAsyncIterable: (params) ->
        uri = new URL '_find', @uri+'/'

        agent = @agent
        our_limit = @limit
        poll_delay = @poll_delay

        do ->
          bookmark = null
          loop
            limit = our_limit
            body = null
            until body?
              {body} = await agent
                .post uri.toString()
                .send Object.assign {bookmark,limit}, params
                .accept 'json'
                .catch (error) ->
                  debug 'findAsyncIterable: error', params, error
                  switch
                    when error.status is 404
                      body: docs: []
                    when error.code is 'ETOOLARGE'
                      limit-- if limit > 1
                      body: null
                    else
                      body: null
              unless body?
                await sleep 100

            {docs} = body
            for doc from docs
              yield doc

            {bookmark} = body

            return if docs.length < limit
            await sleep poll_delay if poll_delay?
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
        fromAsyncIterable @queryAsyncIterable app, view, params

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
        our_limit = @limit

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

              limit = our_limit
              query.sorted = true

              body = null
              until body?
                {body} = await agent
                  .get uri.toString()
                  .query stringify Object.assign {limit}, query
                  .accept 'json'
                  .catch (error) ->
                    debug 'queryAsyncIterable: error', app, view, params, error
                    switch
                      when error.status is 404
                        body: rows: []
                      when error.code is 'ETOOLARGE'
                        limit-- if limit > 1
                        body: null
                      else
                        body: null
                unless body?
                  await sleep 100

              {rows} = body

              if rows.length is limit
                next_row = rows.pop()
              else
                next_row = null

              for row in rows
                yield row
                if query.limit?
                  query.limit--
                  return if query.limit is 0

              if next_row?
                query.startkey = next_row.key
                query.startkey_docid = next_row.id
                # await sleep 0
              else
                done = true
                delete query.startkey_docid
          return

Uses a wrapped client-side map function, returns a stream containing one event for each new row.
Please provide `map_function(emit)`, wrapping the actual `map` function.

      query_changes: (map_function,options) ->
        fromAsyncIterable @query_changesAsyncIterable map_function, options

      query_changesStream:  (map_function,options) ->
        streamify @query_changesAsyncIterable map_function, options

      query_changesAsyncIterable: (map_function,options) ->
        {since,filter,selector,view,include_docs} = options ? {}
        S = @changesAsyncIterable {live:true,include_docs:true,since,filter,selector,view}
        for await {id,seq,deleted,doc} from S

          out = []
          emit = (key,value) ->
            content = {id,seq,deleted,key,value}
            content.doc = doc if include_docs
            out.push content
            return

          fn = map_function emit

          fn Object.assign {}, doc # might throw
          for item in out
            yield item
        return

Build a continuous, non-blocking (`most.js`) stream for changes.

      changes: (options) ->
        fromAsyncIterable @changesAsyncIterable options

Blocking (Stream)

      changesStream: (options) ->
        streamify @changesAsyncIterable options

Async Iterable

      changesAsyncIterable: (options) ->
        uri = new URL '_changes', @uri+'/'

        agent = @agent
        our_limit = @limit
        poll_delay = @poll_delay

        query = {}
        content = {}

        query.feed = 'longpoll'
        query.heartbeat = 5*1000
        query.timeout = 30*1000

        options ?= {}
        query.include_docs = true if options.include_docs
        query.conflicts = true if options.conflicts
        query.attachments = true if options.attachments

        query.filter = options.filter if options.filter?

        switch
          when options.selector?
            query.filter = '_selector'
            content = selector: options.selector

          when options.view?
            query.filter = '_view'
            query.view = options.view

          when options.doc_ids?
            query.filter = '_doc_ids'
            content = doc_ids: options.doc_ids

        since = options.since ? 'now'

        do ->
          loop
            limit = our_limit
            body = null
            until body?
              {body} = await agent
                .post uri.toString()
                .query stringify Object.assign {since,limit}, query
                .send content
                .accept 'json'
                .catch (error) ->
                  debug 'changesAsyncIterable: error', options, error
                  switch
                    when error.status is 404
                      body: results:[], last_seq: null
                    when error.code is 'ETOOLARGE'
                      limit-- if limit > 1
                      body: null
                    else
                      body: null
              unless body?
                await sleep 100

            {results} = body
            for result in results
              yield result

            {last_seq} = body
            return if not last_seq?

            since = last_seq
            await sleep poll_delay if poll_delay?
          return

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
    {URL} = require 'url'
    Request = require 'superagent'
    debug = (require 'debug') 'most-couchdb'
    streamify = require 'async-stream-generator'
    {fromAsyncIterable} = require 'most-async-iterable'

    stringify = (params) ->
      params = Object.assign {}, params ? {}
      ['endkey','end_key','key','keys','startkey','start_key'].forEach (field) ->
        if field of params
          params[field] = JSON.stringify params[field]
        return
      params
