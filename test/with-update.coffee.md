    (require 'chai').should()

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    describe 'with-update', ->
      CouchDB = require '../with-update'
      db = new CouchDB "http://#{process.env.COUCHDB_USER}:#{process.env.COUCHDB_PASSWORD}@couchdb:5984/example2"
      it 'should create the database', ->
        await db.create()

      it 'should update a document', ->
        await db.put _id:'bear'
        await db.update
          _id: 'bear'
          name: 'eay'
        doc = await db.get 'bear'
        doc.should.have.property 'name', 'eay'

        await db.update
          _id: 'bear'
          name: 'iua'
        doc = await db.get 'bear'
        doc.should.have.property 'name', 'iua'
        rev = doc._rev

        await db.update
          _id: 'bear'
          name: 'iua'
        doc = await db.get 'bear'
        doc.should.have.property 'name', 'iua'
        doc.should.have.property '_rev', rev

      it 'should merge a document', ->
        await db.put _id:'tiger'
        await db.merge 'tiger',
          name: 'eay'
        doc = await db.get 'tiger'
        doc.should.have.property 'name', 'eay'

        await db.merge 'tiger',
          name: 'iua'
        doc = await db.get 'tiger'
        doc.should.have.property 'name', 'iua'
        rev = doc._rev

        await db.merge 'tiger',
          name: 'iua'
        doc = await db.get 'tiger'
        doc.should.have.property 'name', 'iua'
        doc.should.have.property '_rev', rev

      it 'should delete the database', ->
        outcome = await db.destroy()
        outcome.should.have.property 'ok', true
