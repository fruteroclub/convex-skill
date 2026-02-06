# Convex Skill

A knowledge skill for building [Convex](https://convex.dev) backends. Provides patterns, schemas, and best practices for writing Convex code — queries, mutations, actions, auth, file storage, and real-time sync.

## What's Included

| File | Description |
|------|-------------|
| `SKILL.md` | Main skill file with quick start and scope overview |
| `references/core-database.md` | Schemas, validators, queries, mutations, indexes |
| `references/advanced-features.md` | Actions, file storage, scheduling, authentication |
| `references/patterns-best-practices.md` | Common patterns, error handling, optimization |

## Quick Start

**Install Convex:**
```bash
npm install convex
npx convex dev
```

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
  }).index("by_timestamp", ["timestamp"]),
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

## For OpenClaw Agents

Clone to your skills directory:
```bash
cd ~/your-workspace/skills
git clone https://github.com/fruteroclub/convex-skill.git convex
```

Or reference when needed — the skill is knowledge-only, no API calls required.

## Scope

This skill covers:

- **Schemas & Validators** — Define tables, field types, indexes
- **Queries** — Reactive functions that auto-update clients
- **Mutations** — Transactional data modifications
- **Actions** — External API calls, file uploads, non-deterministic ops
- **Authentication** — Clerk, Auth0, custom auth integration
- **File Storage** — Uploads, downloads, metadata
- **Scheduling** — Cron jobs and scheduled functions
- **Real-time** — Automatic data synchronization

## Resources

- [Convex Docs](https://docs.convex.dev)
- [Convex Dashboard](https://dashboard.convex.dev)
- [Convex Discord](https://convex.dev/community)

---

*Built for [Frutero](https://frutero.club) agents. MIT License.*
