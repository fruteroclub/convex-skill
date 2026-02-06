# Patterns and Best Practices

Error handling, TypeScript types, code organization, environment variables, and common pitfalls.

## Error Handling

Use `ConvexError` for application errors that preserve structured data in production.

### ConvexError with String

```typescript
// Source: https://docs.convex.dev/functions/error-handling
import { ConvexError } from "convex/values";
import { mutation } from "./_generated/server";
import { v } from "convex/values";

export const deleteTask = mutation({
  args: { taskId: v.id("tasks") },
  handler: async (ctx, args) => {
    const task = await ctx.db.get(args.taskId);
    if (!task) {
      // Simple string error
      throw new ConvexError("Task not found");
    }

    await ctx.db.delete(args.taskId);
  },
});
```

### ConvexError with Structured Data

```typescript
// Source: https://docs.convex.dev/functions/error-handling
import { ConvexError } from "convex/values";
import { mutation } from "./_generated/server";
import { v } from "convex/values";

export const updateTaskStatus = mutation({
  args: {
    taskId: v.id("tasks"),
    status: v.string(),
  },
  handler: async (ctx, args) => {
    const task = await ctx.db.get(args.taskId);
    if (!task) {
      throw new ConvexError({
        message: "Task not found",
        code: "TASK_NOT_FOUND",
        taskId: args.taskId,
      });
    }

    if (task.locked) {
      // Structured error with data field preserved in production
      throw new ConvexError({
        message: "Task is locked by another user",
        code: "TASK_LOCKED",
        lockedBy: task.lockedBy,
        lockedAt: task.lockedAt,
      });
    }

    await ctx.db.patch(args.taskId, { status: args.status });
  },
});
```

### Client-Side Error Handling

```typescript
// Source: https://docs.convex.dev/functions/error-handling
import { useMutation } from "convex/react";
import { api } from "../convex/_generated/api";

function TaskManager() {
  const updateStatus = useMutation(api.tasks.updateTaskStatus);

  const handleUpdate = async (taskId, status) => {
    try {
      await updateStatus({ taskId, status });
    } catch (error) {
      // ConvexError data is preserved
      if (error.data?.code === "TASK_LOCKED") {
        console.error(`Task locked by ${error.data.lockedBy}`);
      } else {
        console.error("Update failed:", error.message);
      }
    }
  };

  // Rest of component...
}
```

### Error Handling in Actions

```typescript
// Source: https://docs.convex.dev/functions/error-handling
import { action } from "./_generated/server";
import { internal } from "./_generated/api";
import { v } from "convex/values";

export const fetchAndStore = action({
  args: { url: v.string() },
  handler: async (ctx, args) => {
    try {
      // External API call
      const response = await fetch(args.url);

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      const data = await response.json();

      // Store in database via mutation
      await ctx.runMutation(internal.data.store, { data });

      return { success: true };
    } catch (error) {
      console.error("External API failed:", error);

      // Log error to database
      await ctx.runMutation(internal.logs.error, {
        source: "fetchAndStore",
        message: error.message,
        url: args.url,
      });

      throw error;
    }
  },
});
```

## TypeScript Types

Use generated types from `convex/_generated/dataModel.ts` for full type safety.

### Generated Document Types

```typescript
// Source: https://docs.convex.dev/generated-api
import { Doc, Id } from "./_generated/dataModel";

// Doc<"tableName"> - Full document type with _id and _creationTime
function processTask(task: Doc<"tasks">) {
  console.log(task._id);           // Id<"tasks">
  console.log(task._creationTime); // number
  console.log(task.title);         // string (from schema)
  console.log(task.status);        // string (from schema)
}

// Id<"tableName"> - Typed document ID
function getTaskById(taskId: Id<"tasks">) {
  // TypeScript ensures you pass the right ID type
}
```

### DataModel Type

```typescript
// Source: https://docs.convex.dev/generated-api
import { DataModel } from "./_generated/dataModel";

// Full database model type
function logAllTables(dataModel: DataModel) {
  // TypeScript knows all your table names
}
```

### Function Return Types

```typescript
// Source: https://docs.convex.dev/generated-api
import { query } from "./_generated/server";
import { v } from "convex/values";

// Return type is inferred from handler
export const getTask = query({
  args: { taskId: v.id("tasks") },
  handler: async (ctx, args) => {
    const task = await ctx.db.get(args.taskId);
    // Return type: Doc<"tasks"> | null (inferred)
    return task;
  },
});

// Explicit return type annotation (optional)
export const listTasks = query({
  args: {},
  handler: async (ctx): Promise<Doc<"tasks">[]> => {
    return await ctx.db.query("tasks").collect();
  },
});
```

### Avoiding Circular Imports

```typescript
// Source: https://docs.convex.dev/generated-api

// ❌ DON'T: Import from schema.ts (causes circular import errors)
import { TaskStatus } from "./schema";

// ✅ DO: Import from generated dataModel
import { Doc } from "./_generated/dataModel";

function getStatus(task: Doc<"tasks">) {
  return task.status;
}
```

