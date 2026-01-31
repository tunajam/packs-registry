# Skill: Convex Patterns

Best practices for building with Convex—the real-time backend.

## When to Use
- Starting a new Convex project
- Designing schemas and queries
- Handling auth and permissions
- Optimizing performance

## Core Concepts

### Convex is Reactive
Queries automatically re-run when data changes. Embrace this.
```typescript
// Frontend
const messages = useQuery(api.messages.list, { channelId });
// Automatically updates when any message changes!
```

### Functions are the API
- **Queries** — Read data (automatically cached, reactive)
- **Mutations** — Write data (transactional, consistent)
- **Actions** — Side effects (call external APIs, non-deterministic)

## Schema Design

### Define Types First
```typescript
// convex/schema.ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  users: defineTable({
    name: v.string(),
    email: v.string(),
    avatarUrl: v.optional(v.string()),
    createdAt: v.number(),
  })
    .index("by_email", ["email"]),
  
  messages: defineTable({
    authorId: v.id("users"),
    channelId: v.id("channels"),
    content: v.string(),
    createdAt: v.number(),
  })
    .index("by_channel", ["channelId", "createdAt"]),
});
```

### Index Strategy
- Index fields you filter/sort by
- Compound indexes for multi-field queries
- Order matters: `["channelId", "createdAt"]` ≠ `["createdAt", "channelId"]`

```typescript
// ✅ Uses index efficiently
const messages = await ctx.db
  .query("messages")
  .withIndex("by_channel", (q) => q.eq("channelId", channelId))
  .order("desc")
  .take(50);
```

## Query Patterns

### Keep Queries Fast
Queries should be fast—they're reactive and re-run often.

```typescript
// ❌ Slow: Fetching everything
export const list = query({
  handler: async (ctx) => {
    const all = await ctx.db.query("messages").collect();
    return all.slice(0, 50);  // Fetched all, returned 50
  },
});

// ✅ Fast: Fetch only what you need
export const list = query({
  args: { channelId: v.id("channels") },
  handler: async (ctx, { channelId }) => {
    return ctx.db
      .query("messages")
      .withIndex("by_channel", (q) => q.eq("channelId", channelId))
      .order("desc")
      .take(50);
  },
});
```

### Join Pattern (Fetching Related Data)
```typescript
export const listWithAuthors = query({
  args: { channelId: v.id("channels") },
  handler: async (ctx, { channelId }) => {
    const messages = await ctx.db
      .query("messages")
      .withIndex("by_channel", (q) => q.eq("channelId", channelId))
      .take(50);
    
    // Fetch authors in parallel
    return Promise.all(
      messages.map(async (message) => ({
        ...message,
        author: await ctx.db.get(message.authorId),
      }))
    );
  },
});
```

## Mutation Patterns

### Validate in Mutations
```typescript
export const send = mutation({
  args: {
    channelId: v.id("channels"),
    content: v.string(),
  },
  handler: async (ctx, { channelId, content }) => {
    // Validate
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");
    
    const channel = await ctx.db.get(channelId);
    if (!channel) throw new Error("Channel not found");
    
    const trimmed = content.trim();
    if (trimmed.length === 0) throw new Error("Message cannot be empty");
    if (trimmed.length > 5000) throw new Error("Message too long");
    
    // Get or create user
    const user = await getOrCreateUser(ctx, identity);
    
    // Write
    return ctx.db.insert("messages", {
      authorId: user._id,
      channelId,
      content: trimmed,
      createdAt: Date.now(),
    });
  },
});
```

### Mutations Are Transactional
Everything in a mutation either succeeds or fails together.

```typescript
export const transfer = mutation({
  args: {
    fromId: v.id("accounts"),
    toId: v.id("accounts"),
    amount: v.number(),
  },
  handler: async (ctx, { fromId, toId, amount }) => {
    const from = await ctx.db.get(fromId);
    const to = await ctx.db.get(toId);
    
    if (from.balance < amount) throw new Error("Insufficient funds");
    
    // Both updates happen atomically
    await ctx.db.patch(fromId, { balance: from.balance - amount });
    await ctx.db.patch(toId, { balance: to.balance + amount });
  },
});
```

## Action Patterns

### When to Use Actions
- Calling external APIs
- Sending emails
- Generating random values
- Anything non-deterministic

```typescript
export const sendWelcomeEmail = action({
  args: { userId: v.id("users") },
  handler: async (ctx, { userId }) => {
    // Read from DB via runQuery
    const user = await ctx.runQuery(api.users.get, { userId });
    
    // External API call
    await resend.emails.send({
      to: user.email,
      subject: "Welcome!",
      html: renderWelcomeEmail(user),
    });
    
    // Write to DB via runMutation
    await ctx.runMutation(api.users.markWelcomeEmailSent, { userId });
  },
});
```

## Auth Patterns

### With Clerk
```typescript
// convex/users.ts
export const current = query({
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) return null;
    
    return ctx.db
      .query("users")
      .withIndex("by_clerk_id", (q) => q.eq("clerkId", identity.subject))
      .unique();
  },
});

// Helper for mutations
async function requireAuth(ctx: MutationCtx) {
  const identity = await ctx.auth.getUserIdentity();
  if (!identity) throw new Error("Not authenticated");
  
  const user = await ctx.db
    .query("users")
    .withIndex("by_clerk_id", (q) => q.eq("clerkId", identity.subject))
    .unique();
  
  if (!user) throw new Error("User not found");
  return user;
}
```

## File Storage

```typescript
// Generate upload URL
export const generateUploadUrl = mutation({
  handler: async (ctx) => {
    await requireAuth(ctx);
    return ctx.storage.generateUploadUrl();
  },
});

// Store reference after upload
export const saveAvatar = mutation({
  args: { storageId: v.id("_storage") },
  handler: async (ctx, { storageId }) => {
    const user = await requireAuth(ctx);
    const url = await ctx.storage.getUrl(storageId);
    await ctx.db.patch(user._id, { avatarUrl: url });
  },
});
```

## Common Gotchas

### Don't Use Date Objects
```typescript
// ❌ Date objects don't serialize
createdAt: new Date()

// ✅ Use timestamps
createdAt: Date.now()
```

### Don't Mutate in Queries
```typescript
// ❌ Never write in a query
export const list = query({
  handler: async (ctx) => {
    await ctx.db.insert("logs", { ... });  // NO!
  },
});
```

### Index Ordering Matters
```typescript
// Index: ["channelId", "createdAt"]

// ✅ Works - using prefix
.withIndex("by_channel", (q) => q.eq("channelId", id))

// ❌ Doesn't work - skipping channelId
.withIndex("by_channel", (q) => q.gt("createdAt", timestamp))
```

## Output

Use these patterns as defaults. Adapt when you have specific needs.
