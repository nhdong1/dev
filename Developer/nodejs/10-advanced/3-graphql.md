# GraphQL — Apollo Server, Resolvers và DataLoader

> GraphQL là query language (ngôn ngữ truy vấn) và runtime cho APIs — cho phép client request chính xác data cần thiết, giảm over-fetching (lấy thừa dữ liệu) và under-fetching (thiếu dữ liệu) so với REST.

## Mục Lục

1. [GraphQL vs REST](#graphql-vs-rest)
2. [Core Concepts](#core-concepts)
3. [Schema Definition](#schema-definition)
4. [Resolvers](#resolvers)
5. [Apollo Server Setup](#apollo-server-setup)
6. [N+1 Problem và DataLoader](#n1-problem-và-dataloader)
7. [Mutations và Subscriptions](#mutations-và-subscriptions)
8. [Authentication và Authorization](#authentication-và-authorization)
9. [Error Handling](#error-handling)
10. [Best Practices](#best-practices)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## GraphQL vs REST

| Aspect | REST | GraphQL |
| ------ | ---- | ------- |
| Endpoints | Nhiều URLs (`/users`, `/posts`) | Single endpoint (`/graphql`) |
| Data fetching | Fixed response shape | Client chọn fields cần |
| Over-fetching | Thường xảy ra | Client control fields |
| Versioning | URL/header versioning | Schema evolution |
| Caching | HTTP caching mature | Phức tạp hơn (cần tools) |
| Learning curve | Thấp | Cao hơn |
| Tooling | Universal | GraphQL-specific (Playground, Apollo Studio) |

```
REST — 3 requests cho dashboard:
GET /users/1          → { id, name, email, ... }  (over-fetch)
GET /users/1/posts    → [{ id, title, ... }]
GET /posts/1/comments → [{ id, text, ... }]

GraphQL — 1 request:
query {
  user(id: 1) {
    name
    posts { title comments { text } }
  }
}
```

**Khi nào chọn GraphQL:**
- Nhiều clients (mobile, web, IoT) với data needs khác nhau
- Complex nested data relationships
- Rapid frontend iteration (không cần backend thay đổi endpoint)

**Khi nào giữ REST:**
- Simple CRUD APIs
- Public APIs cần HTTP caching
- Team chưa có GraphQL experience

---

## Core Concepts

### Schema — Contract Giữa Client và Server

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
  createdAt: DateTime!
}

type Post {
  id: ID!
  title: String!
  content: String
  author: User!
  comments: [Comment!]!
}

type Comment {
  id: ID!
  text: String!
  author: User!
}

type Query {
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
  post(id: ID!): Post
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  createPost(input: CreatePostInput!): Post!
}

input CreateUserInput {
  name: String!
  email: String!
}
```

### Type System

| Syntax | Ý Nghĩa |
| ------ | ------- |
| `String!` | Non-null String |
| `[Post!]!` | Non-null array of non-null Posts |
| `ID` | Unique identifier (serialized as String) |
| `Int`, `Float`, `Boolean` | Scalar types |
| `input` | Input object cho mutations |

---

## Schema Definition

### Code-First với TypeScript (TypeGraphQL / Pothos)

```typescript
// schema.ts — SDL (Schema Definition Language) approach
import { gql } from 'graphql-tag';

export const typeDefs = gql`
  type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
  }

  type Post {
    id: ID!
    title: String!
    author: User!
  }

  type Query {
    user(id: ID!): User
    users: [User!]!
  }

  type Mutation {
    createUser(name: String!, email: String!): User!
  }
`;
```

### Prisma + GraphQL Schema Generation

```bash
npm install @prisma/client graphql-yoga @graphql-yoga/plugin-pothos
# Hoặc dùng nexus, typegraphql để generate schema từ Prisma models
```

---

## Resolvers

Resolver là function resolve từng field trong schema — map GraphQL fields tới data sources.

```typescript
// resolvers.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export const resolvers = {
  Query: {
    user: async (_parent, { id }, context) => {
      return context.prisma.user.findUnique({ where: { id } });
    },
    users: async (_parent, { limit = 10, offset = 0 }, context) => {
      return context.prisma.user.findMany({ take: limit, skip: offset });
    },
  },

  Mutation: {
    createUser: async (_parent, { name, email }, context) => {
      return context.prisma.user.create({
        data: { name, email },
      });
    },
  },

  // Field resolvers — resolve nested relationships
  User: {
    posts: async (parent, _args, context) => {
      return context.prisma.post.findMany({
        where: { authorId: parent.id },
      });
    },
  },

  Post: {
    author: async (parent, _args, context) => {
      return context.prisma.user.findUnique({
        where: { id: parent.authorId },
      });
    },
  },
};
```

### Resolver Function Signature

```typescript
type Resolver = (
  parent: ParentType,    // Result từ parent resolver
  args: ArgsType,        // GraphQL arguments
  context: ContextType,  // Shared per-request (db, user, loaders)
  info: GraphQLResolveInfo // Query metadata
) => ResultType | Promise<ResultType>;
```

---

## Apollo Server Setup

### Với Express

```typescript
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import express from 'express';
import cors from 'cors';
import { PrismaClient } from '@prisma/client';
import { typeDefs } from './schema';
import { resolvers } from './resolvers';
import { createLoaders } from './loaders';

const prisma = new PrismaClient();

async function startServer() {
  const app = express();

  const server = new ApolloServer({
    typeDefs,
    resolvers,
    formatError: (error) => {
      console.error(error);
      return error;
    },
  });

  await server.start();

  app.use(
    '/graphql',
    cors(),
    express.json(),
    expressMiddleware(server, {
      context: async ({ req }) => ({
        prisma,
        user: req.user, // Từ auth middleware
        loaders: createLoaders(prisma),
      }),
    })
  );

  app.listen(4000, () => {
    console.log('GraphQL server at http://localhost:4000/graphql');
  });
}

startServer();
```

### GraphQL Playground / Apollo Sandbox

Truy cập `http://localhost:4000/graphql` để test queries interactively.

```graphql
# Example query
query GetUserWithPosts {
  user(id: "1") {
    name
    email
    posts {
      title
      comments {
        text
        author { name }
      }
    }
  }
}
```

---

## N+1 Problem và DataLoader

### Vấn Đề N+1 trong GraphQL

```graphql
query {
  users {        # 1 query: SELECT * FROM users
    name
    posts {      # N queries: SELECT * FROM posts WHERE author_id = ?
      title
    }
  }
}
```

Với 100 users → 1 + 100 = **101 database queries**.

### DataLoader — Batching và Caching

DataLoader batch multiple requests trong cùng tick của Event Loop và cache kết quả per-request.

```typescript
import DataLoader from 'dataloader';

export function createLoaders(prisma: PrismaClient) {
  return {
    postsByAuthorId: new DataLoader(async (authorIds: readonly string[]) => {
      const posts = await prisma.post.findMany({
        where: { authorId: { in: [...authorIds] } },
      });

      // Group posts by authorId
      const postsByAuthor = new Map<string, Post[]>();
      for (const post of posts) {
        const existing = postsByAuthor.get(post.authorId) || [];
        existing.push(post);
        postsByAuthor.set(post.authorId, existing);
      }

      // Return theo thứ tự authorIds (DataLoader requirement)
      return authorIds.map((id) => postsByAuthor.get(id) || []);
    }),

    userById: new DataLoader(async (ids: readonly string[]) => {
      const users = await prisma.user.findMany({
        where: { id: { in: [...ids] } },
      });
      const userMap = new Map(users.map((u) => [u.id, u]));
      return ids.map((id) => userMap.get(id) || null);
    }),
  };
}
```

### Resolver với DataLoader

```typescript
User: {
  posts: (parent, _args, { loaders }) => {
    return loaders.postsByAuthorId.load(parent.id);
  },
},

Post: {
  author: (parent, _args, { loaders }) => {
    return loaders.userById.load(parent.authorId);
  },
},
```

**Kết quả:** 100 users → 1 query users + 1 batched query posts = **2 queries**.

---

## Mutations và Subscriptions

### Mutations

```graphql
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    id
    title
    author { name }
  }
}
```

```typescript
Mutation: {
  createPost: async (_parent, { input }, { prisma, user }) => {
    if (!user) throw new Error('Unauthorized');

    return prisma.post.create({
      data: {
        title: input.title,
        content: input.content,
        authorId: user.id,
      },
    });
  },
},
```

### Subscriptions (Real-time)

```graphql
type Subscription {
  commentAdded(postId: ID!): Comment!
  postUpdated: Post!
}
```

```typescript
import { PubSub } from 'graphql-subscriptions';

const pubsub = new PubSub();

const resolvers = {
  Subscription: {
    commentAdded: {
      subscribe: (_parent, { postId }) => {
        return pubsub.asyncIterator(`COMMENT_ADDED_${postId}`);
      },
    },
  },

  Mutation: {
    addComment: async (_parent, { postId, text }, { prisma, user }) => {
      const comment = await prisma.comment.create({
        data: { postId, text, authorId: user.id },
      });
      pubsub.publish(`COMMENT_ADDED_${postId}`, { commentAdded: comment });
      return comment;
    },
  },
};
```

**Production:** Dùng Redis PubSub (`graphql-redis-subscriptions`) cho multi-instance scaling.

---

## Authentication và Authorization

### Context-based Auth

```typescript
// auth middleware trước Apollo
app.use('/graphql', async (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (token) {
    try {
      req.user = verifyJwt(token);
    } catch {
      req.user = null;
    }
  }
  next();
});
```

### Field-level Authorization

```typescript
import { shield, rule } from 'graphql-shield';

const isAuthenticated = rule()(async (_parent, _args, { user }) => {
  return user !== null;
});

const isOwner = rule()(async (_parent, { id }, { user, prisma }) => {
  const post = await prisma.post.findUnique({ where: { id } });
  return post?.authorId === user?.id;
});

const permissions = shield({
  Query: {
    users: isAuthenticated,
  },
  Mutation: {
    createPost: isAuthenticated,
    deletePost: isOwner,
  },
});
```

---

## Error Handling

```typescript
import { GraphQLError } from 'graphql';

// Custom error classes
class NotFoundError extends GraphQLError {
  constructor(resource: string) {
    super(`${resource} not found`, {
      extensions: { code: 'NOT_FOUND' },
    });
  }
}

class UnauthorizedError extends GraphQLError {
  constructor() {
    super('Unauthorized', {
      extensions: { code: 'UNAUTHORIZED' },
    });
  }
}

// Resolver usage
user: async (_parent, { id }, { prisma }) => {
  const user = await prisma.user.findUnique({ where: { id } });
  if (!user) throw new NotFoundError('User');
  return user;
},
```

### Error Response Format

```json
{
  "errors": [
    {
      "message": "User not found",
      "extensions": { "code": "NOT_FOUND" },
      "path": ["user"]
    }
  ],
  "data": { "user": null }
}
```

---

## Best Practices

| Practice | Lý Do |
| -------- | ----- |
| DataLoader per-request | Tránh cache leak giữa requests |
| Pagination (cursor-based) | Tránh fetch unlimited lists |
| Query depth/complexity limits | Prevent malicious deep queries |
| Input validation (Zod) | Validate trước khi vào resolver |
| Separate schema cho public/internal | Security boundaries |
| Monitor resolver performance | Identify slow resolvers |

### Query Complexity Limiting

```typescript
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    createComplexityLimitRule(1000, {
      scalarCost: 1,
      objectCost: 2,
      listFactor: 10,
    }),
  ],
});
```

---

## Câu Hỏi Phỏng Vấn

| Câu Hỏi | Trả Lời Ngắn |
| ------- | ------------ |
| GraphQL giải quyết vấn đề gì của REST? | Over-fetching, under-fetching, multiple round trips |
| N+1 trong GraphQL là gì? | 1 query list + N queries cho nested fields |
| DataLoader hoạt động thế nào? | Batch requests trong same tick, cache per-request |
| Query vs Mutation vs Subscription? | Query: read; Mutation: write; Subscription: real-time push |
| GraphQL caching challenges? | Single endpoint, POST requests — cần normalized cache (Apollo Client) |
| Schema-first vs Code-first? | Schema-first: SDL trước; Code-first: TypeScript types generate schema |
| Khi nào KHÔNG dùng GraphQL? | Simple CRUD, public API cần HTTP cache, file upload heavy |
