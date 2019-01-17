    describe 'The modules', ->
      it 'should load', ->
        require '../changes-view'
        require '../db'
        require '../with-update'
        require '../restart'
        require '../view-stream'
