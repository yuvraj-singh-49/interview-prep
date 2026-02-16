# GraphQL & WebSocket APIs

## GraphQL

### Overview

```
REST vs GraphQL:
┌─────────────────────────────────────────────────────────────────┐
│                          REST                                    │
│  GET /users/123                    → User data                   │
│  GET /users/123/posts              → User's posts                │
│  GET /users/123/followers          → User's followers            │
│  = 3 requests, possible over-fetching                            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        GraphQL                                   │
│  query {                                                         │
│    user(id: "123") {                                            │
│      name                                                        │
│      posts { title }                                            │
│      followers { name }                                         │
│    }                                                             │
│  }                                                               │
│  = 1 request, exact data needed                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Schema Definition

```graphql
# Type definitions
type User {
  id: ID!
  email: String!
  name: String!
  avatar: String
  posts(first: Int, after: String): PostConnection!
  followers: [User!]!
  createdAt: DateTime!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments(first: Int): [Comment!]!
  likes: Int!
  published: Boolean!
  createdAt: DateTime!
}

type Comment {
  id: ID!
  content: String!
  author: User!
  post: Post!
  createdAt: DateTime!
}

# Pagination (Relay-style)
type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# Input types
input CreatePostInput {
  title: String!
  content: String!
  published: Boolean = false
}

input UpdatePostInput {
  title: String
  content: String
  published: Boolean
}

# Queries
type Query {
  user(id: ID!): User
  users(first: Int, after: String, filter: UserFilter): UserConnection!
  post(id: ID!): Post
  posts(first: Int, after: String, filter: PostFilter): PostConnection!
  me: User
}

# Mutations
type Mutation {
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post!
  deletePost(id: ID!): Boolean!
  likePost(id: ID!): Post!
  addComment(postId: ID!, content: String!): Comment!
}

# Subscriptions
type Subscription {
  postCreated: Post!
  commentAdded(postId: ID!): Comment!
  userTyping(conversationId: ID!): User!
}
```

### Resolver Implementation

```javascript
const resolvers = {
  Query: {
    user: async (_, { id }, context) => {
      return context.dataSources.userAPI.getUser(id);
    },

    users: async (_, { first = 10, after, filter }, context) => {
      return context.dataSources.userAPI.getUsers({ first, after, filter });
    },

    me: async (_, __, context) => {
      if (!context.user) throw new AuthenticationError('Not authenticated');
      return context.dataSources.userAPI.getUser(context.user.id);
    },

    posts: async (_, args, context) => {
      return context.dataSources.postAPI.getPosts(args);
    }
  },

  Mutation: {
    createPost: async (_, { input }, context) => {
      if (!context.user) throw new AuthenticationError('Not authenticated');

      return context.dataSources.postAPI.createPost({
        ...input,
        authorId: context.user.id
      });
    },

    updatePost: async (_, { id, input }, context) => {
      const post = await context.dataSources.postAPI.getPost(id);

      if (post.authorId !== context.user.id) {
        throw new ForbiddenError('Not authorized');
      }

      return context.dataSources.postAPI.updatePost(id, input);
    },

    likePost: async (_, { id }, context) => {
      return context.dataSources.postAPI.likePost(id, context.user.id);
    }
  },

  // Field resolvers
  User: {
    posts: async (user, { first, after }, context) => {
      return context.dataSources.postAPI.getPostsByUser(user.id, { first, after });
    },

    followers: async (user, _, context) => {
      return context.dataSources.userAPI.getFollowers(user.id);
    }
  },

  Post: {
    author: async (post, _, context) => {
      // Uses DataLoader for batching
      return context.loaders.userLoader.load(post.authorId);
    },

    comments: async (post, { first }, context) => {
      return context.dataSources.commentAPI.getComments(post.id, { first });
    }
  },

  Subscription: {
    postCreated: {
      subscribe: () => pubsub.asyncIterator(['POST_CREATED'])
    },

    commentAdded: {
      subscribe: (_, { postId }) => {
        return pubsub.asyncIterator([`COMMENT_ADDED_${postId}`]);
      }
    }
  }
};
```

### DataLoader for N+1 Prevention

```javascript
const DataLoader = require('dataloader');

// Without DataLoader: N+1 queries
// Query 10 posts → 10 separate author queries

// With DataLoader: Batched queries
// Query 10 posts → 1 batched author query

function createLoaders() {
  return {
    userLoader: new DataLoader(async (userIds) => {
      // Batch fetch all users at once
      const users = await db.query(
        'SELECT * FROM users WHERE id = ANY($1)',
        [userIds]
      );

      // Return in same order as requested
      const userMap = new Map(users.map(u => [u.id, u]));
      return userIds.map(id => userMap.get(id));
    }),

    postLoader: new DataLoader(async (postIds) => {
      const posts = await db.query(
        'SELECT * FROM posts WHERE id = ANY($1)',
        [postIds]
      );
      const postMap = new Map(posts.map(p => [p.id, p]));
      return postIds.map(id => postMap.get(id));
    })
  };
}

