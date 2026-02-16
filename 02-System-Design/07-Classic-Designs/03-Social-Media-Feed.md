# Design: Social Media Feed (Twitter/Facebook)

## Requirements

### Functional Requirements
- Post content (text, images, videos)
- Follow/unfollow users
- View home feed (posts from followed users)
- Like, comment, share posts
- Real-time feed updates

### Non-Functional Requirements
- High availability
- Low latency feed loading (<200ms)
- Scale: 500M users, 10M posts/day
- Eventually consistent (acceptable)

### Capacity Estimation

```
Users: 500M total, 200M daily active
Posts: 10M new posts/day
Feed reads: 500M feed views/day

Storage:
- Post: 1KB avg (text + metadata) = 10GB/day
- Media: stored in object storage (S3)

Timeline:
- Avg user follows 200 people
- Avg user posts 2 posts/day
- Feed shows 100 posts
```

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         Clients                                   │
└────────────────────────────┬─────────────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────────┐
│                      API Gateway                                  │
│              (Auth, Rate Limiting, Routing)                       │
└────────────────────────────┬─────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│  Post Service  │  │  Feed Service  │  │  User Service  │
└───────┬────────┘  └───────┬────────┘  └───────┬────────┘
        │                   │                   │
        ▼                   ▼                   ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│   Posts DB     │  │  Feed Cache    │  │   Users DB     │
│  (Cassandra)   │  │   (Redis)      │  │  (PostgreSQL)  │
└────────────────┘  └────────────────┘  └────────────────┘
```

---

## Feed Generation: Push vs Pull

### Pull Model (Fan-out on Read)

```
When user loads feed:
1. Get list of followed users
2. For each followed user, get recent posts
3. Merge and sort posts
4. Return top N

┌─────────┐
│  User   │  Load Feed
└────┬────┘
     │
     ▼
┌─────────────────────────────────────────────────┐
│  1. Get followed users (200 users)              │
│  2. Query posts for each user                   │
│  3. Merge and rank                              │
│  4. Return feed                                 │
└─────────────────────────────────────────────────┘
```

```javascript
async function getFeed(userId) {
  // Get followed users
  const following = await getFollowing(userId); // ~200 users

  // Get posts from each (in parallel)
  const posts = await Promise.all(
    following.map(followedId =>
      getRecentPosts(followedId, 20) // Last 20 posts each
    )
  );

  // Merge, rank, and return
  return posts
    .flat()
    .sort((a, b) => b.score - a.score) // Ranking algorithm
    .slice(0, 100);
}
```

**Pros:** Simple, always fresh, good for inactive users
**Cons:** Slow for users following many people, high latency

### Push Model (Fan-out on Write)

```
When user posts:
1. Write post to Posts DB
2. Get all followers
3. Push post ID to each follower's feed cache

User posts → Fan out to all followers

┌─────────┐          ┌─────────────────────────────────────┐
│ User A  │ ──Post──▶│ Get followers (1M followers)        │
│ posts   │          │ For each follower:                  │
└─────────┘          │   Add post to their feed cache      │
                     └─────────────────────────────────────┘
                                    │
                                    ▼
                     ┌─────────────────────────────────────┐
                     │  Follower 1: [post_id_new, ...]     │
                     │  Follower 2: [post_id_new, ...]     │
                     │  ...                                │
                     │  Follower 1M: [post_id_new, ...]    │
                     └─────────────────────────────────────┘
```

```javascript
// On new post
async function createPost(userId, content) {
  // Save post
  const post = await savePost(userId, content);

  // Get followers
  const followers = await getFollowers(userId);

  // Push to each follower's feed (async)
  await fanoutQueue.push({
    postId: post.id,
    followers: followers
  });

  return post;
}

// Worker processes fan-out
async function processFanout(job) {
  const { postId, followers } = job;

  for (const followerId of followers) {
    await redis.lpush(`feed:${followerId}`, postId);
    await redis.ltrim(`feed:${followerId}`, 0, 999); // Keep 1000 posts
  }
}
```

**Pros:** Fast feed reads
**Cons:** Slow writes for celebrities (1M+ followers), storage for inactive users

### Hybrid Approach (Recommended)

```
Celebrity (>10K followers): Pull
Regular user: Push

On post:
- Regular user: Fan-out to followers
- Celebrity: Store only, followers pull

On feed read:
- Get pre-computed feed (pushed posts)
- Query celebrity posts
- Merge and return
```

```javascript
const CELEBRITY_THRESHOLD = 10000;

async function createPost(userId, content) {
  const post = await savePost(userId, content);

  const followerCount = await getFollowerCount(userId);

  if (followerCount < CELEBRITY_THRESHOLD) {
    // Push to followers
    await fanoutQueue.push({ postId: post.id, userId });
  }
  // Celebrities don't fan out

  return post;
}

