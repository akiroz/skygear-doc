<a name="overview"></a>
## Overview

Before proceeding, make you have already configured your
[skygear container](/js/guide#set-up-app).

### The Record Class

- Record objects are like dictionaries with keys and values; keys will be
mapped to database columns and values will be stored appropriately
based on the [Data Type](#data-type). Each record has [Reserved Keys](#reserved)
that cannot be used (e.g. `id` or `_id`).

- Records are associated with a user-defined type (e.g. `'blog'`).

- Records are associated with an owner, you must be logged in to save records
(See [Access Control](/js/guide/access-control)).

You can design different `Record` type to model your app. Just like defining
tables in SQL.

``` javascript
const Blog = skygear.Record.extend('blog');

const Note = skygear.Record.extend('note');
const note = new Note({ 'content': 'Hello World' });

```

**Important:** The skygear server will create a database schema for each record
type based on the first record saved and if later records contains fields that doesn't
exist in the current schema, a schema migration is performed.


### Record Database

You are provided with a public and private database:

- Records saved in the public database are readable by everyone by default
(including anonymous users). You may refine this by changing the
[default access control rules](/js/guide/access-control#acl-default)
or setting rules for each record independently.

- Each user has their own private database where records are only accessible
to themselves regardless of what access control rules are set on the records.

- The database interface can be accessed via `skygear.publicDB` and
`skygear.privateDB`.

<a name="basic-crud"></a>
## Basic CRUD

### Creating records

Saving a record to the public database:

``` javascript
const Note = skygear.Record.extend('note');
const note = new Note({ 'content': 'Hello World!' });

skygear.publicDB.save(note)
  .then((saved_record) => {
    console.log(saved_record);
  })
  .catch((error) => {
    console.error(error);
  });
```

Saving multiple records to the public database:

``` javascript
const note1 = new Note({ date: '2016-09-07', content: 'first note' }),
const note2 = new Note({ date: '2016-09-08', content: 'second note'}),
const note3 = new Note({ date: new Date(),   content: 'third note' }),
const note4 = new Note({ date: '2016-09-10', content: 'forth note' }),
const note5 = new Note({ date: new Date(),   content: 'fifth note' }),

skygear.publicDB.save([note1, note2, note3, note4, note5])
  .then((result) => {
    console.log(result.savedRecords);
    // [note1, note2, undefined, note4, undefined]
    console.log(result.errors);
    // [undefined, undefined, error3, undefined, error5]
  })
  .catch((error) => {
    // request error
  });
```

By default, only erroneous records are not saved (e.g. due to data type conflict).
You can use the `atomic` option such that either all or none of the records will be saved:

``` javascript
skygear.publicDB.save([note1, note2, note3, note4, note5], { atomic: true });
// none of the records will be saved
```


### Reading records

Refer to [Query](/js/guide/query) section for details.

Retrieve record using a Query object:

``` javascript
const blogQuery = new skygear.Query(Blog)
  .greaterThan('popular', 10)
  .addDescending('popular');

blogQuery.limit = 10;

skygear.publicDB.query(blogQuery)
  .then((records) => {
    console.log(records)
  })
  .catch((error) => {
    console.error(error);
  });

```

### Updating records

Update a record by ID:

``` javascript
const noteUpdate = new Note({
  _id: 'note/f705fd10-bfd8-47dd-aec9-3e3de32a8f5a',
  content: 'Hello New World'
});

skygear.publicDB.save(noteUpdate)
  .then((record) => {
    console.log('Update Success!')
  })
  .catch((error) => {
    console.error(error);
  });


```

Updating multiple records:


``` javascript
const noteQuery = new skygear.Query(Note)
  .equalTo('content', 'Hello World!');

skygear.publicDB.query(noteQuery)
  .then((records) => {
    for(let i = 0; i < records.length; i++) {
      records[i].content = 'Hello New World!';
    }
    return skygear.publicDB.save(records);
  });

```

- After saving a record, any attributes modified from the server side will
be updated on the saved record object in place.
- The local transient fields of the records are merged with any remote
transient fields applied on the server side.


### Deleting a record

Delete record by ID:

``` javascript
skygear.publicDB.delete({
  id: 'note/f705fd10-bfd8-47dd-aec9-3e3de32a8f5a'
}).then((deletedRecord) => {
  console.log(deletedRecord);
}).catch((error) => {
  console.error(error);
});

```

Deleting multiple records:


``` javascript
const blogQuery = new skygear.Query(Blog)
  .lessThan('rating', 3);

skygear.publicDB.query(blogQuery)
  .then((blogs) => {
    console.log(blogs);
    // [blog1, blog2, blog3]
    return skygear.publicDB.delete(notes);
  })
  .then((errors) => {
    console.log(errors);
    // [undefined, undefined, error3]
  })
  .catch((error) => {
    // request error
  });

```

If some delete operation fails (e.g. due to insufficient permission) the error
message(s) will be reflected in the errors array illustrated above.


<a name="data-type"></a>
## Data Types

Skygear supports most of the built-in JavaScript types:

- String
- Number
- Boolean
- Array
- Object
- Date

In addition, Skygear JS SDK 4 more types:

- [Reference](#reference)
- [Sequence](#sequence)
- [Geo Location](/js/guide/geolocation)
- [Assets](/js/guide/asset) (File Uploads)

Refer to the [server](/server/guide/data-type) documentation for
more detail on supported data types.


<a name="reference"></a>
## Records Relations

Skygear supports relation between records via _references_.
`skygear.Reference` is a class which will translate to foreign keys in
the skygear database for efficient queries.

``` javascript
const Message = skygear.Record.extend('message');
const Person = skygear.Record.extend('person');

const bob = new Person({ name: 'bob' });
const alice = new Person({ name: 'alice' });
const message  = new Message({
  from: new skygear.Reference(bob),
  to: new skygear.Reference(alice),
  content: 'foo bar baz qux'
});

skygear.publicDB.save([bob, alice, message]);

```

**Note:** Since reference objects translates into foreign key constraints, the order of which records are saved matters here, saving `message` before either `bob` or `alice` will result
in an error. Also note that you must delete the `message` record before
`bob` or `alice`.

Using references, you can retrieve the referenced records in a single query using
[transient includes](/js/guide/query#relational-queries).


<a name="sequence"></a>
## Auto-Incrementing Sequences

All skygear records uses a UUIDv4 `id` as the primary key in the database,
you can add a sequence field if you want a sequential "id"
for each record, this field is guaranteed to be unique.

**Note**: You can have multiple sequence fields and they will
not affect the UUID primary key in the database.

``` javascript
const note1 = new Note({
  noteID: new skygear.Sequence(),
  content: 'First Note'
});

skygear.publicDB.save(note1)
  .then((note) => {
    console.log(note.noteID);
    // 0
  });

const note2 = new Note({
  content: 'Second Note'
});

skygear.publicDB.save(note2)
  .then((note) => {
    console.log(note.noteID);
    // 1
  });

```

Sequence fields of newly saved records will take the value of the highest number
plus one. You can also manually override the sequence field of a record as long
as the number doesn't collide with any of the existing ones:


``` javascript
const note3 = new Note({
  noteID: 42,
  content: 'Third Note'
});
skygear.publicDB.save(note3);

```

<a name="reserved"></a>
## Reserved Fields

All records have reserved fields provided by skygear and can be queried
just like any normal field, note the difference between database column
names and the object field.

Database Column | Object Field | Description
--------------- | ------------ | -----------
**N/A**         | `id`         | record type and id
`_id`           | `_id`        | record id
`_created_at`   | `createdAt`  | time of creation
`_updated_at`   | `updatedAt`  | time last modified
`_created_by`   | `createdBy`  | user id
`_updated_by`   | `updatedBy`  | user id
`_owner_id`     | `ownerID`    | user id of owner

Besides these fields, try to avoid naming your fields staring with `_` or `$`.

One quick example:

``` javascript
skygear.publicDB.query(new skygear.Query(Note))
  .then((records) => console.log(records[0]));
```

``` javascript
/* Type: RecordCls */ {
  createdAt: new Date("Thu Jul 07 2016 12:12:42 GMT+0800 (CST)"),
  updatedAt: new Date("Thu Jul 07 2016 12:42:17 GMT+0800 (CST)"),
  createdBy: "118e0217-ffda-49b4-8564-c6c9573259bb",
  updatedBy: "118e0217-ffda-49b4-8564-c6c9573259bb",
  ownerID: "118e0217-ffda-49b4-8564-c6c9573259bb",
  id: "note/3b9f8f98-f993-4e1d-81c3-a451e483306b",
  _id: "3b9f8f98-f993-4e1d-81c3-a451e483306b",
  recordType: "note",
}
```

Querying reserved columns:

``` javascript
const myNotesQuery = new skygear.Query(Note)
  .equalTo('_owner_id', skygear.currentUser.id);

skygear.publicDB.query(myNotesQuery)
  .then((results) => {
    console.log(results);
    // [ ... ]
  });

```

See [database schema](/server/guide/database-schema) for more info.
