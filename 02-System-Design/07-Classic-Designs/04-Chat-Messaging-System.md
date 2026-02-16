# Design: Chat/Messaging System (WhatsApp, Slack)

## Requirements

### Functional Requirements
- 1:1 messaging
- Group chats (up to 500 members)
- Online/offline status
- Message delivery status (sent, delivered, read)
- Push notifications
- Message history
- Media sharing (images, videos, files)

### Non-Functional Requirements
- Real-time delivery (<100ms for online users)
- High availability (99.99%)
- Message ordering within conversation
- End-to-end encryption (optional)
- Scale: 500M DAU, 50B messages/day

### Capacity Estimation

```
Messages: 50B/day = 600K messages/second
Storage: 50B × 100 bytes avg = 5TB/day
Connections: 500M concurrent WebSocket connections

Peak: 3x average = 1.8M messages/second
```

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                           Clients                                     │
│                    (Mobile, Web, Desktop)                             │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────────────┐
│                      Load Balancer                                    │
│                   (WebSocket aware)                                   │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│  Chat Server   │  │  Chat Server   │  │  Chat Server   │
│  (WebSocket)   │  │  (WebSocket)   │  │  (WebSocket)   │
└───────┬────────┘  └───────┬────────┘  └───────┬────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────┐
│                    Message Queue                           │
│                      (Kafka)                               │
└───────────────────────────┬───────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│ Message Store  │  │ Session Store  │  │ Push Service   │
│  (Cassandra)   │  │   (Redis)      │  │  (FCM/APNs)    │
└────────────────┘  └────────────────┘  └────────────────┘
```

---

## Core Components

### Connection Management

```javascript
// WebSocket server
const WebSocket = require('ws');
const Redis = require('ioredis');

const redis = new Redis.Cluster([...]);
const connections = new Map(); // Local connection map

const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', async (ws, req) => {
  const userId = authenticateConnection(req);

  // Store connection locally
  connections.set(userId, ws);

  // Register in Redis (for cross-server routing)
  await redis.hset('user:sessions', userId, {
    serverId: SERVER_ID,
    connectedAt: Date.now()
  });

  // Update online status
  await redis.publish('presence', JSON.stringify({
    userId,
    status: 'online'
  }));

  ws.on('close', async () => {
    connections.delete(userId);
    await redis.hdel('user:sessions', userId);
    await redis.publish('presence', JSON.stringify({
      userId,
      status: 'offline',
      lastSeen: Date.now()
    }));
  });
});
```

### Message Flow

```
Sender (User A) → Chat Server 1 → Kafka → Chat Server 2 → Receiver (User B)
                                    │
                                    ▼
                              Message Store
                              (Cassandra)

