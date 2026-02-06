# Advanced Features

Actions for external APIs, authentication, file storage, scheduling, and real-time subscriptions.

## Action Functions

Actions are for external API calls, side effects, and operations that need non-deterministic behavior.

### Basic Action

```typescript
// Source: https://docs.convex.dev/functions/actions
import { action } from "./_generated/server";
import { v } from "convex/values";

export const sendEmail = action({
  args: {
    to: v.string(),
    subject: v.string(),
    body: v.string(),
  },
  handler: async (ctx, args) => {
    const response = await fetch("https://api.sendgrid.com/v3/mail/send", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${process.env.SENDGRID_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        personalizations: [{ to: [{ email: args.to }] }],
        subject: args.subject,
        content: [{ type: "text/plain", value: args.body }],
      }),
    });

    if (!response.ok) {
      throw new Error(`Failed to send email: ${response.statusText}`);
    }

    return { success: true };
  },
});
```

### Calling Internal Functions

Actions can call mutations and queries using `ctx.runMutation()` and `ctx.runQuery()`:

```typescript
// Source: https://docs.convex.dev/functions/actions
import { action } from "./_generated/server";
import { internal } from "./_generated/api";
import { v } from "convex/values";

export const processPayment = action({
  args: {
    taskId: v.id("tasks"),
    amount: v.number(),
  },
  handler: async (ctx, args) => {
    // Call external payment API
    const response = await fetch("https://api.stripe.com/v1/charges", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${process.env.STRIPE_KEY}`,
      },
      body: JSON.stringify({ amount: args.amount }),
    });

    const payment = await response.json();

    // Update database via internal mutation
    if (response.ok) {
      await ctx.runMutation(internal.tasks.markPaid, {
        taskId: args.taskId,
        paymentId: payment.id,
      });
    }

    return payment;
  },
});
```

### Scheduling from Actions

```typescript
// Source: https://docs.convex.dev/functions/actions
import { action } from "./_generated/server";
import { internal } from "./_generated/api";
import { v } from "convex/values";

export const scheduleReminder = action({
  args: {
    taskId: v.id("tasks"),
    delayMs: v.number(),
  },
  handler: async (ctx, args) => {
    // Schedule future execution (relative delay)
    await ctx.scheduler.runAfter(
      args.delayMs,
      internal.notifications.sendReminder,
      { taskId: args.taskId }
    );

    // Or schedule at specific time (absolute timestamp)
    const reminderTime = Date.now() + args.delayMs;
    await ctx.scheduler.runAt(
      reminderTime,
      internal.notifications.sendReminder,
      { taskId: args.taskId }
    );
  },
});
```

### Anti-Pattern: Sequential ctx.runMutation

```typescript
// BAD: Sequential mutations in actions cause race conditions
for (const item of items) {
  await ctx.runMutation(internal.items.update, { id: item.id });
}

// GOOD: Move batching logic into a single mutation
await ctx.runMutation(internal.items.updateBatch, { items });
```

## Authentication

Verify user identity using `ctx.auth.getUserIdentity()` in any Convex function.

### UserIdentity Type

```typescript
// Source: https://docs.convex.dev/auth
interface UserIdentity {
  subject: string;          // Required: unique user identifier
  issuer?: string;          // Auth provider issuer URL
  tokenIdentifier?: string; // Opaque token identifier
  name?: string;
  email?: string;
  pictureUrl?: string;
}
```

### Authentication Check Pattern

```typescript
// Source: https://docs.convex.dev/auth
import { mutation } from "./_generated/server";
import { v } from "convex/values";

export const createPost = mutation({
  args: {
    title: v.string(),
    content: v.string(),
  },
  handler: async (ctx, args) => {
    // Check authentication at handler start
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) {
      throw new Error("Not authenticated");
    }

    // Use identity.subject for user identification
    const postId = await ctx.db.insert("posts", {
      title: args.title,
      content: args.content,
      author: identity.subject,
      createdAt: Date.now(),
    });

    return postId;
  },
});
```

### Authorization Pattern

Compare authenticated identity against stored user records:

```typescript
// Source: https://docs.convex.dev/auth
export const deletePost = mutation({
  args: { postId: v.id("posts") },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) {
      throw new Error("Not authenticated");
    }

    const post = await ctx.db.get(args.postId);
    if (!post) {
      throw new Error("Post not found");
    }

    // Authorization: check ownership
    if (post.author !== identity.subject) {
      throw new Error("Not authorized");
    }

    await ctx.db.delete(args.postId);
  },
});
```

### Anti-Pattern: Spoofable Auth

```typescript
// BAD: Trusting client-provided user IDs
export const deletePost = mutation({
  args: {
    postId: v.id("posts"),
    userId: v.string(), // ❌ Client can fake this
  },
  handler: async (ctx, args) => {
    const post = await ctx.db.get(args.postId);
    if (post.author === args.userId) { // ❌ Spoofable
      await ctx.db.delete(args.postId);
    }
  },
});