// Context setup
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: ({ req }) => ({
    user: getUser(req),
    loaders: createLoaders(),  // New loaders per request
    dataSources: {
      userAPI: new UserAPI(),
      postAPI: new PostAPI()
    }
  })
});
```

### Query Complexity & Depth Limiting

```javascript
const depthLimit = require('graphql-depth-limit');
const { createComplexityLimitRule } = require('graphql-validation-complexity');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    // Limit query depth
    depthLimit(10),

    // Limit query complexity
    createComplexityLimitRule(1000, {
      scalarCost: 1,
      objectCost: 10,
      listFactor: 20,
      onCost: (cost) => {
        console.log('Query complexity:', cost);
      }
    })
  ]
});

// Custom complexity calculation
const typeDefs = gql`
  type Query {
    posts(first: Int): [Post!]! @complexity(multipliers: ["first"])
  }

  type Post {
    comments: [Comment!]! @complexity(value: 10)
  }
`;
```

### Caching

```javascript
// Response caching
const { ApolloServer } = require('apollo-server');
const responseCachePlugin = require('apollo-server-plugin-response-cache');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    responseCachePlugin({
      sessionId: (context) => context.user?.id || null
    })
  ]
});

// Cache hints in schema
const typeDefs = gql`
  type Query {
    posts: [Post!]! @cacheControl(maxAge: 60)
  }

  type Post @cacheControl(maxAge: 120) {
    id: ID!
    title: String!
    author: User! @cacheControl(maxAge: 300)
    viewCount: Int! @cacheControl(maxAge: 0)  # No cache
  }
`;

// Programmatic cache hints
const resolvers = {
  Post: {
    author: (post, _, context, info) => {
      info.cacheControl.setCacheHint({ maxAge: 300, scope: 'PUBLIC' });
      return context.loaders.userLoader.load(post.authorId);
    }
  }
};
```

---

## WebSocket APIs

### WebSocket Protocol

```
┌─────────────────────────────────────────────────────────────────┐
│                    WebSocket Handshake                           │
│                                                                  │
│  Client → Server: HTTP GET /ws (Upgrade: websocket)             │
│  Server → Client: HTTP 101 Switching Protocols                   │
│                                                                  │
│  ═══════════════════════════════════════════════════════════    │
│                    Full-duplex connection                        │
│  ═══════════════════════════════════════════════════════════    │
│                                                                  │
│  Client ←→ Server: Binary/Text frames                           │
└─────────────────────────────────────────────────────────────────┘
```

### Server Implementation

```javascript
const WebSocket = require('ws');
const http = require('http');

const server = http.createServer();
const wss = new WebSocket.Server({ server });

// Connection management
const clients = new Map();  // userId -> Set of WebSocket connections

wss.on('connection', async (ws, req) => {
  // Authenticate
  const token = new URL(req.url, 'http://localhost').searchParams.get('token');
  const user = await authenticateToken(token);

  if (!user) {
    ws.close(4001, 'Unauthorized');
    return;
  }

  // Track connection
  if (!clients.has(user.id)) {
    clients.set(user.id, new Set());
  }
  clients.get(user.id).add(ws);

  console.log(`User ${user.id} connected. Total connections: ${clients.get(user.id).size}`);

  // Handle messages
  ws.on('message', async (data) => {
    try {
      const message = JSON.parse(data);
      await handleMessage(ws, user, message);
    } catch (error) {
      ws.send(JSON.stringify({ type: 'error', message: error.message }));
    }
  });

  // Handle disconnect
  ws.on('close', () => {
    clients.get(user.id)?.delete(ws);
    if (clients.get(user.id)?.size === 0) {
      clients.delete(user.id);
    }
    console.log(`User ${user.id} disconnected`);
  });

  // Heartbeat
  ws.isAlive = true;
  ws.on('pong', () => { ws.isAlive = true; });
});

// Heartbeat interval
setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) {
      return ws.terminate();
    }
    ws.isAlive = false;
    ws.ping();
  });
}, 30000);

// Message handler
async function handleMessage(ws, user, message) {
  switch (message.type) {
    case 'chat':
      await handleChatMessage(user, message);
      break;
    case 'typing':
      await handleTypingIndicator(user, message);
      break;
    case 'subscribe':
      await handleSubscription(ws, user, message);
      break;
    default:
      ws.send(JSON.stringify({ type: 'error', message: 'Unknown message type' }));
  }
}

// Send to specific user
function sendToUser(userId, message) {
  const userConnections = clients.get(userId);
  if (userConnections) {
    const data = JSON.stringify(message);
    userConnections.forEach(ws => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(data);
      }
    });
  }
}

