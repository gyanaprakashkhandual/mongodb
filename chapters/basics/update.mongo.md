# MongoDB — Update Operations Reference

> **Project:** Task Management Application
> **Scope:** All update operations — updateOne, updateMany, replaceOne, findAndModify, all update operators
> **Shell:** `mongosh` (MongoDB Shell)
> **Prev:** [`Read.md`](./Read.md) — **Next:** [`Delete.md`](./Delete.md)

---

## Table of Contents

1. [How Update Works in MongoDB](#1-how-update-works-in-mongodb)
2. [updateOne](#2-updateone)
3. [updateMany](#3-updatemany)
4. [replaceOne](#4-replaceone)
5. [findOneAndUpdate](#5-findoneandupdate)
6. [findOneAndReplace](#6-findoneandreplace)
7. [Upsert](#7-upsert)
8. [Field Update Operators](#8-field-update-operators)
9. [Numeric Update Operators](#9-numeric-update-operators)
10. [Array Update Operators](#10-array-update-operators)
11. [Array Filters — Updating Specific Array Elements](#11-array-filters--updating-specific-array-elements)
12. [Update on Nested Fields](#12-update-on-nested-fields)
13. [Update Return Values](#13-update-return-values)
14. [Common Update Mistakes](#14-common-update-mistakes)
15. [Real Update Examples — Task Manager](#15-real-update-examples--task-manager)
16. [Quick Cheatsheet](#16-quick-cheatsheet)

---

## 1. How Update Works in MongoDB

When you call an update operation, MongoDB follows this path:

```
db.tasks.updateOne({ filter }, { update })
              │
              ▼
        Find matching document(s)
        using filter — index if available
              │
              ▼
        Apply the update operator
        ($set, $inc, $push, etc.)
              │
              ▼
        Validate against schema
        if a validator exists on the collection
              │
              ▼
        Write the change to WiredTiger storage
              │
              ▼
        Update all affected indexes
              │
              ▼
        Return result object
        { acknowledged, matchedCount, modifiedCount }
```

> NOTE: MongoDB updates are atomic at the single document level. If you update one document, either the full update succeeds or it does not happen at all. Multi-document updates are NOT atomic across documents unless you use transactions.

> NOTE: Without an update operator like `$set`, MongoDB will try to replace the entire document — always use an operator unless you intend a full replacement.

---

## 2. updateOne

`updateOne()` updates the **first document** that matches the filter. Even if 100 documents match, only one gets updated.

### Syntax

```js
db.collection.updateOne(
  { filter }, // which document to find
  { update }, // what change to apply
  { options }, // optional: upsert, arrayFilters, hint
);
```

### Basic updateOne

```js
// Update the status of a specific task by _id
db.tasks.updateOne(
  { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
  { $set: { status: "in-progress" } },
);

// Update multiple fields at once using $set
db.tasks.updateOne(
  { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
  {
    $set: {
      status: "in-progress",
      priority: "high",
      updatedAt: new Date(),
    },
  },
);

// Update by a non-_id field — updates the first match only
db.tasks.updateOne(
  { title: "Design homepage" },
  { $set: { status: "review" } },
);

// Update with a condition on multiple fields
db.tasks.updateOne(
  { status: "todo", priority: "critical" },
  { $set: { assigneeId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") } },
);
```

---

## 3. updateMany

`updateMany()` updates **all documents** that match the filter.

### Basic updateMany

```js
// Mark all todo tasks in a sprint as "backlog"
db.tasks.updateMany(
  { sprintId: ObjectId("64f1a2b3c4d5e6f7a8b9c0db"), status: "todo" },
  { $set: { status: "backlog" } },
);

// Set updatedAt on every document in the collection
db.tasks.updateMany({}, { $set: { updatedAt: new Date() } });

// Assign a default priority to all tasks that have none
db.tasks.updateMany(
  { priority: { $exists: false } },
  { $set: { priority: "medium" } },
);

// Deactivate all users who have not logged in since a date
db.users.updateMany(
  { lastLoginAt: { $lt: new Date("2023-01-01") } },
  { $set: { isActive: false } },
);

// Add a new field to every document in the collection
db.tasks.updateMany({}, { $set: { isArchived: false } });
```

---

## 4. replaceOne

`replaceOne()` replaces the **entire document** with a new one. Only the `_id` is preserved. All other fields are removed and replaced.

```js
// Replace a task document entirely — _id stays the same, everything else is replaced
db.tasks.replaceOne(
  { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
  {
    title: "Redesign homepage layout",
    status: "todo",
    priority: "high",
    projectId: ObjectId("64f1a2b3c4d5e6f7a8b9c0da"),
    assigneeId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d2"),
    createdAt: new Date(),
    updatedAt: new Date(),
  },
);
```

> NOTE: Do not include `_id` in the replacement document unless you want to keep the same value explicitly. MongoDB will throw an error if you include an `_id` that differs from the matched document.

> NOTE: Use `replaceOne` only when you intend to completely overwrite a document. For partial updates, always use `updateOne` with `$set`.

---

## 5. findOneAndUpdate

`findOneAndUpdate()` finds a document, updates it, and returns either the **original** or the **updated** document in a single atomic operation.

```js
// Return the document BEFORE the update (default)
db.tasks.findOneAndUpdate(
  { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
  { $set: { status: "done", completedAt: new Date() } },
);

// Return the document AFTER the update
db.tasks.findOneAndUpdate(
  { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
  { $set: { status: "done", completedAt: new Date() } },
  { returnDocument: "after" },
);

// With projection — return only specific fields of the updated document
db.tasks.findOneAndUpdate(
  { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
  { $set: { status: "done" } },
  {
    returnDocument: "after",
    projection: { title: 1, status: 1, updatedAt: 1 },
  },
);

// With sort — when multiple documents match, update the one sorted first
db.tasks.findOneAndUpdate(
  { status: "todo", priority: "critical" },
  { $set: { status: "in-progress", assigneeId: ObjectId("...") } },
  {
    sort: { createdAt: 1 }, // pick the oldest critical todo task
    returnDocument: "after",
  },
);

// With upsert — insert if no match found
db.tasks.findOneAndUpdate(
  { title: "Write release notes" },
  { $set: { status: "todo", priority: "low" } },
  {
    upsert: true,
    returnDocument: "after",
  },
);
```

| `returnDocument` value | What is returned                         |
| ---------------------- | ---------------------------------------- |
| `"before"` (default)   | The document as it was before the update |
| `"after"`              | The document as it is after the update   |

---

## 6. findOneAndReplace

`findOneAndReplace()` finds a document, replaces it entirely, and returns either the original or the new document.

```js
// Replace and return the document before replacement (default)
db.tasks.findOneAndReplace(
  { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
  {
    title: "New task title",
    status: "todo",
    priority: "medium",
    createdAt: new Date(),
  },
);

// Replace and return the new document
db.tasks.findOneAndReplace(
  { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
  {
    title: "New task title",
    status: "todo",
    priority: "medium",
    createdAt: new Date(),
  },
  { returnDocument: "after" },
);
```

---

## 7. Upsert

An upsert is an update that **inserts the document** if no match is found, and **updates it** if a match exists.

```js
// Upsert a user — create if not found, update if found
db.users.updateOne(
  { email: "arjun@taskmanager.com" },
  {
    $set: {
      name: "Arjun Mehta",
      role: "developer",
      isActive: true,
      updatedAt: new Date(),
    },
    $setOnInsert: {
      createdAt: new Date(), // only written when a new document is INSERTED
    },
  },
  { upsert: true },
);
```

### $setOnInsert

`$setOnInsert` only applies its fields when the upsert results in an **insert**, not when it results in an update.

```js
db.sprints.updateOne(
  { name: "Sprint 1", projectId: ObjectId("64f1a2b3c4d5e6f7a8b9c0da") },
  {
    $set: {
      status: "active",
      updatedAt: new Date(),
    },
    $setOnInsert: {
      createdAt: new Date(), // only set when document is newly created
      order: 1,
    },
  },
  { upsert: true },
);
```

> NOTE: Upsert is safe to run multiple times. If the document already exists, only the `$set` block applies. `$setOnInsert` is ignored on updates. This makes it ideal for idempotent seeding scripts.

---

## 8. Field Update Operators

These operators modify individual fields on a document.

```js
// $set — set a field to a new value, adds the field if it does not exist
db.tasks.updateOne(
  { _id: id },
  { $set: { status: "done", updatedAt: new Date() } },
);

// $unset — remove a field from the document entirely
// The value you pass does not matter — "" or 1 both work
db.tasks.updateOne(
  { _id: id },
  { $unset: { parentTaskId: "", temporaryNote: "" } },
);

// $rename — rename a field without changing its value
// Useful when you decide to rename a field across all documents
db.tasks.updateMany({}, { $rename: { desc: "description" } });

// $setOnInsert — only applies when the operation is an insert (used with upsert)
db.tasks.updateOne(
  { title: "New task" },
  {
    $set: { status: "todo" },
    $setOnInsert: { createdAt: new Date() },
  },
  { upsert: true },
);

// $currentDate — set a field to the current server date
db.tasks.updateOne(
  { _id: id },
  { $currentDate: { updatedAt: true } }, // sets as Date
);

// $currentDate with explicit type
db.tasks.updateOne(
  { _id: id },
  { $currentDate: { updatedAt: { $type: "date" } } }, // same as above
);
```

---

## 9. Numeric Update Operators

These operators work only on numeric fields.

```js
// $inc — increment a field by a positive or negative value
// Adds the field with the given value if it does not exist
db.tasks.updateOne(
  { _id: id },
  { $inc: { storyPoints: 1 } }, // storyPoints + 1
);

db.tasks.updateOne(
  { _id: id },
  { $inc: { storyPoints: -2 } }, // storyPoints - 2
);

// $inc on multiple fields at once
db.sprints.updateOne(
  { _id: sprintId },
  {
    $inc: {
      totalTasks: 1,
      completedTasks: 0, // no change, but field is initialized if missing
    },
  },
);

// $mul — multiply a field by a value
db.tasks.updateOne(
  { _id: id },
  { $mul: { estimatedHours: 1.5 } }, // multiply estimatedHours by 1.5
);

// $min — update only if the new value is LESS than the current value
// Useful for tracking the minimum value seen
db.tasks.updateOne(
  { _id: id },
  { $min: { storyPoints: 3 } }, // only updates if current storyPoints > 3
);

// $max — update only if the new value is GREATER than the current value
// Useful for tracking the maximum value seen
db.tasks.updateOne(
  { _id: id },
  { $max: { storyPoints: 10 } }, // only updates if current storyPoints < 10
);
```

---

## 10. Array Update Operators

These operators add, remove, or modify elements inside array fields.

### $push — Add to Array

```js
// Add a single value to an array
db.tasks.updateOne({ _id: id }, { $push: { tags: "urgent" } });

// Add multiple values using $each
db.tasks.updateOne(
  { _id: id },
  { $push: { tags: { $each: ["frontend", "ui", "responsive"] } } },
);

// Add and keep array sorted using $sort
db.tasks.updateOne(
  { _id: id },
  {
    $push: {
      comments: {
        $each: [{ text: "Looks good", createdAt: new Date() }],
        $sort: { createdAt: -1 }, // sort comments newest first
      },
    },
  },
);

// Add and limit array size using $slice
// Keep only the 5 most recent activity entries
db.users.updateOne(
  { _id: id },
  {
    $push: {
      recentActivity: {
        $each: [{ action: "login", at: new Date() }],
        $sort: { at: -1 },
        $slice: 5, // keep only the latest 5 entries
      },
    },
  },
);

// Add to a specific position using $position
db.tasks.updateOne(
  { _id: id },
  {
    $push: {
      tags: {
        $each: ["pinned"],
        $position: 0, // insert at index 0 (beginning of array)
      },
    },
  },
);
```

### $addToSet — Add Only if Unique

```js
// Add a value only if it does not already exist in the array (no duplicates)
db.tasks.updateOne({ _id: id }, { $addToSet: { tags: "frontend" } });

// Add multiple unique values using $each
db.tasks.updateOne(
  { _id: id },
  { $addToSet: { tags: { $each: ["frontend", "ui", "frontend"] } } },
  // "frontend" appears twice in $each but only added once — duplicates skipped
);
```

### $pull — Remove from Array

```js
// Remove a specific value from an array
db.tasks.updateOne({ _id: id }, { $pull: { tags: "urgent" } });

// Remove elements that match a condition
db.tasks.updateOne(
  { _id: id },
  { $pull: { comments: { isDeleted: true } } }, // remove all deleted comments
);

// Remove elements matching a comparison
db.sprints.updateOne(
  { _id: sprintId },
  { $pull: { taskIds: { $in: [ObjectId("..."), ObjectId("...")] } } },
);
```

### $pullAll — Remove Multiple Specific Values

```js
// Remove all occurrences of any value in the provided list
db.tasks.updateOne(
  { _id: id },
  { $pullAll: { tags: ["urgent", "outdated", "deprecated"] } },
);
```

### $pop — Remove First or Last Element

```js
// Remove the last element of an array
db.tasks.updateOne(
  { _id: id },
  { $pop: { tags: 1 } }, // 1 = last element
);

// Remove the first element of an array
db.tasks.updateOne(
  { _id: id },
  { $pop: { tags: -1 } }, // -1 = first element
);
```

---

## 11. Array Filters — Updating Specific Array Elements

Array filters let you target and update **specific elements inside an array** that match a condition, without replacing the whole array.

### The Positional Operator $

The `$` placeholder refers to the **first array element that matched the query filter**.

```js
// Update the status of the first matching embedded task
db.sprints.updateOne(
  { "tasks.taskId": ObjectId("64f1a2b3c4d5e6f7a8b9c0d5") }, // filter finds the document
  { $set: { "tasks.$.status": "done" } }, // $ refers to matched element
);

// Mark the first comment from a specific user as edited
db.tasks.updateOne(
  {
    _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"),
    "comments.userId": ObjectId("64f1a2b3c4d5e6f7a8b9c0d2"),
  },
  {
    $set: {
      "comments.$.isEdited": true,
      "comments.$.updatedAt": new Date(),
    },
  },
);
```

### arrayFilters — Update All Matching Array Elements

`arrayFilters` lets you update **all array elements** matching a condition, not just the first.

```js
// Update all tasks inside a sprint that have status "todo" to "backlog"
db.sprints.updateOne(
  { _id: sprintId },
  {
    $set: { "tasks.$[task].status": "backlog" },
  },
  {
    arrayFilters: [{ "task.status": "todo" }], // defines what "task" refers to
  },
);

// Update all comments marked as spam inside a task
db.tasks.updateOne(
  { _id: taskId },
  {
    $set: { "comments.$[comment].isHidden": true },
  },
  {
    arrayFilters: [{ "comment.isSpam": true }],
  },
);

// Update array elements matching a numeric condition
db.tasks.updateOne(
  { _id: taskId },
  {
    $set: { "subtasks.$[sub].priority": "high" },
  },
  {
    arrayFilters: [{ "sub.storyPoints": { $gte: 8 } }],
  },
);

// $[*] — update ALL elements in the array unconditionally
db.tasks.updateOne(
  { _id: taskId },
  { $set: { "tags.$[]": "reviewed" } }, // set every tag value to "reviewed"
);
```

---

## 12. Update on Nested Fields

Use **dot notation** to update fields inside embedded documents.

```js
// Update a single nested field
db.users.updateOne(
  { _id: userId },
  { $set: { "profile.bio": "Updated bio text" } },
);

// Update multiple nested fields at once
db.users.updateOne(
  { _id: userId },
  {
    $set: {
      "profile.avatar": "https://cdn.taskmanager.com/new-avatar.jpg",
      "profile.timezone": "Asia/Kolkata",
      updatedAt: new Date(),
    },
  },
);

// Update a nested field inside preferences
db.users.updateOne({ _id: userId }, { $set: { "preferences.theme": "light" } });

// Update three levels deep
db.projects.updateOne(
  { _id: projectId },
  { $set: { "settings.notifications.email": false } },
);

// Unset a nested field
db.users.updateOne({ _id: userId }, { $unset: { "profile.phone": "" } });

// Increment a nested numeric field
db.users.updateOne({ _id: userId }, { $inc: { "stats.tasksCompleted": 1 } });
```

---

## 13. Update Return Values

```js
// updateOne and updateMany return the same shape
{
    acknowledged:  true,    // write was confirmed by the server
    matchedCount:  1,       // number of documents that matched the filter
    modifiedCount: 1,       // number of documents actually changed
    upsertedCount: 0,       // number of documents inserted via upsert
    upsertedId:    null     // _id of the upserted document if applicable
}

// matchedCount vs modifiedCount — they can differ
// If a document matched but the value was already correct, it is not modified
db.tasks.updateOne(
    { _id: id },
    { $set: { status: "todo" } }   // status is already "todo"
)
// Result: { matchedCount: 1, modifiedCount: 0 }
// Matched 1 document but modified 0 because nothing changed

// upsert result when a new document is inserted
db.tasks.updateOne(
    { title: "Brand new task" },
    { $set: { status: "todo" } },
    { upsert: true }
)
// Result: { matchedCount: 0, modifiedCount: 0, upsertedCount: 1, upsertedId: ObjectId("...") }

// findOneAndUpdate returns the document itself, not a result summary
const updated = db.tasks.findOneAndUpdate(
    { _id: id },
    { $set: { status: "done" } },
    { returnDocument: "after" }
)
// updated is the full document object
```

---

## 14. Common Update Mistakes

### Forgetting the update operator — accidental full replacement

```js
// BAD — no $set means the entire document is replaced with just { status: "done" }
// The _id survives but every other field is deleted
db.tasks.updateOne(
  { _id: id },
  { status: "done" }, // this is treated as a full replacement, not a field update
);

// GOOD — use $set to update only the fields you specify
db.tasks.updateOne({ _id: id }, { $set: { status: "done" } });
```

### Using updateOne when you need updateMany

```js
// BAD — only the first matching task gets updated, the rest are left unchanged
db.tasks.updateOne({ status: "todo" }, { $set: { priority: "medium" } });

// GOOD — all matching tasks are updated
db.tasks.updateMany({ status: "todo" }, { $set: { priority: "medium" } });
```

### $push instead of $addToSet when duplicates matter

```js
// BAD — pushing the same tag every time creates duplicates in the array
db.tasks.updateOne({ _id: id }, { $push: { tags: "frontend" } });
// tags: ["design", "frontend", "frontend", "frontend"]

// GOOD — $addToSet ensures no duplicates
db.tasks.updateOne({ _id: id }, { $addToSet: { tags: "frontend" } });
// tags: ["design", "frontend"]
```

### Updating with a string date instead of a Date object

```js
// BAD — stored as a plain string, cannot be compared as a date
db.tasks.updateOne({ _id: id }, { $set: { dueDate: "2024-02-10" } });

// GOOD — stored as a BSON Date, can be queried with $gt, $lt, date range
db.tasks.updateOne({ _id: id }, { $set: { dueDate: new Date("2024-02-10") } });
```

### Not using $currentDate for timestamps

```js
// ACCEPTABLE — works, but ties the timestamp to the client machine's time
db.tasks.updateOne({ _id: id }, { $set: { updatedAt: new Date() } });

// BETTER for server consistency — uses the MongoDB server's clock
db.tasks.updateOne({ _id: id }, { $currentDate: { updatedAt: true } });
```

### Using $ positional operator without the matching field in the filter

```js
// BAD — the $ operator requires the array field to be part of the query filter
db.tasks.updateOne(
  { _id: id }, // does not filter on "comments" field
  { $set: { "comments.$.text": "x" } }, // $ has no matched element to refer to — ERROR
);

// GOOD — the filter must include the array field and condition
db.tasks.updateOne(
  { _id: id, "comments.userId": userId }, // filter on the array field
  { $set: { "comments.$.text": "Updated" } },
);
```

---

## 15. Real Update Examples — Task Manager

```js
use taskmanager

// Move a task to in-progress and assign it to a developer
db.tasks.updateOne(
    { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
    {
        $set: {
            status:     "in-progress",
            assigneeId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d2"),
            startedAt:  new Date(),
            updatedAt:  new Date()
        }
    }
)

// Mark a task as done and record completion time
db.tasks.updateOne(
    { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
    {
        $set: {
            status:      "done",
            completedAt: new Date(),
            updatedAt:   new Date()
        }
    }
)

// Add a comment to a task
db.tasks.updateOne(
    { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
    {
        $push: {
            comments: {
                _id:       new ObjectId(),
                userId:    ObjectId("64f1a2b3c4d5e6f7a8b9c0d2"),
                text:      "Looks good, ready for review.",
                createdAt: new Date()
            }
        }
    }
)

// Add tags to a task without creating duplicates
db.tasks.updateOne(
    { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
    { $addToSet: { tags: { $each: ["frontend", "ui"] } } }
)

// Increment story points on a task
db.tasks.updateOne(
    { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
    { $inc: { storyPoints: 2 } }
)

// Update sprint — mark as completed and set end date to now
db.sprints.updateOne(
    { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0db") },
    {
        $set: {
            status:    "completed",
            endDate:   new Date(),
            updatedAt: new Date()
        }
    }
)

// Move all incomplete tasks from a closed sprint to backlog
db.tasks.updateMany(
    {
        sprintId: ObjectId("64f1a2b3c4d5e6f7a8b9c0db"),
        status:   { $nin: ["done", "cancelled"] }
    },
    {
        $set:   { status: "backlog", updatedAt: new Date() },
        $unset: { sprintId: "" }
    }
)

// Update a user's profile timezone and notification preferences
db.users.updateOne(
    { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
    {
        $set: {
            "profile.timezone":            "America/New_York",
            "preferences.notifications":   false,
            "preferences.theme":           "light",
            updatedAt:                     new Date()
        }
    }
)

// Promote a developer to manager role
db.users.updateOne(
    { email: "rohit@taskmanager.com" },
    {
        $set: {
            role:      "manager",
            updatedAt: new Date()
        }
    }
)

// Remove a tag from all tasks in a project
db.tasks.updateMany(
    { projectId: ObjectId("64f1a2b3c4d5e6f7a8b9c0da") },
    { $pull: { tags: "deprecated" } }
)

// Bulk status update using arrayFilters on embedded sprint tasks
db.sprints.updateOne(
    { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0db") },
    { $set: { "tasks.$[t].status": "backlog" } },
    { arrayFilters: [{ "t.status": "todo" }] }
)

// Upsert a project setting — create if missing, update if present
db.projects.updateOne(
    {
        _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0da"),
        "settings.key": "max_members"
    },
    {
        $set: { "settings.$.value": 25 }
    },
    { upsert: false }
)

// Track the last 10 logins for a user using $push with $slice
db.users.updateOne(
    { _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") },
    {
        $set: { lastLoginAt: new Date() },
        $push: {
            loginHistory: {
                $each:  [{ loginAt: new Date(), ip: "103.21.58.11" }],
                $sort:  { loginAt: -1 },
                $slice: 10    // keep only the 10 most recent entries
            }
        }
    }
)
```

---

## 16. Quick Cheatsheet

| Command                                                        | What it does                              |
| -------------------------------------------------------------- | ----------------------------------------- |
| `db.col.updateOne({filter}, {update})`                         | Update the first matching document        |
| `db.col.updateMany({filter}, {update})`                        | Update all matching documents             |
| `db.col.replaceOne({filter}, {doc})`                           | Replace entire document, keep \_id        |
| `db.col.findOneAndUpdate({}, {}, { returnDocument: "after" })` | Update and return document                |
| `{ $set: { field: val } }`                                     | Set a field value                         |
| `{ $unset: { field: "" } }`                                    | Remove a field                            |
| `{ $rename: { old: "new" } }`                                  | Rename a field                            |
| `{ $currentDate: { field: true } }`                            | Set field to current server date          |
| `{ $inc: { field: n } }`                                       | Increment a number field by n             |
| `{ $mul: { field: n } }`                                       | Multiply a number field by n              |
| `{ $min: { field: n } }`                                       | Update only if new value is smaller       |
| `{ $max: { field: n } }`                                       | Update only if new value is larger        |
| `{ $push: { arr: val } }`                                      | Append a value to an array                |
| `{ $push: { arr: { $each: [] } } }`                            | Append multiple values to an array        |
| `{ $push: { arr: { $each: [], $slice: n } } }`                 | Append and limit array size               |
| `{ $addToSet: { arr: val } }`                                  | Add to array only if not a duplicate      |
| `{ $pull: { arr: val } }`                                      | Remove matching value from array          |
| `{ $pullAll: { arr: [] } }`                                    | Remove multiple values from array         |
| `{ $pop: { arr: 1 } }`                                         | Remove last element of array              |
| `{ $pop: { arr: -1 } }`                                        | Remove first element of array             |
| `"arr.$.field"`                                                | Update first matched array element        |
| `"arr.$[elem].field"` with `arrayFilters`                      | Update all matched array elements         |
| `"arr.$[]"`                                                    | Update all elements in array              |
| `{ upsert: true }`                                             | Insert if no match found                  |
| `{ $setOnInsert: {} }`                                         | Set fields only when inserting via upsert |

---

> **Prev:** [`Read.md`](./Read.md) — Read operations and query operators
> **Next:** [`Delete.md`](./Delete.md) — Deep dive into deleteOne, deleteMany, findOneAndDelete, soft deletes, and bulk deletes
