---
fileID: indexes-working-with
title: Working with Indexes using HTTP
weight: 2095
description: 
layout: default
---
<!-- js/actions/api-index.js -->
## Read index
```http-spec
openapi: 3.0.2
paths:
  /_api/index/{index-id}:
    get:
      description: |2+
        The result is an object describing the index. It has at least the following
        attributes:
        - *id*: the identifier of the index
        - *type*: the index type
        All other attributes are type-dependent. For example, some indexes provide
        *unique* or *sparse* flags, whereas others don't. Some indexes also provide
        a selectivity estimate in the *selectivityEstimate* attribute of the result.
      operationId: ' getIndexes:handle'
      parameters:
      - name: index-id
        schema:
          type: string
        required: true
        description: |+
          The index identifier.
        in: path
      responses:
        '200':
          description: |2
            If the index exists, then a *HTTP 200* is returned.
        '404':
          description: |2
            If the index does not exist, then a *HTTP 404*
            is returned.
      tags:
      - Indexes
```

```curl
---
render: input/output
name: RestIndexPrimaryIndex
release: stable
version: '3.10'
---
    var cn = "products";
    db._drop(cn);
    db._create(cn);
    var url = "/_api/index/" + cn + "/0";
    var response = logCurlRequest('GET', url);
    assert(response.code === 200);
    logJsonResponse(response);
  ~ db._drop(cn);
```


<!-- js/actions/api-index.js -->
##  Create index
```http-spec
openapi: 3.0.2
paths:
  /_api/index#general:
    post:
      description: |2+
        Creates a new index in the collection `collection`. Expects
        an object containing the index details.
        The type of the index to be created must specified in the **type**
        attribute of the index details. Depending on the index type, additional
        other attributes may need to specified in the request in order to create
        the index.
        Indexes require the to be indexed attribute(s) in the **fields** attribute
        of the index details. Depending on the index type, a single attribute or
        multiple attributes can be indexed. In the latter case, an array of
        strings is expected.
        The `.` character denotes sub-attributes in attribute paths. Attributes with
        literal `.` in their name cannot be indexed. Attributes with the name `_id`
        cannot be indexed either, neither as a top-level attribute nor as a sub-attribute.
        Optionally, an index name may be specified as a string in the **name** attribute.
        Index names have the same restrictions as collection names. If no value is
        specified, one will be auto-generated.
        Persistent indexes (including vertex-centric indexes) can be created as unique
        or non-unique variants. Uniqueness can be controlled by specifying the
        **unique** option for the index definition. Setting it to `true` creates a
        unique index. Setting it to `false` or omitting the `unique` attribute creates a
        non-unique index.
        **Note**: Unique indexes on non-shard keys are not supported in a cluster.
        Persistent indexes can optionally be created in a sparse
        variant. A sparse index will be created if the **sparse** attribute in
        the index details is set to `true`. Sparse indexes do not index documents
        for which any of the index attributes is either not set or is `null`.
        The optional **deduplicate** attribute is supported by persistent array indexes.
        It controls whether inserting duplicate index values
        from the same document into a unique array index will lead to a unique constraint
        error or not. The default value is `true`, so only a single instance of each
        non-unique index value will be inserted into the index per document. Trying to
        insert a value into the index that already exists in the index always fails,
        regardless of the value of this attribute.
        The optional **estimates** attribute is supported by persistent indexes.
        This attribute controls whether index selectivity estimates are
        maintained for the index. Not maintaining index selectivity estimates can have
        a slightly positive impact on write performance.
        The downside of turning off index selectivity estimates will be that
        the query optimizer will not be able to determine the usefulness of different
        competing indexes in AQL queries when there are multiple candidate indexes to
        choose from.
        The `estimates` attribute is optional and defaults to `true` if not set. It will
        have no effect on indexes other than persistent indexes.
        The optional attribute **cacheEnabled** is supported by indexes of type
        *persistent*. This attribute controls whether an extra in-memory hash cache is
        created for the index. The hash cache can be used to speed up index lookups.
        The cache can only be used for queries that look up all index attributes via
        an equality lookup (`==`). The hash cache cannot be used for range scans,
        partial lookups or sorting.
        The cache will be populated lazily upon reading data from the index. Writing data
        into the collection or updating existing data will invalidate entries in the
        cache. The cache may have a negative effect on performance in case index values
        are updated more often than they are read.
        The maximum size of cache entries that can be stored is currently 4 MB, i.e.
        the cumulated size of all index entries for any index lookup value must be
        less than 4 MB. This limitation is there to avoid storing the index entries
        of "super nodes" in the cache.
        `cacheEnabled` defaults to `false` and should only be used for indexes that
        are known to benefit from an extra layer of caching.
        The optional attribute **inBackground** can be set to `true` to create the index
        in the background, which will not write-lock the underlying collection for
        as long as if the index is built in the foreground.
      operationId: ' createIndex:general'
      parameters:
      - name: collection
        schema:
          type: string
        required: true
        description: |+
          The collection name.
        in: query
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                index-details:
                  type: object
                  description: |2+
              required:
              - index-details
      responses:
        '200':
          description: |2
            If the index already exists, then an *HTTP 200* is returned.
        '201':
          description: |2
            If the index does not already exist and could be created, then an *HTTP 201*
            is returned.
        '400':
          description: |2
            If an invalid index description is posted or attributes are used that the
            target index will not support, then an *HTTP 400* is returned.
        '404':
          description: |2
            If *collection* is unknown, then an *HTTP 404* is returned.
      tags:
      - Indexes
```