async function getFeed(userId) {
  // Get pushed feed
  const pushedPostIds = await redis.lrange(`feed:${userId}`, 0, 99);
  const pushedPosts = await getPosts(pushedPostIds);

  // Get celebrity posts (pull)
  const celebrities = await getFollowedCelebrities(userId);
  const celebrityPosts = await Promise.all(
    celebrities.map(c => getRecentPosts(c.id, 20))
  );

  // Merge
  return [...pushedPosts, ...celebrityPosts.flat()]
    .sort((a, b) => b.score - a.score)
    .slice(0, 100);
}
```

---

## Database Schema

### Posts Table (Cassandra)

```sql
CREATE TABLE posts (
  post_id UUID,
  user_id UUID,
  content TEXT,
  media_urls LIST<TEXT>,
  created_at TIMESTAMP,
  like_count COUNTER,
  comment_count COUNTER,
  share_count COUNTER,
  PRIMARY KEY ((user_id), created_at, post_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- For fetching posts by user, sorted by time
```

### User Timeline (Redis)

```javascript
// Feed stored as sorted set
// Key: feed:{userId}
// Score: timestamp
// Value: postId

ZADD feed:user123 1640000000 "post_abc"
ZADD feed:user123 1640000100 "post_def"

// Get feed
ZREVRANGE feed:user123 0 99  // Latest 100 posts
```

### Social Graph (Neo4j or Graph table)

```sql
CREATE TABLE follows (
  follower_id UUID,
  followee_id UUID,
  created_at TIMESTAMP,
  PRIMARY KEY (follower_id, followee_id)
);

CREATE INDEX ON follows (followee_id);  -- Get followers
```

---

## Feed Ranking

```javascript
function calculateScore(post, user) {
  const recencyScore = getRecencyScore(post.createdAt);
  const engagementScore = getEngagementScore(post);
  const affinityScore = getAffinityScore(user, post.author);
  const diversityBonus = getDiversityBonus(post);

  return (
    recencyScore * 0.4 +
    engagementScore * 0.3 +
    affinityScore * 0.2 +
    diversityBonus * 0.1
  );
}

function getRecencyScore(createdAt) {
  const ageHours = (Date.now() - createdAt) / 3600000;
  return 1 / (1 + 0.1 * ageHours); // Decay over time
}

function getEngagementScore(post) {
  const likes = post.likeCount;
  const comments = post.commentCount * 2;
  const shares = post.shareCount * 3;
  return Math.log(1 + likes + comments + shares);
}

function getAffinityScore(user, author) {
  // Based on interaction history
  const interactions = getInteractionCount(user.id, author.id);
  return Math.min(1, interactions / 10);
}
```

---

## Real-Time Updates

### WebSocket for Live Feed

```javascript
// Server
const wss = new WebSocket.Server({ port: 8080 });
const userConnections = new Map();

wss.on('connection', (ws, req) => {
  const userId = authenticate(req);
  userConnections.set(userId, ws);

  ws.on('close', () => {
    userConnections.delete(userId);
  });
});

// When new post is created
async function notifyFollowers(post) {
  const followers = await getFollowers(post.userId);

  for (const followerId of followers) {
    const ws = userConnections.get(followerId);
    if (ws) {
      ws.send(JSON.stringify({
        type: 'NEW_POST',
        post: post
      }));
    }
  }
}
```

### Server-Sent Events (SSE)

```javascript
app.get('/feed/stream', authenticate, (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');

  const userId = req.user.id;

  const subscriber = (message) => {
    res.write(`data: ${JSON.stringify(message)}\n\n`);
  };

  pubsub.subscribe(`feed:${userId}`, subscriber);

  req.on('close', () => {
    pubsub.unsubscribe(`feed:${userId}`, subscriber);
  });
});
```

---

## Media Handling

```
┌──────────┐     ┌─────────────┐     ┌─────────────┐
│  Client  │────▶│ Upload URL  │────▶│     S3      │
│          │     │  (presigned)│     │   (upload)  │
└──────────┘     └─────────────┘     └──────┬──────┘
                                            │
                                     ┌──────▼──────┐
                                     │  Lambda     │
                                     │ (process)   │
                                     └──────┬──────┘
                                            │
                                     ┌──────▼──────┐
                                     │    CDN      │
                                     │  (serve)    │
                                     └─────────────┘
```

```javascript
// Generate presigned URL for upload
app.post('/posts/upload-url', async (req, res) => {
  const key = `uploads/${req.user.id}/${uuid()}.jpg`;

  const uploadUrl = await s3.getSignedUrl('putObject', {
    Bucket: 'media-bucket',
    Key: key,
    ContentType: 'image/jpeg',
    Expires: 300
  });

  res.json({
    uploadUrl,
    key,
    cdnUrl: `https://cdn.example.com/${key}`
  });
});
```

---

## Caching Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    Caching Layers                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Feed Cache (Redis)                                         │
│  - Pre-computed feeds for active users                      │
│  - TTL: 24 hours, refresh on activity                       │
│                                                             │
│  Post Cache (Redis)                                         │
│  - Recent/popular posts                                     │
│  - TTL: 1 hour                                              │
│                                                             │
│  User Cache (Redis)                                         │
│  - User profiles, follower counts                           │
│  - TTL: 15 minutes                                          │
│                                                             │
│  CDN                                                        │
│  - Media files                                              │
│  - Static assets                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Discussion Points

### How to handle celebrity posts?

1. Don't fan out - followers pull
2. Cache celebrity posts aggressively
3. Separate celebrity feed service
4. Pre-compute for top fans only

### How to ensure feed freshness?

1. Background refresh jobs
2. Invalidate on new post/follow
3. Mix of push (real-time) and pull (refresh)
4. WebSocket for immediate updates

### How to handle delete/edit?

1. Mark post as deleted (soft delete)
2. Background job removes from feeds
3. Accept eventual consistency
4. Cache invalidation

### How to rank posts?

1. Recency + engagement hybrid
2. User affinity (interaction history)
3. Content type preference
4. Diversity (avoid same author)
5. ML-based personalization