// GOOD: Always use ctx.auth.getUserIdentity()
export const deletePost = mutation({
  args: { postId: v.id("posts") },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");

    const post = await ctx.db.get(args.postId);
    if (post.author !== identity.subject) { // ✅ Verified by Convex
      throw new Error("Not authorized");
    }

    await ctx.db.delete(args.postId);
  },
});
```

## File Storage

Upload and manage files using pre-signed URLs and storageIds.

### Generate Upload URL

```typescript
// Source: https://docs.convex.dev/file-storage/upload-files
import { mutation } from "./_generated/server";

export const generateUploadUrl = mutation({
  args: {},
  handler: async (ctx) => {
    // Returns pre-signed URL valid for 1 hour
    return await ctx.storage.generateUploadUrl();
  },
});
```

### Client Upload Flow

```typescript
// Source: https://docs.convex.dev/file-storage/upload-files
// Step 1: Get upload URL from mutation
const uploadUrl = await generateUploadUrl();

// Step 2: POST file to upload URL
const response = await fetch(uploadUrl, {
  method: "POST",
  headers: { "Content-Type": file.type },
  body: file,
});

const { storageId } = await response.json();

// Step 3: Save storageId to database
await saveFile({ storageId, filename: file.name });
```

### Save StorageId to Database

```typescript
// Source: https://docs.convex.dev/file-storage
import { mutation } from "./_generated/server";
import { v } from "convex/values";

export const saveFile = mutation({
  args: {
    storageId: v.id("_storage"),
    filename: v.string(),
  },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) {
      throw new Error("Not authenticated");
    }

    const fileId = await ctx.db.insert("files", {
      storageId: args.storageId,
      filename: args.filename,
      uploadedBy: identity.subject,
      uploadedAt: Date.now(),
    });

    return fileId;
  },
});
```

### Get File URL

```typescript
// Source: https://docs.convex.dev/file-storage
import { query } from "./_generated/server";
import { v } from "convex/values";

export const getFileUrl = query({
  args: { fileId: v.id("files") },
  handler: async (ctx, args) => {
    const file = await ctx.db.get(args.fileId);
    if (!file) {
      return null;
    }

    // Returns download URL (or null if file deleted)
    return await ctx.storage.getUrl(file.storageId);
  },
});
```

### File Metadata and Deletion

```typescript
// Source: https://docs.convex.dev/file-storage
export const deleteFile = mutation({
  args: { fileId: v.id("files") },
  handler: async (ctx, args) => {
    const file = await ctx.db.get(args.fileId);
    if (!file) {
      throw new Error("File not found");
    }

    // Get metadata (size, contentType)
    const metadata = await ctx.storage.getMetadata(file.storageId);
    console.log(`Deleting file: ${metadata?.size} bytes`);

    // Delete from storage
    await ctx.storage.delete(file.storageId);

    // Delete database record
    await ctx.db.delete(args.fileId);
  },
});
```

## Scheduling

Set up recurring tasks with cron jobs and scheduled functions.

### Cron Job Configuration

```typescript
// Source: https://docs.convex.dev/scheduling/cron-jobs
// convex/crons.ts
import { cronJobs } from "convex/server";
import { internal } from "./_generated/api";

const crons = cronJobs();

// Interval-based scheduling
crons.interval(
  "clear old messages",
  { hours: 24 }, // Run every 24 hours
  internal.messages.clearOld
);

crons.interval(
  "send notifications",
  { minutes: 5 }, // Run every 5 minutes
  internal.notifications.process
);

// Named schedule patterns
crons.hourly(
  "hourly sync",
  { minuteUTC: 30 }, // 30 minutes past each hour
  internal.sync.run
);

crons.daily(
  "daily report",
  { hourUTC: 9, minuteUTC: 0 }, // 9:00 AM UTC
  internal.reports.generate
);

crons.weekly(
  "weekly summary",
  { dayOfWeek: "monday", hourUTC: 8, minuteUTC: 0 },
  internal.reports.weeklySummary
);

crons.monthly(
  "monthly billing",
  { day: 1, hourUTC: 0, minuteUTC: 0 }, // First day of month at midnight UTC
  internal.billing.process
);

// Traditional cron syntax (UTC timezone)
crons.cron(
  "complex schedule",
  "0 2,14 * * *", // 2:00 AM and 2:00 PM UTC daily
  internal.maintenance.run
);

