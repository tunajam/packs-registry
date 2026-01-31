# Drizzle ORM Context

Comprehensive guide to Drizzle ORM - SQL-like TypeScript ORM with zero dependencies.

## Setup

```bash
# PostgreSQL
npm install drizzle-orm postgres
npm install -D drizzle-kit

# MySQL
npm install drizzle-orm mysql2
npm install -D drizzle-kit

# SQLite
npm install drizzle-orm better-sqlite3
npm install -D drizzle-kit @types/better-sqlite3
```

## Schema Definition

### PostgreSQL Schema
```typescript
// db/schema.ts
import {
  pgTable,
  serial,
  text,
  varchar,
  timestamp,
  boolean,
  integer,
  uuid,
  json,
  pgEnum,
  index,
  uniqueIndex,
} from 'drizzle-orm/pg-core'
import { relations } from 'drizzle-orm'

// Enums
export const roleEnum = pgEnum('role', ['user', 'admin', 'moderator'])
export const postStatusEnum = pgEnum('post_status', ['draft', 'published', 'archived'])

// Users table
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  name: varchar('name', { length: 255 }),
  role: roleEnum('role').default('user').notNull(),
  metadata: json('metadata').$type<{ preferences: Record<string, unknown> }>(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
}, (table) => ({
  emailIdx: uniqueIndex('email_idx').on(table.email),
}))

// Posts table
export const posts = pgTable('posts', {
  id: uuid('id').primaryKey().defaultRandom(),
  title: varchar('title', { length: 255 }).notNull(),
  content: text('content'),
  status: postStatusEnum('status').default('draft').notNull(),
  published: boolean('published').default(false).notNull(),
  publishedAt: timestamp('published_at'),
  authorId: uuid('author_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
}, (table) => ({
  authorIdx: index('author_idx').on(table.authorId),
  publishedIdx: index('published_idx').on(table.published, table.publishedAt),
}))

// Categories table
export const categories = pgTable('categories', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 100 }).notNull().unique(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
})

// Many-to-many junction table
export const postsToCategories = pgTable('posts_to_categories', {
  postId: uuid('post_id').notNull().references(() => posts.id, { onDelete: 'cascade' }),
  categoryId: uuid('category_id').notNull().references(() => categories.id, { onDelete: 'cascade' }),
}, (table) => ({
  pk: { columns: [table.postId, table.categoryId] },
}))

// Comments with self-referential relation
export const comments = pgTable('comments', {
  id: uuid('id').primaryKey().defaultRandom(),
  content: text('content').notNull(),
  authorId: uuid('author_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  postId: uuid('post_id').notNull().references(() => posts.id, { onDelete: 'cascade' }),
  parentId: uuid('parent_id').references((): any => comments.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at').defaultNow().notNull(),
})
```

### Relations
```typescript
// db/relations.ts
import { relations } from 'drizzle-orm'
import { users, posts, categories, postsToCategories, comments } from './schema'

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
  comments: many(comments),
}))

export const postsRelations = relations(posts, ({ one, many }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
  comments: many(comments),
  postsToCategories: many(postsToCategories),
}))

export const categoriesRelations = relations(categories, ({ many }) => ({
  postsToCategories: many(postsToCategories),
}))

export const postsToCategoriesRelations = relations(postsToCategories, ({ one }) => ({
  post: one(posts, {
    fields: [postsToCategories.postId],
    references: [posts.id],
  }),
  category: one(categories, {
    fields: [postsToCategories.categoryId],
    references: [categories.id],
  }),
}))

export const commentsRelations = relations(comments, ({ one, many }) => ({
  author: one(users, {
    fields: [comments.authorId],
    references: [users.id],
  }),
  post: one(posts, {
    fields: [comments.postId],
    references: [posts.id],
  }),
  parent: one(comments, {
    fields: [comments.parentId],
    references: [comments.id],
    relationName: 'replies',
  }),
  replies: many(comments, { relationName: 'replies' }),
}))
```

## Database Client

```typescript
// db/index.ts
import { drizzle } from 'drizzle-orm/postgres-js'
import postgres from 'postgres'
import * as schema from './schema'

const connectionString = process.env.DATABASE_URL!

// For queries
const queryClient = postgres(connectionString)
export const db = drizzle(queryClient, { schema })

// For migrations (separate client)
export const migrationClient = postgres(connectionString, { max: 1 })
```

## Query Patterns

### Basic CRUD
```typescript
import { db } from '@/db'
import { users, posts } from '@/db/schema'
import { eq, and, or, like, desc, asc, isNull } from 'drizzle-orm'

// Insert
const [newUser] = await db.insert(users).values({
  email: 'user@example.com',
  name: 'John Doe',
}).returning()

// Insert many
await db.insert(posts).values([
  { title: 'Post 1', authorId: newUser.id },
  { title: 'Post 2', authorId: newUser.id },
])

// Select
const allUsers = await db.select().from(users)

const user = await db.select().from(users).where(eq(users.id, 'user-id')).limit(1)

// Update
const [updatedUser] = await db.update(users)
  .set({ name: 'Jane Doe', updatedAt: new Date() })
  .where(eq(users.id, 'user-id'))
  .returning()

// Delete
await db.delete(users).where(eq(users.id, 'user-id'))
```

