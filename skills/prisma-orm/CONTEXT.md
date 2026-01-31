# Prisma ORM Context

Comprehensive guide to Prisma schema design, queries, and best practices.

## Setup

```bash
npm install prisma @prisma/client
npx prisma init
```

## Schema Patterns

### Basic Schema
```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  avatar    String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  posts     Post[]
  comments  Comment[]
  
  @@index([email])
}

model Post {
  id          String    @id @default(cuid())
  title       String
  content     String?
  published   Boolean   @default(false)
  publishedAt DateTime?
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  
  author      User      @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId    String
  
  categories  Category[]
  comments    Comment[]
  
  @@index([authorId])
  @@index([published, publishedAt])
}

model Category {
  id    String @id @default(cuid())
  name  String @unique
  slug  String @unique
  posts Post[]
}

model Comment {
  id        String   @id @default(cuid())
  content   String
  createdAt DateTime @default(now())
  
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId  String
  
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId    String
  
  parent    Comment?  @relation("CommentReplies", fields: [parentId], references: [id])
  parentId  String?
  replies   Comment[] @relation("CommentReplies")
  
  @@index([postId])
  @@index([authorId])
}
```

### Enums and Types
```prisma
enum Role {
  USER
  ADMIN
  MODERATOR
}

enum PostStatus {
  DRAFT
  PENDING
  PUBLISHED
  ARCHIVED
}

model User {
  id   String @id @default(cuid())
  role Role   @default(USER)
}

model Post {
  id     String     @id @default(cuid())
  status PostStatus @default(DRAFT)
}
```

### JSON Fields
```prisma
model User {
  id       String @id @default(cuid())
  settings Json   @default("{}")
  metadata Json?
}
```

### Multi-field IDs and Uniques
```prisma
model Subscription {
  userId    String
  productId String
  createdAt DateTime @default(now())
  
  user    User    @relation(fields: [userId], references: [id])
  product Product @relation(fields: [productId], references: [id])
  
  @@id([userId, productId])
}

model Post {
  id     String @id @default(cuid())
  slug   String
  orgId  String
  
  @@unique([orgId, slug])
}
```

## Client Singleton

```typescript
// lib/db.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma = globalForPrisma.prisma ?? new PrismaClient({
  log: process.env.NODE_ENV === 'development' 
    ? ['query', 'error', 'warn'] 
    : ['error'],
})

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma
}

export default prisma
```

## Query Patterns

### Basic CRUD
```typescript
// Create
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John Doe',
  },
})

// Read
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
})

const user = await prisma.user.findUniqueOrThrow({
  where: { email: 'user@example.com' },
})

// Update
const user = await prisma.user.update({
  where: { id: 'user-id' },
  data: { name: 'Jane Doe' },
})

// Delete
await prisma.user.delete({
  where: { id: 'user-id' },
})
```

### Relations
```typescript
// Include relations
const user = await prisma.user.findUnique({
  where: { id: 'user-id' },
  include: {
    posts: true,
    comments: {
      include: { post: true },
    },
  },
})

// Nested create
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    posts: {
      create: [
        { title: 'First Post' },
        { title: 'Second Post' },
      ],
    },
  },
  include: { posts: true },
})

// Connect existing
const post = await prisma.post.create({
  data: {
    title: 'New Post',
    author: { connect: { id: 'user-id' } },
    categories: {
      connect: [{ id: 'cat-1' }, { id: 'cat-2' }],
    },
  },
})
```

### Filtering
```typescript
const posts = await prisma.post.findMany({
  where: {
    AND: [
      { published: true },
      { authorId: 'user-id' },
    ],
    OR: [
      { title: { contains: 'prisma' } },
      { content: { contains: 'prisma' } },
    ],
    NOT: { status: 'ARCHIVED' },
    categories: {
      some: { slug: 'tech' },
    },
    createdAt: {
      gte: new Date('2024-01-01'),
      lt: new Date('2025-01-01'),
    },
  },
  orderBy: [
    { publishedAt: 'desc' },
    { title: 'asc' },
  ],
  take: 10,
  skip: 0,
})
```

### Select Fields
```typescript
const users = await prisma.user.findMany({
  select: {
    id: true,
    email: true,
    name: true,
    _count: {
      select: { posts: true },
    },
  },
})
```

### Aggregations
```typescript
const stats = await prisma.post.aggregate({
  _count: { id: true },
  _avg: { viewCount: true },
  _max: { viewCount: true },
  where: { published: true },
})

const grouped = await prisma.post.groupBy({
  by: ['authorId'],
  _count: { id: true },
  _sum: { viewCount: true },
  having: {
    id: { _count: { gt: 5 } },
  },
})
```

### Transactions
```typescript
// Sequential transactions
const [user, post] = await prisma.$transaction([
  prisma.user.create({ data: { email: 'user@example.com' } }),
  prisma.post.create({ data: { title: 'Post', authorId: 'temp' } }),
])

// Interactive transactions
const result = await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({
    data: { email: 'user@example.com' },
  })
  
  const post = await tx.post.create({
    data: { title: 'Post', authorId: user.id },
  })
  
  return { user, post }
}, {
  maxWait: 5000,
  timeout: 10000,
})
```

### Raw Queries
```typescript
const users = await prisma.$queryRaw<User[]>`
  SELECT * FROM "User" WHERE email LIKE ${`%@example.com`}
`

await prisma.$executeRaw`
  UPDATE "User" SET "lastActiveAt" = NOW() WHERE id = ${userId}
`
```

## Migration Commands

```bash
# Create migration
npx prisma migrate dev --name add_user_table

# Apply migrations (production)
npx prisma migrate deploy

# Reset database
npx prisma migrate reset

# Generate client
npx prisma generate

# Push schema (dev only, no migration)
npx prisma db push

# View database
npx prisma studio

# Format schema
npx prisma format
```

## Middleware & Extensions

### Soft Delete Extension
```typescript
const prisma = new PrismaClient().$extends({
  query: {
    user: {
      async delete({ args, query }) {
        return prisma.user.update({
          where: args.where,
          data: { deletedAt: new Date() },
        })
      },
      async findMany({ args, query }) {
        args.where = { ...args.where, deletedAt: null }
        return query(args)
      },
    },
  },
})
```

### Audit Log Extension
```typescript
const prisma = new PrismaClient().$extends({
  query: {
    $allModels: {
      async create({ model, args, query }) {
        const result = await query(args)
        await logAudit(model, 'CREATE', result)
        return result
      },
    },
  },
})
```

## Best Practices

1. **Use cuid() or uuid()** - Not autoincrement for distributed systems
2. **Add indexes** - For frequently queried fields
3. **Use @updatedAt** - Auto-track modifications
4. **Cascade deletes carefully** - Explicit onDelete behavior
5. **Use transactions** - For related operations
6. **Select only needed fields** - Reduce payload size
7. **Pagination** - Always use take/skip for lists
8. **Singleton client** - Prevent connection pool exhaustion

## Common Gotchas

1. **JSON fields** - Need type assertions in TypeScript
2. **Optional relations** - Check for null before accessing
3. **Connection pooling** - Use PgBouncer in serverless
4. **Migration naming** - Use descriptive names
5. **Dev vs Deploy** - Different commands for different environments
