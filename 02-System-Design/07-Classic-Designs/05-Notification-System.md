# Design: Notification System

## Requirements

### Functional Requirements
- Support multiple channels: Push, SMS, Email, In-app
- User notification preferences
- Notification templates
- Scheduling (immediate and delayed)
- Rate limiting per user
- Notification history and analytics

### Non-Functional Requirements
- High availability
- At-least-once delivery
- Scale: 10B notifications/day
- Latency: <1 second for real-time notifications
- Deduplication

### Capacity Estimation

```
Notifications: 10B/day = 115K notifications/second
Peak: 3x = 350K/second

Storage:
- Notification record: 500 bytes
- 10B × 500 bytes = 5TB/day
- Keep 30 days = 150TB
```

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Event Sources                                   │
│      (Order Service, User Service, Marketing, etc.)                      │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Notification Service                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   API       │  │  Validator  │  │  Priority   │  │  Template   │    │
│  │   Gateway   │→ │  & Filter   │→ │  Router     │→ │  Renderer   │    │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Message Queues (Kafka)                            │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│   │ Priority │  │ Push     │  │ Email    │  │ SMS      │               │
│   │ Queue    │  │ Queue    │  │ Queue    │  │ Queue    │               │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘               │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         ▼                       ▼                       ▼
┌────────────────┐      ┌────────────────┐      ┌────────────────┐
│  Push Worker   │      │  Email Worker  │      │   SMS Worker   │
│  (FCM, APNs)   │      │  (SendGrid)    │      │  (Twilio)      │
└────────────────┘      └────────────────┘      └────────────────┘
```

---

## API Design

```javascript
// Send notification
POST /api/v1/notifications
{
  "userId": "user-123",           // or userIds for bulk
  "type": "ORDER_SHIPPED",
  "channels": ["push", "email"],  // optional, use preferences
  "priority": "high",             // high, medium, low
  "data": {
    "orderId": "order-456",
    "trackingNumber": "1Z999AA10123456784"
  },
  "scheduleAt": null              // null for immediate
}

// Bulk notification
POST /api/v1/notifications/bulk
{
  "userIds": ["user-1", "user-2", ...],
  "type": "PROMOTION",
  "channels": ["push"],
  "data": {...}
}

// Get user preferences
GET /api/v1/users/:userId/notification-preferences
{
  "channels": {
    "push": { "enabled": true, "quiet_hours": {"start": "22:00", "end": "08:00"} },
    "email": { "enabled": true, "frequency": "instant" },
    "sms": { "enabled": false }
  },
  "categories": {
    "marketing": { "push": false, "email": true },
    "orders": { "push": true, "email": true, "sms": true },
    "social": { "push": true, "email": false }
  }
}
```

---

## Core Components

### Notification Service

```javascript
class NotificationService {
  async send(request) {
    // 1. Validate request
    this.validate(request);

    // 2. Get user preferences
    const preferences = await this.getPreferences(request.userId);

    // 3. Determine channels
    const channels = this.resolveChannels(request, preferences);

    // 4. Check rate limits
    if (await this.isRateLimited(request.userId)) {
      throw new RateLimitError();
    }

    // 5. Create notification record
    const notification = await this.createNotification(request);

    // 6. Route to channel queues
    for (const channel of channels) {
      await this.routeToChannel(channel, notification);
    }

    return notification;
  }

  resolveChannels(request, preferences) {
    const requestedChannels = request.channels || ['push', 'email'];
    const category = this.getCategory(request.type);

    return requestedChannels.filter(channel => {
      // Check if channel is enabled globally
      if (!preferences.channels[channel]?.enabled) return false;

      // Check category preferences
      if (preferences.categories[category]?.[channel] === false) return false;

      // Check quiet hours for push
      if (channel === 'push' && this.isQuietHours(preferences.channels.push)) {
        // Queue for later or skip
        return false;
      }

      return true;
    });
  }

  async routeToChannel(channel, notification) {
    const queue = this.getQueueForChannel(channel);
    const priority = notification.priority;

    await kafka.publish(`notifications.${channel}.${priority}`, {
      notificationId: notification.id,
      userId: notification.userId,
      channel,
      payload: notification
    });
  }
}
```

### Template Rendering

```javascript
// Template storage
const templates = {
  ORDER_SHIPPED: {
    push: {
      title: "Your order is on the way!",
      body: "Order #{{orderId}} has shipped. Track: {{trackingNumber}}"
    },
    email: {
      subject: "Your order #{{orderId}} has shipped",
      template: "order-shipped.html"
    },
    sms: {
      body: "Your order {{orderId}} shipped! Track at: {{trackingUrl}}"
    }
  }
};

