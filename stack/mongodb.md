# MongoDB (2025)

> **Last updated**: January 2026
> **Versions covered**: MongoDB 7.x
> **Purpose**: Document database for flexible, scalable applications

---

## Philosophy (2025-2026)

MongoDB is the **flexible document database** that stores data as JSON-like documents, offering schema flexibility, horizontal scaling, and powerful aggregation.

**Key philosophical shifts:**
- **Design for access patterns** — Schema serves queries, not normalization
- **Embed by default** — Denormalize for read performance
- **Reference when necessary** — Large/frequently updated data
- **Sharding for scale** — Horizontal scaling built-in
- **Aggregation pipelines** — Complex queries in the database
- **Atlas everywhere** — Managed service as default

---

## TL;DR

- Design schema based on query patterns, not entity relationships
- Embed data accessed together; reference data updated independently
- Avoid unbounded arrays (16MB document limit)
- Create compound indexes for common queries
- Use aggregation pipelines for complex queries
- Enable authentication from day one
- Monitor index usage and remove unused indexes
- Consider TTL indexes for expiring data

---

## Best Practices

### Project Structure (Node.js/Mongoose)

```
my-mongodb-app/
├── src/
│   ├── models/
│   │   ├── User.ts
│   │   ├── Post.ts
│   │   └── Comment.ts
│   ├── services/
│   │   ├── user.service.ts
│   │   └── post.service.ts
│   ├── repositories/
│   │   ├── user.repository.ts
│   │   └── post.repository.ts
│   ├── db/
│   │   └── connection.ts
│   └── types/
│       └── index.ts
├── scripts/
│   └── seed.ts
└── package.json
```

### Connection Setup

```typescript
// src/db/connection.ts
import mongoose from 'mongoose';

const MONGODB_URI = process.env.MONGODB_URI!;

export async function connectDB() {
  try {
    await mongoose.connect(MONGODB_URI, {
      // Connection pool settings
      maxPoolSize: 10,
      minPoolSize: 5,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    console.log('Connected to MongoDB');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
}

// Handle connection events
mongoose.connection.on('error', (err) => {
  console.error('MongoDB error:', err);
});

mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected');
});

// Graceful shutdown
process.on('SIGINT', async () => {
  await mongoose.connection.close();
  process.exit(0);
});
```

### Schema Design: Embedding

```typescript
// src/models/Post.ts
import mongoose, { Schema, Document } from 'mongoose';

// Embedded comment schema (good for bounded, co-located data)
const CommentSchema = new Schema({
  author: {
    _id: { type: Schema.Types.ObjectId, required: true },
    name: { type: String, required: true },
    avatar: String,
  },
  content: { type: String, required: true, maxlength: 2000 },
  createdAt: { type: Date, default: Date.now },
});

// Main post schema
const PostSchema = new Schema({
  title: {
    type: String,
    required: true,
    maxlength: 200,
    index: true,
  },
  slug: {
    type: String,
    required: true,
    unique: true,
  },
  content: {
    type: String,
    required: true,
  },
  author: {
    _id: { type: Schema.Types.ObjectId, ref: 'User', required: true },
    name: { type: String, required: true },
    avatar: String,
  },
  tags: [{
    type: String,
    index: true,
  }],
  // Embed comments (bounded — limit to 100)
  comments: {
    type: [CommentSchema],
    validate: [
      (arr: any[]) => arr.length <= 100,
      'Maximum 100 comments per post',
    ],
  },
  commentsCount: { type: Number, default: 0 },
  likesCount: { type: Number, default: 0 },
  status: {
    type: String,
    enum: ['draft', 'published', 'archived'],
    default: 'draft',
    index: true,
  },
  publishedAt: Date,
}, {
  timestamps: true,
});

// Compound index for common queries
PostSchema.index({ status: 1, publishedAt: -1 });
PostSchema.index({ 'author._id': 1, createdAt: -1 });

// Text index for search
PostSchema.index({ title: 'text', content: 'text' });

export interface IPost extends Document {
  title: string;
  slug: string;
  content: string;
  author: {
    _id: mongoose.Types.ObjectId;
    name: string;
    avatar?: string;
  };
  tags: string[];
  comments: Array<{
    author: { _id: mongoose.Types.ObjectId; name: string; avatar?: string };
    content: string;
    createdAt: Date;
  }>;
  commentsCount: number;
  likesCount: number;
  status: 'draft' | 'published' | 'archived';
  publishedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

export const Post = mongoose.model<IPost>('Post', PostSchema);
```

### Schema Design: Referencing

