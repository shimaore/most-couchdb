CouchDB view as a stream of `row`
---------------

    view_stream = (db_uri,app,view,params) ->

      if app?
        uri = "#{db_uri}/_design/#{app}/_view/#{view}"
      else
        uri = "#{db_uri}/#{view}"

      n = oboe_stream_request url:uri,qs:params

      r = oboe_stream 'rows.*', n
      n.node 'rows.*', -> oboe.drop
      r

    module.exports = view_stream
    oboe_stream = require 'oboe-as-stream'
    oboe = require 'oboe'
    request = require 'request'
    oboe_stream_request = (require 'oboe-stream-request') oboe, request