class TemplateRenderer {
  render(type, channel, data) {
    const template = templates[type][channel];

    if (channel === 'email') {
      return this.renderHTML(template.template, data);
    }

    // Simple variable substitution for push/sms
    let result = template.body || template.title;
    for (const [key, value] of Object.entries(data)) {
      result = result.replace(new RegExp(`{{${key}}}`, 'g'), value);
    }

    return {
      ...template,
      body: result
    };
  }
}
```

---

## Channel Workers

### Push Notification Worker

```javascript
class PushWorker {
  async process(job) {
    const { userId, payload } = job;

    // Get user's device tokens
    const devices = await this.getDeviceTokens(userId);

    const results = await Promise.allSettled(
      devices.map(device => this.sendToDevice(device, payload))
    );

    // Handle failures
    for (let i = 0; i < results.length; i++) {
      if (results[i].status === 'rejected') {
        const error = results[i].reason;

        if (this.isInvalidToken(error)) {
          // Remove invalid token
          await this.removeDeviceToken(devices[i].token);
        } else if (this.isRetryable(error)) {
          // Retry later
          await this.scheduleRetry(job, devices[i]);
        }
      }
    }

    // Update notification status
    await this.updateStatus(payload.notificationId, 'delivered');
  }

  async sendToDevice(device, payload) {
    if (device.platform === 'ios') {
      return this.sendAPNs(device.token, payload);
    } else {
      return this.sendFCM(device.token, payload);
    }
  }

  async sendFCM(token, payload) {
    return firebase.messaging().send({
      token,
      notification: {
        title: payload.title,
        body: payload.body
      },
      data: payload.data,
      android: {
        priority: payload.priority === 'high' ? 'high' : 'normal',
        ttl: 3600 * 1000 // 1 hour
      }
    });
  }
}
```

### Email Worker

```javascript
class EmailWorker {
  async process(job) {
    const { userId, payload, notificationId } = job;

    // Get user email
    const user = await this.getUser(userId);

    // Render email template
    const html = await this.renderTemplate(payload.template, payload.data);

    // Send via SendGrid
    const result = await sendgrid.send({
      to: user.email,
      from: 'notifications@example.com',
      subject: payload.subject,
      html,
      trackingSettings: {
        clickTracking: { enable: true },
        openTracking: { enable: true }
      },
      customArgs: {
        notificationId
      }
    });

    await this.updateStatus(notificationId, 'delivered');

    return result;
  }
}
```

---

## Rate Limiting

```javascript
class NotificationRateLimiter {
  constructor(redis) {
    this.redis = redis;
  }

  async checkLimit(userId, channel, type) {
    const limits = {
      push: { perHour: 10, perDay: 50 },
      email: { perHour: 5, perDay: 20 },
      sms: { perHour: 3, perDay: 10 }
    };

    const limit = limits[channel];
    const hourKey = `ratelimit:${channel}:${userId}:hour`;
    const dayKey = `ratelimit:${channel}:${userId}:day`;

    const multi = this.redis.multi();
    multi.incr(hourKey);
    multi.expire(hourKey, 3600);
    multi.incr(dayKey);
    multi.expire(dayKey, 86400);

    const [hourCount, , dayCount] = await multi.exec();

    if (hourCount[1] > limit.perHour || dayCount[1] > limit.perDay) {
      return { allowed: false, retryAfter: 3600 };
    }

    return { allowed: true };
  }
}
```

---

## Deduplication

```javascript
class DeduplicationService {
  async isDuplicate(userId, type, data, windowSeconds = 60) {
    // Create hash of notification content
    const hash = crypto
      .createHash('sha256')
      .update(JSON.stringify({ userId, type, data }))
      .digest('hex');

    const key = `dedup:${hash}`;

    // Check if we've sent this recently
    const exists = await redis.exists(key);

    if (exists) {
      return true;
    }

    // Mark as sent with TTL
    await redis.setex(key, windowSeconds, '1');

    return false;
  }
}

