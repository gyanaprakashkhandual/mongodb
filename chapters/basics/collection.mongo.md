# MongoDB — Collection Commands Reference

> **Project:** Task Management Application
> **Scope:** Collection-level commands — the place where 90% of your MongoDB work happens
> **Shell:** `mongosh` (MongoDB Shell)
> **Prev:** [`Database.md`](./Database.md) → **Next:** [`Create.md`](./Create.md)

---

## Table of Contents

1. [What is a Collection?](#1-what-is-a-collection)
2. [Create Collection](#2-create-collection)
3. [List & Inspect Collections](#3-list--inspect-collections)
4. [Collection Stats & Info](#4-collection-stats--info)
5. [Insert Documents](#5-insert-documents)
6. [Read Documents — find()](#6-read-documents--find)
7. [Query Operators](#7-query-operators)
8. [Projection — Control Which Fields Return](#8-projection--control-which-fields-return)
9. [Sort, Limit & Skip](#9-sort-limit--skip)
10. [Update Documents](#10-update-documents)
11. [Update Operators](#11-update-operators)
12. [Delete Documents](#12-delete-documents)
13. [Indexes](#13-indexes)
14. [Aggregation Pipeline](#14-aggregation-pipeline)
15. [Collection Utilities](#15-collection-utilities)
16. [Quick Cheatsheet](#16-quick-cheatsheet)

---

## 1. What is a Collection?

A **collection** is a group of MongoDB documents — equivalent to a table in SQL, but without a fixed schema. Documents inside the same collection can have completely different fields.

```
db.tasks         ← you are accessing the "tasks" collection inside the current database
  └── .find()    ← calling a method on that collection
```

> You never need to "create" a collection manually. The moment you insert the first document, MongoDB creates it automatically.

---

## 2. Create Collection

```js
// Auto-created on first insert — most common approach
db.tasks.insertOne({ title: "Design homepage" });

// Explicit creation — use when you need options
db.createCollection("tasks");

// Capped collection — fixed size, old docs auto-removed (great for logs)
db.createCollection("activity_logs", {
  capped: true, // Enable fixed-size mode
  size: 10485760, // Max size in bytes (10 MB)
  max: 5000, // Max number of documents
});

// Collection with schema validation — enforce structure at DB level
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email", "role"],
      properties: {
        name: { bsonType: "string" },
        email: { bsonType: "string" },
        role: {
          bsonType: "string",
          enum: ["admin", "manager", "developer", "viewer"],
        },
        isActive: { bsonType: "bool" },
      },
    },
  },
  validationLevel: "strict", // Validate ALL inserts and updates
  validationAction: "error", // Reject docs that fail validation
});

// Rename a collection
db.tasks.renameCollection("archived_tasks");

// Drop (permanently delete) a collection and all its documents
db.tasks.drop();
```

---

## 3. List & Inspect Collections

```js
// List all collection names in the current database
show collections

// Get collection names as an array (useful in scripts)
db.getCollectionNames()

// Get full collection info — UUID, options, type
db.getCollectionInfos()

// Get info for one specific collection
db.getCollectionInfos({ name: "tasks" })
```

---

## 4. Collection Stats & Info

```js
// Get detailed stats — document count, size, index info
db.tasks.stats();

// Get total number of documents in the collection
db.tasks.countDocuments();

// Count with a filter — count only "in-progress" tasks
db.tasks.countDocuments({ status: "in-progress" });

// Faster estimated count (uses metadata, not full scan)
db.tasks.estimatedDocumentCount();

// Get the indexes defined on the collection
db.tasks.getIndexes();

// Get the total size of all indexes on the collection
db.tasks.totalIndexSize();

// Check if collection is capped
db.tasks.isCapped();

// Validate the collection — checks for structural errors
db.tasks.validate();
```

---

## 5. Insert Documents

```js
// Insert a single document
db.users.insertOne({
  name: "Arjun Mehta",
  email: "arjun@taskmanager.com",
  role: "developer",
  isActive: true,
  createdAt: new Date(),
});

// Insert multiple documents at once
db.users.insertMany([
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

// Insert with a custom _id (not recommended — usually let MongoDB generate it)
db.roles.insertOne({
  _id: "admin", // Custom string _id
  permissions: ["read", "write", "delete", "manage_users"],
});
```

> `insertOne()` returns `{ acknowledged: true, insertedId: ObjectId(...) }`
> `insertMany()` returns `{ acknowledged: true, insertedIds: { 0: ObjectId, 1: ObjectId, ... } }`

---

## 6. Read Documents — find()

```js
// Get ALL documents in the collection
db.tasks.find();

// Pretty-print the output (easier to read in mongosh)
db.tasks.find().pretty();

// Get only ONE document (returns first match)
db.tasks.findOne();

// Find with a filter — all tasks with status "todo"
db.tasks.find({ status: "todo" });

// Find one specific document by _id
db.tasks.findOne({ _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") });

// Find with multiple conditions (implicit AND)
db.tasks.find({
  status: "in-progress",
  priority: "high",
});

// Find tasks assigned to a specific user (using ObjectId reference)
db.tasks.find({
  assigneeId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"),
});
```

---

## 7. Query Operators

### Comparison Operators

```js
// $eq  — equal to (same as { status: "todo" })
db.tasks.find({ status: { $eq: "todo" } });

// $ne  — not equal to
db.tasks.find({ status: { $ne: "done" } });

// $gt  — greater than
db.tasks.find({ storyPoints: { $gt: 5 } });

// $gte — greater than or equal to
db.tasks.find({ storyPoints: { $gte: 5 } });

// $lt  — less than
db.tasks.find({ storyPoints: { $lt: 3 } });

// $lte — less than or equal to
db.tasks.find({ storyPoints: { $lte: 3 } });

// $in  — value is in an array of options
db.tasks.find({ status: { $in: ["todo", "in-progress"] } });

// $nin — value is NOT in an array of options
db.tasks.find({ status: { $nin: ["done", "cancelled"] } });
```

### Logical Operators

```js
// $and — all conditions must be true
db.tasks.find({
  $and: [{ status: "in-progress" }, { priority: "high" }],
});

// $or — at least one condition must be true
db.tasks.find({
  $or: [{ priority: "high" }, { priority: "critical" }],
});

// $nor — none of the conditions must be true
db.tasks.find({
  $nor: [{ status: "done" }, { status: "cancelled" }],
});

// $not — negate a condition
db.tasks.find({
  priority: { $not: { $eq: "low" } },
});
```

### Element Operators

```js
// $exists — check if a field exists on the document
db.tasks.find({ parentTaskId: { $exists: true } }); // only subtasks (have a parent)
db.tasks.find({ parentTaskId: { $exists: false } }); // only root-level tasks

// $type — check the BSON type of a field
db.tasks.find({ storyPoints: { $type: "int" } });
db.tasks.find({ createdAt: { $type: "date" } });
```

### Array Operators

```js
// $all — document's array must contain ALL these values
db.tasks.find({ tags: { $all: ["frontend", "urgent"] } });

// $elemMatch — at least one array element must match all conditions
db.sprints.find({
  tasks: {
    $elemMatch: { status: "in-progress", priority: "high" },
  },
});

// $size — array must have exactly this many elements
db.tasks.find({ tags: { $size: 3 } });
```

### Evaluation Operators

```js
// $regex — pattern matching on string fields
db.tasks.find({ title: { $regex: "design", $options: "i" } }); // "i" = case-insensitive

// $text — full-text search (requires a text index on the field)
db.tasks.find({ $text: { $search: "homepage design" } });

// $expr — compare two fields within the same document
db.sprints.find({
  $expr: { $gt: ["$completedTasks", "$totalTasks"] }, // completedTasks > totalTasks
});

// $where — use a JavaScript expression (slow — avoid in production)
db.tasks.find({ $where: "this.title.length > 20" });
```

---

## 8. Projection — Control Which Fields Return

> The second argument to `find()` is the **projection** — it controls which fields appear in the result.

```js
// Return only title and status — _id is included by default
db.tasks.find({}, { title: 1, status: 1 });

// Exclude _id — return title and status only
db.tasks.find({}, { title: 1, status: 1, _id: 0 });

// Exclude specific fields — return everything EXCEPT password
db.users.find({}, { password: 0 });

// Combine filter + projection
db.tasks.find(
  { status: "in-progress" }, // filter
  { title: 1, priority: 1, assigneeId: 1, _id: 0 }, // projection
);
```

> You cannot mix `1` (include) and `0` (exclude) in the same projection — except for `_id` which can always be explicitly excluded even when other fields are included.

---

## 9. Sort, Limit & Skip

```js
// Sort by createdAt ascending (oldest first)
db.tasks.find().sort({ createdAt: 1 });

// Sort by createdAt descending (newest first)
db.tasks.find().sort({ createdAt: -1 });

// Sort by multiple fields — priority desc, then createdAt asc
db.tasks.find().sort({ priority: -1, createdAt: 1 });

// Limit — return only the first 10 documents
db.tasks.find().limit(10);

// Skip — skip the first 20 documents (used for pagination)
db.tasks.find().skip(20);

// Pagination pattern — page 3, 10 items per page
db.tasks
  .find()
  .sort({ createdAt: -1 })
  .skip(20) // (page - 1) * limit  →  (3 - 1) * 10 = 20
  .limit(10);

// Chain everything together
db.tasks
  .find({ status: "todo" }, { title: 1, priority: 1, _id: 0 })
  .sort({ priority: -1 })
  .skip(0)
  .limit(5);
```

| Value | Sort order                           |
| ----- | ------------------------------------ |
| `1`   | Ascending (A → Z, 0 → 9, old → new)  |
| `-1`  | Descending (Z → A, 9 → 0, new → old) |

---

## 10. Update Documents

```js
// updateOne — update the FIRST document that matches the filter
db.tasks.updateOne(
  { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") }, // filter
  { $set: { status: "in-progress" } }, // update
);

// updateMany — update ALL documents that match the filter
db.tasks.updateMany(
  { status: "todo" }, // filter
  { $set: { priority: "medium" } }, // update
);

// replaceOne — replace the ENTIRE document (only _id is kept)
db.tasks.replaceOne(
  { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
  {
    title: "Replaced task",
    status: "todo",
    priority: "low",
    createdAt: new Date(),
  },
);

// upsert — update if found, INSERT if not found
db.tasks.updateOne(
  { title: "Write unit tests" }, // filter
  { $set: { status: "todo", priority: "high" } }, // update
  { upsert: true }, // option — insert if not found
);

// findOneAndUpdate — update and return the document
db.tasks.findOneAndUpdate(
  { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
  { $set: { status: "done" } },
  { returnDocument: "after" }, // "before" returns old doc, "after" returns updated doc
);
```

---

## 11. Update Operators

### Field Operators

```js
// $set — set a field value (adds the field if it doesn't exist)
db.tasks.updateOne(
  { _id: id },
  { $set: { status: "done", updatedAt: new Date() } },
);

// $unset — remove a field from the document
db.tasks.updateOne({ _id: id }, { $unset: { parentTaskId: "" } });

// $rename — rename a field
db.tasks.updateMany({}, { $rename: { desc: "description" } });

// $inc — increment a numeric field by a value
db.tasks.updateOne({ _id: id }, { $inc: { storyPoints: 1 } }); // +1
db.tasks.updateOne({ _id: id }, { $inc: { storyPoints: -2 } }); // -2

// $mul — multiply a numeric field by a value
db.tasks.updateOne({ _id: id }, { $mul: { estimatedHours: 1.5 } });

// $min — update only if new value is LESS than current value
db.tasks.updateOne({ _id: id }, { $min: { storyPoints: 3 } });

// $max — update only if new value is GREATER than current value
db.tasks.updateOne({ _id: id }, { $max: { storyPoints: 10 } });

// $currentDate — set a field to the current date/time
db.tasks.updateOne({ _id: id }, { $currentDate: { updatedAt: true } });
```

### Array Update Operators

```js
// $push — add a value to an array
db.tasks.updateOne({ _id: id }, { $push: { tags: "urgent" } });

// $push with $each — add multiple values to an array
db.tasks.updateOne(
  { _id: id },
  {
    $push: { tags: { $each: ["frontend", "ui"] } },
  },
);

// $push with $sort — push and sort the array
db.sprints.updateOne(
  { _id: id },
  {
    $push: {
      tasks: {
        $each: [{ taskId: ObjectId(), order: 3 }],
        $sort: { order: 1 },
      },
    },
  },
);

// $addToSet — add value to array ONLY if it doesn't already exist (no duplicates)
db.tasks.updateOne({ _id: id }, { $addToSet: { tags: "frontend" } });

// $pull — remove a specific value from an array
db.tasks.updateOne({ _id: id }, { $pull: { tags: "urgent" } });

// $pullAll — remove multiple specific values from an array
db.tasks.updateOne({ _id: id }, { $pullAll: { tags: ["urgent", "outdated"] } });

// $pop — remove first (-1) or last (1) element from array
db.tasks.updateOne({ _id: id }, { $pop: { tags: 1 } }); // remove last
db.tasks.updateOne({ _id: id }, { $pop: { tags: -1 } }); // remove first
```

---

## 12. Delete Documents

```js
// deleteOne — delete the FIRST document that matches the filter
db.tasks.deleteOne({ _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") });

// deleteMany — delete ALL documents that match the filter
db.tasks.deleteMany({ status: "cancelled" });

// deleteMany with date filter — delete tasks older than 1 year
db.tasks.deleteMany({
  createdAt: { $lt: new Date("2023-01-01") },
});

// Delete ALL documents in a collection (collection itself remains)
db.tasks.deleteMany({});

// findOneAndDelete — delete and return the deleted document
db.tasks.findOneAndDelete({ _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") });

// drop — delete the entire collection including all documents and indexes
db.tasks.drop();
```

> **`deleteMany({})` vs `drop()`:**
>
> - `deleteMany({})` removes all documents but **keeps** the collection and its indexes
> - `drop()` removes the collection, all documents, AND all indexes — completely gone

---

## 13. Indexes

> Indexes make queries fast. Without an index, MongoDB does a full **collection scan** (reads every document).

```js
// Create a single-field index on email (ascending)
db.users.createIndex({ email: 1 });

// Create a unique index — no two documents can have the same email
db.users.createIndex({ email: 1 }, { unique: true });

// Create a compound index — queries on both fields benefit
db.tasks.createIndex({ projectId: 1, status: 1 });

// Create a text index — enables full-text search on a field
db.tasks.createIndex({ title: "text", description: "text" });

// Create a TTL index — auto-delete documents after N seconds
db.activity_logs.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 2592000 }, // auto-delete after 30 days
);

// Create a sparse index — only index documents where the field EXISTS
db.tasks.createIndex({ parentTaskId: 1 }, { sparse: true });

// Create a partial index — index only documents matching a filter
db.tasks.createIndex(
  { assigneeId: 1 },
  { partialFilterExpression: { status: { $ne: "done" } } },
);

// List all indexes on a collection
db.tasks.getIndexes();

// Drop a specific index by name
db.tasks.dropIndex("email_1");

// Drop all indexes except _id (use with caution)
db.tasks.dropIndexes();

// See if a query is using an index (EXPLAIN the query)
db.tasks.find({ status: "in-progress" }).explain("executionStats");
```

| Index Type   | Use Case                                       |
| ------------ | ---------------------------------------------- |
| Single field | Queries filtering on one field                 |
| Compound     | Queries filtering/sorting on multiple fields   |
| Unique       | Enforce no duplicate values                    |
| Text         | Full-text search (`$text`)                     |
| TTL          | Auto-expire documents (logs, sessions, tokens) |
| Sparse       | Field only exists on some documents            |
| Partial      | Index a subset of documents (saves space)      |

> **Golden rule:** Only create indexes for fields you actually query or sort on. Too many indexes slow down writes.

---

## 14. Aggregation Pipeline

> Aggregation is MongoDB's most powerful feature — it processes documents through a series of **stages**, like a pipeline.

```
db.tasks.aggregate([
    { stage 1 },    ←  documents flow in
    { stage 2 },    ←  transformed
    { stage 3 }     ←  final result comes out
])
```

### Common Pipeline Stages

```js
// $match — filter documents (like find() — put this FIRST for performance)
db.tasks.aggregate([{ $match: { status: "in-progress" } }]);

// $project — reshape documents, include/exclude/compute fields
db.tasks.aggregate([
  {
    $project: {
      title: 1,
      status: 1,
      _id: 0,
      isUrgent: { $eq: ["$priority", "critical"] }, // computed field
    },
  },
]);

// $sort — sort documents
db.tasks.aggregate([{ $sort: { createdAt: -1 } }]);

// $limit — limit the number of results
db.tasks.aggregate([{ $limit: 10 }]);

// $skip — skip N documents (for pagination)
db.tasks.aggregate([{ $skip: 20 }, { $limit: 10 }]);

// $group — group documents and aggregate values
db.tasks.aggregate([
  {
    $group: {
      _id: "$status", // group BY status field
      totalTasks: { $sum: 1 }, // count documents per group
      avgPoints: { $avg: "$storyPoints" },
    },
  },
]);

// $count — count total documents passing through the pipeline
db.tasks.aggregate([
  { $match: { status: "done" } },
  { $count: "completedTaskCount" },
]);

// $unwind — deconstruct an array field into separate documents
db.tasks.aggregate([
  { $unwind: "$tags" }, // one document per tag value
]);

// $lookup — JOIN another collection (like SQL JOIN)
db.tasks.aggregate([
  {
    $lookup: {
      from: "users", // collection to join
      localField: "assigneeId", // field from tasks
      foreignField: "_id", // field from users
      as: "assignee", // output array field name
    },
  },
]);

// $addFields — add new computed fields without removing existing ones
db.tasks.aggregate([
  {
    $addFields: {
      isOverdue: {
        $lt: ["$dueDate", new Date()], // true if dueDate is in the past
      },
    },
  },
]);

// $replaceRoot — replace the root document with a nested field
db.tasks.aggregate([{ $replaceRoot: { newRoot: "$assignee" } }]);

// $out — write aggregation result to a new collection
db.tasks.aggregate([
  { $match: { status: "done" } },
  { $out: "completed_tasks" }, // saves results to this collection
]);

// $merge — merge aggregation results into an existing collection
db.tasks.aggregate([
  { $group: { _id: "$projectId", count: { $sum: 1 } } },
  { $merge: { into: "project_stats", whenMatched: "merge" } },
]);
```

### Full Pipeline Example — Task Summary by Project

```js
// For each project, count tasks by status
db.tasks.aggregate([
  // Stage 1 — filter only active tasks
  {
    $match: { status: { $ne: "cancelled" } },
  },
  // Stage 2 — join with projects collection
  {
    $lookup: {
      from: "projects",
      localField: "projectId",
      foreignField: "_id",
      as: "project",
    },
  },
  // Stage 3 — flatten the project array into a single object
  {
    $unwind: "$project",
  },
  // Stage 4 — group by projectId + status and count
  {
    $group: {
      _id: {
        projectId: "$projectId",
        projectName: "$project.title",
        status: "$status",
      },
      count: { $sum: 1 },
    },
  },
  // Stage 5 — sort by project name
  {
    $sort: { "_id.projectName": 1 },
  },
  // Stage 6 — return only relevant fields
  {
    $project: {
      _id: 0,
      project: "$_id.projectName",
      status: "$_id.status",
      taskCount: "$count",
    },
  },
]);
```

---

## 15. Collection Utilities

```js
// Count all documents in a collection
db.tasks.countDocuments();

// Count with filter
db.tasks.countDocuments({ status: "done" });

// Get distinct values of a field
db.tasks.distinct("status");
// → [ "todo", "in-progress", "review", "done" ]

// Distinct with filter
db.tasks.distinct("assigneeId", { status: "in-progress" });

// Bulk write — run multiple operations in one call (most efficient)
db.tasks.bulkWrite([
  { insertOne: { document: { title: "New task", status: "todo" } } },
  { updateOne: { filter: { _id: id1 }, update: { $set: { status: "done" } } } },
  { deleteOne: { filter: { _id: id2 } } },
]);

// Watch for real-time changes on a collection (Change Streams)
const changeStream = db.tasks.watch();
changeStream.on("change", (event) => {
  console.log(event); // fires on insert, update, delete
});

// Copy all documents from one collection to another
db.tasks.find().forEach((doc) => db.tasks_backup.insertOne(doc));
```

---

## 16. Quick Cheatsheet

| Command                               | What it does                 |
| ------------------------------------- | ---------------------------- |
| `db.col.insertOne({})`                | Insert one document          |
| `db.col.insertMany([])`               | Insert multiple documents    |
| `db.col.find()`                       | Get all documents            |
| `db.col.findOne({})`                  | Get first matching document  |
| `db.col.find({ field: val })`         | Filter documents             |
| `db.col.find({}, { field: 1 })`       | Projection — select fields   |
| `db.col.find().sort({ field: -1 })`   | Sort results                 |
| `db.col.find().limit(10)`             | Limit results                |
| `db.col.find().skip(20)`              | Skip results (pagination)    |
| `db.col.countDocuments({})`           | Count matching documents     |
| `db.col.distinct("field")`            | Get unique values of a field |
| `db.col.updateOne({}, { $set: {} })`  | Update first match           |
| `db.col.updateMany({}, { $set: {} })` | Update all matches           |
| `db.col.replaceOne({}, {})`           | Replace entire document      |
| `db.col.deleteOne({})`                | Delete first match           |
| `db.col.deleteMany({})`               | Delete all matches           |
| `db.col.drop()`                       | Delete entire collection     |
| `db.col.createIndex({ field: 1 })`    | Create an index              |
| `db.col.getIndexes()`                 | List all indexes             |
| `db.col.explain("executionStats")`    | Analyze query performance    |
| `db.col.aggregate([])`                | Run aggregation pipeline     |
| `db.col.stats()`                      | Collection size and stats    |
| `db.col.bulkWrite([])`                | Run multiple ops in one call |

---

> **Prev →** [`Database.md`](./Database.md) — Database-level commands
> **Next →** [`Create.md`](./Create.md) — Deep dive into insertOne, insertMany, bulk inserts, and document design patterns
