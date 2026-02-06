# Core Database

Schema definitions, queries, and mutations — the foundation of Convex development.

## Quick Start

Install and initialize a Convex project:

```bash
npm install convex
npx convex dev
```

This creates the required directory structure:
- `convex/` — Backend functions and schema
- `convex/_generated/` — Auto-generated TypeScript types (git-ignored)
- `convex/schema.ts` — Database schema definition

First query function:

```typescript
// Source: https://docs.convex.dev/functions
import { query } from "./_generated/server";

export const list = query({
  args: {},
  handler: async (ctx) => {
    return await ctx.db.query("tasks").collect();
  },
});
```

## Schema Definitions

Define database tables with typed validators in `convex/schema.ts`.

### Schema Structure

```typescript
// Source: https://docs.convex.dev/database/schemas
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  tableName: defineTable({
    field: v.string(),
    // ... more fields
  })
    .index("index_name", ["field1", "field2"])
    .index("another_index", ["field3"]),
});
```

### All Validators

```typescript
// Source: https://docs.convex.dev/functions/validation

// Primitives
v.string()      // UTF-8 strings
v.number()      // IEEE-754 floating point
v.int64()       // BigInt values
v.boolean()     // true/false
v.null()        // null values
v.bytes()       // ArrayBuffer

// Collections
v.array(v.string())              // Arrays (max 8,192 items)
v.object({ key: v.string() })    // Objects (max 1,024 fields)
v.record(v.string(), v.number()) // Dynamic-key objects

// Composite
v.union(v.string(), v.number())  // Multiple allowed types
v.nullable(v.string())           // Equivalent to v.union(v.string(), v.null())
v.literal("active")              // Constant values
v.optional(v.string())           // Optional fields
v.any()                          // Accept any value

// Special
v.id("tableName")                // Document IDs with type checking
```

### Complete Schema Example

```typescript
// Source: https://docs.convex.dev/database/schemas
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  users: defineTable({
    name: v.string(),
    email: v.string(),
    tokenIdentifier: v.string(),
    createdAt: v.optional(v.number()),
  })
    .index("by_token", ["tokenIdentifier"])
    .index("by_email", ["email"]),

  channels: defineTable({
    name: v.string(),
    description: v.optional(v.string()),
    isPrivate: v.boolean(),
  }).index("by_name", ["name"]),

  messages: defineTable({
    channel: v.id("channels"),
    author: v.id("users"),
    body: v.string(),
    attachments: v.optional(v.array(v.string())),
  })
    .index("by_channel", ["channel"])
    .index("by_channel_creation", ["channel", "_creationTime"])
    .index("by_author", ["author"]),
});
```

### Index Definitions

Indexes enable efficient queries. Define indexes on fields you'll query:

```typescript
// Source: https://docs.convex.dev/database/indexes
defineTable({
  status: v.string(),
  priority: v.number(),
  assignee: v.id("users"),
})
  .index("by_status", ["status"])
  .index("by_assignee", ["assignee"])
  .index("by_status_priority", ["status", "priority"]);
```

**Key point:** Every `.withIndex()` query must have a matching `.index()` definition.

## Query Functions

Read-only functions with automatic caching and real-time subscriptions.

### Basic Query

```typescript
// Source: https://docs.convex.dev/database/reading-data
import { query } from "./_generated/server";
import { v } from "convex/values";

export const listTasks = query({
  args: {},
  handler: async (ctx) => {
    return await ctx.db.query("tasks").collect();
  },
});
```

### Query with Arguments

```typescript
// Source: https://docs.convex.dev/database/reading-data
export const getByStatus = query({
  args: {
    status: v.string(),
  },
  handler: async (ctx, args) => {
    return await ctx.db
      .query("tasks")
      .filter((q) => q.eq(q.field("status"), args.status))
      .collect();
  },
});
```

### Query API Chain

```typescript
// Source: https://docs.convex.dev/database/reading-data
const results = await ctx.db
  .query("tableName")
  .withIndex("index_name", (q) => q.eq("field", value))  // Use index (best performance)
  .filter((q) => q.eq(q.field("otherField"), value))    // Additional filtering
  .order("asc")                                          // "asc" or "desc"
  .collect();                                            // Returns Doc[]

// Alternative terminations:
.first()     // Returns Doc | null
.unique()    // Returns Doc (throws if 0 or >1 results)
.take(10)    // Returns Doc[] (max 10)
```

### Index Queries

**Always prefer `.withIndex()` over `.filter()` for performance.**

```typescript
// Source: https://docs.convex.dev/database/indexes
export const listByChannel = query({
  args: {
    channelId: v.id("channels"),
  },
  handler: async (ctx, args) => {
    return await ctx.db
      .query("messages")
      .withIndex("by_channel", (q) => q.eq("channel", args.channelId))
      .order("desc")
      .collect();
  },
});
```

### Range Queries

