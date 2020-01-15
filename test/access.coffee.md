    ({expect} = require 'chai').should()

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    describe 'Simple queries', ->
      CouchDB = require '..'
      db = new CouchDB "http://#{process.env.COUCHDB_USER ? 'admin'}:#{process.env.COUCHDB_PASSWORD ? 'password'}@couchdb:5984/example"

      it 'should create the database', ->
        await db.create(2)
        return

      it 'should get info', ->
        info = await db.info()
        info.should.have.property 'update_seq'
        return

      it 'should put a document', ->
        await db.put _id:'hello', description:'world'
        return

      it 'should retrieve the document', ->
        doc = await db.get 'hello'
        doc.should.have.property '_id', 'hello'
        doc.should.have.property '_rev'
        doc.should.have.property 'description', 'world'
        return

      it 'should test for the document', ->
        truth = await db.has 'hello'
        truth.should.eql true
        return

      it 'should delete the document', ->
        doc = await db.get 'hello'
        await db.delete doc
        return

      it 'should test for the document', ->
        truth = await db.has 'hello'
        truth.should.eql false
        return

      it 'should query (no docs)', ->
        count = 0
        await db.query null, '_all_docs', include_docs:true
          .observe -> count++
        (expect count).to.eql 0
        return

      it.skip 'should queryStream (error)', ->
        count = 0
        db
        .queryStream null, '_all_doc', include_docs:true
        .on 'error', (error) ->
          count++
        (expect count).to.eql 1
        return

      it.skip 'should query (error)', ->
        count = 0
        await db.query null, '_all_doc', include_docs:true
          .recoverWith (error) ->
            count++
            most.empty()
          .observe (value) ->
            count++

        (expect count).to.eql 1
        return

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
        return

      it 'should queryStream', ->
        S = db.queryStream null, '_all_docs', include_docs:true
        len = 0
        for await row from S
          (expect row).to.have.property 'id', 'hola'
          (expect row).to.have.property 'value'
          (expect row).to.have.property 'doc'
          (expect row.doc).to.have.property 'kisses'
          (expect row.doc.kisses).to.have.length 2
          ++len
        len.should.eql 1
        return

      it 'should queryAsyncIterable', ->
        S = db.queryAsyncIterable null, '_all_docs', include_docs:true
        len = 0
        for await row from S
          (expect row).to.have.property 'id', 'hola'
          (expect row).to.have.property 'value'
          (expect row).to.have.property 'doc'
          (expect row.doc).to.have.property 'kisses'
          (expect row.doc.kisses).to.have.length 2
          ++len
        len.should.eql 1
        return

      [1,10,98,99,100,101,102,198,199,200,201,202,2367].forEach (number) ->
        it "should queryAsyncIterable (pages) for #{number}", ->
          await db.put
            _id: "_design/test#{number}"
            language: 'coffeescript'
            views:
              pages:
                map: """
                  (doc) ->
                    [1..#{number}].map emit
                """
          S = db.queryAsyncIterable "test#{number}", 'pages', include_docs:true
          len = 0
          for await row from S
            (expect row).to.have.property 'id', 'hola'
            (expect row).to.have.property 'value'
            (expect row).to.have.property 'doc'
            (expect row.doc).to.have.property 'kisses'
            (expect row.doc.kisses).to.have.length 2
            ++len
          len.should.eql number
          return

      it "should queryAsyncIterable (with restrictions)", ->
          S = db.queryAsyncIterable "test2367", 'pages', start_key: 34, endkey: 168, inclusive_end:false
          len = 0
          for await row from S
            ++len
          len.should.eql 168-34
          return

      it "should queryAsyncIterable (with restrictions)", ->
          S = db.queryAsyncIterable "test2367", 'pages', start_key: 34
          len = 0
          for await row from S
            ++len
          len.should.eql 2367-34+1
          return

      it "should queryAsyncIterable (with restrictions)", ->
          S = db.queryAsyncIterable "test2367", 'pages', endkey: 28, inclusive_end:true
          len = 0
          for await row from S
            ++len
          len.should.eql 28
          return

      it "should queryAsyncIterable (reverse)", ->
          S = db.queryAsyncIterable "test2367", 'pages', startkey: 168, endkey: 34, inclusive_end:false, descending:true
          len = 0
          for await row from S
            ++len
          len.should.eql 168-34
          return

      it "should queryAsyncIterable (key)", ->
          S = db.queryAsyncIterable "test2367", 'pages', key: 34
          len = 0
          for await row from S
            ++len
          len.should.eql 1
          return

      it "should queryAsyncIterable (keys)", ->
          @timeout 6000
          S = db.queryAsyncIterable "test2367", 'pages', keys: [34...664]
          len = 0
          for await row from S
            ++len
          len.should.eql 664-34
          return

      it "should queryAsyncIterable (keys,cancel)", ->
          @timeout 6000
          cancel = false
          S = db.queryAsyncIterable "test2367", 'pages', keys: [34...664], -> cancel
          len = 0
          for await row from S
            ++len
            cancel = true if len > 10
          len.should.be.below 15
          return

      it 'should find', ->
        await db.put _id:'yippee', name:'coocoo'
        db.find selector: name: 'coocoo'
        .take 1
        .observe (doc) ->
          doc.should.have.property '_id', 'yippee'
          doc.should.have.property 'name', 'coocoo'
        return

      it 'should findStream', ->
        S = db.findStream selector: name: 'coocoo'
        len = 0
        for await doc from S
          doc.should.have.property '_id', 'yippee'
          doc.should.have.property 'name', 'coocoo'
          ++len
        len.should.eql 1
        return

      it 'should findAsyncIterable', ->
        S = db.findAsyncIterable selector: name: 'coocoo'
        len = 0
        for await doc from S
          doc.should.have.property '_id', 'yippee'
          doc.should.have.property 'name', 'coocoo'
          ++len
        len.should.eql 1
        return

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
        return

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
        return

      it 'should query-changes (multiple times on the same)', ->
        map = (emit) ->
          (doc) ->
            if doc.name?
              emit 'pet', doc.name

        result = db.query_changes map
          .take 3
          .observe (row) ->
            row.should.have.property 'id'
            row.should.have.property 'seq'
            row.should.not.have.property 'doc'
            row.should.have.property 'key', 'pet'
            row.should.have.property 'value', 'jay'

        await sleep 500

        await db.put _id:'jay1', name:'jay'
        await db.put _id:'jay2', name:'jay'
        await db.put _id:'jay3', name:'jay'
        await result
        return

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
        return

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
        return

      it 'should query-changes (with timeout)', ->
        @timeout 47*1000
        map = (emit) ->
          (doc) ->
            if doc.name?
              emit 'blurb', doc.name

        result = db.query_changes map
          .take 1
          .observe (row) ->
            row.should.have.property 'id', 'gutenTag'
            row.should.have.property 'seq'
            row.should.not.have.property 'doc'
            row.should.have.property 'key', 'blurb'
            row.should.have.property 'value', 'Katz'

        await sleep 45*1000

        await db.put _id:'gutenTag', name:'Katz'
        await result
        return

      it 'should query-changes (with selector, with timeout)', ->
        @timeout 47*1000

        map = (emit) ->
          (doc) ->
            if doc.name?
              emit 'blob', doc.name

        result = db.query_changes map, include_docs:true, selector: _id: 'bumbling'
          .take 1
          .observe (row) ->
            row.should.have.property 'id', 'bumbling'
            row.should.have.property 'seq'
            row.should.have.property 'doc'
            row.should.have.property 'key', 'blob'
            row.should.have.property 'value', 'larva'

        await sleep 45*1000

        await db.put _id:'massive', name:'mosquita'
        await db.put _id:'bumbling', name:'larva'
        await result
        return

      it 'should query (with params)', ->
        for await row from db.queryStream null, '_all_docs', include_docs:true, startkey: 'hall', endkey: 'hello'
          (expect row).to.have.property 'id', 'hallo'
        return

      it 'should put attachment', ->
        await db.put _id:'bob'
        {_rev} = await db.get 'bob'
        await db.putAttachment 'bob', 'hello/world.png', _rev, new Buffer([0xbe,0xfe]), 'image/png'
        buf = await db.getAttachment 'bob', 'hello/world.png'
        expect(buf[0]).to.equal 0xbe
        expect(buf[1]).to.equal 0xfe
        {_rev,_attachments} = await db.get 'bob'
        expect(_attachments).to.have.property 'hello/world.png'
        await db.deleteAttachment 'bob', 'hello/world.png', _rev
        {_attachments} = await db.get 'bob'
        expect(_attachments).to.be.undefined
        return

      it 'should insert a whole bunch of documents', ->
        @timeout 20000
        for i in [1..1000]
          await db.put _id:"cat #{i.toString(10).padStart(4)}", countme: true
        return

      it 'should insert the view', ->
        await db.put
          _id: "_design/countme"
          language: 'coffeescript'
          views:
            pages:
              map: """
                (doc) ->
                  if doc.countme
                    emit 1
                    emit 2
                    emit 3
              """
        return

      it "should queryAsyncIterable (identical keys)", ->
          @timeout 4000
          S = db.queryAsyncIterable 'countme', 'pages', key: 3
          len = 0
          for await row from S
            row.should.have.property 'key', 3
            ++len
          len.should.eql 1000
          return

      it "should queryAsyncIterable (identical keys, array)", ->
          @timeout 4000
          S = db.queryAsyncIterable 'countme', 'pages', keys: [1,2]
          len = 0
          for await row from S
            ++len
          len.should.eql 2000
          return

      it 'should delete the database', ->
        outcome = await db.destroy()
        outcome.should.have.property 'ok', true
        return
