<a name="what-are-records"></a>
## What are Records?
In Skygear, you can store data using **Records**. They provide a reliable
way store *structured*, *persistent* data on the backend.

Each Skygear record is associated with a named **type** which is bound to
an extendable data schema. In the JS SDK, records are simply JS objects.

All Skygear records are also assigned a unique UUID during creation for
identification.

<a name="record-owner"></a>
### Record Owner
By design, each Skygear record is associated a user for easy access control,
this is known as the record "owner" or "creator" so you must be logged-in
to be able to create and save records. However, if fine grained access
control is not required, you can login as the anonymous user. 

Refer to the [User & Authentication][] section for how to sign up / login
or the [Access Control][] section for how you can restrict Record access.

<a name="record-object"></a>
### Record Object
Skygear records are JS objects, you can easily create records using
the following 2 lines:
```js
// define record type "person"
const PersonRecord = skygear.Record.extend("person");
// create "person" record: bob
const bob = new PersonRecord({ name: "bob", age: 20 });
```
```js
{ // bob object:
  name: "bob"
  age: 20

  // metadata provided by skygear on creation (read-only)
  recordType: "person"
  id: "person/45b0272b-41dc-483c-a07d-798a95cabbbd"

  // metadata provided by skygear on save (read-only)
  ownerID: "7888d764-a20a-4c2c-89c2-8eb02de0a079"
  createdAt: Mon Oct 17 2016 16:49:14 GMT+0800 (HKT)
  createdBy: "7888d764-a20a-4c2c-89c2-8eb02de0a079"
  updatedAt: Mon Oct 17 2016 16:49:14 GMT+0800 (HKT)
  updatedBy: "7888d764-a20a-4c2c-89c2-8eb02de0a079"
}
```
When creating Skygear records, you should avoid starting your type name with
`_` and the fields names with either `_` or `$` as they are reserved by
Skygear for internal usage. You should also avoid naming your fields the
same as the metadata shown above.

<a name="public-private-database"></a>
## Public/Private Database
Skygear provides 2 places where you could save records: a `privateDB` that
can only be accessed by the user and a `publicDB` that's controlled by
access rules attached to each record. By default, every record in the
`publicDB` is readable by everyone and only writable by its creator.

Refer to the [Access Control][] section for more detail.

<a name="sql-database-backend"></a>
### SQL Database Backend
On the Skygear server backend, your data is stored in a PostgreSQL database.
The record's type name is mapped to a table name while field names are mapped
to table columns. You can view and modify skygear's data by connecting
directly to the database.

<a name="save"></a>
## Save
You can save skygear records using a single API call, here's how you could
save the `bob` record:
```js
const PersonRecord = skygear.Record.extend("person");
const bob = new PersonRecord({ name: "bob", age: 20 });
const savePromise = skygear.publicDB.save(bob); // or skygear.privateDB.save()
```
`savePromise` is an [A+ Promise Object][] that either returns the record
that was successfully saved or the error when the save operation failed.
You can find out more about errors in the [Error Handling][] guide.

When we save the `bob` record to the server, it will help us create a table
named `person` like the following:

| _id &lt;String&gt;                   | name &lt;String&gt; | age &lt;Integer&gt; |
|--------------------------------------|---------------------|---------------------|
| 45b0272b-41dc-483c-a07d-798a95cabbbd | bob                 | 20                  |

Data types are inferred from the first saved record of its type (e.g. `bob` is the first
saved record of the `person` type), skygear supports most of JS' built-in types.
You can read more about them in the [Data Types]() section.

<a name="batch-save"></a>
### Batch Save
You can also save multiple records at the same time like so:

```js
const alice = new PersonRecord({ name: 'alice', age: 12, height: 141 });
const frederica = new PersonRecord({ name: 'frederica', age: 19, height: 164 }); 
const multiSavePromise = skygear.publicDB.save([alice, frederica]);
```
A few things to note here:

- We have just dynamically added a new field to the `person` type called `height`,
the skygear server will help us extend the database schema automatically such that
the table now looks something like this:

