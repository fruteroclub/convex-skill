---
name: convex
description: Build Convex backends with queries, mutations, actions, and schemas. Use when writing Convex code, defining database schemas, implementing real-time sync, or needing Convex patterns.
license: MIT
metadata:
  clawdbot:
    emoji: "⚡"
  moltbot:
    emoji: "⚡"
---

# convex

A knowledge-only skill for building Convex backends. Agents use this skill to learn patterns, schemas, and best practices for writing Convex code — not to call APIs. This skill provides the knowledge needed to implement database schemas, queries, mutations, actions, authentication, file storage, and real-time subscriptions.

## Quick Start

**Install Convex:**
```bash
npm install convex
```

**Initialize your project:**
```bash
npx convex dev
```

This creates a `convex/` directory with:
- `schema.ts` — define your database tables and validators
- `_generated/` — TypeScript types auto-generated from your schema

**Define a schema:**
```typescript
// convex/schema.ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  messages: defineTable({
    text: v.string(),
    author: v.string(),
    timestamp: v.number(),
  })
    .index("by_timestamp", ["timestamp"]),
});
```

**Write a query:**
```typescript
// convex/messages.ts
import { query } from "./_generated/server";

export const list = query({
  args: {},
  handler: async (ctx) => {
    return await ctx.db.query("messages")
      .withIndex("by_timestamp")
      .order("desc")
      .collect();
  },
});
```

Your Convex backend is now running locally and ready to receive queries.

## Scope

This skill covers:

- **Database schemas & validators**: Define tables, field types, indexes, and validation rules using Convex schema builder
- **Query functions**: Write reactive query functions that automatically update clients when data changes
- **Mutation functions**: Implement mutations to modify data with transactional guarantees
- **Action functions**: Create actions for external API calls, file uploads, and non-deterministic operations
- **Authentication patterns**: Integrate Clerk, Auth0, or custom auth providers with Convex
- **File storage**: Handle file uploads, downloads, and metadata using Convex file storage
- **Scheduling**: Set up cron jobs and scheduled functions for background tasks
- **Real-time subscriptions**: Build reactive UIs with automatic data synchronization

## References

This skill provides three detailed reference documents. Load the appropriate one based on your task:

**Core Database Operations** — `references/core-database.md`
Schema definitions, query functions, mutation functions. Use this for:
- Defining tables and field validators (v.string, v.number, v.id, etc.)
- Writing query functions with indexing and pagination
- Implementing mutation operations (insert, patch, replace, delete)

**Advanced Features** — `references/advanced-features.md`
Actions, authentication, file storage, scheduling, subscriptions. Use this for:
- Action functions for external API calls (ctx.runMutation, ctx.runQuery)
- Authentication with ctx.auth.getUserIdentity() and authorization patterns
- File storage with generateUploadUrl() and storageId workflow
- Scheduling with cronJobs() and interval/cron/named schedules
- Real-time subscriptions with useQuery/useMutation hooks

**Patterns & Best Practices** — `references/patterns-best-practices.md`
Error handling, TypeScript types, code organization, pitfalls. Use this for:
- Structured error handling with ConvexError
- TypeScript types from _generated/dataModel (Doc<>, Id<>)
- Code organization patterns with model/ layer and internal functions
- Environment variables via process.env in Convex functions
- Common pitfalls and how to avoid them
