---
title: HTTP interface for databases
menuTitle: Databases
weight: 20
description: >-
  The HTTP API for databases lets you create and delete databases, list
  available databases, and get information about specific databases
---
The HTTP interface for databases provides operations to create and drop
individual databases. These are mapped to the standard `POST` and `DELETE`
HTTP methods. There is also the `GET` method to retrieve an array of existing
databases.

{{< info >}}
All database management operations can only be accessed via the default
`_system` database and none of the other databases.
{{< /info >}}

## Addresses of databases

Any operation triggered via ArangoDB's RESTful HTTP API is executed in the
context of exactly one database. The database name is read from the first part
of the request URI path (e.g. `/_db/mydb/...`). If the request URI does not
contain a database name, it defaults to `/_db/_system`.

To explicitly specify the database in a request, the request URI must contain
the database name at the beginning of the path:

```
http://localhost:8529/_db/mydb/...
```

The `...` placeholder is the actual path to the accessed resource. In the example,
the resource is accessed in the context of the `mydb` database. Actual URLs in
the context of `mydb` could look like this:

```
http://localhost:8529/_db/mydb/_api/version
http://localhost:8529/_db/mydb/_api/document/test/12345
http://localhost:8529/_db/mydb/myapp/get
```

{{< info >}}
Database management operations like listing, creating, and dropping databases
can only be executed with the `_system` database as the context.
{{< /info >}}
 
Special characters in database names must be properly URL-encoded, e.g.
`a + b = c` needs to be encoded as `a%20%2B%20b%20%3D%20c`:

```
http://localhost:8529/_db/a%20%2B%20b%20%3D%20c/_api/version
```