```typescript
// src/models/User.ts
import mongoose, { Schema, Document } from 'mongoose';

const UserSchema = new Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
  },
  name: {
    type: String,
    required: true,
    trim: true,
  },
  avatar: String,
  passwordHash: {
    type: String,
    required: true,
    select: false,  // Don't return by default
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user',
  },
  // Reference posts (unbounded, so don't embed)
  // Query posts separately: Post.find({ 'author._id': userId })

  // Stats (denormalized for quick access)
  stats: {
    postsCount: { type: Number, default: 0 },
    followersCount: { type: Number, default: 0 },
    followingCount: { type: Number, default: 0 },
  },
  lastLoginAt: Date,
}, {
  timestamps: true,
});

// Indexes
UserSchema.index({ email: 1 });
UserSchema.index({ createdAt: -1 });

export interface IUser extends Document {
  email: string;
  name: string;
  avatar?: string;
  passwordHash: string;
  role: 'user' | 'admin';
  stats: {
    postsCount: number;
    followersCount: number;
    followingCount: number;
  };
  lastLoginAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

export const User = mongoose.model<IUser>('User', UserSchema);
```

### Bucket Pattern (Time-Series Data)

```typescript
// For high-frequency data, bucket into documents
const MetricsBucketSchema = new Schema({
  sensorId: { type: String, required: true },
  date: { type: Date, required: true },  // Bucket date (e.g., hourly)
  measurements: [{
    timestamp: Date,
    value: Number,
  }],
  count: { type: Number, default: 0 },
  sum: { type: Number, default: 0 },
  min: Number,
  max: Number,
});

// One document per sensor per hour
MetricsBucketSchema.index({ sensorId: 1, date: 1 }, { unique: true });
```

### Repository Pattern

```typescript
// src/repositories/post.repository.ts
import { Post, IPost } from '../models/Post';
import { FilterQuery, UpdateQuery } from 'mongoose';

interface FindPostsOptions {
  page?: number;
  limit?: number;
  status?: 'draft' | 'published' | 'archived';
  authorId?: string;
  tags?: string[];
}

export class PostRepository {
  async findById(id: string): Promise<IPost | null> {
    return Post.findById(id);
  }

  async findBySlug(slug: string): Promise<IPost | null> {
    return Post.findOne({ slug });
  }

  async find(options: FindPostsOptions = {}): Promise<{ posts: IPost[]; total: number }> {
    const { page = 1, limit = 10, status, authorId, tags } = options;
    const skip = (page - 1) * limit;

    const filter: FilterQuery<IPost> = {};
    if (status) filter.status = status;
    if (authorId) filter['author._id'] = authorId;
    if (tags?.length) filter.tags = { $in: tags };

    const [posts, total] = await Promise.all([
      Post.find(filter)
        .sort({ publishedAt: -1 })
        .skip(skip)
        .limit(limit)
        .lean(),
      Post.countDocuments(filter),
    ]);

    return { posts, total };
  }

  async create(data: Partial<IPost>): Promise<IPost> {
    const post = new Post(data);
    return post.save();
  }

  async update(id: string, data: UpdateQuery<IPost>): Promise<IPost | null> {
    return Post.findByIdAndUpdate(id, data, { new: true, runValidators: true });
  }

  async delete(id: string): Promise<boolean> {
    const result = await Post.deleteOne({ _id: id });
    return result.deletedCount > 0;
  }

  async addComment(postId: string, comment: IPost['comments'][0]): Promise<IPost | null> {
    return Post.findByIdAndUpdate(
      postId,
      {
        $push: { comments: { $each: [comment], $slice: -100 } },
        $inc: { commentsCount: 1 },
      },
      { new: true }
    );
  }

  async search(query: string, limit = 10): Promise<IPost[]> {
    return Post.find(
      { $text: { $search: query }, status: 'published' },
      { score: { $meta: 'textScore' } }
    )
      .sort({ score: { $meta: 'textScore' } })
      .limit(limit)
      .lean();
  }
}
```

### Aggregation Pipelines

```typescript
// Get posts with author details and comment count
const posts = await Post.aggregate([
  { $match: { status: 'published' } },
  { $sort: { publishedAt: -1 } },
  { $limit: 10 },
  {
    $lookup: {
      from: 'users',
      localField: 'author._id',
      foreignField: '_id',
      as: 'authorDetails',
    },
  },
  { $unwind: '$authorDetails' },
  {
    $project: {
      title: 1,
      slug: 1,
      excerpt: { $substr: ['$content', 0, 200] },
      author: {
        _id: '$authorDetails._id',
        name: '$authorDetails.name',
        avatar: '$authorDetails.avatar',
      },
      commentsCount: 1,
      likesCount: 1,
      publishedAt: 1,
    },
  },
]);

// Get tag statistics
const tagStats = await Post.aggregate([
  { $match: { status: 'published' } },
  { $unwind: '$tags' },
  { $group: { _id: '$tags', count: { $sum: 1 } } },
  { $sort: { count: -1 } },
  { $limit: 20 },
  { $project: { tag: '$_id', count: 1, _id: 0 } },
]);

// Get user activity summary
const userActivity = await Post.aggregate([
  { $match: { 'author._id': new mongoose.Types.ObjectId(userId) } },
  {
    $group: {
      _id: {
        year: { $year: '$createdAt' },
        month: { $month: '$createdAt' },
      },
      posts: { $sum: 1 },
      totalComments: { $sum: '$commentsCount' },
      totalLikes: { $sum: '$likesCount' },
    },
  },
  { $sort: { '_id.year': -1, '_id.month': -1 } },
]);
```

