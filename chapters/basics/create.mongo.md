# MongoDB — Create (Insert) Reference

> **Project:** Task Management Application
> **Scope:** All insert operations — insertOne, insertMany, bulk inserts, document design patterns
> **Shell:** `mongosh` (MongoDB Shell)
> **Prev:** [`Collection.md`](./Collection.md) → **Next:** [`Read.md`](./Read.md)

---

## Table of Contents

1. [How Insert Works in MongoDB](#1-how-insert-works-in-mongodb)
2. [insertOne](#2-insertone)
3. [insertMany](#3-insertmany)
4. [The `_id` Field — Deep Dive](#4-the-_id-field--deep-dive)
5. [Insert with Ordered vs Unordered](#5-insert-with-ordered-vs-unordered)
6. [Upsert — Insert or Update](#6-upsert--insert-or-update)
7. [bulkWrite — Insert at Scale](#7-bulkwrite--insert-at-scale)
8. [Document Design Patterns](#8-document-design-patterns)
9. [Inserting Real Collections — Task Manager](#9-inserting-real-collections--task-manager)
10. [Insert Return Values](#10-insert-return-values)
11. [Common Insert Mistakes](#11-common-insert-mistakes)
12. [Quick Cheatsheet](#12-quick-cheatsheet)

---

## 1. How Insert Works in MongoDB

When you insert a document, MongoDB does the following in order:

```
Your document
      │
      ▼
  Validate _id          ← auto-generates ObjectId if _id is missing
      │
      ▼
  Schema validation     ← runs if validator is set on the collection
      │
      ▼
  Write to WiredTiger   ← the default MongoDB storage engine
      │
      ▼
  Update all indexes    ← every index on the collection gets updated
      │
      ▼
  Acknowledge write     ← returns { acknowledged: true, insertedId: ... }
```

> **Key point:** Every insert automatically updates ALL indexes on that collection. This is why too many indexes slow down writes — each insert has more index trees to update.

---

## 2. insertOne

`insertOne()` inserts a **single document** and returns a result object.

### Basic Insert

```js
// Switch to your database first
use taskmanager

// Insert one user
db.users.insertOne({
    name:      "Arjun Mehta",
    email:     "arjun@taskmanager.com",
    password:  "$2b$10$hashedpassword",
    role:      "developer",
    isActive:  true,
    createdAt: new Date()
})
```

### Insert with Nested Object (Embedded Document)

```js
// Insert a user with an embedded address object
db.users.insertOne({
  name: "Priya Sharma",
  email: "priya@taskmanager.com",
  role: "manager",
  profile: {
    avatar: "https://cdn.taskmanager.com/avatars/priya.jpg",
    bio: "Senior project manager with 8 years of experience",
    timezone: "Asia/Kolkata",
    phone: "+91-9876543210",
  },
  preferences: {
    theme: "dark",
    notifications: true,
    language: "en",
  },
  createdAt: new Date(),
  updatedAt: new Date(),
});
```

### Insert with Arrays

```js
// Insert a role document with an array of permissions
db.roles.insertOne({
  name: "manager",
  permissions: [
    "create_project",
    "edit_project",
    "delete_project",
    "assign_tasks",
    "view_reports",
  ],
  createdAt: new Date(),
});
```

### Insert with References (ObjectId)

```js
// First insert a project
const projectResult = db.projects.insertOne({
  title: "Website Redesign",
  description: "Complete overhaul of the company website",
  status: "active",
  createdAt: new Date(),
});

// Use the inserted project's _id to create a sprint that references it
db.sprints.insertOne({
  name: "Sprint 1 — Discovery",
  projectId: projectResult.insertedId, // ← reference to projects._id
  startDate: new Date("2024-02-01"),
  endDate: new Date("2024-02-14"),
  status: "active",
  createdAt: new Date(),
});
```

---

## 3. insertMany

`insertMany()` inserts **multiple documents at once** in a single operation — much faster than calling `insertOne()` in a loop.

### Basic insertMany

```js
// Insert multiple users in one call
db.users.insertMany([
  {
    name: "Arjun Mehta",
    email: "arjun@taskmanager.com",
    role: "developer",
    isActive: true,
    createdAt: new Date(),
  },
  {
    name: "Priya Sharma",
    email: "priya@taskmanager.com",
    role: "manager",
    isActive: true,
    createdAt: new Date(),
  },
  {
    name: "Rohit Desai",
    email: "rohit@taskmanager.com",
    role: "developer",
    isActive: false,
    createdAt: new Date(),
  },
]);
```

### insertMany — Mixed Structure (Flexible Schema)

```js
// MongoDB allows different fields across documents in the same collection
// This is valid — not all tasks need the same fields
db.tasks.insertMany([
  {
    title: "Design homepage wireframe",
    status: "todo",
    priority: "high",
    tags: ["design", "frontend"],
    storyPoints: 5,
    createdAt: new Date(),
  },
  {
    title: "Set up CI/CD pipeline",
    status: "in-progress",
    priority: "critical",
    assigneeId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"),
    dueDate: new Date("2024-02-10"),
    createdAt: new Date(),
    // Note: no tags, no storyPoints — totally fine in MongoDB
  },
  {
    title: "Write API documentation",
    status: "todo",
    priority: "low",
    parentTaskId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d2"), // this is a subtask
    createdAt: new Date(),
  },
]);
```

### insertMany — Storing Result for Reference

```js
// Store the result to access all inserted _ids
const result = db.users.insertMany([
  { name: "Kavya Nair", email: "kavya@taskmanager.com", role: "developer" },
  { name: "Sameer Khan", email: "sameer@taskmanager.com", role: "viewer" },
  { name: "Deepa Menon", email: "deepa@taskmanager.com", role: "manager" },
]);

// Access individual inserted _ids
console.log(result.insertedIds);
// {
//   '0': ObjectId("..."),
//   '1': ObjectId("..."),
//   '2': ObjectId("...")
// }

// Use them to insert related documents
db.projects.insertOne({
  title: "Mobile App Launch",
  teamIds: Object.values(result.insertedIds), // array of all 3 user _ids
  createdAt: new Date(),
});
```

---

## 4. The `_id` Field — Deep Dive

Every document in MongoDB **must** have an `_id` field. It is the **primary key** — always unique, always indexed.

### Auto-Generated ObjectId (default — recommended)

```js
// MongoDB auto-generates _id if you don't provide one
db.tasks.insertOne({ title: "Task A" });
// → _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1")
```

### Custom String \_id

```js
// Use a custom string as _id — useful for role names, config keys
db.roles.insertOne({
  _id: "admin", // human-readable _id
  permissions: ["read", "write", "delete", "manage_users"],
});

db.roles.insertOne({
  _id: "developer",
  permissions: ["read", "write"],
});
```

### Custom Number \_id

```js
// Use a number as _id — only if you manage uniqueness yourself
db.sprints.insertOne({
  _id: 1, // sprint number as _id
  name: "Sprint 1",
  projectId: ObjectId("..."),
});
```

### UUID as \_id

```js
// Use a UUID string as _id — common when integrating with external systems
db.users.insertOne({
  _id: "550e8400-e29b-41d4-a716-446655440000",
  name: "External User",
  email: "user@external.com",
});
```

### ObjectId Utilities

```js
// Create a new ObjectId manually
const newId = new ObjectId();

// Create an ObjectId from an existing string
const existingId = ObjectId("64f1a2b3c4d5e6f7a8b9c0d1");

// Extract the creation timestamp from an ObjectId — built-in created time
ObjectId("64f1a2b3c4d5e6f7a8b9c0d1").getTimestamp();
// → ISODate("2023-09-01T10:30:00.000Z")

// Check if two ObjectIds are equal
ObjectId("64f1...").equals(ObjectId("64f1..."));
```

> **Duplicate `_id` error:** If you try to insert a document with an `_id` that already exists, MongoDB throws:
> `E11000 duplicate key error collection: taskmanager.tasks index: _id_ dup key`

---

## 5. Insert with Ordered vs Unordered

When using `insertMany()`, MongoDB lets you control what happens when one document in the batch fails (e.g., duplicate `_id`).

### Ordered Insert (default — stops on first error)

```js
// ordered: true is the DEFAULT
// If document 2 fails, documents 3, 4, 5... are NOT inserted
db.users.insertMany(
  [
    { _id: 1, name: "Arjun" },
    { _id: 2, name: "Priya" },
    { _id: 2, name: "Duplicate!" }, // ← this will FAIL (dup _id)
    { _id: 3, name: "Rohit" }, // ← this will NOT run
  ],
  { ordered: true }, // default — stops at first failure
);
```

### Unordered Insert (continues on error)

```js
// ordered: false — skip failures and continue inserting the rest
// Use this for bulk imports where some duplicates are expected
db.users.insertMany(
  [
    { _id: 1, name: "Arjun" },
    { _id: 2, name: "Priya" },
    { _id: 2, name: "Duplicate!" }, // ← this will FAIL (skipped)
    { _id: 3, name: "Rohit" }, // ← this WILL be inserted
  ],
  { ordered: false }, // skip failures, continue with remaining
);
```

| Option                    | Behavior on error                          | Use case                                        |
| ------------------------- | ------------------------------------------ | ----------------------------------------------- |
| `ordered: true` (default) | Stops immediately — remaining docs skipped | When all inserts must succeed together          |
| `ordered: false`          | Skips failed doc — continues with rest     | Bulk imports where some duplicates are expected |

---

## 6. Upsert — Insert or Update

An **upsert** is a special update operation that:

- **Updates** the document if it already exists
- **Inserts** it as a new document if it does NOT exist

```js
// Upsert a sprint — create it if it doesn't exist, update it if it does
db.sprints.updateOne(
  { name: "Sprint 1", projectId: ObjectId("64f1...") }, // filter
  {
    $set: {
      status: "active",
      startDate: new Date("2024-02-01"),
      endDate: new Date("2024-02-14"),
      updatedAt: new Date(),
    },
    $setOnInsert: {
      createdAt: new Date(), // only set this field on INSERT, not on update
    },
  },
  { upsert: true }, // insert if not found
);
```

### `$setOnInsert` — Very Useful with Upsert

```js
// $setOnInsert only applies when a new document is CREATED
// $set applies on both insert and update
db.users.updateOne(
  { email: "arjun@taskmanager.com" },
  {
    $set: {
      lastLoginAt: new Date(), // always update this
      isActive: true,
    },
    $setOnInsert: {
      createdAt: new Date(), // only set on first insert
      role: "developer",
    },
  },
  { upsert: true },
);
```

> Use upsert for **idempotent seeding** — run the same script multiple times without creating duplicates. Great for seeding initial roles, config, and admin users.

---

## 7. bulkWrite — Insert at Scale

`bulkWrite()` lets you run **multiple different operations** in a single database call — most efficient for batch processing.

```js
db.tasks.bulkWrite([
  // insertOne inside bulkWrite
  {
    insertOne: {
      document: {
        title: "Create login page",
        status: "todo",
        priority: "high",
        createdAt: new Date(),
      },
    },
  },

  // insertOne — another document
  {
    insertOne: {
      document: {
        title: "Set up database",
        status: "done",
        priority: "critical",
        createdAt: new Date(),
      },
    },
  },

  // updateOne inside the same bulk call
  {
    updateOne: {
      filter: { title: "Write unit tests" },
      update: { $set: { status: "in-progress" } },
      upsert: true, // insert if not found
    },
  },

  // deleteOne inside the same bulk call
  {
    deleteOne: {
      filter: { status: "cancelled" },
    },
  },
]);
```

### bulkWrite Return Value

```js
// bulkWrite returns a detailed summary
{
    acknowledged:    true,
    insertedCount:   2,      // how many insertOne succeeded
    matchedCount:    1,      // how many updateOne filters matched
    modifiedCount:   1,      // how many documents were actually changed
    deletedCount:    1,      // how many deleteOne succeeded
    upsertedCount:   0,      // how many upserts created new documents
    upsertedIds:     {},
    insertedIds:     {
        '0': ObjectId("..."),
        '1': ObjectId("...")
    }
}
```

> Use `bulkWrite()` when seeding data, processing imports, or syncing large batches. It's far faster than individual calls in a loop.

---

## 8. Document Design Patterns

This is where MongoDB thinking differs most from SQL. You have two choices for related data:

### Pattern 1 — Embedding (Denormalized)

Store related data **inside** the same document as a nested object or array.

```js
// Task with comments embedded directly inside the document
db.tasks.insertOne({
  title: "Design homepage",
  status: "in-progress",
  priority: "high",

  // Comments embedded — no separate collection needed
  comments: [
    {
      _id: new ObjectId(),
      userId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"),
      text: "Let's use the new brand colors here",
      createdAt: new Date("2024-02-01T09:00:00Z"),
    },
    {
      _id: new ObjectId(),
      userId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d2"),
      text: "Agreed. Also add mobile breakpoints.",
      createdAt: new Date("2024-02-01T09:15:00Z"),
    },
  ],

  createdAt: new Date(),
});
```

**When to embed:**

- The related data is always read together with the parent
- The sub-data is small and doesn't grow unboundedly
- You don't need to query the sub-data independently

### Pattern 2 — Referencing (Normalized)

Store related data in a **separate collection** and link via `ObjectId`.

```js
// Task stores only a reference to the assignee — not the full user object
db.tasks.insertOne({
  title: "Design homepage",
  status: "in-progress",
  priority: "high",
  projectId: ObjectId("64f1a2b3c4d5e6f7a8b9c0da"), // → projects._id
  sprintId: ObjectId("64f1a2b3c4d5e6f7a8b9c0db"), // → sprints._id
  assigneeId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"), // → users._id
  createdAt: new Date(),
});

// User lives in its own collection — updated once, correct everywhere
db.users.insertOne({
  _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"),
  name: "Arjun Mehta",
  email: "arjun@taskmanager.com",
  role: "developer",
});
```

**When to reference:**

- The related data is large or grows over time (e.g., thousands of comments)
- The related data is shared across many documents (e.g., one user assigned to 100 tasks)
- You need to query the related data independently

### Pattern 3 — Hybrid (Most Common in Real Apps)

Embed the fields you always need, reference the rest.

```js
// Embed just enough user info to display the task card
// but keep the full user in its own collection
db.tasks.insertOne({
  title: "Design homepage",
  status: "in-progress",
  priority: "high",

  // Embed a snapshot of the assignee (for fast display without a JOIN)
  assignee: {
    _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"), // still store the reference
    name: "Arjun Mehta", // embed name for display
    avatar: "https://cdn.taskmanager.com/arjun.jpg", // embed avatar for UI
  },

  createdAt: new Date(),
});
```

> **Tradeoff:** If the user updates their name, you need to update it in both places. Only embed data that rarely changes.

### Pattern 4 — Self-Referencing (Parent-Child / Tree)

```js
// Tasks can have subtasks — parentTaskId references another task in the same collection
db.tasks.insertMany([
  {
    _id: ObjectId("aaaa000000000000000000a1"),
    title: "Build Authentication",
    status: "in-progress",
    parentTaskId: null, // null = root-level task, no parent
    createdAt: new Date(),
  },
  {
    _id: ObjectId("aaaa000000000000000000a2"),
    title: "Design login page UI",
    status: "todo",
    parentTaskId: ObjectId("aaaa000000000000000000a1"), // child of "Build Authentication"
    createdAt: new Date(),
  },
  {
    _id: ObjectId("aaaa000000000000000000a3"),
    title: "Implement JWT token logic",
    status: "todo",
    parentTaskId: ObjectId("aaaa000000000000000000a1"), // also child of "Build Authentication"
    createdAt: new Date(),
  },
]);
```

---

## 9. Inserting Real Collections — Task Manager

Here is a complete, realistic seed for the full Task Management Application — all 6 collections in the right order (parent data before child data).

```js
use taskmanager

// ── Step 1: Roles ──────────────────────────────────────────
db.roles.insertMany([
    {
        _id:         "admin",
        permissions: ["create_project", "delete_project", "manage_users",
                      "assign_tasks", "view_reports", "manage_billing"]
    },
    {
        _id:         "manager",
        permissions: ["create_project", "edit_project", "assign_tasks",
                      "view_reports"]
    },
    {
        _id:         "developer",
        permissions: ["view_project", "update_task_status", "add_comments"]
    },
    {
        _id:         "viewer",
        permissions: ["view_project", "view_tasks"]
    }
])

// ── Step 2: Users ──────────────────────────────────────────
const users = db.users.insertMany([
    {
        name:      "Arjun Mehta",
        email:     "arjun@taskmanager.com",
        password:  "$2b$10$hashed1",
        role:      "admin",
        isActive:  true,
        createdAt: new Date()
    },
    {
        name:      "Priya Sharma",
        email:     "priya@taskmanager.com",
        password:  "$2b$10$hashed2",
        role:      "manager",
        isActive:  true,
        createdAt: new Date()
    },
    {
        name:      "Rohit Desai",
        email:     "rohit@taskmanager.com",
        password:  "$2b$10$hashed3",
        role:      "developer",
        isActive:  true,
        createdAt: new Date()
    }
])

const [adminId, managerId, devId] = Object.values(users.insertedIds)

// ── Step 3: Project ────────────────────────────────────────
const project = db.projects.insertOne({
    title:       "Website Redesign",
    description: "Complete overhaul of the company website",
    status:      "active",
    ownerId:     adminId,
    memberIds:   [adminId, managerId, devId],
    createdAt:   new Date()
})

const projectId = project.insertedId

// ── Step 4: Sprint ─────────────────────────────────────────
const sprint = db.sprints.insertOne({
    name:      "Sprint 1 — Discovery & Design",
    projectId: projectId,
    startDate: new Date("2024-02-01"),
    endDate:   new Date("2024-02-14"),
    status:    "active",
    createdAt: new Date()
})

const sprintId = sprint.insertedId

// ── Step 5: Tasks ──────────────────────────────────────────
db.tasks.insertMany([
    {
        title:       "Design homepage wireframe",
        description: "Create low and high fidelity wireframes for the homepage",
        status:      "in-progress",
        priority:    "high",
        storyPoints: 5,
        projectId:   projectId,
        sprintId:    sprintId,
        assigneeId:  devId,
        parentTaskId: null,
        tags:        ["design", "frontend"],
        dueDate:     new Date("2024-02-07"),
        createdAt:   new Date()
    },
    {
        title:        "Design mobile breakpoints",
        description:  "Ensure the homepage wireframe is responsive",
        status:       "todo",
        priority:     "medium",
        storyPoints:  3,
        projectId:    projectId,
        sprintId:     sprintId,
        assigneeId:   devId,
        parentTaskId: null,   // will be updated to parent task _id
        tags:         ["design", "mobile"],
        createdAt:    new Date()
    },
    {
        title:       "Set up CI/CD pipeline",
        description: "Configure GitHub Actions for automated testing and deployment",
        status:      "todo",
        priority:    "critical",
        storyPoints: 8,
        projectId:   projectId,
        sprintId:    sprintId,
        assigneeId:  managerId,
        parentTaskId: null,
        tags:        ["devops", "backend"],
        createdAt:   new Date()
    }
])

// ── Step 6: Activity Log ───────────────────────────────────
db.activity_logs.insertMany([
    {
        userId:    adminId,
        action:    "created_project",
        targetId:  projectId,
        targetType: "project",
        createdAt: new Date()
    },
    {
        userId:    managerId,
        action:    "created_sprint",
        targetId:  sprintId,
        targetType: "sprint",
        createdAt: new Date()
    }
])
```

---

## 10. Insert Return Values

```js
// insertOne return value
{
    acknowledged: true,                              // write was confirmed by server
    insertedId:   ObjectId("64f1a2b3c4d5e6f7a8b9c0d1")   // _id of the new document
}

// insertMany return value
{
    acknowledged: true,
    insertedIds: {
        '0': ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"),   // index → _id
        '1': ObjectId("64f1a2b3c4d5e6f7a8b9c0d2"),
        '2': ObjectId("64f1a2b3c4d5e6f7a8b9c0d3")
    }
}

// Access the inserted _id after insertOne
const result = db.tasks.insertOne({ title: "New Task" })
const newId   = result.insertedId
console.log(newId)   // ObjectId("...")

// Access all inserted _ids after insertMany
const result  = db.tasks.insertMany([{ title: "A" }, { title: "B" }])
const allIds  = Object.values(result.insertedIds)   // [ ObjectId, ObjectId ]
```

---

## 11. Common Insert Mistakes

### Inserting in a loop instead of insertMany

```js
// BAD — one network round-trip per document (slow)
tasks.forEach((task) => {
  db.tasks.insertOne(task);
});

// GOOD — one network round-trip for all documents (fast)
db.tasks.insertMany(tasks);
```

### Not storing the result when you need the \_id

```js
// BAD — you lose the _id and can't reference it
db.projects.insertOne({ title: "My Project" })
db.sprints.insertOne({ projectId: ??? })   // you don't know the _id

// GOOD — store the result and use insertedId
const project = db.projects.insertOne({ title: "My Project" })
db.sprints.insertOne({ projectId: project.insertedId })
```

### Using `new Date()` as a string

```js
// BAD — stores as a plain string, can't query by date range
db.tasks.insertOne({ dueDate: "2024-02-10" });

// GOOD — stores as a proper BSON Date type
db.tasks.insertOne({ dueDate: new Date("2024-02-10") });
```

### Forgetting `ordered: false` on large bulk imports

```js
// BAD — one duplicate stops your entire 10,000-doc import
db.tasks.insertMany(tenThousandTasks); // ordered: true by default

// GOOD — skip duplicates, insert the rest
db.tasks.insertMany(tenThousandTasks, { ordered: false });
```

### Embedding data that grows unboundedly

```js
// BAD — comments array will grow forever, hitting the 16MB doc limit
db.tasks.insertOne({
  title: "Big Task",
  comments: [], // over time, 10,000 comments get pushed here — document bloat
});

// GOOD — store comments in a separate collection, reference the taskId
db.comments.insertOne({
  taskId: ObjectId("..."),
  text: "First comment",
  userId: ObjectId("..."),
  createdAt: new Date(),
});
```

---

## 12. Quick Cheatsheet

| Command                                      | What it does                                        |
| -------------------------------------------- | --------------------------------------------------- |
| `db.col.insertOne({})`                       | Insert one document                                 |
| `db.col.insertMany([])`                      | Insert multiple documents in one call               |
| `db.col.insertMany([], { ordered: false })`  | Insert batch, skip failures                         |
| `db.col.updateOne({}, {}, { upsert: true })` | Insert if not found, update if found                |
| `db.col.bulkWrite([])`                       | Run mixed insert/update/delete in one call          |
| `result.insertedId`                          | Get the `_id` of the inserted document              |
| `result.insertedIds`                         | Get all `_ids` from insertMany                      |
| `new ObjectId()`                             | Manually create a new ObjectId                      |
| `new Date()`                                 | Create a BSON Date for timestamps                   |
| `ObjectId("...").getTimestamp()`             | Extract creation time from an ObjectId              |
| `$setOnInsert`                               | Set fields only on insert (not on update) in upsert |

---

> **Prev →** [`Collection.md`](./Collection.md) — Collection-level commands
> **Next →** [`Read.md`](./Read.md) — Deep dive into find, filters, projection, sort, pagination & more
