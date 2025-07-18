---
title: HTTP interface for data import
menuTitle: Import
weight: 70
description: >-
  The Import HTTP API allows you to load JSON data in bulk into ArangoDB
---
## Import JSON data as documents

```openapi
paths:
  /_db/{database-name}/_api/import:
    post:
      operationId: importData
      description: |
        Load JSON data and store it as documents into the specified collection.

        If you import documents into edge collections, all documents require a `_from`
        and a `_to` attribute.
      requestBody:
        content:
          'text/plain; charset=utf-8':
            schema:
              description: |
                The request body can have different JSON formats depending on
                the `type` parameter:
                - One JSON object per line (JSONL)
                - A JSON array of objects
                - One JSON array per line (CSV-like)
      parameters:
        # Purposefully undocumented:
        #   line (compensate for arangoimport --skip-lines in error messages)
        - name: database-name
          in: path
          required: true
          example: _system
          description: |
            The name of the database.
          schema:
            type: string
        - name: collection
          in: query
          required: true
          description: |
            The name of the target collection. The collection needs to exist already.
          schema:
            type: string
        - name: type
          in: query
          required: false
          description: |
            Determines how the body of the request is interpreted.

            - `documents`: JSON Lines (JSONL) format. Each line is expected to be one
              JSON object.

              Example:

              ```json
              {"_key":"john","name":"John Smith","age":35}
              {"_key":"katie","name":"Katie Foster","age":28}
              ```

            - `array` (or `list`): JSON format. The request body is expected to be a
              JSON array of objects. This format requires ArangoDB to parse the complete
              array and keep it in memory for the duration of the import. This is more
              resource-intensive than the line-wise JSONL processing.

              Any whitespace outside of strings is ignored, which means the JSON data can be
              a single line or be formatted as multiple lines.

              Example:

              ```json
              [
                {"_key":"john","name":"John Smith","age":35},
                {"_key":"katie","name":"Katie Foster","age":28}
              ]
              ```

            - `auto`: automatically determines the type (either `documents` or `array`).

            - Omit the `type` parameter entirely (or set it to an empty string)
              to import JSON arrays of tabular data, similar to CSV.

              The first line is an array of strings that defines the attribute keys. The
              subsequent lines are arrays with the attribute values. The keys and values
              are matched by the order of the array elements.

              Example:

              ```json
              ["_key","name","age"]
              ["john","John Smith",35]
              ["katie","Katie Foster",28]
              ```
          schema:
            type: string
            enum: ["", documents, array, auto]
            default: ""
        - name: ignoreMissing
          in: query
          required: false
          description: |
            When importing JSON arrays of tabular data (`type` parameter is omitted),
            the first line of the request body defines the attribute keys and the
            subsequent lines the attribute values for each document. Subsequent lines
            with a different number of elements than the first line are not imported
            by default.

            ```js
            ["attr1", "attr2"]
            [1, 2]     // matching number of elements
            [1]        // misses 2nd element
            [1, 2, 3]  // excess 3rd element
            ```

            You can enable this option to import them anyway. For the missing elements,
            the document attributes are omitted. Excess elements are ignored.
          schema:
            type: boolean
            default: false
        - name: fromPrefix
          in: query
          required: false
          description: |
            The collection name prefix to prepend to all values in the `_from`
            attribute that only specify a document key.
          schema:
            type: string
        - name: toPrefix
          in: query
          required: false
          description: |
            The collection name prefix to prepend to all values in the `_to`
            attribute that only specify a document key.
          schema:
            type: string
        - name: overwriteCollectionPrefix
          in: query
          required: false
          description: |
            Force the `fromPrefix` and `toPrefix`, possibly replacing existing
            collection name prefixes.
          schema:
            type: boolean
            default: false
        - name: overwrite
          in: query
          required: false
          description: |
            If enabled, then all data in the collection is removed prior to the
            import. Any existing index definitions are preserved.
          schema:
            type: boolean
            default: false
        - name: waitForSync
          in: query
          required: false
          description: |
            Wait until documents have been synced to disk before returning.
          schema:
            type: boolean
            default: false
        - name: onDuplicate
          in: query
          required: false
          description: |
            Controls what action is carried out in case of a unique key constraint
            violation.

            - `error`: this will not import the current document because of the unique
              key constraint violation. This is the default setting.
            - `update`: this will update an existing document in the database with the
              data specified in the request. Attributes of the existing document that
              are not present in the request will be preserved.
            - `replace`: this will replace an existing document in the database with the
              data specified in the request.
            - `ignore`: this will not update an existing document and simply ignore the
              error caused by a unique key constraint violation.

            Note that `update`, `replace` and `ignore` will only work when the
            import document in the request contains the `_key` attribute. `update` and
            `replace` may also fail because of secondary unique key constraint violations.
          schema:
            type: string
            enum: [error, update, replace, ignore]
            default: error
        - name: complete
          in: query
          required: false
          description: |
            If set to `true`, the whole import fails if any error occurs. Otherwise, the
            import continues even if some documents are invalid and cannot be imported,
            skipping the problematic documents.
          schema:
            type: boolean
            default: false
        - name: details
          in: query
          required: false
          description: |
            If set to `true`, the result includes a `details` attribute with information
            about documents that could not be imported.
          schema:
            type: boolean
            default: false
      responses:
        '201':
          description: |
            is returned if all documents could be imported successfully.

            The response is a JSON object with the following attributes:
          content:
            application/json:
              schema:
                type: object
                required:
                  - created
                  - errors
                  - empty
                  - updated
                  - ignored
                properties:
                  created:
                    description: |
                      The number of imported documents.
                    type: integer
                  errors:
                    description: |
                      The number of documents that were not imported due to errors.
                    type: integer
                  empty:
                    description: |
                      The number of empty lines found in the input. Only greater than zero for the
                      types `documents` and `auto`.
                    type: integer
                  updated:
                    description: |
                      The number of updated/replaced documents. Only greater than zero if `onDuplicate`
                      is set to either `update` or `replace`.
                    type: integer
                  ignored:
                    description: |
                      The number of failed but ignored insert operations. Only greater than zero if
                      `onDuplicate` is set to `ignore`.
                    type: integer
                  details:
                    description: |
                      An array with the error messages caused by documents that could not be imported.
                      Only present if `details` is set to `true`.
                    type: array
                    items:
                      type: string
        '400':
          description: |
            The `type` contains an invalid value, no `collection` is
            specified, the documents are incorrectly encoded, or the request
            is malformed.
        '404':
          description: |
            The `collection` parameter or the `_from` or `_to` attributes of an
            imported edge refer to an unknown collection.
        '409':
          description: |
            The `complete` option is enabled and the import triggers a
            unique key violation.
        '500':
          description: |
            The `complete` option is enabled and the input is invalid,
            or the server cannot auto-generate a document key (out of keys error)
            for a document with no user-defined key.
      tags:
        - Import
```

