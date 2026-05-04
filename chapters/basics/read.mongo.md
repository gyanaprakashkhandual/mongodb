# MongoDB — Read Operations Reference

> **Project:** Task Management Application
> **Scope:** All read operations — find, filters, projection, sort, pagination, query operators
> **Shell:** `mongosh` (MongoDB Shell)
> **Prev:** [`Create.md`](./Create.md) — **Next:** [`Update.md`](./Update.md)

---

## Table of Contents

1. [How Read Works in MongoDB](#1-how-read-works-in-mongodb)
2. [find() and findOne()](#2-find-and-findone)
3. [Comparison Operators](#3-comparison-operators)
4. [Logical Operators](#4-logical-operators)
5. [Element Operators](#5-element-operators)
6. [Array Operators](#6-array-operators)
7. [Evaluation Operators](#7-evaluation-operators)
8. [Projection — Field Selection](#8-projection--field-selection)
9. [Sort](#9-sort)
10. [Limit and Skip — Pagination](#10-limit-and-skip--pagination)
11. [Cursor Methods](#11-cursor-methods)
12. [countDocuments and distinct](#12-countdocuments-and-distinct)
13. [Query on Nested Fields](#13-query-on-nested-fields)
14. [Query on Arrays](#14-query-on-arrays)
15. [explain() — Query Analysis](#15-explain--query-analysis)
16. [Real Query Examples — Task Manager](#16-real-query-examples--task-manager)
17. [Quick Cheatsheet](#17-quick-cheatsheet)

---

## 1. How Read Works in MongoDB

When you call `find()`, MongoDB follows this path:

```
db.tasks.find({ status: "todo" })
         │
         ▼
   Check query plan     -- does an index exist for this field?
         │
    YES  │  NO
         │   └──> Collection scan  -- reads every document (slow on large data)
         ▼
   Index scan           -- uses the index to jump directly to matching docs
         │
         ▼
   Apply projection     -- strip fields based on your second argument
         │
         ▼
   Return cursor        -- a pointer to the result set, not the data itself
         │
         ▼
   Iterate cursor       -- mongosh auto-iterates; in code you call .toArray() or loop
```

> NOTE: `find()` does NOT return an array directly. It returns a **cursor** — a pointer to the results. MongoDB streams documents through the cursor in batches of 101 by default.

---

## 2. find() and findOne()

```js
// Return ALL documents in the collection (use with caution on large collections)
db.tasks.find();

// Return all with pretty print — easier to read in mongosh
db.tasks.find().pretty();

// Return the FIRST document that matches — returns a document object, not a cursor
db.tasks.findOne();

// Find all documents matching a filter
db.tasks.find({ status: "todo" });

// findOne with a filter — returns the first match only
db.tasks.findOne({ status: "todo" });

// Find by _id — the most precise and fastest query possible
db.tasks.findOne({ _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") });

// Find with multiple fields — implicit AND between all conditions
db.tasks.find({
  status: "in-progress",
  priority: "high",
});

// Find tasks assigned to a specific user by their ObjectId reference
db.tasks.find({
  assigneeId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"),
});

// Find tasks belonging to a specific project and sprint
db.tasks.find({
  projectId: ObjectId("64f1a2b3c4d5e6f7a8b9c0da"),
  sprintId: ObjectId("64f1a2b3c4d5e6f7a8b9c0db"),
});
```

---

## 3. Comparison Operators

These operators compare a field's value against a given value.

```js
// $eq — equal to (explicit form, same result as { status: "todo" })
db.tasks.find({ status: { $eq: "todo" } });

// $ne — not equal to
db.tasks.find({ status: { $ne: "done" } });

// $gt — greater than
db.tasks.find({ storyPoints: { $gt: 5 } });

// $gte — greater than or equal to
db.tasks.find({ storyPoints: { $gte: 5 } });

// $lt — less than
db.tasks.find({ storyPoints: { $lt: 3 } });

// $lte — less than or equal to
db.tasks.find({ storyPoints: { $lte: 3 } });

// $in — field value must be one of the values in the array
db.tasks.find({ status: { $in: ["todo", "in-progress"] } });

// $nin — field value must NOT be any of the values in the array
db.tasks.find({ status: { $nin: ["done", "cancelled"] } });

// Range query — combine $gte and $lte for between
db.tasks.find({ storyPoints: { $gte: 3, $lte: 8 } });

// Date range — find tasks due this week
db.tasks.find({
  dueDate: {
    $gte: new Date("2024-02-01"),
    $lte: new Date("2024-02-07"),
  },
});
```

---

## 4. Logical Operators

These operators combine multiple conditions.

```js
// $and — ALL conditions must be true
// Implicit AND: just listing fields in the filter object works the same
db.tasks.find({
  $and: [{ status: "in-progress" }, { priority: "high" }],
});

// Implicit AND — same result, cleaner syntax when fields are different
db.tasks.find({ status: "in-progress", priority: "high" });

// Use explicit $and when you need multiple conditions on the SAME field
db.tasks.find({
  $and: [{ storyPoints: { $gte: 3 } }, { storyPoints: { $lte: 8 } }],
});

// $or — at least ONE condition must be true
db.tasks.find({
  $or: [{ priority: "high" }, { priority: "critical" }],
});

// $nor — NONE of the conditions must be true
// Returns docs where status is not "done" AND not "cancelled"
db.tasks.find({
  $nor: [{ status: "done" }, { status: "cancelled" }],
});

// $not — negates a single condition
db.tasks.find({
  priority: { $not: { $eq: "low" } },
});

// Combine $or and $and — tasks that are (high or critical) AND are overdue
db.tasks.find({
  $and: [
    { $or: [{ priority: "high" }, { priority: "critical" }] },
    { dueDate: { $lt: new Date() } },
  ],
});
```

---

## 5. Element Operators

These operators check whether a field exists or what type it is.

```js
// $exists: true — only return documents where the field is present
db.tasks.find({ parentTaskId: { $exists: true } }); // subtasks only

// $exists: false — only return documents where the field is absent
db.tasks.find({ parentTaskId: { $exists: false } }); // root-level tasks only

// $exists with a value check — field exists AND has a specific value
db.tasks.find({
  assigneeId: { $exists: true },
  status: "todo",
});

// $type — match documents where the field is a specific BSON type
db.tasks.find({ storyPoints: { $type: "int" } });
db.tasks.find({ storyPoints: { $type: "double" } });
db.tasks.find({ dueDate: { $type: "date" } });
db.tasks.find({ title: { $type: "string" } });
db.tasks.find({ assigneeId: { $type: "objectId" } });
db.tasks.find({ tags: { $type: "array" } });
db.tasks.find({ isActive: { $type: "bool" } });

// $type with numeric alias — same as above
db.tasks.find({ storyPoints: { $type: 16 } }); // 16 = 32-bit integer
```

| BSON Type        | String alias | Number |
| ---------------- | ------------ | ------ |
| Double           | `"double"`   | 1      |
| String           | `"string"`   | 2      |
| Object           | `"object"`   | 3      |
| Array            | `"array"`    | 4      |
| ObjectId         | `"objectId"` | 7      |
| Boolean          | `"bool"`     | 8      |
| Date             | `"date"`     | 9      |
| Null             | `"null"`     | 10     |
| Integer (32-bit) | `"int"`      | 16     |
| Integer (64-bit) | `"long"`     | 18     |
| Decimal128       | `"decimal"`  | 19     |

---

## 6. Array Operators

These operators query fields that hold arrays.

```js
// Match documents where the array field contains a specific value
// (works even if the array has other values too)
db.tasks.find({ tags: "frontend" });

// Match documents where the array contains ALL of these values
db.tasks.find({ tags: { $all: ["frontend", "urgent"] } });

// $size — match documents where the array has exactly N elements
db.tasks.find({ tags: { $size: 0 } }); // tasks with no tags
db.tasks.find({ tags: { $size: 3 } }); // tasks with exactly 3 tags

// $elemMatch — at least one element in the array must match ALL conditions
// Use when you need multiple conditions on the same array element
db.sprints.find({
  tasks: {
    $elemMatch: {
      status: "in-progress",
      priority: "high",
    },
  },
});

// Without $elemMatch — conditions can match DIFFERENT elements (usually wrong)
// This finds sprints where status matches one element AND priority matches another
db.sprints.find({
  "tasks.status": "in-progress", // can be on element 1
  "tasks.priority": "high", // can be on element 2 (different element)
});

// Query a specific array index — find tasks where the first tag is "frontend"
db.tasks.find({ "tags.0": "frontend" });
```

---

## 7. Evaluation Operators

These operators evaluate expressions or run pattern matching.

```js
// $regex — pattern match on string fields
db.tasks.find({ title: { $regex: "design" } }); // case-sensitive
db.tasks.find({ title: { $regex: "design", $options: "i" } }); // case-insensitive
db.tasks.find({ title: { $regex: "^Design" } }); // starts with "Design"
db.tasks.find({ title: { $regex: "page$" } }); // ends with "page"
db.tasks.find({ email: { $regex: "@taskmanager\\.com$" } }); // email domain match

// $text — full-text search (requires a text index on the field first)
// Create the index: db.tasks.createIndex({ title: "text", description: "text" })
db.tasks.find({ $text: { $search: "homepage design" } }); // matches either word
db.tasks.find({ $text: { $search: '"homepage design"' } }); // exact phrase match
db.tasks.find({ $text: { $search: "design -homepage" } }); // design but NOT homepage

// Sort by text relevance score
db.tasks
  .find(
    { $text: { $search: "homepage design" } },
    { score: { $meta: "textScore" } },
  )
  .sort({ score: { $meta: "textScore" } });

// $expr — compare two fields in the same document using aggregation expressions
// Find sprints where completedTasks exceeds totalTasks (data anomaly)
db.sprints.find({
  $expr: { $gt: ["$completedTasks", "$totalTasks"] },
});

// $expr with arithmetic — find tasks where actual hours exceeded estimate
db.tasks.find({
  $expr: { $gt: ["$actualHours", "$estimatedHours"] },
});

// $mod — modulo operation — find every 5th task (storyPoints divisible by 5)
db.tasks.find({ storyPoints: { $mod: [5, 0] } });

// $where — JavaScript expression (very slow, avoid in production)
db.tasks.find({ $where: "this.title.length > 50" });
```

---

## 8. Projection — Field Selection

Projection is the second argument to `find()` — it controls which fields come back in the result.

```js
// Include specific fields — _id is included by default
db.tasks.find({}, { title: 1, status: 1, priority: 1 });

// Include fields and explicitly exclude _id
db.tasks.find({}, { title: 1, status: 1, priority: 1, _id: 0 });

// Exclude specific fields — return everything EXCEPT these
db.users.find({}, { password: 0, __v: 0 });

// Combine filter and projection
db.tasks.find(
  { status: "in-progress" },
  { title: 1, priority: 1, assigneeId: 1, dueDate: 1, _id: 0 },
);

// Project nested fields using dot notation
db.users.find({}, { "profile.avatar": 1, "profile.bio": 1, name: 1 });

// Project a specific array element using $slice
db.tasks.find({}, { comments: { $slice: 5 } }); // first 5 comments
db.tasks.find({}, { comments: { $slice: -3 } }); // last 3 comments
db.tasks.find({}, { comments: { $slice: [10, 5] } }); // skip 10, return next 5

// $elemMatch in projection — return only the FIRST matching array element
db.tasks.find(
  { "comments.userId": ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
  {
    comments: { $elemMatch: { userId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") } },
  },
);
```

> NOTE: You cannot mix include (1) and exclude (0) in the same projection — the only exception is `_id` which can always be set to 0 even when other fields are set to 1.

```js
// This will throw an error — mixing 1 and 0 is not allowed
db.tasks.find({}, { title: 1, status: 0 }); // ERROR

// This is allowed — _id is the only exception
db.tasks.find({}, { title: 1, status: 1, _id: 0 }); // VALID
```

---

## 9. Sort

```js
// Sort ascending — 1 means A to Z, 0 to 9, oldest to newest
db.tasks.find().sort({ createdAt: 1 });

// Sort descending — -1 means Z to A, 9 to 0, newest to oldest
db.tasks.find().sort({ createdAt: -1 });

// Sort by multiple fields — primary sort, then secondary sort
// First sort by priority descending, then by createdAt ascending
db.tasks.find().sort({ priority: -1, createdAt: 1 });

// Sort with a filter
db.tasks.find({ status: "todo" }).sort({ priority: -1 });

// Sort with filter and projection
db.tasks
  .find({ status: "todo" }, { title: 1, priority: 1, dueDate: 1, _id: 0 })
  .sort({ dueDate: 1 });

// Sort by a nested field using dot notation
db.users.find().sort({ "profile.joinedAt": -1 });

// Sort alphabetically by name
db.users.find().sort({ name: 1 });
```

| Sort value | Order                                   |
| ---------- | --------------------------------------- |
| `1`        | Ascending — A to Z, 0 to 9, old to new  |
| `-1`       | Descending — Z to A, 9 to 0, new to old |

---

## 10. Limit and Skip — Pagination

```js
// limit — return only the first N documents
db.tasks.find().limit(10);

// skip — skip the first N documents
db.tasks.find().skip(20);

// Pagination pattern — always sort first for consistent pages
// Page 1: skip 0,  limit 10
db.tasks.find().sort({ createdAt: -1 }).skip(0).limit(10);

// Page 2: skip 10, limit 10
db.tasks.find().sort({ createdAt: -1 }).skip(10).limit(10);

// Page 3: skip 20, limit 10
db.tasks.find().sort({ createdAt: -1 }).skip(20).limit(10);

// Formula: skip = (pageNumber - 1) * pageSize
// Page 5 with 10 items per page: skip = (5 - 1) * 10 = 40
db.tasks.find().sort({ createdAt: -1 }).skip(40).limit(10);

// Full chain — filter + projection + sort + pagination
db.tasks
  .find(
    { status: { $ne: "done" } },
    { title: 1, status: 1, priority: 1, assigneeId: 1, _id: 0 },
  )
  .sort({ priority: -1, createdAt: -1 })
  .skip(0)
  .limit(10);
```

> NOTE: `skip()` becomes slow on very large collections because MongoDB still has to read and discard the skipped documents. For large datasets, use cursor-based pagination instead — store the `_id` of the last document on the current page and use it as a filter for the next page.

```js
// Cursor-based pagination — faster than skip() on large datasets

// Page 1 — no filter needed for the first page
const page1 = db.tasks.find().sort({ _id: 1 }).limit(10);

// Get the last _id from page 1 results
const lastId = page1[page1.length - 1]._id;

// Page 2 — use the last _id as the anchor
db.tasks
  .find({ _id: { $gt: lastId } })
  .sort({ _id: 1 })
  .limit(10);
```

---

## 11. Cursor Methods

`find()` returns a cursor. These methods operate on the cursor.

```js
// hasNext() — check if there are more documents in the cursor
const cursor = db.tasks.find({ status: "todo" });
while (cursor.hasNext()) {
  printjson(cursor.next()); // next() retrieves the next document
}

// forEach() — iterate over all documents in the cursor
db.tasks.find({ status: "todo" }).forEach((task) => {
  printjson(task);
});

// toArray() — load all cursor documents into a JavaScript array
const tasks = db.tasks.find({ status: "todo" }).toArray();
tasks.length; // total count

// map() — transform each document as you iterate
const titles = db.tasks.find().map((task) => task.title);

// count() — count documents matching the query (deprecated, use countDocuments)
db.tasks.find({ status: "todo" }).count();

// size() — similar to count but accounts for limit() and skip()
db.tasks.find({ status: "todo" }).limit(10).size();

// explain() — show how the query was executed (index used, docs scanned)
db.tasks.find({ status: "todo" }).explain();
db.tasks.find({ status: "todo" }).explain("executionStats");

// close() — manually close the cursor (frees server resources)
const cursor = db.tasks.find();
cursor.close();

// batchSize() — control how many documents are returned per batch
db.tasks.find().batchSize(50); // default is 101
```

---

## 12. countDocuments and distinct

```js
// Count ALL documents in the collection
db.tasks.countDocuments();

// Count documents matching a filter
db.tasks.countDocuments({ status: "todo" });
db.tasks.countDocuments({ status: "in-progress", priority: "high" });

// Count with a date filter — tasks created this month
db.tasks.countDocuments({
  createdAt: {
    $gte: new Date("2024-02-01"),
    $lt: new Date("2024-03-01"),
  },
});

// estimatedDocumentCount — fast approximate count using collection metadata
// Does not accept a filter — use only when exact count is not critical
db.tasks.estimatedDocumentCount();

// distinct — get all unique values of a field
db.tasks.distinct("status");
// Returns: [ "todo", "in-progress", "review", "done", "cancelled" ]

db.tasks.distinct("priority");
// Returns: [ "low", "medium", "high", "critical" ]

// distinct with a filter — unique assignees on active tasks only
db.tasks.distinct("assigneeId", { status: { $ne: "done" } });

// distinct on a nested field
db.users.distinct("profile.timezone");

// distinct on an array field — returns each unique value inside the arrays
db.tasks.distinct("tags");
// Returns all unique tag values across all task documents
```

---

## 13. Query on Nested Fields

Use **dot notation** to query inside embedded documents.

```js
// Query a nested field — find users in a specific timezone
db.users.find({ "profile.timezone": "Asia/Kolkata" });

// Query a nested field with a comparison operator
db.users.find({ "profile.experienceYears": { $gte: 5 } });

// Multiple nested field conditions
db.users.find({
  "preferences.notifications": true,
  "preferences.theme": "dark",
});

// Query two levels deep
db.tasks.find({ "metadata.source.platform": "web" });

// Nested field with exists — find users who have set a bio
db.users.find({ "profile.bio": { $exists: true } });

// Sort by a nested field
db.users.find().sort({ "profile.experienceYears": -1 });

// Project a nested field — return only the nested fields you need
db.users.find(
  { role: "developer" },
  { name: 1, "profile.timezone": 1, "profile.avatar": 1, _id: 0 },
);
```

---

## 14. Query on Arrays

```js
// Find documents where the array contains a specific value
db.tasks.find({ tags: "frontend" });

// Find documents where the array contains ANY of these values ($in)
db.tasks.find({ tags: { $in: ["frontend", "backend"] } });

// Find documents where the array contains ALL of these values ($all)
db.tasks.find({ tags: { $all: ["frontend", "urgent"] } });

// Find documents where a specific value is NOT in the array
db.tasks.find({ tags: { $nin: ["blocked", "deprecated"] } });

// Find documents where the array is empty
db.tasks.find({ tags: { $size: 0 } });

// Find documents where the array has a specific number of elements
db.tasks.find({ tags: { $size: 2 } });

// Query a specific index position in the array
db.tasks.find({ "tags.0": "design" }); // first tag must be "design"
db.tasks.find({ "tags.2": "urgent" }); // third tag must be "urgent"

// Query on an array of embedded documents
// Find sprints that have at least one task with status "done"
db.sprints.find({ "tasks.status": "done" });

// $elemMatch — all conditions must match on the SAME array element
db.sprints.find({
  tasks: {
    $elemMatch: {
      status: "in-progress",
      priority: "critical",
    },
  },
});
```

---

## 15. explain() — Query Analysis

`explain()` shows you exactly how MongoDB executes a query — whether it used an index or did a full scan.

```js
// Basic explain — shows the winning query plan
db.tasks.find({ status: "todo" }).explain();

// executionStats — shows detailed performance stats
db.tasks.find({ status: "todo" }).explain("executionStats");

// allPlansExecution — shows stats for all candidate plans considered
db.tasks.find({ status: "todo" }).explain("allPlansExecution");
```

### Key Fields to Read in executionStats

```js
// After running explain("executionStats"), look for these fields:

{
    executionStats: {
        nReturned:         10,     // number of documents returned
        totalDocsExamined: 10,     // documents MongoDB had to look at
        totalKeysExamined: 10,     // index keys scanned
        executionTimeMillis: 0     // how long the query took in milliseconds
    },
    queryPlanner: {
        winningPlan: {
            stage: "IXSCAN"        // IXSCAN = used an index (good)
                                   // COLLSCAN = full scan (bad on large collections)
        }
    }
}
```

| Stage        | Meaning                                                    |
| ------------ | ---------------------------------------------------------- |
| `IXSCAN`     | Used an index to find documents — fast                     |
| `COLLSCAN`   | Scanned the entire collection — slow on large data         |
| `FETCH`      | Retrieved full documents after an index scan               |
| `SORT`       | Performed an in-memory sort — add an index if this is slow |
| `PROJECTION` | Applied field projection                                   |
| `LIMIT`      | Applied a limit                                            |

```js
// The golden rule — totalDocsExamined should equal nReturned
// If totalDocsExamined >> nReturned, you are missing an index

// Example of a bad query — no index on status
db.tasks.find({ status: "todo" }).explain("executionStats");
// totalDocsExamined: 50000, nReturned: 120 → reading 50k to find 120 → add index

// Fix — create an index, then re-run explain
db.tasks.createIndex({ status: 1 });
db.tasks.find({ status: "todo" }).explain("executionStats");
// totalDocsExamined: 120, nReturned: 120 → perfect
```

---

## 16. Real Query Examples — Task Manager

These are realistic production queries for the Task Management Application.

```js
use taskmanager

// Get all incomplete tasks in a specific sprint, newest first
db.tasks.find(
    {
        sprintId: ObjectId("64f1a2b3c4d5e6f7a8b9c0db"),
        status:   { $nin: ["done", "cancelled"] }
    },
    { title: 1, status: 1, priority: 1, assigneeId: 1, dueDate: 1, _id: 1 }
).sort({ priority: -1, createdAt: -1 })

// Get all high and critical priority tasks assigned to a specific developer
db.tasks.find({
    assigneeId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"),
    priority:   { $in: ["high", "critical"] },
    status:     { $ne: "done" }
}).sort({ dueDate: 1 })

// Get overdue tasks — dueDate is in the past and task is not done
db.tasks.find({
    dueDate: { $lt: new Date() },
    status:  { $nin: ["done", "cancelled"] }
}).sort({ dueDate: 1 })

// Get all root-level tasks (not subtasks) in a project
db.tasks.find({
    projectId:    ObjectId("64f1a2b3c4d5e6f7a8b9c0da"),
    parentTaskId: { $exists: false }
}).sort({ createdAt: 1 })

// Get all subtasks of a specific parent task
db.tasks.find({
    parentTaskId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d5")
}).sort({ createdAt: 1 })

// Get all active projects with their owner info (two queries — no JOIN in simple MQL)
const activeProjects = db.projects.find({ status: "active" }).toArray()
const ownerIds = activeProjects.map(p => p.ownerId)
const owners = db.users.find({ _id: { $in: ownerIds } }).toArray()

// Search tasks by title keyword (requires text index)
// db.tasks.createIndex({ title: "text", description: "text" })
db.tasks.find(
    { $text: { $search: "authentication login" } },
    { score: { $meta: "textScore" }, title: 1, status: 1 }
).sort({ score: { $meta: "textScore" } })

// Get tasks created in the last 7 days
const sevenDaysAgo = new Date()
sevenDaysAgo.setDate(sevenDaysAgo.getDate() - 7)
db.tasks.find({
    createdAt: { $gte: sevenDaysAgo }
}).sort({ createdAt: -1 })

// Get users who have not logged in since a specific date
db.users.find({
    lastLoginAt: { $lt: new Date("2024-01-01") },
    isActive:    true
}).sort({ lastLoginAt: 1 })

// Paginated task list — page 2, 10 tasks per page, filtered by project
db.tasks.find(
    { projectId: ObjectId("64f1a2b3c4d5e6f7a8b9c0da") },
    { title: 1, status: 1, priority: 1, _id: 1 }
)
.sort({ createdAt: -1 })
.skip(10)
.limit(10)

// Count tasks by status in a project (without aggregation)
db.tasks.countDocuments({ projectId: ObjectId("64f1a2b3c4d5e6f7a8b9c0da"), status: "todo" })
db.tasks.countDocuments({ projectId: ObjectId("64f1a2b3c4d5e6f7a8b9c0da"), status: "in-progress" })
db.tasks.countDocuments({ projectId: ObjectId("64f1a2b3c4d5e6f7a8b9c0da"), status: "done" })
```

---

## 17. Quick Cheatsheet

| Command                                      | What it does                                    |
| -------------------------------------------- | ----------------------------------------------- |
| `db.col.find()`                              | Return all documents (as cursor)                |
| `db.col.find({})`                            | Same as above — empty filter matches everything |
| `db.col.findOne({})`                         | Return first matching document object           |
| `db.col.find({ field: val })`                | Filter by exact field value                     |
| `db.col.find({ f: { $gt: val } })`           | Greater than comparison                         |
| `db.col.find({ f: { $in: [] } })`            | Field value must be in array                    |
| `db.col.find({ $or: [] })`                   | At least one condition must match               |
| `db.col.find({ $and: [] })`                  | All conditions must match                       |
| `db.col.find({ f: { $exists: true } })`      | Field must exist                                |
| `db.col.find({ f: { $regex: "pattern" } })`  | Regex pattern match                             |
| `db.col.find({ $text: { $search: "..." } })` | Full-text search                                |
| `db.col.find({}, { field: 1 })`              | Include only specific fields                    |
| `db.col.find({}, { field: 0 })`              | Exclude specific fields                         |
| `db.col.find().sort({ f: -1 })`              | Sort descending                                 |
| `db.col.find().limit(10)`                    | Return max 10 documents                         |
| `db.col.find().skip(20)`                     | Skip first 20 documents                         |
| `db.col.find().skip(20).limit(10)`           | Pagination — page 3 of 10                       |
| `db.col.countDocuments({})`                  | Count matching documents                        |
| `db.col.estimatedDocumentCount()`            | Fast approximate total count                    |
| `db.col.distinct("field")`                   | Get all unique values of a field                |
| `db.col.find().explain("executionStats")`    | Analyze query performance                       |
| `db.col.find().forEach(fn)`                  | Iterate cursor with a function                  |
| `db.col.find().toArray()`                    | Convert cursor to array                         |

---

> **Prev:** [`Create.md`](./Create.md) — Insert operations and document design
> **Next:** [`Update.md`](./Update.md) — Deep dive into updateOne, updateMany, all update operators, and array updates
