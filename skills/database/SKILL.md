---
name: ink-database
description: Create and manage SQLite databases on Ink (ml.ink). Use when the user wants to create a database, manage data, or connect a database to a service.
argument-hint: "[database-name]"
---

# Ink Databases

Help the user create and manage SQLite databases on Ink (ml.ink).

## Creating a database

### 1. Create the resource

```
mcp__ink__create_resource(name: "<database-name>")
```

Use `$ARGUMENTS` as the name if provided, otherwise ask the user or derive from the project name.

### 2. Get connection details

```
mcp__ink__get_resource(name: "<database-name>")
```

This returns:
- **connectionString**: The URL to connect to the database
- **authToken**: Authentication token for the connection

### 3. Connect to a service

To use the database from a deployed service, add the connection details as environment variables:

```
mcp__ink__update_service(name: "<service>", envVars: [
  { key: "DATABASE_URL", value: "<connectionString>" },
  { key: "DATABASE_AUTH_TOKEN", value: "<authToken>" }
])
```

### 4. Client libraries

Recommend the appropriate client library based on the project:

- **Node.js**: `@libsql/client`
  ```js
  import { createClient } from "@libsql/client";
  const db = createClient({
    url: process.env.DATABASE_URL,
    authToken: process.env.DATABASE_AUTH_TOKEN,
  });
  ```

- **Python**: `libsql-experimental`
  ```python
  import libsql_experimental as libsql
  conn = libsql.connect(
    os.environ["DATABASE_URL"],
    auth_token=os.environ["DATABASE_AUTH_TOKEN"]
  )
  ```

## Listing databases

```
mcp__ink__list_resources
```

## Deleting a database

```
mcp__ink__delete_resource(name: "<database-name>")
```

Warn the user that this is destructive and cannot be undone.