| _id &lt;String&gt;                   | name &lt;String&gt; | age &lt;Integer&gt; | height &lt;Integer&gt; |
|--------------------------------------|---------------------|---------------------|------------------------|
| 45b0272b-41dc-483c-a07d-798a95cabbbd | bob                 | 20                  | NULL                   |
| b4400221-0bed-4ab2-b536-10be20a8f129 | alice               | 12                  | 141                    |
| d0bcbb3a-c379-460a-9dba-1f46eb980df1 | frederica           | 19                  | 164                    |

- `multiSavePromise` either returns an object that looks like the following
or an error if the skygear server can't be reached.
```js
{ // both records saved successfully
    savedRecords: [alice, frederica],
    errors: [null, null]
}
```
In case any of the records fails to save (e.g. due to schema conflict),
the corresponding error will be available in the `errors` array at its index:
```js
{ // failed to save frederica
    savedRecords: [alice, null],
    errors: [null, error]
}
```

<a name="retrieve"></a>
## Retrieve
You can retrieve records using **Queries**, they help you retrieve record(s)
from the database based on logical rules.

To create a query, you must provide a record class and optionally specify
rules to narrow your search. Using our `PersonRecord` class from previous
examples, we can retrieve the `bob` record from the public database like so:
```js
const queryPromise = skygear.publicDB.query(
  new skygear.Query(PersonRecord)
    .equalTo('name', 'bob')
);
```
`queryPromise` is an [A+ Promise Object][] that either returns an array
of record results or an error if the query operation failed.
(See [Error Handling][].)

We can also retrieve the `bob` record by ID using the special
query parameter `_id`:
```js
const queryPromise = skygear.publicDB.query(
  new skygear.Query(PersonRecord)
    .equalTo('_id', '45b0272b-41dc-483c-a07d-798a95cabbbd')
);
```

More in-depth usage of the query API is covered in the [Queries][] section.

<a name="update"></a>
## Update
In Skygear, save and update operations are the same thing. You can update
previously saved record objects and save them again like you would in a
text editor. Say we now know bob's height:
```js
bob.height = 169;
skygear.publicDB.save(bob);
```
This will update our database to something like the following:

| _id &lt;String&gt;                   | name &lt;String&gt; | age &lt;Integer&gt; | height &lt;Integer&gt; |
|--------------------------------------|---------------------|---------------------|------------------------|
| 45b0272b-41dc-483c-a07d-798a95cabbbd | bob                 | 20                  | 169                    |
| b4400221-0bed-4ab2-b536-10be20a8f129 | alice               | 12                  | 141                    |
| d0bcbb3a-c379-460a-9dba-1f46eb980df1 | frederica           | 19                  | 164                    |

<a name="update-by-id"></a>
### Update by ID
Sometimes you may also want to update a record by ID,
you can do this using the following snippet. Again, setting
bob's height but without the record object:
```js
skygear.publicDB.save(
  new PersonRecord({
    _id: 'person/45b0272b-41dc-483c-a07d-798a95cabbbd',
    height: 169
  })
);
```
Note: the special `_id` field is used in this case.

<a name="delete"></a>
## Delete
The delete API works in a similar fashion to the save API, you can
delete a record object; array of records or by ID.
```js
// delete previously defined record object from the database
skygear.publicDB.delete(frederica)

// delete record by ID (in this case: bob)
skygear.publicDB.delete({
  id: 'person/45b0272b-41dc-483c-a07d-798a95cabbbd'
})
```
Note: the delete operation uses the normal `id` field without underscore.

This API call returns a promise object containing the deleted record or
array of deleted records.

<a name="atomic-save-updates"></a>
## Atomic Save/Updates
In a multi-save/update operation, you can make it such that either all or
none of the records will be saved if an error occurs using the `atomic`
option. This call will behave just like a [single-item save][Save] operation:
```js
skygear.publicDB.save(arrayOfRecords, { atomic: true });
```

[Record Object]: #record-object
[Save]: #save
[Queries]: /cloud-database/js/queries
[Data Types]: /cloud-database/js/data-types
[Access Control]: /cloud-database/js/access-control
[Database Schema]: /anvanced/skygear-server#database-schema
[User & Authentication]: /user-authentication/js/basic-authentication
[Error Handling]: /cloud-database/js/error-handling
[A+ Promise Object]: https://github.com/promises-aplus/promises-spec
