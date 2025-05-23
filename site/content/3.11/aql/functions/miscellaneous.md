---
title: Miscellaneous functions in AQL
menuTitle: Miscellaneous
weight: 40
description: >-
  AQL functions that do not fall into other categories are listed here
---
## Control flow functions

### FIRST_DOCUMENT()

`FIRST_DOCUMENT(alternative, ...) → doc`

Return the first alternative that is a document, and *null* if none of the
alternatives is a document.

- **alternative** (any, *repeatable*): input of arbitrary type
- returns **doc** (object\|null): document / object or null

### FIRST_LIST()

`FIRST_LIST(alternative, ...) → list`

Return the first alternative that is an array, and *null* if none of the
alternatives is an array.

- **alternative** (any, *repeatable*): input of arbitrary type
- returns **list** (array\|null): array / list or null

### MIN_MATCH()

`MIN_MATCH(expr1, ... exprN, minMatchCount) → fulfilled`

Match documents where at least **minMatchCount** of the specified
AQL expressions are satisfied.

There is a corresponding [`MIN_MATCH()` ArangoSearch function](arangosearch.md#min_match)
that can utilize View indexes.

- **expr** (expression, _repeatable_): any valid AQL expression
- **minMatchCount** (number): minimum number of expressions that should
  be satisfied
- returns **fulfilled** (bool): whether at least **minMatchCount** of the
  specified expressions are `true`

You can use `MIN_MATCH()` to filter if two out of three conditions evaluate to
`true` for instance:

```aql
LET members = [
  { name: "Carol", age: 41, active: true },
  { name: "Doug", age: 56, active: true },
]
FOR doc IN members
  FILTER MIN_MATCH(LENGTH(doc.name) == 5, doc.age >= 50, doc.active, 2)
  RETURN doc
```

An equivalent filter expression without `MIN_MATCH()` would be more cumbersome:

```aql
  FILTER (LENGTH(doc.name) == 5 AND doc.age >= 50)
    OR (doc.age >= 50 AND doc.active)
    OR (doc.active AND LENGTH(doc.name) == 5)
```

### NOT_NULL()

`NOT_NULL(alternative, ...) → value`

Return the first element that is not *null*, and *null* if all alternatives
are *null* themselves. It is also known as `COALESCE()` in SQL.

- **alternative** (any, *repeatable*): input of arbitrary type
- returns **value** (any): first non-null parameter, or *null* if all arguments
  are *null*

### Ternary operator

For conditional evaluation, check out the
[ternary operator](../operators.md#ternary-operator).

## Database functions

### CHECK_DOCUMENT()

`CHECK_DOCUMENT(document) → checkResult`

Returns *true* if *document* is a valid document object, i.e. a document
without any duplicate attribute names. Will return *false* for any
non-objects/non-documents or documents with duplicate attribute names.

{{< warning >}}
This is an internal function for validating database objects and
is not supposed to be useful for anything else.
{{< /warning >}}

The primary use case for this function is to apply it on all
documents in a given collection as follows:

```aql
FOR doc IN collection
  FILTER !CHECK_DOCUMENT(doc)
  RETURN JSON_STRINGIFY(doc)
```

This query will return all documents in the given collection with redundant
attribute names and export them. This output can be used for subsequent
cleanup operations.

{{< info >}}
When using object literals in AQL, there will be an automatic 
removal/cleanup of duplicate attribute names, so the function will be effective
only for **already stored** database documents. Therefore,
`RETURN CHECK_DOCUMENT( { a: 1, a: 2 } )` is expected to return `true`.
{{< /info >}}

- **document** (object): an arbitrary document / object
- returns **checkResult** (bool): *true* for any valid objects/documents without
  duplicate attribute names, and *false* for any non-objects/non-documents or 
  objects/documents with duplicate attribute names

### COLLECTION_COUNT()

`COLLECTION_COUNT(coll) → count`

Determine the amount of documents in a collection. [`LENGTH()`](#length)
is preferred.

### COLLECTIONS()

`COLLECTIONS() → docArray`

Return an array of collections.

- returns **docArray** (array): each collection as a document with attributes
  *name* and *_id* in an array

### COUNT()

This is an alias for [`LENGTH()`](#length).

### CURRENT_DATABASE()

`CURRENT_DATABASE() → databaseName`

Returns the name of the current database.

The current database is the database name that was specified in the URL path of the request (or defaults to _system database).

- returns **databaseName** (string): the current database name

### CURRENT_USER()

`CURRENT_USER() → userName`

Return the name of the current user.

The current user is the user account name that was specified in the
*Authorization* HTTP header of the request. It will only be populated if
authentication on the server is turned on, and if the query was executed inside
a request context. Otherwise, the return value of this function will be *null*.

- returns **userName** (string\|null): the current user name, or *null* if
  authentication is disabled

### DECODE_REV()

`DECODE_REV(revision) → details`

Decompose the specified `revision` string into its components.
The resulting object has a `date` and a `count` attribute.
This function is supposed to be called with the `_rev` attribute value
of a database document as argument.

- **revision** (string): revision ID string
- returns **details** (object\|null): object with two attributes
  *date* (string in ISO 8601 format) and *count* (integer number),
  or *null*

If the input revision ID is not a string or cannot be processed, the function
issues a warning and returns *null*.

Please note that the result structure may change in future versions of
ArangoDB in case the internal format of revision strings is modified. Please 
also note that the *date* value in the current result provides the date and
time of when the document record was put together on the server, but not
necessarily the time of insertion into the underlying storage engine. Therefore
in case of concurrent document operations the exact document storage order 
cannot be derived unambiguously from the revision value. It should thus be
treated as a rough estimate of when a document was created or last updated.

```aql
DECODE_REV( "_YU0HOEG---" )
// { "date" : "2019-03-11T16:15:05.314Z", "count" : 0 }
```

### DOCUMENT()

Dynamically look up one or multiple documents from any collections, either using
a collection name and one or more document keys, or one or more document
identifiers. The collections do not need to be known at query compile time, they
can be computed at runtime.

{{< info >}}
It is recommended to use subqueries with the [`FOR` operation](../high-level-operations/for.md)
and filters over `DOCUMENT()` whenever the collections are known in advance,
especially for [joins](../examples-and-query-patterns/joins.md), because they perform better, you
can add additional filters, and combine it with sorting to get an array of
documents in a guaranteed order.

Queries that use the `DOCUMENT()` function cannot be
[**cached**](../execution-and-performance/caching-query-results.md), each lookup is executed as
a single operation, the lookups need to be executed on Coordinators for
sharded collections in cluster deployments, and only primary indexes and no
projections can be utilized.
{{< /info >}}

`DOCUMENT(collection, id) → doc`

Return the document identified by `id` (document key or identifier) from the
specified `collection`.

If the document cannot be found, `null` will be returned.
If there is a mismatch between the `collection` passed and the collection in
the document identifier, then `null` will be returned, too.

The `id` parameter can also be an array of document keys or identifiers. In this
case, the function will return an array of all documents that could be found.
The results are not guaranteed to be in the requested order. Documents that
could not be found are not indicated in the result (no `null` values) and do
also not raise warnings.

- **collection** (string): name of a collection
- **id** (string\|array): a document key, a document identifier, or an array of
  document keys, identifiers, or both
- returns **doc** (document\|array\|null): the found document (or `null` if it
  was not found), or an array of all found documents **in any order**

**Examples**

```aql
---
name: FUNCTION_DOCUMENT_1
description: ''
dataset: knows_graph
---
RETURN DOCUMENT( persons, "persons/alice" )
```

```aql
---
name: FUNCTION_DOCUMENT_2
description: ''
dataset: knows_graph
---
RETURN DOCUMENT( persons, "alice" )
```

```aql
---
name: FUNCTION_DOCUMENT_3
description: ''
dataset: knows_graph
---
RETURN DOCUMENT( persons, [ "persons/alice", "persons/bob" ] )
```

```aql
---
name: FUNCTION_DOCUMENT_4
description: ''
dataset: knows_graph
---
RETURN DOCUMENT( persons, [ "alice", "bob" ] )
```

```aql
---
name: FUNCTION_DOCUMENT_5
description: ''
dataset: knows_graph
bindVars: 
  {
    "@coll": "persons",
    "key": "alice"
  }
---
RETURN DOCUMENT( @@coll, @key ) 
```

```aql
---
name: FUNCTION_DOCUMENT_6
description: ''
dataset: knows_graph
bindVars: 
  {
    "@coll": "persons",
    "keys": ["alice", "bob"]
  }
---
RETURN DOCUMENT( @@coll, @keys )
```

---

`DOCUMENT(id) → doc`

The function can also be used with a single `id` parameter as follows:

- **id** (string\|array): a document identifier, or an array of identifiers
- returns **doc** (document\|array\|null): the found document (or `null` if it
  was not found), or an array of the found documents **in any order**

**Examples**

```aql
---
name: FUNCTION_DOCUMENT_7
description: ''
dataset: knows_graph
---
RETURN DOCUMENT("persons/alice")
```

```aql
---
name: FUNCTION_DOCUMENT_8
description: ''
dataset: knows_graph
---
RETURN DOCUMENT( [ "persons/alice", "persons/bob" ] )
```

```aql
---
name: FUNCTION_DOCUMENT_9
description: ''
dataset: knows_graph
bindVars: 
  {
    "key": "persons/alice"
  }
---
RETURN DOCUMENT( @key )
```

```aql
---
name: FUNCTION_DOCUMENT_10
description: ''
dataset: knows_graph
bindVars: 
  {
    "keys": ["persons/alice", "persons/bob"]
  }
---
RETURN DOCUMENT( @keys )
```

```aql
---
name: FUNCTION_DOCUMENT_11
description: ''
dataset: knows_graph
bindVars: 
  {
    "key": "bob"
  }
---
RETURN DOCUMENT( CONCAT("persons/", @key) )
```

### LENGTH()

`LENGTH(coll) → documentCount`

Determine the amount of documents in a collection.

It calls [`COLLECTION_COUNT()`](#collection_count) internally.

- **coll** (collection): a collection (not string)
- returns **documentCount** (number): the total amount of documents in *coll*

`LENGTH()` can also determine the [number of elements](array.md#length) in an array,
the [number of attribute keys](document-object.md#length) of an object / document and
the [character length](string.md#length) of a string.

### SHARD_ID()

`SHARD_ID(collection, shardKeys) → shardId`

Return the shard in a collection that contains the specified shard keys.

- **collection** (string): a collection name
- **shardKeys** (object): a set of shard keys and values. Any missing shard key
  is substituted with the `null` value.
- returns **shardId** (string): the responsible shard for the specified
  shard keys in the given collection. On deployments other than clusters,
  the collection name itself is returned.

```aql
---
name: shard_id1
description: ''
type: cluster
dataset: observationsSampleDataset
---
RETURN SHARD_ID("observations", { "time": "2021-05-25 07:15:00", "subject": "xh458", "val": 10 })
```

## Hash functions

### HASH()

`HASH(value) → hashNumber`

Calculate a hash value for *value*.

- **value** (any): an element of arbitrary type
- returns **hashNumber** (number): a hash value of *value*

*value* is not required to be a string, but can have any data type. The calculated
hash value will take the data type of *value* into account, so for example the
number *1* and the string *"1"* will have different hash values. For arrays the
hash values will be equal if the arrays contain exactly the same values
(including value types) in the same order. For objects the same hash values will
be created if the objects have exactly the same attribute names and values
(including value types). The order in which attributes appear inside objects
is not important for hashing.

The hash value returned by this function is a number. The hash algorithm is not
guaranteed to remain the same in future versions of ArangoDB. The hash values
should therefore be used only for temporary calculations, e.g. to compare if two
documents are the same, or for grouping values in queries.

### MINHASH()

`MINHASH(values, numHashes) → hashes`

Calculate MinHash signatures for the *values* using locality-sensitive hashing.
The result can be used to approximate the Jaccard similarity of sets.

- **values** (array): an array with elements of arbitrary type to hash
- **numHashes** (number): the size of the MinHash signature. Must be
  greater or equal to `1`. The signature size defines the probabilistic error
  (`err = rsqrt(numHashes)`). For an error amount that does not exceed 5%
  (`0.05`), use a size of `1 / (0.05 * 0.05) = 400`.
- returns **hashes** (array): an array of strings with the encoded hash values

**Examples**

```aql
---
name: aqlMinHash
description: ''
---
RETURN MINHASH(["foo", "bar", "baz"], 5)
```

### MINHASH_COUNT()

`MINHASH_COUNT(error) → numHashes`

Calculate the number of hashes (MinHash signature size) needed to not exceed the
specified error amount.

- **error** (number): the probabilistic error you can tolerate in the range `[0, 1)`
- returns **numHashes** (number): the required number of hashes to not exceed
  the specified error amount

**Examples**

```aql
---
name: aqlMinHashCount
description: ''
---
RETURN MINHASH_COUNT(0.05)
```

### MINHASH_ERROR()

`MINHASH_ERROR(numHashes) → error`

Calculate the error amount based on the number of hashes (MinHash signature size).

- **numHashes** (number): the number of hashes you want to check
- returns **error** (number): the probabilistic error to expect with the specified
  number of hashes

**Examples**

```aql
---
name: aqlMinHashError
description: ''
---
RETURN MINHASH_ERROR(400)
```

### String-based hashing

See the following string functions:

- [`CRC32()`](string.md#crc32)
- [`FNV64()`](string.md#fnv64)
- [`MD5()`](string.md#md5)
- [`SHA1()`](string.md#sha1)
- [`SHA512()`](string.md#sha512)

## Function calling

### APPLY()

`APPLY(functionName, arguments) → retVal`

Dynamically call the function *funcName* with the arguments specified.
Arguments are given as array and are passed as separate parameters to
the called function.

Both built-in and user-defined functions can be called. 

- **funcName** (string): a function name
- **arguments** (array, *optional*): an array with elements of arbitrary type
- returns **retVal** (any): the return value of the called function

```aql
APPLY( "SUBSTRING", [ "this is a test", 0, 7 ] )
// "this is"
```

### CALL()

`CALL(funcName, arg1, arg2, ... argN) → retVal`

Dynamically call the function *funcName* with the arguments specified.
Arguments are given as multiple parameters and passed as separate
parameters to the called function.

Both built-in and user-defined functions can be called.

- **funcName** (string): a function name
- **args** (any, *repeatable*): an arbitrary number of elements as
  multiple arguments, can be omitted
- returns **retVal** (any): the return value of the called function

```aql
CALL( "SUBSTRING", "this is a test", 0, 4 )
// "this"
```

## Other functions

### ASSERT() / WARN()

`ASSERT(expr, message) → retVal`\
`WARN(expr, message) → retVal`

The two functions evaluate an expression. In case the expression evaluates to
*true* both functions will return *true*. If the expression evaluates to
*false* *ASSERT* will throw an error and *WARN* will issue a warning and return
*false*. This behavior allows the use of *ASSERT* and *WARN* in `FILTER`
conditions.

- **expr** (expression): AQL expression to be evaluated
- **message** (string): message that will be used in exception or warning if expression evaluates to false
- returns **retVal** (bool): returns true if expression evaluates to true

```aql
FOR i IN 1..3 FILTER ASSERT(i > 0, "i is not greater 0") RETURN i
FOR i IN 1..3 FILTER WARN(i < 2, "i is not smaller 2") RETURN i
```

### IN_RANGE()

`IN_RANGE(value, low, high, includeLow, includeHigh) → included`

Returns true if *value* is greater than (or equal to) *low* and less than
(or equal to) *high*. The values can be of different types. They are compared
as described in [Type and value order](../fundamentals/type-and-value-order.md) and
is thus identical to the comparison operators `<`, `<=`, `>` and `>=` in
behavior.

- **value** (any): an element of arbitrary type
- **low** (any): minimum value of the desired range
- **high** (any): maximum value of the desired range
- **includeLow** (bool): whether the minimum value shall be included in
  the range (left-closed interval) or not (left-open interval)
- **includeHigh** (bool): whether the maximum value shall be included in
  the range (right-closed interval) or not (right-open interval)
- returns **included** (bool): whether *value* is in the range

If *low* and *high* are the same, but *includeLow* and/or *includeHigh* is set
to `false`, then nothing will match. If *low* is greater than *high* nothing will
match either.

{{< info >}}
The regular `IN_RANGE()` function cannot utilize indexes, unlike its
ArangoSearch counterpart which can use the View index.
{{< /info >}}

```aql
---
name: aqlMiscInRange_1
description: ''
---
LET value = 4
RETURN IN_RANGE(value, 3, 5, true, true)
/* same as:
   RETURN value >= 3 AND value <= 5
*/
```

<!-- separator -->

```aql
---
name: aqlMiscInRange_2
description: ''
---
FOR value IN 2..6
  RETURN { value, in_range: IN_RANGE(value, 3, 5, false, true) }
  /* same as:
     RETURN { value, in_range: value > 3 AND value <= 5 }
  */
```

<!-- separator -->

```aql
---
name: aqlMiscInRange_3
description: ''
---
LET coll = [
  { text: "fennel" },
  { text: "fox grape" },
  { text: "forest strawberry" },
  { text: "fungus" }
]
FOR doc IN coll
  FILTER IN_RANGE(doc.text,"fo", "fp", true, false) // values with prefix "fo"
  /* same as:
     FILTER doc.text >= "fo" AND doc.text < "fp"
  */
  RETURN doc
```

### PREGEL_RESULT()

`PREGEL_RESULT(jobId, withId) → results`

Allows to access results of a Pregel job that are only held in memory.
See [Pregel AQL integration](../../data-science/pregel/_index.md#aql-integration).

- **jobId** (string): the `id` of a Pregel job
- **withId** (bool): if enabled, then the document `_id` is returned in
  addition to the `_key` for each vertex
- returns **results** (array): an array of objects, one element per vertex, with
  the attributes computed by the Pregel algorithm and the document key (and
  optionally identifier)

## Internal functions

The following functions are used during development of ArangoDB as a database
system, primarily for unit testing. They are not intended to be used by end
users, especially not in production environments.

### FAIL()

`FAIL(reason)`

Let a query fail on purpose. Can be used in a conditional branch, or to verify
if lazy evaluation / short circuiting is used for instance.

- **reason** (string): an error message
- returns nothing, because the query is aborted

```aql
RETURN 1 == 1 ? "okay" : FAIL("error") // "okay"
RETURN 1 == 1 || FAIL("error") ? true : false // true
RETURN 1 == 2 && FAIL("error") ? true : false // false
RETURN 1 == 1 && FAIL("error") ? true : false // aborted with error
```

### NOOPT() / NOEVAL()

`NOOPT(value) → retVal`

No-operation that prevents certain query compile-time and run-time optimizations. 
Constant expressions can be forced to be evaluated at runtime with this.
This function is marked as non-deterministic so its argument withstands
query optimization.

`NOEVAL(value) → retVal`

Same as `NOOPT()`, except that it is marked as deterministic.

There is no need to call these functions explicitly, they are mainly used for
internal testing.

- **value** (any): a value of arbitrary type
- returns **retVal** (any): *value*

```aql
// differences in execution plan (explain)
FOR i IN 1..3 RETURN (1 + 1)       // const assignment
FOR i IN 1..3 RETURN NOOPT(1 + 1)  // simple expression
FOR i IN 1..3 RETURN NOEVAL(1 + 1) // simple expression

RETURN NOOPT( 123 ) // evaluates 123 at runtime
RETURN NOOPT( CONCAT("a", "b") ) // evaluates concatenation at runtime
```

### PASSTHRU()

`PASSTHRU(value) → retVal`

Simply returns its call argument unmodified. There is no need to call this function 
explicitly, it is mainly used for internal testing.

- **value** (any): a value of arbitrary type
- returns **retVal** (any): *value*

### SCHEMA_GET()

`SCHEMA_GET(collection) → schema`

Return the schema definition as defined in the properties of the
specified collection.

- **collection** (string): name of a collection
- returns **schema** (object): schema definition object

```aql
RETURN SCHEMA_GET("myColl")
```

### SCHEMA_VALIDATE()

`SCHEMA_VALIDATE(doc, schema) → result`

Test if the given document is valid according to the schema definition.

- **doc** (doc): document
- **schema** (object): schema definition object
- returns **result** (object): an object with the following attributes:
  - **valid** (bool): `true` if the document fulfills the schema's requirements,
    otherwise it will be `false` and *errorMessage* will be set
  - **errorMessage** (string): details about the validation failure

If the input document **doc** is not an object, the function will return
a *null* value and register a warning in the query.

Using an empty **schema** object is equivalent to specifying a **schema**
value of *null*, which will make all input objects successfully pass the 
validation.

### SLEEP()

`SLEEP(seconds) → null`

Wait for a certain amount of time before continuing the query.

- **seconds** (number): amount of time to wait
- returns a *null* value

```aql
SLEEP(1)    // wait 1 second
SLEEP(0.02) // wait 20 milliseconds
```

### V8()

`V8(expression) → retVal`

No-operation that enforces the usage of the V8 JavaScript engine. There is
no need to call this function explicitly, it is mainly used for internal
testing.

- **expression** (any): arbitrary expression
- returns **retVal** (any): the return value of the *expression*

```aql
// differences in execution plan (explain)
FOR i IN 1..3 RETURN (1 + 1)          // const assignment
FOR i IN 1..3 RETURN V8(1 + 1)        // simple expression
```

### VERSION()

`VERSION() → serverVersion`

Returns the server version as a string. In a cluster, returns the version
of the Coordinator.

- returns **serverVersion** (string): the server version string

```aql
RETURN VERSION()        // e.g. "3.10.0" 
```