1. User A sends message via WebSocket
2. Chat Server validates and publishes to Kafka
3. Message stored in Cassandra
4. Kafka delivers to Chat Server hosting User B
5. Chat Server sends via WebSocket to User B
6. User B sends ACK
7. Delivery receipt sent back to User A
```

```javascript
// Message handling
async function handleMessage(senderId, message) {
  const { conversationId, recipientId, content, clientMessageId } = message;

  // Generate server message ID
  const messageId = generateMessageId();

  // Create message object
  const msg = {
    id: messageId,
    clientMessageId,
    conversationId,
    senderId,
    recipientId,
    content,
    timestamp: Date.now(),
    status: 'sent'
  };

  // Store in Cassandra
  await cassandra.execute(`
    INSERT INTO messages (conversation_id, message_id, sender_id, content, timestamp)
    VALUES (?, ?, ?, ?, ?)
  `, [conversationId, messageId, senderId, content, msg.timestamp]);

  // Publish to Kafka for delivery
  await kafka.publish('messages', {
    key: recipientId,
    value: msg
  });

  // Send ACK to sender
  sendToUser(senderId, {
    type: 'message_sent',
    clientMessageId,
    messageId,
    timestamp: msg.timestamp
  });

  return msg;
}
```

### Message Delivery

```javascript
// Kafka consumer on each chat server
async function processMessage(msg) {
  const { recipientId } = msg;

  // Check if user is connected to this server
  const ws = connections.get(recipientId);

  if (ws && ws.readyState === WebSocket.OPEN) {
    // User is online on this server
    ws.send(JSON.stringify({
      type: 'new_message',
      message: msg
    }));

    // Update delivery status
    await updateMessageStatus(msg.id, 'delivered');
    notifySender(msg.senderId, msg.id, 'delivered');
  } else {
    // Check if user is on another server
    const session = await redis.hget('user:sessions', recipientId);

    if (session) {
      // User is online on different server - Kafka will route
      // The correct server's consumer will handle it
    } else {
      // User is offline - send push notification
      await sendPushNotification(recipientId, msg);
    }
  }
}
```

---

## Database Schema

### Messages (Cassandra)

```cql
-- Messages by conversation (for chat history)
CREATE TABLE messages (
  conversation_id UUID,
  message_id TIMEUUID,
  sender_id UUID,
  content TEXT,
  content_type TEXT,  -- text, image, video, file
  media_url TEXT,
  timestamp TIMESTAMP,
  PRIMARY KEY ((conversation_id), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Message status tracking
CREATE TABLE message_status (
  message_id TIMEUUID,
  recipient_id UUID,
  status TEXT,  -- sent, delivered, read
  timestamp TIMESTAMP,
  PRIMARY KEY ((message_id), recipient_id)
);
```

### User Sessions (Redis)

```javascript
// Online user sessions
HSET user:sessions user123 '{"serverId":"server-1","connectedAt":1640000000}'

// User presence
SET presence:user123 "online" EX 300  // 5 min TTL, heartbeat refreshes

// Typing indicators
SET typing:conv123:user456 1 EX 3  // 3 second TTL
```

### Conversations (PostgreSQL)

```sql
CREATE TABLE conversations (
  id UUID PRIMARY KEY,
  type VARCHAR(10),  -- 'direct' or 'group'
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

CREATE TABLE conversation_members (
  conversation_id UUID REFERENCES conversations(id),
  user_id UUID,
  joined_at TIMESTAMP,
  role VARCHAR(20),  -- 'admin', 'member'
  last_read_message_id UUID,
  PRIMARY KEY (conversation_id, user_id)
);

CREATE INDEX idx_user_conversations ON conversation_members(user_id);
```

---

## Group Chat

```javascript
// Group message fan-out
async function sendGroupMessage(groupId, senderId, content) {
  const messageId = generateMessageId();

  // Get all group members
  const members = await getGroupMembers(groupId);

  // Store message once
  await cassandra.execute(`
    INSERT INTO messages (conversation_id, message_id, sender_id, content, timestamp)
    VALUES (?, ?, ?, ?, ?)
  `, [groupId, messageId, senderId, content, Date.now()]);

  // Fan-out to all members (except sender)
  for (const member of members) {
    if (member.userId !== senderId) {
      await kafka.publish('messages', {
        key: member.userId,
        value: {
          id: messageId,
          conversationId: groupId,
          senderId,
          content,
          type: 'group'
        }
      });
    }
  }
}

// For very large groups, consider:
// 1. Lazy delivery (deliver on app open)
// 2. Separate group message queue
// 3. Pub/sub per group
```

---

## Read Receipts & Typing Indicators

```javascript
// Read receipt
async function markAsRead(userId, conversationId, messageId) {
  // Update last read pointer
  await db.query(`
    UPDATE conversation_members
    SET last_read_message_id = $1
    WHERE conversation_id = $2 AND user_id = $3
  `, [messageId, conversationId, userId]);

  // Notify message senders
  const messages = await getUnreadMessages(conversationId, userId, messageId);
  const senderIds = [...new Set(messages.map(m => m.senderId))];

  for (const senderId of senderIds) {
    sendToUser(senderId, {
      type: 'read_receipt',
      conversationId,
      readBy: userId,
      upToMessageId: messageId
    });
  }
}

// Typing indicator (ephemeral, not stored)
function sendTypingIndicator(userId, conversationId) {
  // Set short TTL in Redis
  redis.set(`typing:${conversationId}:${userId}`, 1, 'EX', 3);

  // Broadcast to conversation members
  const members = getConversationMembers(conversationId);
  for (const member of members) {
    if (member.userId !== userId) {
      sendToUser(member.userId, {
        type: 'typing',
        conversationId,
        userId
      });
    }
  }
}
```

---

## Push Notifications

```javascript
// For offline users
async function sendPushNotification(userId, message) {
  // Get user's push tokens
  const tokens = await db.query(
    'SELECT token, platform FROM push_tokens WHERE user_id = $1',
    [userId]
  );

  for (const { token, platform } of tokens) {
    if (platform === 'ios') {
      await apns.send(token, {
        alert: {
          title: message.senderName,
          body: message.content.substring(0, 100)
        },
        data: {
          conversationId: message.conversationId,
          messageId: message.id
        }
      });
    } else if (platform === 'android') {
      await fcm.send({
        token,
        notification: {
          title: message.senderName,
          body: message.content.substring(0, 100)
        },
        data: {
          conversationId: message.conversationId,
          messageId: message.id
        }
      });
    }
  }
}
```

---

## Message Sync (Multi-Device)

```javascript
// When device comes online, sync messages
async function syncMessages(userId, lastSyncTimestamp) {
  // Get all conversations for user
  const conversations = await getUserConversations(userId);

  const updates = [];
  for (const conv of conversations) {
    // Get messages after last sync
    const messages = await cassandra.execute(`
      SELECT * FROM messages
      WHERE conversation_id = ?
      AND timestamp > ?
      ORDER BY message_id ASC
      LIMIT 100
    `, [conv.id, lastSyncTimestamp]);

    if (messages.length > 0) {
      updates.push({
        conversationId: conv.id,
        messages: messages.rows
      });
    }
  }

  return updates;
}
```

---

## End-to-End Encryption

```javascript
// Signal Protocol overview
// 1. Each user has identity key pair (long-term)
// 2. Each device has signed pre-key + one-time pre-keys
// 3. Double Ratchet for forward secrecy

// Key exchange (simplified)
async function initiateSession(senderId, recipientId) {
  // Get recipient's keys from server
  const recipientKeys = await getPublicKeys(recipientId);

  // X3DH key agreement
  const sharedSecret = x3dh(
    senderIdentityKey,
    senderEphemeralKey,
    recipientKeys.identityKey,
    recipientKeys.signedPreKey,
    recipientKeys.oneTimePreKey
  );

  // Initialize ratchet
  const session = initializeDoubleRatchet(sharedSecret);
  saveSession(senderId, recipientId, session);
}

// Encrypt message
function encryptMessage(session, plaintext) {
  const { messageKey, newSession } = ratchetStep(session);
  const ciphertext = aesEncrypt(messageKey, plaintext);
  return { ciphertext, newSession };
}
```

---

## Scaling Considerations

### Connection Scaling

```
Each server: ~100K WebSocket connections
500M connections / 100K = 5,000 chat servers

Use consistent hashing to route users to servers
Session state in Redis for cross-server communication
```

### Message Storage

```
50B messages/day × 100 bytes = 5TB/day
5TB × 365 = 1.8PB/year

Strategies:
- Cassandra cluster with replication factor 3
- Time-based partitioning (messages older than 1 year archived)
- Media stored in S3, only URLs in DB
```

---

## Interview Discussion Points

### How to handle message ordering?

1. Client-generated monotonic message IDs per conversation
2. Server timestamp as tiebreaker
3. Kafka partition per conversation ensures ordering
4. Cassandra clustering key orders messages

### How to ensure message delivery?

1. Persistent queue (Kafka) - messages survive server restart
2. ACK from client before marking delivered
3. Push notification for offline users
4. Sync on reconnect

### How to handle user on multiple devices?

1. All devices register with server
2. Message fanned out to all devices
3. Read receipts synced across devices
4. Each device maintains its own encryption session