// Broadcast to room
function broadcastToRoom(roomId, message, excludeUserId = null) {
  const data = JSON.stringify(message);
  const roomMembers = getRoomMembers(roomId);

  roomMembers.forEach(userId => {
    if (userId !== excludeUserId) {
      sendToUser(userId, message);
    }
  });
}
```

### Chat Application Protocol

```javascript
// Message types
const MessageTypes = {
  // Client → Server
  SEND_MESSAGE: 'send_message',
  TYPING_START: 'typing_start',
  TYPING_STOP: 'typing_stop',
  READ_RECEIPT: 'read_receipt',
  JOIN_ROOM: 'join_room',
  LEAVE_ROOM: 'leave_room',

  // Server → Client
  NEW_MESSAGE: 'new_message',
  USER_TYPING: 'user_typing',
  MESSAGE_DELIVERED: 'message_delivered',
  MESSAGE_READ: 'message_read',
  USER_ONLINE: 'user_online',
  USER_OFFLINE: 'user_offline',
  ERROR: 'error'
};

// Message format
const messageSchema = {
  send_message: {
    type: 'send_message',
    roomId: 'string',
    content: 'string',
    clientMessageId: 'string',  // For deduplication
    replyTo: 'string?'  // Optional message ID
  },

  new_message: {
    type: 'new_message',
    messageId: 'string',
    roomId: 'string',
    senderId: 'string',
    content: 'string',
    timestamp: 'number',
    clientMessageId: 'string'  // Echo back for confirmation
  }
};

// Server-side handler
async function handleChatMessage(user, message) {
  const { roomId, content, clientMessageId, replyTo } = message;

  // Validate room membership
  if (!await isRoomMember(user.id, roomId)) {
    throw new Error('Not a member of this room');
  }

  // Check for duplicate (idempotency)
  const existing = await redis.get(`msg:${clientMessageId}`);
  if (existing) {
    return JSON.parse(existing);
  }

  // Store message
  const newMessage = await db.messages.create({
    roomId,
    senderId: user.id,
    content,
    replyTo,
    createdAt: new Date()
  });

  // Cache for deduplication
  await redis.setex(`msg:${clientMessageId}`, 3600, JSON.stringify(newMessage));

  // Broadcast to room
  broadcastToRoom(roomId, {
    type: MessageTypes.NEW_MESSAGE,
    messageId: newMessage.id,
    roomId,
    senderId: user.id,
    senderName: user.name,
    content,
    timestamp: newMessage.createdAt.getTime(),
    clientMessageId
  });

  // Send push notifications to offline users
  await sendPushToOfflineMembers(roomId, user, content);

  return newMessage;
}
```

### Scaling WebSockets

```
┌─────────────────────────────────────────────────────────────────┐
│                     Load Balancer                                │
│              (Sticky sessions / IP hash)                         │
└───────────────────────────┬─────────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         ▼                  ▼                  ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  WS Server 1    │ │  WS Server 2    │ │  WS Server 3    │
│  (connections)  │ │  (connections)  │ │  (connections)  │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             ▼
                    ┌─────────────────┐
                    │   Redis Pub/Sub │
                    │  (message bus)  │
                    └─────────────────┘
```

```javascript
// Redis Pub/Sub for cross-server messaging
const Redis = require('ioredis');
const publisher = new Redis();
const subscriber = new Redis();

// Subscribe to messages for this server's users
subscriber.subscribe('chat:broadcast');

subscriber.on('message', (channel, data) => {
  const message = JSON.parse(data);

  // Only send to users connected to THIS server
  const userConnections = clients.get(message.targetUserId);
  if (userConnections) {
    const payload = JSON.stringify(message.payload);
    userConnections.forEach(ws => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(payload);
      }
    });
  }
});

// Send message (publishes to Redis)
async function sendToUser(userId, payload) {
  // Try local first
  const localConnections = clients.get(userId);
  if (localConnections?.size > 0) {
    const data = JSON.stringify(payload);
    localConnections.forEach(ws => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(data);
      }
    });
  }

  // Also publish to Redis for other servers
  await publisher.publish('chat:broadcast', JSON.stringify({
    targetUserId: userId,
    payload
  }));
}
```

### Connection State Management

```javascript
// Track user presence
class PresenceManager {
  constructor(redis) {
    this.redis = redis;
    this.serverId = process.env.SERVER_ID || uuid();
  }

  async setOnline(userId) {
    const key = `presence:${userId}`;
    await this.redis.hset(key, this.serverId, Date.now());
    await this.redis.expire(key, 60);  // Auto-expire

    // Start heartbeat
    this.startHeartbeat(userId);

    // Notify friends
    await this.notifyPresenceChange(userId, 'online');
  }

