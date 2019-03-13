CouchDB view as a Stream of `row`
---------------

    view_stream = (db_uri,app,view,params) ->

      if app?
        uri = "#{db_uri}/_design/#{app}/_view/#{view}"
      else
        uri = "#{db_uri}/#{view}"

      req = Request
        .get uri
        .query params

      A = rows req
      B = map.obj ({value}) -> value
      C = pump A, B
      if Symbol.asyncIterator of Stream.Readable.prototype
        C[Symbol.asyncIterator] ?= Stream.Readable.prototype[Symbol.asyncIterator]
      C

    module.exports = view_stream
    json = require 'json-parser-transform'
    rows = json.thru (prefix) -> prefix.length is 2 and prefix[0] is 'rows'
    map = require 'through2-map'
    pump = require 'pump'
    Stream = require 'stream'
    Request = require 'superagent'
