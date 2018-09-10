    (require 'chai').should()

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    describe 'Simple queries', ->
      CouchDB = require '..'
      db = new CouchDB 'http://admin:password@couchdb:5984/example'
      it 'should create the database', ->
        await db.agent.put db.uri

      it 'should put a document', ->
        await db.put _id:'hello', description:'world'

      it 'should retrieve the document', ->
        doc = await db.get 'hello'
        doc.should.have.property '_id', 'hello'
        doc.should.have.property '_rev'
        doc.should.have.property 'description', 'world'

      it 'should delete the document', ->
        doc = await db.get 'hello'
        await db.delete doc

      it 'should query', ->
        await db.put _id:'hola', name:'bear'
        db.query '_all_docs', null, include_docs:true
        .take 1
        .observe (row) ->
          row.should.have.property 'id', 'hola'
          row.should.have.property 'value'
          row.should.have.property 'doc'

      it 'should query-changes', ->
        @timeout 4000
        map = (emit) ->
          (doc) ->
            if doc.name?
              emit 'pet', doc.name

        result = db.query_changes map
          .take 1
          .observe (row) ->
            row.should.have.property 'id', 'hallo'
            row.should.have.property 'key', 'pet'
            row.should.have.property 'value', 'dog'

        await sleep 10

        await db.put _id:'hallo', name:'dog'
        await result

      it 'should delete the database', ->
        await db.agent.delete db.uri
