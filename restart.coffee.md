When the stream finishes, we restart it with a small delay.

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

The argument `s` is a function creating a new version of the stream for us.

    module.exports = autoRestart = (s) ->

      most
      .generate -> yield 200
      .startWith 0
      .map (delay) ->
        await sleep delay
        s()
      .chain most.fromPromise
      .switchLatest()

    most = require 'most'
