# MongoDB & Mongoose — Document Schema, Aggregation và Indexing

> Mongoose là ODM (Object-Document Mapper — Ánh Xạ Đối Tượng–Tài Liệu) phổ biến nhất cho MongoDB trong Node.js. Cung cấp schema validation, middleware, và aggregation pipeline.

## Mục Lục

1. [MongoDB vs PostgreSQL](#mongodb-vs-postgresql)
2. [Mongoose Setup](#mongoose-setup)
3. [Schema và Model](#schema-và-model)
4. [CRUD Operations](#crud-operations)
5. [Population (Tham Chiếu Document)](#population-tham-chiếu-document)
6. [Aggregation Pipeline](#aggregation-pipeline)
7. [Indexing (Đánh Chỉ Mục)](#indexing-đánh-chỉ-mục)
8. [Middleware và Validation](#middleware-và-validation)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## MongoDB vs PostgreSQL

| Tiêu Chí | MongoDB | PostgreSQL |
| -------- | ------- | ---------- |
| **Data Model** | BSON documents (JSON-like) | Relational tables |
| **Schema** | Flexible — schema optional | Strict schema |
| **Joins** | `$lookup` hoặc embed documents | Native SQL JOIN |
| **Transactions** | Multi-document từ v4.0 | Native ACID |
| **Scaling** | Horizontal sharding built-in | Read replicas, partitioning |
| **Use Case** | CMS, logs, IoT, flexible models | Finance, reporting, complex relations |

---

## Mongoose Setup

```bash
npm install mongoose
```

```typescript
import mongoose from 'mongoose';

const MONGODB_URI = process.env.MONGODB_URI || 'mongodb://localhost:27017/myapp';

export async function connectDB() {
  try {
    await mongoose.connect(MONGODB_URI, {
      maxPoolSize: 10,
      serverSelectionTimeoutMS: 5000,
    });
    console.log('MongoDB connected');
  } catch (err) {
    console.error('MongoDB connection error:', err);
    process.exit(1);
  }
}

mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected');
});

process.on('SIGINT', async () => {
  await mongoose.connection.close();
  process.exit(0);
});
```

---

## Schema và Model

```typescript
import mongoose, { Schema, Document, Types } from 'mongoose';

// Interface cho TypeScript
interface IUser extends Document {
  _id: Types.ObjectId;
  email: string;
  name?: string;
  role: 'USER' | 'ADMIN';
  posts: Types.ObjectId[];
  profile?: IProfile;
  createdAt: Date;
  updatedAt: Date;
}

interface IProfile {
  bio?: string;
  avatar?: string;
}

const profileSchema = new Schema<IProfile>({
  bio: { type: String, maxlength: 500 },
  avatar: { type: String },
}, { _id: false });

const userSchema = new Schema<IUser>(
  {
    email: {
      type: String,
      required: [true, 'Email is required'],
      unique: true,
      lowercase: true,
      trim: true,
      match: [/^\S+@\S+\.\S+$/, 'Invalid email'],
    },
    name: { type: String, trim: true },
    role: {
      type: String,
      enum: ['USER', 'ADMIN'],
      default: 'USER',
    },
    posts: [{ type: Schema.Types.ObjectId, ref: 'Post' }],
    profile: profileSchema, // Embedded document
  },
  {
    timestamps: true,
    collection: 'users',
  }
);

// Virtual — không lưu DB
userSchema.virtual('postCount', {
  ref: 'Post',
  localField: '_id',
  foreignField: 'author',
  count: true,
});

userSchema.set('toJSON', { virtuals: true });

export const User = mongoose.model<IUser>('User', userSchema);
```

### Embed vs Reference

```
Embed (nhúng):                    Reference (tham chiếu):
┌─────────────────────┐          ┌──────────┐    ┌──────────┐
│ User                │          │ User     │    │ Post     │
│  email: "..."       │          │  posts:  │───►│ author:  │
│  profile: {         │          │  [id1,   │    │  userId  │
│    bio: "..."       │          │   id2]   │    └──────────┘
│  }                  │          └──────────┘
└─────────────────────┘

Embed: 1 query, data luôn đi cùng nhau
Reference: normalize, tránh duplicate, cần populate/$lookup
```

---

## CRUD Operations

```typescript
// CREATE
const user = await User.create({
  email: 'alice@example.com',
  name: 'Alice',
  profile: { bio: 'Developer' },
});

// READ
const users = await User.find({ role: 'USER' })
  .select('email name createdAt')
  .sort({ createdAt: -1 })
  .limit(20)
  .lean(); // Plain JS object — nhanh hơn, không có Mongoose methods

const user = await User.findById(userId);
const user = await User.findOne({ email: 'alice@example.com' });

// UPDATE
await User.findByIdAndUpdate(
  userId,
  { name: 'Alice Updated' },
  { new: true, runValidators: true } // Trả document mới, chạy validation
);

// UPDATE MANY
await User.updateMany(
  { role: 'USER' },
  { $set: { verified: true } }
);

// DELETE
await User.findByIdAndDelete(userId);

// UPSERT
await User.findOneAndUpdate(
  { email: 'bob@example.com' },
  { name: 'Bob' },
  { upsert: true, new: true }
);
```

### Query Operators

```typescript
await User.find({
  email: { $regex: '@gmail.com$', $options: 'i' },
  createdAt: { $gte: new Date('2024-01-01') },
  role: { $in: ['USER', 'ADMIN'] },
  name: { $exists: true, $ne: null },
});
```

---

## Population (Tham Chiếu Document)

```typescript
const postSchema = new Schema({
  title: { type: String, required: true },
  content: String,
  author: { type: Schema.Types.ObjectId, ref: 'User', required: true },
  tags: [{ type: Schema.Types.ObjectId, ref: 'Tag' }],
}, { timestamps: true });

export const Post = mongoose.model('Post', postSchema);

// Populate author khi query posts
const posts = await Post.find({ published: true })
  .populate('author', 'email name') // Chỉ lấy email, name
  .populate('tags', 'name')
  .sort({ createdAt: -1 });

// Nested populate
const users = await User.find()
  .populate({
    path: 'posts',
    match: { published: true },
    options: { sort: { createdAt: -1 }, limit: 5 },
  });
```

> **Population tương đương JOIN** — cẩn thận N+1 nếu populate trong loop.

---

## Aggregation Pipeline

Cho analytics, reporting, complex transforms:

```typescript
const stats = await Post.aggregate([
  // Stage 1: Filter
  { $match: { published: true, createdAt: { $gte: new Date('2024-01-01') } } },

  // Stage 2: Join users
  {
    $lookup: {
      from: 'users',
      localField: 'author',
      foreignField: '_id',
      as: 'authorInfo',
    },
  },
  { $unwind: '$authorInfo' },

  // Stage 3: Group
  {
    $group: {
      _id: '$authorInfo._id',
      authorName: { $first: '$authorInfo.name' },
      postCount: { $sum: 1 },
      avgTitleLength: { $avg: { $strLenCP: '$title' } },
    },
  },

  // Stage 4: Sort & limit
  { $sort: { postCount: -1 } },
  { $limit: 10 },

  // Stage 5: Project output shape
  {
    $project: {
      _id: 0,
      authorId: '$_id',
      authorName: 1,
      postCount: 1,
      avgTitleLength: { $round: ['$avgTitleLength', 0] },
    },
  },
]);
```

### Common Aggregation Stages

| Stage | Mục Đích |
| ----- | -------- |
| `$match` | Filter documents (như WHERE) |
| `$group` | Group và aggregate (như GROUP BY) |
| `$lookup` | Join collections (như LEFT JOIN) |
| `$unwind` | Deconstruct array field |
| `$project` | Reshape documents (như SELECT) |
| `$sort` | Sort results |
| `$limit` / `$skip` | Pagination |

---

## Indexing (Đánh Chỉ Mục)

```typescript
// Schema-level indexes
userSchema.index({ email: 1 }, { unique: true });
userSchema.index({ role: 1, createdAt: -1 }); // Compound index
userSchema.index({ name: 'text' }); // Text search

// TTL index — auto-delete sau thời gian
sessionSchema.index({ createdAt: 1 }, { expireAfterSeconds: 86400 });

// Tạo index trong code hoặc MongoDB shell
await User.createIndexes();

// Explain query plan
const explain = await User.find({ email: 'test@example.com' }).explain('executionStats');
console.log(explain.executionStats);
```

### Index Best Practices

1. **Index fields thường query** — `where`, `sort`, `lookup` fields
2. **Compound index order** — equality fields trước, range/sort sau
3. **ESR rule** — Equality, Sort, Range cho compound index
4. **Không over-index** — mỗi index tốn write performance và disk
5. **Dùng `.explain()`** — verify index được sử dụng (`IXSCAN` not `COLLSCAN`)

---

## Middleware và Validation

```typescript
// Pre-save hook
userSchema.pre('save', async function (next) {
  if (this.isModified('password')) {
    const bcrypt = require('bcrypt');
    this.password = await bcrypt.hash(this.password, 12);
  }
  next();
});

// Post-save hook
userSchema.post('save', function (doc) {
  console.log(`User ${doc._id} saved`);
});

// Custom validator
userSchema.path('email').validate(async function (email) {
  const count = await mongoose.models.User.countDocuments({ email, _id: { $ne: this._id } });
  return count === 0;
}, 'Email already exists');
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Khi nào embed vs reference trong MongoDB?

**Trả lời:** Embed khi data luôn đọc cùng nhau, one-to-few, không cần query độc lập (address trong user). Reference khi one-to-many lớn, data shared (author → nhiều posts), hoặc cần update độc lập.

### Câu 2: `.lean()` có lợi ích gì?

**Trả lời:** Trả plain JavaScript object thay vì Mongoose document — nhanh hơn ~5x, ít memory. Trade-off: không có `.save()`, virtuals, getters/setters, middleware.

### Câu 3: MongoDB có ACID không?

**Trả lời:** Single-document operations luôn atomic. Multi-document transactions từ MongoDB 4.0+ với replica set — nhưng overhead cao hơn PostgreSQL. Thiết kế schema embed để minimize need for transactions.

### Câu 4: `$lookup` vs populate?

**Trả lời:** Cùng mục đích (join collections). `populate` là Mongoose abstraction — dễ dùng trong app code. `$lookup` trong aggregation pipeline — mạnh hơn cho complex transforms, filtering sau join.

---

**Xem tiếp:** [6-redis-caching.md](./6-redis-caching.md) — in-memory data store cho caching và sessions.