```typescript
// Source: https://docs.convex.dev/database/indexes
export const recentMessages = query({
  args: {
    channelId: v.id("channels"),
    startTime: v.number(),
    endTime: v.number(),
  },
  handler: async (ctx, args) => {
    return await ctx.db
      .query("messages")
      .withIndex("by_channel_creation", (q) =>
        q.eq("channel", args.channelId)
         .gt("_creationTime", args.startTime)
         .lt("_creationTime", args.endTime)
      )
      .collect();
  },
});
```

### Pagination

```typescript
// Source: https://docs.convex.dev/database/pagination
import { query } from "./_generated/server";
import { paginationOptsValidator } from "convex/server";
import { v } from "convex/values";

export const listPaginated = query({
  args: {
    channelId: v.id("channels"),
    paginationOpts: paginationOptsValidator,
  },
  handler: async (ctx, args) => {
    return await ctx.db
      .query("messages")
      .withIndex("by_channel", (q) => q.eq("channel", args.channelId))
      .order("desc")
      .paginate(args.paginationOpts);
  },
});

// Returns: { page: Doc[], continueCursor: string | null, isDone: boolean }
```

### Get by ID

```typescript
// Source: https://docs.convex.dev/database/reading-data
export const getTask = query({
  args: {
    taskId: v.id("tasks"),
  },
  handler: async (ctx, args) => {
    const task = await ctx.db.get(args.taskId);
    if (!task) {
      throw new Error("Task not found");
    }
    return task;
  },
});
```

### Anti-Patterns

**Don't use `.filter()` when you can use `.withIndex()`:**

```typescript
// BAD: Scans all documents
const tasks = await ctx.db
  .query("tasks")
  .filter((q) => q.eq(q.field("status"), "active"))
  .collect();

// GOOD: Uses index (add .index("by_status", ["status"]) to schema)
const tasks = await ctx.db
  .query("tasks")
  .withIndex("by_status", (q) => q.eq("status", "active"))
  .collect();
```

**Don't use unbounded `.collect()` on large datasets:**

```typescript
// BAD: Could return millions of documents
const allMessages = await ctx.db.query("messages").collect();

// GOOD: Use pagination
const result = await ctx.db
  .query("messages")
  .order("desc")
  .paginate(paginationOpts);
```

## Mutation Functions

Transactional write operations for creating, updating, and deleting data.

### Insert

```typescript
// Source: https://docs.convex.dev/database/writing-data
import { mutation } from "./_generated/server";
import { v } from "convex/values";

export const createTask = mutation({
  args: {
    title: v.string(),
    description: v.optional(v.string()),
  },
  handler: async (ctx, args) => {
    const taskId = await ctx.db.insert("tasks", {
      title: args.title,
      description: args.description,
      status: "pending",
      createdAt: Date.now(),
    });
    return taskId;  // Returns Id<"tasks">
  },
});
```

### Patch (Partial Update)

```typescript
// Source: https://docs.convex.dev/database/writing-data
export const updateTaskStatus = mutation({
  args: {
    taskId: v.id("tasks"),
    status: v.string(),
  },
  handler: async (ctx, args) => {
    await ctx.db.patch(args.taskId, {
      status: args.status,
      updatedAt: Date.now(),
    });
  },
});
```

### Replace (Full Update)

```typescript
// Source: https://docs.convex.dev/database/writing-data
export const replaceTask = mutation({
  args: {
    taskId: v.id("tasks"),
    title: v.string(),
    description: v.string(),
    status: v.string(),
  },
  handler: async (ctx, args) => {
    await ctx.db.replace(args.taskId, {
      title: args.title,
      description: args.description,
      status: args.status,
      updatedAt: Date.now(),
    });
  },
});
```

### Delete

```typescript
// Source: https://docs.convex.dev/database/writing-data
export const deleteTask = mutation({
  args: {
    taskId: v.id("tasks"),
  },
  handler: async (ctx, args) => {
    await ctx.db.delete(args.taskId);
  },
});
```

### Transaction Guarantees

All operations within a mutation are atomic — either all succeed or all fail:

```typescript
// Source: https://docs.convex.dev/database/writing-data
export const transferOwnership = mutation({
  args: {
    taskId: v.id("tasks"),
    newOwnerId: v.id("users"),
  },
  handler: async (ctx, args) => {
    const task = await ctx.db.get(args.taskId);
    if (!task) {
      throw new Error("Task not found");
    }

    // Both operations succeed or both fail
    await ctx.db.patch(args.taskId, { owner: args.newOwnerId });
    await ctx.db.insert("audit_log", {
      action: "transfer",
      taskId: args.taskId,
      newOwner: args.newOwnerId,
      timestamp: Date.now(),
    });
  },
});
```

### Mutation with Authentication

```typescript
// Source: https://docs.convex.dev/auth
export const createMessage = mutation({
  args: {
    channelId: v.id("channels"),
    body: v.string(),
  },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) {
      throw new Error("Not authenticated");
    }

    const messageId = await ctx.db.insert("messages", {
      channel: args.channelId,
      body: args.body,
      author: identity.subject,
    });

    return messageId;
  },
});
```