// Usage in notification service
async send(request) {
  if (await this.dedup.isDuplicate(request.userId, request.type, request.data)) {
    logger.info('Duplicate notification suppressed', request);
    return { status: 'deduplicated' };
  }

  // Continue with normal flow...
}
```

---

## Scheduling

```javascript
// Scheduled notifications
class NotificationScheduler {
  async schedule(notification) {
    const scheduledAt = notification.scheduleAt;

    if (!scheduledAt) {
      // Immediate - send directly
      return this.notificationService.send(notification);
    }

    // Store in scheduled jobs table
    await db.query(`
      INSERT INTO scheduled_notifications
      (id, user_id, type, payload, scheduled_at, status)
      VALUES ($1, $2, $3, $4, $5, 'pending')
    `, [notification.id, notification.userId, notification.type,
        JSON.stringify(notification), scheduledAt]);

    return { status: 'scheduled', scheduledAt };
  }
}

// Scheduler worker (runs every minute)
class SchedulerWorker {
  async tick() {
    const dueNotifications = await db.query(`
      SELECT * FROM scheduled_notifications
      WHERE scheduled_at <= NOW()
      AND status = 'pending'
      LIMIT 1000
    `);

    for (const notification of dueNotifications.rows) {
      try {
        await this.notificationService.send(JSON.parse(notification.payload));
        await this.markComplete(notification.id);
      } catch (error) {
        await this.markFailed(notification.id, error);
      }
    }
  }
}
```

---

## Analytics & Tracking

```javascript
// Track notification events
class NotificationAnalytics {
  async trackEvent(notificationId, event, metadata = {}) {
    await kafka.publish('notification-events', {
      notificationId,
      event,  // 'sent', 'delivered', 'opened', 'clicked', 'failed'
      timestamp: Date.now(),
      metadata
    });
  }
}

// Webhook for email events (SendGrid)
app.post('/webhooks/email', (req, res) => {
  for (const event of req.body) {
    analytics.trackEvent(
      event.customArgs.notificationId,
      event.event,  // 'delivered', 'open', 'click', 'bounce'
      { email: event.email }
    );
  }
  res.sendStatus(200);
});

// Aggregated analytics
async function getNotificationStats(timeRange) {
  return db.query(`
    SELECT
      type,
      channel,
      COUNT(*) as total,
      SUM(CASE WHEN status = 'delivered' THEN 1 ELSE 0 END) as delivered,
      SUM(CASE WHEN status = 'opened' THEN 1 ELSE 0 END) as opened,
      SUM(CASE WHEN status = 'clicked' THEN 1 ELSE 0 END) as clicked
    FROM notification_events
    WHERE timestamp > $1
    GROUP BY type, channel
  `, [timeRange.start]);
}
```

---

## Database Schema

```sql
-- Notifications
CREATE TABLE notifications (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  type VARCHAR(50) NOT NULL,
  channel VARCHAR(20) NOT NULL,
  priority VARCHAR(10) DEFAULT 'medium',
  payload JSONB NOT NULL,
  status VARCHAR(20) DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT NOW(),
  sent_at TIMESTAMP,
  delivered_at TIMESTAMP
);

CREATE INDEX idx_notifications_user ON notifications(user_id, created_at DESC);
CREATE INDEX idx_notifications_status ON notifications(status) WHERE status = 'pending';

-- User preferences
CREATE TABLE notification_preferences (
  user_id UUID PRIMARY KEY,
  preferences JSONB NOT NULL,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Device tokens
CREATE TABLE device_tokens (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  token VARCHAR(255) NOT NULL UNIQUE,
  platform VARCHAR(20) NOT NULL,
  app_version VARCHAR(20),
  created_at TIMESTAMP DEFAULT NOW(),
  last_used_at TIMESTAMP
);

CREATE INDEX idx_device_tokens_user ON device_tokens(user_id);
```

---

## Interview Discussion Points

### How to handle notification storms?

1. Rate limiting per user and globally
2. Queue prioritization (critical > normal > marketing)
3. Backpressure handling (shed low-priority when overloaded)
4. Batch similar notifications

### How to ensure delivery?

1. Persistent queues (Kafka)
2. Retry with exponential backoff
3. Dead letter queue for failed notifications
4. Multiple delivery channels as fallback

### How to handle user preferences?

1. Global enable/disable per channel
2. Category-based preferences
3. Quiet hours
4. Frequency controls (instant, daily digest, weekly)