## Code Organization

Organize Convex functions with thin public APIs and business logic in helper modules.

### Directory Structure

```typescript
// Source: Project structure from research
convex/
├── schema.ts              // Database schema (required)
├── _generated/            // Auto-generated (git-ignored)
├── model/                 // Helper functions and business logic
│   ├── tasks.ts
│   └── users.ts
├── tasks/                 // Resource-based organization
│   ├── queries.ts
│   └── mutations.ts
├── users/
│   ├── queries.ts
│   └── mutations.ts
├── crons.ts               // Scheduled functions (if needed)
└── http.ts                // HTTP actions (if needed)
```

### Thin Public API Pattern

```typescript
// convex/tasks/mutations.ts
// Public API: validation + access control only
import { mutation } from "../_generated/server";
import { v } from "convex/values";
import { createTask, updateTaskStatus } from "../model/tasks";

export const create = mutation({
  args: {
    title: v.string(),
    description: v.optional(v.string()),
  },
  handler: async (ctx, args) => {
    // Authentication check
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) {
      throw new Error("Not authenticated");
    }

    // Delegate to model layer
    return await createTask(ctx, {
      title: args.title,
      description: args.description,
      createdBy: identity.subject,
    });
  },
});
```

### Business Logic in Model Layer

```typescript
// convex/model/tasks.ts
// Business logic: reusable functions, no direct export to client
import { MutationCtx } from "../_generated/server";

export async function createTask(
  ctx: MutationCtx,
  args: {
    title: string;
    description?: string;
    createdBy: string;
  }
) {
  // Business logic here
  const taskId = await ctx.db.insert("tasks", {
    title: args.title,
    description: args.description,
    status: "pending",
    createdBy: args.createdBy,
    createdAt: Date.now(),
  });

  // Trigger side effects
  await notifyAssignees(ctx, taskId);

  return taskId;
}

async function notifyAssignees(ctx: MutationCtx, taskId: string) {
  // Additional business logic
}
```

### Internal vs Public Functions

```typescript
// Source: https://docs.convex.dev/functions
import { internalMutation, mutation } from "./_generated/server";
import { v } from "convex/values";

// Public: callable from client
export const createTask = mutation({
  args: { title: v.string() },
  handler: async (ctx, args) => {
    // Validate auth
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) {
      throw new Error("Not authenticated");
    }

    return await ctx.db.insert("tasks", {
      title: args.title,
      createdBy: identity.subject,
    });
  },
});

// Internal: only callable from other Convex functions (actions, crons)
export const markPaid = internalMutation({
  args: {
    taskId: v.id("tasks"),
    paymentId: v.string(),
  },
  handler: async (ctx, args) => {
    // No auth check needed - only called internally
    await ctx.db.patch(args.taskId, {
      status: "paid",
      paymentId: args.paymentId,
    });
  },
});
```

## Environment Variables

Access environment variables via `process.env` in any Convex function.

### Basic Usage

```typescript
// Source: https://docs.convex.dev/production/environment-variables
import { action } from "./_generated/server";

export const callExternalApi = action({
  args: {},
  handler: async (ctx) => {
    const apiKey = process.env.API_KEY;
    const apiUrl = process.env.API_URL;

    const response = await fetch(`${apiUrl}/data`, {
      headers: { "Authorization": `Bearer ${apiKey}` },
    });

    return await response.json();
  },
});
```

### Built-in Environment Variables

```typescript
// Source: https://docs.convex.dev/production/environment-variables

// Always available in Convex functions:
process.env.CONVEX_CLOUD_URL  // Your deployment URL
process.env.CONVEX_SITE_URL   // Site URL for HTTP actions
```

### Setting Environment Variables

```bash
# Via CLI
npx convex env set API_KEY "your-key-here"
npx convex env set API_URL "https://api.example.com"

# Via Dashboard
# Go to Settings > Environment Variables in Convex dashboard
```

### Important Limitation

```typescript
// Source: https://docs.convex.dev/production/environment-variables

// ❌ CANNOT conditionally export functions based on env vars
// This will NOT work - exports determined at deployment time
export const myFunc = process.env.DEBUG
  ? mutation({ /* ... */ })
  : internalMutation({ /* ... */ });

// ✅ CAN use env vars inside function logic
export const myFunc = mutation({
  handler: async (ctx) => {
    if (process.env.DEBUG === "true") {
      console.log("Debug mode enabled");
    }
    // ...
  },
});
```

## Common Pitfalls

### Pitfall 1: Using .filter() Instead of .withIndex()

**Problem:** Scans all documents, hits 32,000 document limit, causes timeouts.

**Fix:** Add index to schema and use `.withIndex()` for filtering.

