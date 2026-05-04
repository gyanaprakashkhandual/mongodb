# MongoDB — Getting Started

> **Project:** Task Management Application
> **File:** `GetStarted.md`
> **Purpose:** Understand what MongoDB is, how it's structured, and how to think in documents before writing a single command.

---

## Table of Contents

1. [What is MongoDB?](#1-what-is-mongodb)
2. [SQL vs MongoDB — Terminology](#2-sql-vs-mongodb--terminology)
3. [How MongoDB is Structured](#3-how-mongodb-is-structured)
4. [Architecture Tree](#4-architecture-tree)
5. [What is a Document?](#5-what-is-a-document)
6. [What is a Collection?](#6-what-is-a-collection)
7. [What is a Database?](#7-what-is-a-database)
8. [BSON vs JSON](#8-bson-vs-json)
9. [ObjectId — The Default \_id](#9-objectid--the-default-_id)
10. [Install & Connect](#10-install--connect)
11. [Your First Commands](#11-your-first-commands)
12. [How Our Task Manager Maps to MongoDB](#12-how-our-task-manager-maps-to-mongodb)

---

## 1. What is MongoDB?

MongoDB is a **NoSQL document database** — it stores data as **JSON-like documents** instead of rows and columns like a traditional SQL database.

| Feature        | MongoDB                                |
| -------------- | -------------------------------------- |
| Type           | NoSQL — Document Database              |
| Storage format | BSON (Binary JSON)                     |
| Schema         | Flexible — no fixed structure required |
| Query language | MQL (MongoDB Query Language)           |
| Scaling        | Horizontal (sharding across servers)   |
| Relationships  | Embedding or referencing (manual)      |

> **In simple terms:** Instead of a spreadsheet with fixed columns, MongoDB stores each record as a flexible JSON object. Each record can have different fields.

---

## 2. SQL vs MongoDB — Terminology

If you're coming from SQL, here's how the concepts map:

| SQL         | MongoDB                | Description                         |
| ----------- | ---------------------- | ----------------------------------- |
| Database    | Database               | A named container of data           |
| Table       | Collection             | A group of related documents        |
| Row         | Document               | A single record (JSON object)       |
| Column      | Field                  | A key inside a document             |
| Primary Key | `_id`                  | Unique identifier for each document |
| Index       | Index                  | Speeds up queries                   |
| JOIN        | `$lookup` / Embedding  | Combine related data                |
| Schema      | Validator (optional)   | Rules for document structure        |
| Foreign Key | Reference (`ObjectId`) | Link between documents              |

---

## 3. How MongoDB is Structured

MongoDB has a **3-level hierarchy:**

```
MongoDB Server
    └── Database
            └── Collection
                    └── Document
```

Think of it like this:

```
MongoDB Server        →   The running MongoDB process
   Database           →   Like a project or application namespace
      Collection      →   Like a table — groups similar documents
         Document     →   A single JSON record (one user, one task, etc.)
```

---

## 4. Architecture Tree

This is the full picture of how MongoDB organizes data — from the server all the way down to individual fields inside a document:

```
MongoDB Server
│
├── Database (e.g., taskmanager)
│   │
│   ├── Collection (e.g., users)
│   │   │
│   │   ├── Document (JSON Object)
│   │   │   ├── _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1")
│   │   │   ├── name: "Arjun Mehta"
│   │   │   ├── email: "arjun@taskmanager.com"
│   │   │   └── password: "$2b$10$hashedpassword"
│   │   │
│   │   ├── Document
│   │   │   ├── _id: ObjectId
│   │   │   ├── name: "Priya Sharma"
│   │   │   ├── email: "priya@taskmanager.com"
│   │   │   └── password: "$2b$10$hashedpassword"
│   │   │
│   │   └── Document ...
│   │
│   ├── Collection (e.g., projects)
│   │   │
│   │   ├── Document
│   │   │   ├── _id: ObjectId
│   │   │   ├── title: "Website Redesign"
│   │   │   ├── description: "Revamp the company website"
│   │   │   ├── ownerId: ObjectId  ← references users._id
│   │   │   └── createdAt: Date
│   │   │
│   │   └── Document ...
│   │
│   ├── Collection (e.g., tasks)
│   │   │
│   │   ├── Document
│   │   │   ├── _id: ObjectId
│   │   │   ├── title: "Design homepage"
│   │   │   ├── status: "in-progress"
│   │   │   ├── priority: "high"
│   │   │   ├── projectId: ObjectId  ← references projects._id
│   │   │   ├── assigneeId: ObjectId ← references users._id
│   │   │   ├── parentTaskId: ObjectId  ← self-reference (subtask)
│   │   │   └── createdAt: Date
│   │   │
│   │   └── Document ...
│   │
│   ├── Collection (e.g., sprints)
│   │   ├── Document
│   │   └── Document ...
│   │
│   └── Collection (e.g., roles)
│       ├── Document
│       └── Document ...
│
└── Database (e.g., admin)        ← MongoDB internal — manages auth & cluster
    └── Database (e.g., local)    ← MongoDB internal — stores replication data
    └── Database (e.g., config)   ← MongoDB internal — stores sharding metadata
```

> **Never touch** `admin`, `local`, or `config` databases manually. These are MongoDB's internal system databases.

---

## 5. What is a Document?

A **document** is the basic unit of data in MongoDB. It is a **JSON object** — a set of key-value pairs.

```json
{
  "_id": "ObjectId(\"64f1a2b3c4d5e6f7a8b9c0d1\")",
  "name": "Arjun Mehta",
  "email": "arjun@taskmanager.com",
  "role": "developer",
  "isActive": true,
  "createdAt": "2024-01-15T10:30:00Z",
  "skills": ["JavaScript", "Node.js", "MongoDB"],
  "address": {
    "city": "Mumbai",
    "state": "Maharashtra",
    "country": "India"
  }
}
```

Key things about documents:

| Rule     | Detail                                                                    |
| -------- | ------------------------------------------------------------------------- |
| Format   | JSON (stored as BSON internally)                                          |
| Max size | **16 MB** per document                                                    |
| `_id`    | Every document must have one — auto-generated if not provided             |
| Fields   | Can be strings, numbers, arrays, nested objects, dates, booleans, null    |
| Schema   | Flexible — two documents in the same collection can have different fields |

---

## 6. What is a Collection?

A **collection** is a group of related documents — like a table in SQL, but without a fixed schema.

```
Collection: tasks
┌─────────────────────────────────────────────────────┐
│  Document 1: { _id, title, status, assigneeId }     │
│  Document 2: { _id, title, status, priority, tags } │  ← different fields, totally fine
│  Document 3: { _id, title, status }                 │
└─────────────────────────────────────────────────────┘
```

Key things about collections:

| Rule            | Detail                                                  |
| --------------- | ------------------------------------------------------- |
| Auto-created    | MongoDB creates a collection on the first insert        |
| No fixed schema | Documents inside can have different structures          |
| Naming          | Lowercase, no spaces — e.g. `tasks`, `users`, `sprints` |
| Indexes         | Created on collections to speed up queries              |

---

## 7. What is a Database?

A **database** is a named container that holds collections. For our app, we use one database called `taskmanager`.

```
taskmanager (database)
├── users
├── roles
├── projects
├── sprints
├── tasks
└── activity_logs
```

Key things about databases:

| Rule         | Detail                                                        |
| ------------ | ------------------------------------------------------------- |
| Auto-created | MongoDB creates a DB when you first write data into it        |
| Isolation    | Each database has its own collections, users, and permissions |
| Naming       | Lowercase, no special characters — e.g. `taskmanager`         |
| `show dbs`   | Lists all databases that have at least one document           |

---

## 8. BSON vs JSON

MongoDB stores data as **BSON** (Binary JSON) internally, but you write and read it as **JSON**.

| Feature     | JSON                      | BSON                               |
| ----------- | ------------------------- | ---------------------------------- |
| Format      | Text                      | Binary                             |
| Used for    | Writing queries (mongosh) | Internal storage                   |
| Extra types | Nope                      | ObjectId, Date, Binary, Decimal128 |
| Speed       | Slower to parse           | Faster to traverse                 |
| Size        | Smaller text              | Slightly larger binary             |

> You don't need to think about BSON day-to-day. Just write JSON — MongoDB handles the conversion.

---

## 9. ObjectId — The Default `_id`

Every document in MongoDB has a unique `_id` field. By default, MongoDB auto-generates an **ObjectId**.

```
ObjectId("64f1a2b3c4d5e6f7a8b9c0d1")
          └──────────────────────────┘
          24-character hexadecimal string
          = 12 bytes of data
```

### What's inside an ObjectId?

```
64f1a2b3   c4d5e6   f7a8b9c0d1
└────────┘ └──────┘ └──────────┘
4 bytes     3 bytes   5 bytes
Timestamp   Machine   Random
(Unix time)   ID      value
```

| Part       | Bytes | Info                                              |
| ---------- | ----- | ------------------------------------------------- |
| Timestamp  | 4     | Seconds since Unix epoch — built-in creation time |
| Machine ID | 3     | Unique to the machine/process                     |
| Random     | 5     | Random value for uniqueness                       |

> Because the timestamp is embedded, you can extract the **creation time** from any `_id` using `ObjectId("...").getTimestamp()` — no need for a separate `createdAt` field unless you want it human-readable.

---

## 10. Install & Connect

### Install MongoDB (Ubuntu / macOS)

```bash
# macOS — using Homebrew
brew tap mongodb/brew
brew install mongodb-community
brew services start mongodb-community

# Ubuntu
sudo apt-get install -y mongodb
sudo systemctl start mongod
```

### Install mongosh (MongoDB Shell)

```bash
# macOS
brew install mongosh

# npm (any OS)
npm install -g mongosh
```

### Connect to MongoDB

```bash
# Connect to local MongoDB (default port 27017)
mongosh

# Connect with a URI
mongosh "mongodb://localhost:27017"

# Connect to MongoDB Atlas (cloud)
mongosh "mongodb+srv://username:password@cluster0.mongodb.net/taskmanager"
```

---

## 11. Your First Commands

Once connected in `mongosh`, try these in order:

```js
// 1. Check which database you're on
db

// 2. Switch to your app database
use taskmanager

// 3. Insert your first document
db.users.insertOne({
    name: "Arjun Mehta",
    email: "arjun@taskmanager.com",
    role: "admin"
})

// 4. Now check — taskmanager DB is created
show dbs

// 5. See the document you just inserted
db.users.find()

// 6. See all collections now
show collections
```

> Notice: the database `taskmanager` only appeared in `show dbs` AFTER you inserted a document. That's MongoDB's lazy creation in action.

---

## 12. How Our Task Manager Maps to MongoDB

Here's the full data model for our **Task Management Application**:

| Collection      | Purpose                          | Key Fields                                                  |
| --------------- | -------------------------------- | ----------------------------------------------------------- |
| `users`         | All registered users             | `name`, `email`, `password`, `role`                         |
| `roles`         | Role definitions and permissions | `name`, `permissions[]`                                     |
| `projects`      | Projects created in the app      | `title`, `description`, `ownerId`                           |
| `sprints`       | Sprints belonging to a project   | `name`, `startDate`, `endDate`, `projectId`                 |
| `tasks`         | Tasks within sprints/projects    | `title`, `status`, `priority`, `assigneeId`, `parentTaskId` |
| `activity_logs` | Audit trail of all actions       | `userId`, `action`, `targetId`, `timestamp`                 |

### Relationships

```
users ──────────────────┐
  │                     │
  │ (ownerId)           │ (assigneeId)
  ▼                     ▼
projects ──── sprints ──── tasks
                              │
                              └── tasks (parentTaskId → self-reference for subtasks)
```

---

> **Next →** [`Database.md`](./Database.md) — All database-level commands with examples
