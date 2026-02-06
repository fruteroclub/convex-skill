---
name: convex
description: Build Convex backends with queries, mutations, actions, and schemas. Use when writing Convex code, defining database schemas, implementing real-time sync, or needing Convex patterns.
homepage: https://docs.convex.dev
metadata: {"openclaw":{"emoji":"⚡"}}
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

Detailed guides in `{baseDir}/references/`:

- `core-database.md` — Schemas, validators, queries, mutations, indexes
- `advanced-features.md` — Actions, file storage, scheduling, authentication
- `patterns-best-practices.md` — Common patterns, error handling, optimization

## Resources

- [Convex Docs](https://docs.convex.dev)
- [Convex Dashboard](https://dashboard.convex.dev)
- [Convex Discord](https://convex.dev/community)

## Convex Rules (LLM Reference)

The `{baseDir}/references/convex-rules.txt` file contains comprehensive coding guidelines for Convex:

- Function syntax (queries, mutations, actions, internal functions)
- HTTP endpoint syntax
- Validators and types
- Schema guidelines
- Pagination patterns
- Full-text search
- File storage
- Scheduling and crons
- Complete chat-app example with AI integration

Load this file when writing Convex code for syntax and patterns.