```typescript
// BAD: Full table scan
const tasks = await ctx.db
  .query("tasks")
  .filter((q) => q.eq(q.field("status"), "active"))
  .collect();

// GOOD: Indexed query (add .index("by_status", ["status"]) to schema)
const tasks = await ctx.db
  .query("tasks")
  .withIndex("by_status", (q) => q.eq("status", "active"))
  .collect();
```

### Pitfall 2: Circular Imports Through schema.ts

**Problem:** "validator is undefined" errors at runtime despite TypeScript passing.

**Fix:** Never import from `schema.ts`. Use generated types from `_generated/dataModel`.

```typescript
// BAD: Circular import
import { taskSchema } from "./schema";

// GOOD: Use generated types
import { Doc, Id } from "./_generated/dataModel";
```

### Pitfall 3: Not Awaiting Promises

**Problem:** Mutations fail silently, scheduled functions never execute, errors go unhandled.

**Fix:** Always await promises. Enable ESLint rule `@typescript-eslint/no-floating-promises`.

```typescript
// BAD: Missing await
ctx.db.insert("tasks", { title: "New Task" }); // Fails silently

// GOOD: Properly awaited
await ctx.db.insert("tasks", { title: "New Task" });
```

### Pitfall 4: Spoofable Auth (Trusting Client-Provided IDs)

**Problem:** Security vulnerability — anyone can impersonate any user by passing the right ID.

**Fix:** Always use `ctx.auth.getUserIdentity()` for authorization, never trust `args`.

```typescript
// BAD: Trusting client
export const deletePost = mutation({
  args: { postId: v.id("posts"), userId: v.string() },
  handler: async (ctx, args) => {
    const post = await ctx.db.get(args.postId);
    if (post.author === args.userId) { // ❌ Spoofable
      await ctx.db.delete(args.postId);
    }
  },
});

// GOOD: Verified identity
export const deletePost = mutation({
  args: { postId: v.id("posts") },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");

    const post = await ctx.db.get(args.postId);
    if (post.author !== identity.subject) { // ✅ Verified
      throw new Error("Not authorized");
    }

    await ctx.db.delete(args.postId);
  },
});
```

### Pitfall 5: Sequential ctx.runMutation in Actions

**Problem:** Race conditions, inconsistent state, slow performance.

**Fix:** Move batching logic into a single mutation instead of calling multiple mutations sequentially.

```typescript
// BAD: Sequential mutations
for (const item of items) {
  await ctx.runMutation(internal.items.update, { id: item.id });
}

// GOOD: Single batch mutation
await ctx.runMutation(internal.items.updateBatch, { items });
```

### Pitfall 6: Unbounded .collect()

**Problem:** Queries can return millions of documents, causing memory issues and timeouts.

**Fix:** Use pagination for large result sets.

```typescript
// BAD: Unbounded query
const allMessages = await ctx.db.query("messages").collect();

// GOOD: Paginated query
const result = await ctx.db
  .query("messages")
  .order("desc")
  .paginate(paginationOpts);
```

### Pitfall 7: Date.now() in Queries

**Problem:** Subscriptions won't re-run when time changes, causing stale data.

**Fix:** Use scheduled functions or actions for time-based logic, not queries.

```typescript
// BAD: Query won't re-run when time changes
export const getActiveOffers = query({
  handler: async (ctx) => {
    const now = Date.now();
    return await ctx.db
      .query("offers")
      .filter((q) => q.gt(q.field("expiresAt"), now))
      .collect();
  },
});

// GOOD: Use cron to update offer status
export const updateExpiredOffers = internalMutation({
  handler: async (ctx) => {
    const now = Date.now();
    const expired = await ctx.db
      .query("offers")
      .filter((q) =>
        q.and(
          q.lt(q.field("expiresAt"), now),
          q.eq(q.field("status"), "active")
        )
      )
      .collect();

    for (const offer of expired) {
      await ctx.db.patch(offer._id, { status: "expired" });
    }
  },
});

// Then query by status field instead
export const getActiveOffers = query({
  handler: async (ctx) => {
    return await ctx.db
      .query("offers")
      .withIndex("by_status", (q) => q.eq("status", "active"))
      .collect();
  },
});
```

### Pitfall 8: Manual Refetch After Mutations

**Problem:** Unnecessary code complexity, race conditions with real-time updates.

**Fix:** Trust `useQuery` reactivity — it automatically updates when data changes.

```typescript
// BAD: Manual refetching
const messages = useQuery(api.messages.list, { channelId });
const sendMessage = useMutation(api.messages.send);

const handleSend = async (body) => {
  await sendMessage({ channelId, body });
  refetch(); // ❌ Unnecessary
};

// GOOD: Let useQuery handle updates
const messages = useQuery(api.messages.list, { channelId });
const sendMessage = useMutation(api.messages.send);

const handleSend = async (body) => {
  await sendMessage({ channelId, body });
  // ✅ Messages automatically update via subscription
};
```