// With arguments
crons.monthly(
  "payment reminder",
  { day: 1, hourUTC: 16, minuteUTC: 0 },
  internal.payments.sendReminder,
  { channel: "email" }
);

export default crons;
```

### Scheduled Function Example

```typescript
// Source: https://docs.convex.dev/scheduling/cron-jobs
import { internalMutation } from "./_generated/server";

export const clearOld = internalMutation({
  args: {},
  handler: async (ctx) => {
    const cutoff = Date.now() - 30 * 24 * 60 * 60 * 1000; // 30 days ago

    const oldMessages = await ctx.db
      .query("messages")
      .filter((q) => q.lt(q.field("_creationTime"), cutoff))
      .collect();

    for (const message of oldMessages) {
      await ctx.db.delete(message._id);
    }

    console.log(`Deleted ${oldMessages.length} old messages`);
  },
});
```

## Real-Time Subscriptions

Client hooks for reactive data synchronization.

### useQuery Hook

```typescript
// Source: https://docs.convex.dev/functions
import { useQuery } from "convex/react";
import { api } from "../convex/_generated/api";

function MessageList({ channelId }) {
  // Auto-subscribes to query, returns undefined while loading
  const messages = useQuery(api.messages.list, { channelId });

  if (messages === undefined) {
    return <div>Loading...</div>;
  }

  return (
    <ul>
      {messages.map((msg) => (
        <li key={msg._id}>{msg.body}</li>
      ))}
    </ul>
  );
}
```

### useMutation Hook

```typescript
// Source: https://docs.convex.dev/functions
import { useMutation } from "convex/react";
import { api } from "../convex/_generated/api";

function MessageForm({ channelId }) {
  const sendMessage = useMutation(api.messages.send);

  const handleSubmit = async (e) => {
    e.preventDefault();
    const form = e.target;
    const body = form.body.value;

    try {
      await sendMessage({ channelId, body });
      form.reset();
    } catch (error) {
      console.error("Failed to send:", error);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="body" placeholder="Type a message..." />
      <button type="submit">Send</button>
    </form>
  );
}
```

### useAction Hook

```typescript
// Source: https://docs.convex.dev/functions
import { useAction } from "convex/react";
import { api } from "../convex/_generated/api";

function PaymentButton({ amount }) {
  const processPayment = useAction(api.payments.process);

  const handleClick = async () => {
    try {
      const result = await processPayment({ amount });
      console.log("Payment successful:", result);
    } catch (error) {
      console.error("Payment failed:", error);
    }
  };

  return <button onClick={handleClick}>Pay ${amount}</button>;
}
```

### usePaginatedQuery Hook

```typescript
// Source: https://docs.convex.dev/functions
import { usePaginatedQuery } from "convex/react";
import { api } from "../convex/_generated/api";

function InfiniteList({ channelId }) {
  const { results, status, loadMore } = usePaginatedQuery(
    api.messages.listPaginated,
    { channelId },
    { initialNumItems: 20 }
  );

  return (
    <div>
      {results.map((msg) => (
        <div key={msg._id}>{msg.body}</div>
      ))}
      {status === "CanLoadMore" && (
        <button onClick={() => loadMore(10)}>Load More</button>
      )}
    </div>
  );
}
```

### Optimistic Updates

```typescript
// Source: https://docs.convex.dev/functions
import { useMutation } from "convex/react";
import { api } from "../convex/_generated/api";

function MessageForm({ channelId }) {
  const sendMessage = useMutation(api.messages.send).withOptimisticUpdate(
    (localStore, args) => {
      // Immediately add optimistic message to local state
      const currentMessages = localStore.getQuery(api.messages.list, {
        channelId: args.channelId,
      });

      if (currentMessages !== undefined) {
        localStore.setQuery(
          api.messages.list,
          { channelId: args.channelId },
          [
            ...currentMessages,
            {
              _id: "optimistic-id",
              _creationTime: Date.now(),
              body: args.body,
              channel: args.channelId,
            },
          ]
        );
      }
    }
  );

  // Rest of component...
}
```

### Anti-Pattern: Manual Refetch

```typescript
// BAD: Manually refetching after mutations
const messages = useQuery(api.messages.list, { channelId });
const sendMessage = useMutation(api.messages.send);

const handleSend = async (body) => {
  await sendMessage({ channelId, body });
  // ❌ DON'T refetch manually - useQuery handles this automatically
  refetch();
};

// GOOD: Trust useQuery reactivity
const messages = useQuery(api.messages.list, { channelId });
const sendMessage = useMutation(api.messages.send);

const handleSend = async (body) => {
  await sendMessage({ channelId, body });
  // ✅ Messages automatically update via subscription
};
```
