    (require 'chai').should()

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    describe 'with-update', ->
      CouchDB = require '../with-update'
      db = new CouchDB "http://#{process.env.COUCHDB_USER ? 'admin'}:#{process.env.COUCHDB_PASSWORD ? 'password'}@couchdb:5984/example2"
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

      it 'should (bulk) update a document', ->
        await db.bulk_update [
          {_id: 'lion', name: 'gri'}
          {_id: 'bear', name: 'gro'}
        ]
        doc = await db.get 'lion'
        doc.should.have.property 'name', 'gri'
        rev1 = doc._rev

        doc = await db.get 'bear'
        doc.should.have.property 'name', 'gro'
        rev2 = doc._rev

        await db.bulk_update [
          {_id: 'lion', name: 'gri'}
          {_id: 'bear', name: 'gro'}
        ]
        doc = await db.get 'lion'
        doc.should.have.property 'name', 'gri'
        doc.should.have.property '_rev', rev1
        doc = await db.get 'bear'
        doc.should.have.property 'name', 'gro'
        doc.should.have.property '_rev', rev2

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

      it 'should (bulk) merge a document', ->
        await db.put _id:'cat'
        await db.bulk_merge ['tiger','cat'], [
          {name: 'gui'}
          {name: 'pop'}
        ]
        doc = await db.get 'tiger'
        doc.should.have.property 'name', 'gui'
        doc = await db.get 'cat'
        doc.should.have.property 'name', 'pop'

        await db.bulk_merge ['tiger','cat'], [
          {name: 'iua'}
          {name: 'plp'}
        ]
        doc = await db.get 'tiger'
        doc.should.have.property 'name', 'iua'
        rev1 = doc._rev
        doc = await db.get 'cat'
        doc.should.have.property 'name', 'plp'
        rev2 = doc._rev

        await db.bulk_merge ['tiger','cat'], [
          {name: 'iua'}
          {name: 'plp'}
        ]
        doc = await db.get 'tiger'
        doc.should.have.property 'name', 'iua'
        doc.should.have.property '_rev', rev1
        doc = await db.get 'cat'
        doc.should.have.property 'name', 'plp'
        doc.should.have.property '_rev', rev2

      it 'should delete the database', ->
        outcome = await db.destroy()
        outcome.should.have.property 'ok', true