### Filtering & Operators
```typescript
import { eq, ne, gt, gte, lt, lte, like, ilike, between, inArray, isNull, isNotNull, and, or, not, sql } from 'drizzle-orm'

// Complex where clause
const filteredPosts = await db.select().from(posts).where(
  and(
    eq(posts.published, true),
    or(
      like(posts.title, '%drizzle%'),
      like(posts.content, '%drizzle%')
    ),
    isNotNull(posts.publishedAt),
    inArray(posts.status, ['published', 'archived']),
    gte(posts.createdAt, new Date('2024-01-01'))
  )
)

// Order and pagination
const paginatedPosts = await db.select()
  .from(posts)
  .where(eq(posts.published, true))
  .orderBy(desc(posts.publishedAt), asc(posts.title))
  .limit(10)
  .offset(20)
```

### Joins
```typescript
// Inner join
const postsWithAuthors = await db
  .select({
    post: posts,
    author: users,
  })
  .from(posts)
  .innerJoin(users, eq(posts.authorId, users.id))

// Left join
const usersWithPosts = await db
  .select({
    user: users,
    post: posts,
  })
  .from(users)
  .leftJoin(posts, eq(users.id, posts.authorId))

// Partial select
const postTitles = await db
  .select({
    title: posts.title,
    authorName: users.name,
    authorEmail: users.email,
  })
  .from(posts)
  .innerJoin(users, eq(posts.authorId, users.id))
```

### Query API (with Relations)
```typescript
// Relational queries (needs relations defined)
const usersWithPosts = await db.query.users.findMany({
  with: {
    posts: {
      where: eq(posts.published, true),
      orderBy: [desc(posts.publishedAt)],
      limit: 5,
      with: {
        comments: true,
      },
    },
  },
})

const user = await db.query.users.findFirst({
  where: eq(users.email, 'user@example.com'),
  with: { posts: true },
})
```

### Aggregations
```typescript
import { count, sum, avg, min, max, sql } from 'drizzle-orm'

const stats = await db
  .select({
    totalPosts: count(posts.id),
    publishedPosts: count(sql`CASE WHEN ${posts.published} THEN 1 END`),
  })
  .from(posts)

// Group by
const postsByAuthor = await db
  .select({
    authorId: posts.authorId,
    authorName: users.name,
    postCount: count(posts.id),
  })
  .from(posts)
  .innerJoin(users, eq(posts.authorId, users.id))
  .groupBy(posts.authorId, users.name)
  .having(gt(count(posts.id), 5))
```

### Transactions
```typescript
const result = await db.transaction(async (tx) => {
  const [user] = await tx.insert(users).values({
    email: 'user@example.com',
  }).returning()

  const [post] = await tx.insert(posts).values({
    title: 'First Post',
    authorId: user.id,
  }).returning()

  return { user, post }
})

// Nested transactions (savepoints)
await db.transaction(async (tx) => {
  await tx.insert(users).values({ email: 'user1@example.com' })
  
  await tx.transaction(async (tx2) => {
    await tx2.insert(users).values({ email: 'user2@example.com' })
    // If this fails, only this savepoint rolls back
  })
})
```

### Raw SQL
```typescript
import { sql } from 'drizzle-orm'

// Raw query
const result = await db.execute(sql`
  SELECT * FROM users WHERE email LIKE ${`%@example.com`}
`)

// SQL in select
const usersWithPostCount = await db
  .select({
    id: users.id,
    email: users.email,
    postCount: sql<number>`(SELECT COUNT(*) FROM posts WHERE posts.author_id = users.id)`,
  })
  .from(users)
```

## Drizzle Kit Configuration

```typescript
// drizzle.config.ts
import type { Config } from 'drizzle-kit'

export default {
  schema: './db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
  verbose: true,
  strict: true,
} satisfies Config
```

## Migration Commands

```bash
# Generate migration
npx drizzle-kit generate

# Apply migrations
npx drizzle-kit migrate

# Push schema (dev, no migration files)
npx drizzle-kit push

# Open Drizzle Studio
npx drizzle-kit studio

# Drop everything (danger!)
npx drizzle-kit drop
```

## Type Inference

```typescript
import { InferSelectModel, InferInsertModel } from 'drizzle-orm'
import { users, posts } from './schema'

// Infer types from schema
type User = InferSelectModel<typeof users>
type NewUser = InferInsertModel<typeof users>
type Post = InferSelectModel<typeof posts>
type NewPost = InferInsertModel<typeof posts>

// Usage
function createUser(data: NewUser): Promise<User> {
  return db.insert(users).values(data).returning().then(([user]) => user)
}
```

## Best Practices

1. **Define relations separately** - Keep schema clean, relations in dedicated file
2. **Use .returning()** - Get inserted/updated rows
3. **Type your JSON columns** - Use `.$type<T>()`
4. **Create indexes** - Use the table callback for indexes
5. **Use transactions** - For related operations
6. **Infer types** - Use `InferSelectModel`/`InferInsertModel`
7. **Prefer query API** - For nested data fetching
8. **Use sql template** - For raw SQL with proper escaping