Database names containing Unicode must be properly
[NFC-normalized](https://en.wikipedia.org/wiki/Unicode_equivalence#Normal_forms).
Non-NFC-normalized names are rejected by the server.

## Manage databases

### Get information about the current database

```openapi
paths:
  /_db/{database-name}/_api/database/current:
    get:
      operationId: getCurrentDatabase
      description: |
        Retrieves the properties of the current database

        The response is a JSON object with the following attributes:

        - `name`: the name of the current database
        - `id`: the id of the current database
        - `path`: the filesystem path of the current database
        - `isSystem`: whether or not the current database is the `_system` database
        - `sharding`: the default sharding method for collections created in this database
        - `replicationFactor`: the default replication factor for collections in this database
        - `writeConcern`: the default write concern for collections in this database
      parameters:
        - name: database-name
          in: path
          required: true
          example: _system
          description: |
            The name of the database.
          schema:
            type: string
      responses:
        '200':
          description: |
            is returned if the information was retrieved successfully.
        '400':
          description: |
            is returned if the request is invalid.
        '404':
          description: |
            is returned if the database could not be found.
      tags:
        - Databases
```

**Examples**

```curl
---
description: ''
name: RestDatabaseGetInfo
---
var url = "/_api/database/current";
var response = logCurlRequest('GET', url);

assert(response.code === 200);

logJsonResponse(response);
```

### List the accessible databases

```openapi
paths:
  /_db/{database-name}/_api/database/user:
    get:
      operationId: listUserAccessibleDatabases
      description: |
        Retrieves the list of all databases the current user can access without
        specifying a different username or password.
      parameters:
        - name: database-name
          in: path
          required: true
          example: _system
          description: |
            The name of a database. Which database you use doesn't matter as long
            as the user account you authenticate with has at least read access
            to this database.
          schema:
            type: string
      responses:
        '200':
          description: |
            is returned if the list of database was compiled successfully.
        '400':
          description: |
            is returned if the request is invalid.
      tags:
        - Databases
```

**Examples**

```curl
---
description: ''
name: RestDatabaseGetUser
---
var url = "/_api/database/user";
var response = logCurlRequest('GET', url);

assert(response.code === 200);

logJsonResponse(response);
```

### List all databases

```openapi
paths:
  /_db/_system/_api/database:
    get:
      operationId: listDatabases
      description: |
        Retrieves the list of all existing databases

        {{</* info */>}}
        Retrieving the list of databases is only possible from within the `_system` database.
        {{</* /info */>}}
      responses:
        '200':
          description: |
            is returned if the list of database was compiled successfully.
        '400':
          description: |
            is returned if the request is invalid.
        '403':
          description: |
            is returned if the request was not executed in the `_system` database.
      tags:
        - Databases
```

**Examples**

```curl
---
description: ''
name: RestDatabaseGet
---
var url = "/_db/_system/_api/database";
var response = logCurlRequest('GET', url);

assert(response.code === 200);

logJsonResponse(response);
```

### Create a database

```openapi
paths:
  /_db/_system/_api/database:
    post:
      operationId: createDatabase
      description: |
        Creates a new database.

        The response is a JSON object with the attribute `result` set to `true`.

        {{</* info */>}}
        Creating a new database is only possible from within the `_system` database.
        {{</* /info */>}}
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required:
                - name
              properties:
                name:
                  description: |
                    Has to contain a valid database name. The name must conform to the selected
                    naming convention for databases. If the name contains Unicode characters, the
                    name must be [NFC-normalized](https://en.wikipedia.org/wiki/Unicode_equivalence#Normal_forms).
                    Non-normalized names will be rejected by arangod.
                  type: string
                options:
                  description: |
                    Optional object which can contain the following attributes:
                  type: object
                  properties:
                    sharding:
                      description: |
                        The sharding method to use for new collections in this database. Valid values
                        are: "", "flexible", or "single". The first two are equivalent. _(cluster only)_
                      type: string
                    replicationFactor:
                      description: |
                        Default replication factor for new collections created in this database.
                        Special values include "satellite", which will replicate the collection to
                        every DB-Server, and 1, which disables replication.
                        _(cluster only)_
                      type: integer
                    writeConcern:
                      description: |
                        Default write concern for new collections created in this database.
                        It determines how many copies of each shard are required to be
                        in sync on the different DB-Servers. If there are less than these many copies
                        in the cluster, a shard refuses to write. Writes to shards with enough
                        up-to-date copies succeed at the same time, however. The value of
                        `writeConcern` cannot be greater than `replicationFactor`.

                        For SatelliteCollections, the `writeConcern` is automatically controlled to
                        equal the number of DB-Servers and has a value of `0`.
                        Otherwise, the default value is controlled by the `--cluster.write-concern`
                        startup option, which defaults to `1`. _(cluster only)_
                      type: number
                users:
                  description: |
                    An array of user objects. The users will be granted *Administrate* permissions
                    for the new database. Users that do not exist yet will be created.
                    If `users` is not specified or does not contain any users, the default user
                    `root` will be used to ensure that the new database will be accessible after it
                    is created. The `root` user is created with an empty password should it not
                    exist. Each user object can contain the following attributes:
                  type: array
                  items:
                    type: object
                    required:
                      - username
                    properties:
                      username:
                        description: |
                          Login name of an existing user or one to be created.
                        type: string
                      passwd:
                        description: |
                          The user password as a string. If not specified, it will default to an empty
                          string. The attribute is ignored for users that already exist.
                        type: string
                      active:
                        description: |
                          A flag indicating whether the user account should be activated or not.
                          The default value is `true`. If set to `false`, then the user won't be able to
                          log into the database. The default is `true`. The attribute is ignored for users
                          that already exist.
                        type: boolean
                      extra:
                        description: |
                          A JSON object with extra user information. It is used by the web interface
                          to store graph viewer settings and saved queries. Should not be set or
                          modified by end users, as custom attributes will not be preserved.
                        type: object
      responses:
        '201':
          description: |
            is returned if the database was created successfully.
        '400':
          description: |
            is returned if the request parameters are invalid, if a database with the
            specified name already exists, or if the configured limit to the number
            of databases has been reached.
        '403':
          description: |
            is returned if the request was not executed in the `_system` database.
        '409':
          description: |
            is returned if a database with the specified name already exists.
      tags:
        - Databases
```

**Examples**

```curl
---
description: |-
  Creating a database named `example`.
name: RestDatabaseCreate
---
var url = "/_db/_system/_api/database";
var name = "example";
try {
  db._dropDatabase(name);
}
catch (err) {
}

var data = {
  name: name,
  options: {
    sharding: "flexible",
    replicationFactor: 3
  }
};
var response = logCurlRequest('POST', url, data);

db._dropDatabase(name);
assert(response.code === 201);

logJsonResponse(response);
```

```curl
---
description: |-
  Creating a database named `mydb` with two users, flexible sharding and
  default replication factor of 3 for collections that will be part of
  the newly created database.
name: RestDatabaseCreateUsers
---
var url = "/_db/_system/_api/database";
var name = "mydb";
try {
  db._dropDatabase(name);
}
catch (err) {
}

var data = {
  name: name,
  users: [
    {
      username: "admin",
      passwd: "secret",
      active: true
    },
    {
      username: "tester",
      passwd: "test001",
      active: false
    }
  ]
};
var response = logCurlRequest('POST', url, data);

db._dropDatabase(name);
assert(response.code === 201);

logJsonResponse(response);
```

### Drop a database

```openapi
paths:
  /_db/_system/_api/database/{database-name}:
    delete:
      operationId: deleteDatabase
      description: |
        Drops the database along with all data stored in it.

        {{</* info */>}}
        Dropping a database is only possible from within the `_system` database.
        The `_system` database itself cannot be dropped.
        {{</* /info */>}}
      parameters:
        - name: database-name
          in: path
          required: true
          description: |
            The name of the database
          schema:
            type: string
      responses:
        '200':
          description: |
            is returned if the database was dropped successfully.
        '400':
          description: |
            is returned if the request is malformed.
        '403':
          description: |
            is returned if the request was not executed in the `_system` database.
        '404':
          description: |
            is returned if the database could not be found.
      tags:
        - Databases
```

**Examples**

```curl
---
description: ''
name: RestDatabaseDrop
---
var url = "/_db/_system/_api/database";
var name = "example";

db._createDatabase(name);
var response = logCurlRequest('DELETE', url + '/' + name);

assert(response.code === 200);

logJsonResponse(response);
```