<!-- js/actions/api-index.js -->
##  Delete index
```http-spec
openapi: 3.0.2
paths:
  /_api/index/{index-id}:
    delete:
      description: |2+
        Deletes an index with *index-id*.
      operationId: ' dropIndex'
      parameters:
      - name: index-id
        schema:
          type: string
        required: true
        description: |+
          The index id.
        in: path
      responses:
        '200':
          description: |2
            If the index could be deleted, then an *HTTP 200* is
            returned.
        '404':
          description: |2
            If the *index-id* is unknown, then an *HTTP 404* is returned.
      tags:
      - Indexes
```

```curl
---
render: input/output
name: RestIndexDeleteUniquePersistent
release: stable
version: '3.10'
---
    var cn = "products";
    db._drop(cn);
    db._create(cn);
    var url = "/_api/index/" + db.products.ensureIndex({ type: "persistent", fields: ["a", "b"] }).id;
    var response = logCurlRequest('DELETE', url);
    assert(response.code === 200);
    logJsonResponse(response);
  ~ db._drop(cn);
```


<!-- js/actions/api-index.js -->
##  Read all indexes of a collection
```http-spec
openapi: 3.0.2
paths:
  /_api/index:
    get:
      description: |2+
        Returns an object with an attribute *indexes* containing an array of all
        index descriptions for the given collection. The same information is also
        available in the *identifiers* as an object with the index handles as
        keys.
      operationId: ' getIndexes'
      parameters:
      - name: collection
        schema:
          type: string
        required: true
        description: |+
          The collection name.
        in: query
      - name: withStats
        schema:
          type: boolean
        required: false
        description: |+
          Whether to include figures and estimates in the result.
        in: query
      - name: withHidden
        schema:
          type: boolean
        required: false
        description: |+
          Whether to include hidden indexes in the result.
        in: query
      responses:
        '200':
          description: |2
            returns a JSON object containing a list of indexes on that collection.
      tags:
      - Indexes
```

```curl
---
render: input/output
name: RestIndexAllIndexes
release: stable
version: '3.10'
---
    var cn = "products";
    db._drop(cn);
    db._create(cn);
    db[cn].ensureIndex({ type: "persistent", fields: ["name"] });
    db[cn].ensureIndex({ type: "persistent", fields: ["price"], sparse: true });
    var url = "/_api/index?collection=" + cn;
    var response = logCurlRequest('GET', url);
    assert(response.code === 200);
    logJsonResponse(response);
  ~ db._drop(cn);
```