**Examples**

```curl
---
description: |-
  Importing documents with heterogenous attributes from an array of JSON objects:
name: RestImportJsonList
---
db._flushCache();
var cn = "products";
db._drop(cn);
db._create(cn);
db._flushCache();

var body = [
  { _key: "abc", value1: 25, value2: "test", allowed: true },
  { _key: "foo", name: "baz" },
  { name: { detailed: "detailed name", short: "short name" } }
];

var response = logCurlRequest('POST', "/_api/import?collection=" + cn + "&type=list", body);

assert(response.code === 201);
assert(response.parsedBody.created === 3);
assert(response.parsedBody.errors === 0);
assert(response.parsedBody.empty === 0);

logJsonResponse(response);
db._drop(cn);
```

```curl
---
description: |-
  Importing documents using JSON objects separated by new lines (JSONL):
name: RestImportJsonLines
---
db._flushCache();
var cn = "products";
db._drop(cn);
db._create(cn);
db._flushCache();

var body = '{ "_key": "abc", "value1": 25, "value2": "test",' +
           '"allowed": true }\n' +
           '{ "_key": "foo", "name": "baz" }\n\n' +
           '{ "name": {' +
           ' "detailed": "detailed name", "short": "short name" } }\n';
var response = logCurlRequest('POST', "/_api/import?collection=" + cn + "&type=documents", body);

assert(response.code === 201);
assert(response.parsedBody.created === 3);
assert(response.parsedBody.errors === 0);
assert(response.parsedBody.empty === 1);

logJsonResponse(response);
db._drop(cn);
```

```curl
---
description: |-
  Using the `auto` type detection:
name: RestImportJsonType
---
db._flushCache();
var cn = "products";
db._drop(cn);
db._create(cn);
db._flushCache();

var body = [
  { _key: "abc", value1: 25, value2: "test", allowed: true },
  { _key: "foo", name: "baz" },
  { name: { detailed: "detailed name", short: "short name" } }
];

var response = logCurlRequestRaw('POST', "/_api/import?collection=" + cn + "&type=auto", body);

assert(response.code === 201);
assert(response.parsedBody.created === 3);
assert(response.parsedBody.errors === 0);
assert(response.parsedBody.empty === 0);

logJsonResponse(response);
db._drop(cn);
```

```curl
---
description: |-
  Importing JSONL into an edge collection, with `_from`, `_to` and `name`
  attributes:
name: RestImportJsonEdge
---
db._flushCache();
var cn = "links";
db._drop(cn);
db._createEdgeCollection(cn);
db._drop("products");
db._create("products");
db._flushCache();

var body = '{ "_from": "products/123", "_to": "products/234" }\n' +
           '{"_from": "products/332", "_to": "products/abc", ' +
           '  "name": "other name" }';

var response = logCurlRequestRaw('POST', "/_api/import?collection=" + cn + "&type=documents", body);

assert(response.code === 201);
assert(response.parsedBody.created === 2);
assert(response.parsedBody.errors === 0);
assert(response.parsedBody.empty === 0);

logJsonResponse(response);
db._drop(cn);
db._drop("products");
```

