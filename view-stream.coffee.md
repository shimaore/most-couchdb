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

      rows req
      .pipe map.obj ({value}) -> value

    module.exports = view_stream
    json = require 'json-parser-transform'
    rows = json.thru (prefix) -> prefix.length is 2 and prefix[0] is 'rows'
    map = require 'through2-map'
    Request = require 'superagent'