  async setOffline(userId) {
    const key = `presence:${userId}`;
    await this.redis.hdel(key, this.serverId);

    // Check if user is still online on other servers
    const remaining = await this.redis.hlen(key);
    if (remaining === 0) {
      await this.notifyPresenceChange(userId, 'offline');
    }
  }

  async isOnline(userId) {
    const key = `presence:${userId}`;
    const servers = await this.redis.hlen(key);
    return servers > 0;
  }

  async getOnlineUsers(userIds) {
    const pipeline = this.redis.pipeline();
    userIds.forEach(id => pipeline.hlen(`presence:${id}`));

    const results = await pipeline.exec();
    return userIds.filter((_, i) => results[i][1] > 0);
  }

  startHeartbeat(userId) {
    const interval = setInterval(async () => {
      if (!clients.has(userId)) {
        clearInterval(interval);
        return;
      }

      await this.redis.hset(`presence:${userId}`, this.serverId, Date.now());
      await this.redis.expire(`presence:${userId}`, 60);
    }, 30000);
  }
}
```

---

## Real-Time Patterns

### Event Sourcing with WebSocket

```javascript
// Real-time collaborative editing
class CollaborativeDocument {
  constructor(documentId, wss) {
    this.documentId = documentId;
    this.operations = [];  // Operation log
    this.subscribers = new Set();
  }

  subscribe(ws, userId) {
    this.subscribers.add({ ws, userId });

    // Send current state
    ws.send(JSON.stringify({
      type: 'document_state',
      documentId: this.documentId,
      operations: this.operations
    }));
  }

  async applyOperation(userId, operation) {
    // Transform operation against concurrent operations (OT)
    const transformed = this.transform(operation);

    // Store operation
    this.operations.push({
      ...transformed,
      userId,
      timestamp: Date.now(),
      version: this.operations.length
    });

    // Persist to database
    await this.persistOperation(transformed);

    // Broadcast to all subscribers
    this.broadcast({
      type: 'operation',
      operation: transformed,
      userId
    }, userId);
  }

  broadcast(message, excludeUserId) {
    const data = JSON.stringify(message);
    this.subscribers.forEach(({ ws, userId }) => {
      if (userId !== excludeUserId && ws.readyState === WebSocket.OPEN) {
        ws.send(data);
      }
    });
  }
}
```

### Rate Limiting WebSocket Messages

```javascript
class WebSocketRateLimiter {
  constructor(options = {}) {
    this.windowMs = options.windowMs || 1000;
    this.maxMessages = options.maxMessages || 10;
    this.counters = new Map();
  }

  isAllowed(userId) {
    const now = Date.now();
    const windowStart = now - this.windowMs;

    // Get or create counter
    let counter = this.counters.get(userId);
    if (!counter) {
      counter = { timestamps: [] };
      this.counters.set(userId, counter);
    }

    // Remove old timestamps
    counter.timestamps = counter.timestamps.filter(t => t > windowStart);

    // Check limit
    if (counter.timestamps.length >= this.maxMessages) {
      return false;
    }

    // Add new timestamp
    counter.timestamps.push(now);
    return true;
  }
}

// Usage
const rateLimiter = new WebSocketRateLimiter({ maxMessages: 20, windowMs: 1000 });

ws.on('message', async (data) => {
  if (!rateLimiter.isAllowed(user.id)) {
    ws.send(JSON.stringify({
      type: 'error',
      code: 'RATE_LIMITED',
      message: 'Too many messages'
    }));
    return;
  }

  // Process message
});
```

---

## Interview Discussion Points

### When to use GraphQL vs REST?

**GraphQL is better when:**
- Mobile apps with varying data needs
- Complex data relationships
- Rapid frontend iteration
- Avoiding over-fetching/under-fetching

**REST is better when:**
- Simple CRUD operations
- Caching is critical (HTTP caching)
- File uploads (though GraphQL can handle)
- Team unfamiliar with GraphQL

### How to prevent GraphQL abuse?

1. **Query depth limiting** - Prevent deeply nested queries
2. **Query complexity analysis** - Calculate cost before execution
3. **Rate limiting** - Per-user/IP limits
4. **Persisted queries** - Only allow pre-approved queries
5. **Timeout enforcement** - Kill long-running queries

### How to scale WebSocket connections?

1. **Horizontal scaling** - Multiple WS servers behind load balancer
2. **Sticky sessions** - Route same user to same server
3. **Redis Pub/Sub** - Cross-server message delivery
4. **Connection pooling** - Limit connections per user
5. **Graceful degradation** - Fallback to long-polling

### How to handle WebSocket reconnection?

1. **Client-side retry** - Exponential backoff
2. **Session resumption** - Store last message ID
3. **Message queue** - Buffer messages during disconnect
4. **State sync** - Full state refresh on reconnect