```curl
---
description: |-
  Importing an array of JSON objects into an edge collection,
  omitting `_from` or `_to`:
name: RestImportJsonEdgeInvalid
---
db._flushCache();
var cn = "links";
db._drop(cn);
db._createEdgeCollection(cn);
db._flushCache();

var body = [ { name: "some name" } ];

var response = logCurlRequestRaw('POST', "/_api/import?collection=" + cn + "&type=list&details=true", body);

assert(response.code === 201);
assert(response.parsedBody.created === 0);
assert(response.parsedBody.errors === 1);
assert(response.parsedBody.empty === 0);

logJsonResponse(response);
db._drop(cn);
```

```curl
---
description: |-
  Violating a unique constraint, but allowing partial imports:
name: RestImportJsonUniqueContinue
---
var cn = "products";
db._drop(cn);
db._create(cn);
db._flushCache();

var body = '{ "_key": "abc", "value1": 25, "value2": "test" }\n' +
           '{ "_key": "abc", "value1": "bar", "value2": "baz" }';

var response = logCurlRequestRaw('POST', "/_api/import?collection=" + cn
+ "&type=documents&details=true", body);

assert(response.code === 201);
assert(response.parsedBody.created === 1);
assert(response.parsedBody.errors === 1);
assert(response.parsedBody.empty === 0);

logJsonResponse(response);
db._drop(cn);
```

```curl
---
description: |-
  Violating a unique constraint, not allowing partial imports:
name: RestImportJsonUniqueFail
---
var cn = "products";
db._drop(cn);
db._create(cn);
db._flushCache();

var body = '{ "_key": "abc", "value1": 25, "value2": "test" }\n' +
           '{ "_key": "abc", "value1": "bar", "value2": "baz" }';

var response = logCurlRequestRaw('POST', "/_api/import?collection=" + cn + "&type=documents&complete=true", body);

assert(response.code === 409);

logJsonResponse(response);
db._drop(cn);
```

```curl
---
description: |-
  Using a non-existing collection:
name: RestImportJsonInvalidCollection
---
var cn = "products";
db._drop(cn);

var body = '{ "name": "test" }';

var response = logCurlRequestRaw('POST', "/_api/import?collection=" + cn + "&type=documents", body);

assert(response.code === 404);

logJsonResponse(response);
```

```curl
---
description: |-
  Using a malformed body with an array of JSON objects being expected:
name: RestImportJsonInvalidBody
---
var cn = "products";
db._drop(cn);
db._create(cn);
db._flushCache();

var body = '{ }';

var response = logCurlRequestRaw('POST', "/_api/import?collection=" + cn + "&type=list", body);

assert(response.code === 400);

logJsonResponse(response);
db._drop(cn);
```

```curl
---
description: |-
  Importing two documents using the JSON arrays format. The documents have a
  `_key`, `value1`, and `value2` attribute each. One line in the import data is
  empty and skipped:
name: RestImportCsvExample
---
var cn = "products";
db._drop(cn);
db._create(cn);

var body = '[ "_key", "value1", "value2" ]\n' +
           '[ "abc", 25, "test" ]\n\n' +
           '[ "foo", "bar", "baz" ]';

var response = logCurlRequestRaw('POST', "/_api/import?collection=" + cn, body);

assert(response.code === 201);
assert(response.parsedBody.created === 2);
assert(response.parsedBody.errors === 0);
assert(response.parsedBody.empty === 1);

logJsonResponse(response);
db._drop(cn);
```

```curl
---
description: |-
  Importing JSON arrays into an edge collection, with `_from`, `_to`, and `name`
  attributes:
name: RestImportCsvEdge
---
var cn = "links";
db._drop(cn);
db._createEdgeCollection(cn);
db._drop("products");
db._create("products");

var body = '[ "_from", "_to", "name" ]\n' +
           '[ "products/123","products/234", "some name" ]\n' +
           '[ "products/332", "products/abc", "other name" ]';

var response = logCurlRequestRaw('POST', "/_api/import?collection=" + cn, body);

assert(response.code === 201);
assert(response.parsedBody.created === 2);
assert(response.parsedBody.errors === 0);
assert(response.parsedBody.empty === 0);

logJsonResponse(response);
db._drop(cn);
db._drop("products");
```

```curl
---
description: |-
  Importing JSON arrays into an edge collection, omitting `_from` or `_to`:
name: RestImportCsvEdgeInvalid
---
var cn = "links";
db._drop(cn);
db._createEdgeCollection(cn);

var body = '[ "name" ]\n[ "some name" ]\n[ "other name" ]';

var response = logCurlRequestRaw('POST', "/_api/import?collection=" + cn + "&details=true", body);

assert(response.code === 201);
assert(response.parsedBody.created === 0);
assert(response.parsedBody.errors === 2);
assert(response.parsedBody.empty === 0);

logJsonResponse(response);
db._drop(cn);
```

```curl
---
description: |-
  Using a malformed body with JSON arrays being expected:
name: RestImportCsvInvalidBody
---
var cn = "products";
db._drop(cn);
db._create(cn);

var body = '{ "_key": "foo", "value1": "bar" }';

var response = logCurlRequest('POST', "/_api/import?collection=" + cn, body);

assert(response.code === 400);

logJsonResponse(response);
db._drop(cn);
```
