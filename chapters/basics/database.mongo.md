# MongoDB — Database Commands Reference

> **Project:** Task Management Application
> **Scope:** Database-level commands only
> **Shell:** `mongosh` (MongoDB Shell)
> **Next:** `Collection.md` → collection-level commands

---

## Table of Contents

1. [Connection](#1-connection)
2. [Create & Select Database](#2-create--select-database)
3. [Info & Stats](#3-info--stats)
4. [Collections](#4-collections)
5. [Drop & Repair](#5-drop--repair)
6. [Users & Roles](#6-users--roles)
7. [Profiler](#7-profiler)
8. [Logging](#8-logging)
9. [Backup & Restore](#9-backup--restore)
10. [Misc Utilities](#10-misc-utilities)

---

## 1. Connection

```js
// Connect to MongoDB server (default: localhost:27017)
mongosh

// Connect to a remote MongoDB server with URI
mongosh "mongodb://localhost:27017"

// Connect with authentication credentials
mongosh "mongodb://admin:password@localhost:27017"

// Connect using a full connection string (Atlas / remote)
mongosh "mongodb+srv://admin:password@cluster0.mongodb.net/taskmanager"
```

---

## 2. Create & Select Database

```js
// Switch to a database — MongoDB is lazy, creates it only on first write
use taskmanager

// Show the currently selected database
db

// Show all databases — only lists DBs that have at least one collection with data
show dbs
```

> 💡 **Key concept:** `use taskmanager` does NOT create the DB immediately.
> MongoDB creates it only when you insert the first document.

---

## 3. Info & Stats

```js
// Get full statistics of the current DB (size, collections, indexes, etc.)
db.stats();

// Get the name of the current database programmatically
db.getName();

// Get the MongoDB server version
db.version();

// Get server status (connections, memory, uptime, opcounters, etc.)
db.serverStatus();

// Get current host info (hostname, OS, CPU, memory)
db.hostInfo();

// List all commands available on the current database
db.listCommands();

// Run a raw database-level admin command
db.runCommand({ connectionStatus: 1 });

// Check replication status — useful in replica sets
db.adminCommand({ replSetGetStatus: 1 });

// Get the current operation being executed on the server
db.currentOp();

// Kill a specific running operation by its opid
db.killOp(12345);
```

---

## 4. Collections

### List Collections

```js
// List all collections in the current database
show collections

// Programmatic way — returns collection names as an array
db.getCollectionNames()

// Get full collection info including options and UUID
db.getCollectionInfos()

// Get info for a specific collection by name filter
db.getCollectionInfos({ name: "tasks" })
```

### Create Collection

```js
// Explicitly create a collection (optional — MongoDB auto-creates on insert)
db.createCollection("tasks");
```

### Create Capped Collection

```js
// Capped = fixed-size collection — old docs auto-deleted when full (circular overwrite)
db.createCollection("activity_logs", {
  capped: true, // Enables fixed-size mode
  size: 10485760, // Max size in bytes (10 MB)
  max: 5000, // Max number of documents allowed
});
```

> 💡 Use capped collections for **logs, audit trails, and activity feeds** where you only care about recent data.

### Create Collection with Schema Validation

```js
// Enforce document structure at the DB level using JSON Schema
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email", "role"], // These fields MUST exist
      properties: {
        name: { bsonType: "string", description: "User full name — required" },
        email: { bsonType: "string", description: "User email — required" },
        role: {
          bsonType: "string",
          enum: ["admin", "manager", "developer", "viewer"],
        },
      },
    },
  },
  validationLevel: "strict", // Apply validation on ALL inserts and updates
  validationAction: "error", // Reject documents that fail (use "warn" to only log)
});
```

| `validationLevel` | Meaning                                       |
| ----------------- | --------------------------------------------- |
| `strict`          | Validate all inserts & updates                |
| `moderate`        | Validate inserts + updates to valid docs only |

| `validationAction` | Meaning                       |
| ------------------ | ----------------------------- |
| `error`            | Reject the document (default) |
| `warn`             | Allow but log the violation   |

### Rename & Drop Collection

```js
// Rename an existing collection
db.tasks.renameCollection("archived_tasks");

// Drop a collection and permanently delete all its documents
db.tasks.drop();
```

---

## 5. Drop & Repair

```js
// ⚠️  Drop the entire current database — IRREVERSIBLE, deletes everything
db.dropDatabase();

// Repair the database and reclaim disk space (run on admin db)
db.repairDatabase();
```

> ⚠️ **Warning:** `dropDatabase()` is permanent. Always take a backup first.

---

## 6. Users & Roles

### Create User

```js
db.createUser({
  user: "taskapp_user",
  pwd: "StrongPassword@123",
  roles: [
    { role: "readWrite", db: "taskmanager" }, // Read + write on taskmanager
    { role: "read", db: "reporting" }, // Read-only on reporting DB
  ],
});
```

### Manage Users

```js
// View all users created in the current database
db.getUsers();

// Update a user's password
db.updateUser("taskapp_user", {
  pwd: "NewPassword@456",
});

// Delete a user from the current database
db.dropUser("taskapp_user");
```

### Manage Roles

```js
// Grant additional roles to a user
db.grantRolesToUser("taskapp_user", [
  { role: "dbAdmin", db: "taskmanager" }, // Grants schema/index management
]);

// Revoke roles from a user
db.revokeRolesFromUser("taskapp_user", [{ role: "read", db: "reporting" }]);

// Show all built-in roles available in MongoDB
db.getRoles({ showBuiltinRoles: true });
```

### Built-in Role Reference

| Role              | Access Level                           |
| ----------------- | -------------------------------------- |
| `read`            | Read-only on a database                |
| `readWrite`       | Read + write on a database             |
| `dbAdmin`         | Schema, indexes, collection management |
| `userAdmin`       | Create and manage users & roles        |
| `clusterAdmin`    | Full cluster management                |
| `readAnyDatabase` | Read on all databases                  |
| `root`            | Superuser — full access to everything  |

---

## 7. Profiler

> The profiler helps you find **slow queries** and optimize performance.

```js
// Check current profiler status (level + slowms threshold)
db.getProfilingStatus();

// Set profiler level:
//   0 = off
//   1 = log slow queries only (recommended for production)
//   2 = log ALL queries (use only for debugging)
db.setProfilingLevel(1, { slowms: 100 }); // Log queries slower than 100ms

// Read the 10 most recent profiler entries (sorted by timestamp descending)
db.system.profile.find().sort({ ts: -1 }).limit(10);
```

> 💡 **Tip:** Use level `1` in production — level `2` logs everything and can hurt performance.

---

## 8. Logging

```js
// Get the current log verbosity level per component
db.adminCommand({ getLog: "global" });

// Set global log verbosity (0 = default, 1–5 = increasingly verbose)
db.adminCommand({
  setParameter: 1,
  logLevel: 1,
});
```

---

## 9. Backup & Restore

> ⚠️ Run these in your **terminal**, NOT inside `mongosh`.

```bash
# Export entire database to a folder
mongodump --uri="mongodb://localhost:27017" --db=taskmanager --out=./backup

# Restore a database from a backup folder
mongorestore --uri="mongodb://localhost:27017" --db=taskmanager ./backup/taskmanager

# Export a single collection to a JSON file
mongoexport --db=taskmanager --collection=tasks --out=tasks.json

# Import a JSON file into a collection
mongoimport --db=taskmanager --collection=tasks --file=tasks.json
```

| Tool           | Purpose                                     |
| -------------- | ------------------------------------------- |
| `mongodump`    | Binary backup of an entire DB or collection |
| `mongorestore` | Restore from a `mongodump` backup           |
| `mongoexport`  | Export collection as JSON / CSV             |
| `mongoimport`  | Import JSON / CSV into a collection         |

---

## 10. Misc Utilities

```js
// Check if there was an error on the last operation
db.getPrevError();

// Reset the last error state
db.resetError();

// Ping the database — confirm connection is alive (returns { ok: 1 })
db.adminCommand({ ping: 1 });

// Force a checkpoint — flushes all pending writes to disk
db.adminCommand({ fsync: 1 });

// Get MongoDB server build info (version, OS, modules, architecture)
db.adminCommand({ buildInfo: 1 });

// List all active sessions on the server
db.adminCommand({ listSessions: 1 });
```

---

## Quick Command Cheatsheet

| Command                        | What it does                   |
| ------------------------------ | ------------------------------ |
| `use <db>`                     | Switch / create database       |
| `show dbs`                     | List all databases             |
| `db.stats()`                   | Database size and stats        |
| `db.dropDatabase()`            | ⚠️ Delete entire database      |
| `show collections`             | List all collections           |
| `db.createCollection()`        | Explicitly create a collection |
| `db.getCollectionNames()`      | Get collection names as array  |
| `db.createUser()`              | Add a new user                 |
| `db.getUsers()`                | List all users                 |
| `db.setProfilingLevel()`       | Enable slow query logging      |
| `db.adminCommand({ ping: 1 })` | Check DB connection            |
| `mongodump`                    | Backup database (terminal)     |
| `mongorestore`                 | Restore database (terminal)    |

---

> **Next →** [`Collection.md`](./Collection.md) — Collection-level commands (find, insert, update, delete, indexes, aggregation)