### Transactions

```typescript
import mongoose from 'mongoose';

async function transferFollower(fromUserId: string, toUserId: string) {
  const session = await mongoose.startSession();

  try {
    session.startTransaction();

    await User.updateOne(
      { _id: fromUserId },
      { $inc: { 'stats.followersCount': -1 } },
      { session }
    );

    await User.updateOne(
      { _id: toUserId },
      { $inc: { 'stats.followersCount': 1 } },
      { session }
    );

    await session.commitTransaction();
  } catch (error) {
    await session.abortTransaction();
    throw error;
  } finally {
    session.endSession();
  }
}
```

### TTL Index (Expiring Data)

```typescript
// Sessions that expire after 24 hours
const SessionSchema = new Schema({
  userId: { type: Schema.Types.ObjectId, required: true },
  token: { type: String, required: true },
  createdAt: { type: Date, default: Date.now },
});

// Documents automatically deleted 24 hours after createdAt
SessionSchema.index({ createdAt: 1 }, { expireAfterSeconds: 86400 });
```

---

## Anti-Patterns

### ❌ Unbounded Arrays

**Why it's bad**: Document size limit (16MB), slow updates.

```typescript
// ❌ DON'T — Unbounded followers array
const UserSchema = new Schema({
  followers: [{ type: Schema.Types.ObjectId, ref: 'User' }],
  // Can grow to millions!
});

// ✅ DO — Separate collection with references
const FollowSchema = new Schema({
  follower: { type: Schema.Types.ObjectId, ref: 'User' },
  following: { type: Schema.Types.ObjectId, ref: 'User' },
  createdAt: { type: Date, default: Date.now },
});
FollowSchema.index({ follower: 1, following: 1 }, { unique: true });
```

### ❌ No Indexes on Query Fields

**Why it's bad**: Full collection scans.

```typescript
// ❌ DON'T — Query without index
await Post.find({ status: 'published', 'author._id': authorId });

// ✅ DO — Create compound index
PostSchema.index({ status: 1, 'author._id': 1 });
```

### ❌ Fetching Entire Documents

**Why it's bad**: Unnecessary data transfer.

```typescript
// ❌ DON'T — Fetch everything
const posts = await Post.find({ status: 'published' });

// ✅ DO — Select only needed fields
const posts = await Post.find({ status: 'published' })
  .select('title slug author publishedAt')
  .lean();
```

### ❌ No Authentication

**Why it's bad**: Open database to attacks.

```bash
# ❌ DON'T — Default no-auth connection
mongodb://localhost:27017/mydb

# ✅ DO — Enable authentication
mongodb://user:password@localhost:27017/mydb?authSource=admin
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 7.0 | 2023 | Queryable encryption, compound indexes |
| 7.x | 2024-2025 | Improved aggregation, vector search |
| Atlas | 2025 | AI integrations, serverless instances |

---

## Quick Reference

| Operation | Mongoose | Native Driver |
|-----------|----------|---------------|
| Find one | `Model.findById(id)` | `collection.findOne({ _id })` |
| Find many | `Model.find(filter)` | `collection.find(filter)` |
| Insert | `new Model(data).save()` | `collection.insertOne(data)` |
| Update | `Model.findByIdAndUpdate()` | `collection.updateOne()` |
| Delete | `Model.deleteOne()` | `collection.deleteOne()` |
| Count | `Model.countDocuments()` | `collection.countDocuments()` |
| Aggregate | `Model.aggregate([...])` | `collection.aggregate([...])` |

| Index Type | Use Case |
|------------|----------|
| Single field | Simple queries |
| Compound | Multi-field queries |
| Text | Full-text search |
| TTL | Expiring data |
| Unique | Enforce uniqueness |
| Sparse | Index only documents with field |

---

## Resources

- [Official MongoDB Documentation](https://www.mongodb.com/docs/)
- [Schema Design Best Practices](https://www.mongodb.com/developer/products/mongodb/mongodb-schema-design-best-practices/)
- [Data Modeling Guide](https://www.mongodb.com/docs/manual/data-modeling/)
- [MongoDB University](https://learn.mongodb.com/)
- [Mongoose Documentation](https://mongoosejs.com/docs/)
- [Performance Best Practices](https://www.mongodb.com/resources/products/capabilities/performance-best-practices)
