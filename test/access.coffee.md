    ({expect} = require 'chai').should()

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    describe 'Simple queries', ->
      CouchDB = require '..'
      db = new CouchDB "http://#{process.env.COUCHDB_USER}:#{process.env.COUCHDB_PASSWORD}@couchdb:5984/example"
      it 'should create the database', ->
        await db.create()

      it 'should get info', ->
        info = await db.info()
        info.should.have.property 'update_seq'

      it 'should put a document', ->
        await db.put _id:'hello', description:'world'

      it 'should retrieve the document', ->
        doc = await db.get 'hello'
        doc.should.have.property '_id', 'hello'
        doc.should.have.property '_rev'
        doc.should.have.property 'description', 'world'

      it 'should test for the document', ->
        truth = await db.has 'hello'
        truth.should.eql true

      it 'should delete the document', ->
        doc = await db.get 'hello'
        await db.delete doc

      it 'should test for the document', ->
        truth = await db.has 'hello'
        truth.should.eql false

      it 'should query', ->
        await db.put _id:'hola', name:'bear', nothing: null, kisses: ['dog','cat']
        await db.query null, '_all_docs', include_docs:true
        .take 1
        .observe (row) ->
          (expect row).to.have.property 'id', 'hola'
          (expect row).to.have.property 'value'
          (expect row).to.have.property 'doc'
          (expect row.doc).to.have.property 'kisses'
          (expect row.doc.kisses).to.have.length 2

      it 'should find', ->
        await db.put _id:'yippee', name:'coocoo'
        db.find selector: name: 'coocoo'
        .take 1
        .observe (doc) ->
          doc.should.have.property '_id', 'yippee'
          doc.should.have.property 'name', 'coocoo'

      it 'should findStream', ->
        S = db.findStream selector: name: 'coocoo'
        len = 0
        for await doc from S
          doc.should.have.property '_id', 'yippee'
          doc.should.have.property 'name', 'coocoo'
          ++len
        len.should.eql 1

      it 'should findAsyncIterable', ->
        S = db.findAsyncIterable selector: name: 'coocoo'
        len = 0
        for await doc from S
          doc.should.have.property '_id', 'yippee'
          doc.should.have.property 'name', 'coocoo'
          ++len
        len.should.eql 1

      it 'should query-changes', ->
        map = (emit) ->
          (doc) ->
            if doc.name?
              emit 'pet', doc.name

        result = db.query_changes map
          .take 1
          .observe (row) ->
            row.should.have.property 'id', 'hallo'
            row.should.have.property 'seq'
            row.should.not.have.property 'doc'
            row.should.have.property 'key', 'pet'
            row.should.have.property 'value', 'dog'

        await sleep 500

        await db.put _id:'hallo', name:'dog'
        await result

      it 'should query-changes (twice)', ->
        map = (emit) ->
          (doc) ->
            if doc.name?
              emit 'pet', doc.name

        result1 = db.query_changes map
          .take 1
          .observe (row) ->
            row.should.have.property 'id', 'kitty'
            row.should.have.property 'seq'
            row.should.not.have.property 'doc'
            row.should.have.property 'key', 'pet'
            row.should.have.property 'value', 'poo'

        result2 = db.query_changes map
          .take 1
          .observe (row) ->
            row.should.have.property 'id', 'kitty'
            row.should.have.property 'seq'
            row.should.not.have.property 'doc'
            row.should.have.property 'key', 'pet'
            row.should.have.property 'value', 'poo'

        await sleep 500

        await db.put _id:'kitty', name:'poo'
        await result1
        await result2

      it 'should query-changes (with docs explicitely)', ->
        map = (emit) ->
          (doc) ->
            if doc.name?
              emit 'pet', doc.name

        result = db.query_changes map, include_docs:true
          .take 1
          .observe (row) ->
            row.should.have.property 'id', 'bonjour'
            row.should.have.property 'seq'
            row.should.have.property 'doc'
            row.should.have.property 'key', 'pet'
            row.should.have.property 'value', 'cat'

        await sleep 500

        await db.put _id:'bonjour', name:'cat'
        await result

      it 'should query-changes (with selector)', ->
        map = (emit) ->
          (doc) ->
            if doc.name?
              emit 'pet', doc.name

        result = db.query_changes map, include_docs:true, selector: _id: 'tag'
          .take 1
          .observe (row) ->
            row.should.have.property 'id', 'tag'
            row.should.have.property 'seq'
            row.should.have.property 'doc'
            row.should.have.property 'key', 'pet'
            row.should.have.property 'value', 'lion'

        await sleep 500

        await db.put _id:'ignored', name:'mosquito'
        await db.put _id:'tag', name:'lion'
        await result

      it 'should delete the database', ->
        outcome = await db.destroy()
        outcome.should.have.property 'ok', true
