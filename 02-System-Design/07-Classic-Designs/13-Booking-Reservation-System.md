# Design: Booking/Reservation System (Hotel/Flight/Ticket)

## Requirements

### Functional Requirements
- Search available inventory
- Create reservations with time-limited holds
- Handle payments
- Manage cancellations and modifications
- Prevent double-booking
- Waitlist management

### Non-Functional Requirements
- Strong consistency (no overbooking)
- High availability (99.99%)
- Low latency (<200ms for availability check)
- Handle flash sales/high concurrency
- Scale: 10K bookings/second peak

### Capacity Estimation

```
Daily bookings: 1M
Peak QPS: 10K bookings/second (flash sales)
Availability checks: 100K/second

Inventory:
- Hotels: 1M properties × 100 rooms = 100M room-nights
- Flights: 100K flights × 200 seats = 20M seats
- Events: 10K events × 10K seats = 100M tickets
```

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Clients (Web/Mobile)                            │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────┐
│                            API Gateway                                   │
│                   (Auth, Rate Limiting, Routing)                         │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────────────────────┐
         ▼                       ▼                       ▼               │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐       │
│  Search         │    │  Booking        │    │  Payment        │       │
│  Service        │    │  Service        │    │  Service        │       │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘       │
         │                      │                      │                 │
         ▼                      ▼                      │                 │
┌─────────────────┐    ┌─────────────────┐            │                 │
│  Elasticsearch  │    │  Inventory      │◄───────────┘                 │
│  (Catalog)      │    │  Service        │                              │
└─────────────────┘    └────────┬────────┘                              │
                                │                                        │
                       ┌────────┴────────┐                              │
                       ▼                 ▼                              │
              ┌─────────────────┐ ┌─────────────────┐                   │
              │  Inventory DB   │ │  Distributed    │                   │
              │  (PostgreSQL)   │ │  Lock (Redis)   │                   │
              └─────────────────┘ └─────────────────┘                   │
```

---

## Inventory Management

### Database Schema

```sql
-- Properties (Hotels)
CREATE TABLE properties (
  id UUID PRIMARY KEY,
  name VARCHAR(255),
  location GEOGRAPHY(POINT, 4326),
  address TEXT,
  rating DECIMAL(2,1),
  amenities TEXT[],
  created_at TIMESTAMP DEFAULT NOW()
);

-- Room Types
CREATE TABLE room_types (
  id UUID PRIMARY KEY,
  property_id UUID REFERENCES properties(id),
  name VARCHAR(100),
  description TEXT,
  base_price DECIMAL(10,2),
  max_occupancy INT,
  amenities TEXT[]
);

-- Inventory (available units per date)
CREATE TABLE inventory (
  id UUID PRIMARY KEY,
  room_type_id UUID REFERENCES room_types(id),
  date DATE NOT NULL,
  total_rooms INT NOT NULL,
  available_rooms INT NOT NULL,
  price DECIMAL(10,2),
  UNIQUE(room_type_id, date),
  CHECK(available_rooms >= 0),
  CHECK(available_rooms <= total_rooms)
);

-- Create index for fast lookups
CREATE INDEX idx_inventory_date ON inventory(room_type_id, date);

-- Reservations
CREATE TABLE reservations (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  room_type_id UUID REFERENCES room_types(id),
  check_in DATE NOT NULL,
  check_out DATE NOT NULL,
  guests INT,
  total_price DECIMAL(10,2),
  status VARCHAR(20) DEFAULT 'pending',  -- pending, confirmed, cancelled
  held_until TIMESTAMP,  -- For temporary holds
  created_at TIMESTAMP DEFAULT NOW(),
  confirmed_at TIMESTAMP,
  cancelled_at TIMESTAMP
);

CREATE INDEX idx_reservations_status ON reservations(status, held_until);
CREATE INDEX idx_reservations_user ON reservations(user_id);
```

### Availability Check

```javascript
class InventoryService {
  async checkAvailability(roomTypeId, checkIn, checkOut, roomsNeeded = 1) {
    const nights = this.getNightsBetween(checkIn, checkOut);

    const availability = await db.query(`
      SELECT date, available_rooms, price
      FROM inventory
      WHERE room_type_id = $1
        AND date >= $2
        AND date < $3
      ORDER BY date
    `, [roomTypeId, checkIn, checkOut]);

    // Check if all nights have availability
    if (availability.length !== nights) {
      return { available: false, reason: 'INCOMPLETE_INVENTORY' };
    }

    const minAvailable = Math.min(...availability.map(a => a.available_rooms));
    if (minAvailable < roomsNeeded) {
      return {
        available: false,
        reason: 'INSUFFICIENT_INVENTORY',
        availableRooms: minAvailable
      };
    }

    // Calculate total price
    const totalPrice = availability.reduce((sum, a) => sum + parseFloat(a.price), 0);

    return {
      available: true,
      roomsAvailable: minAvailable,
      totalPrice,
      priceBreakdown: availability.map(a => ({
        date: a.date,
        price: a.price
      }))
    };
  }

  getNightsBetween(checkIn, checkOut) {
    const msPerDay = 24 * 60 * 60 * 1000;
    return (new Date(checkOut) - new Date(checkIn)) / msPerDay;
  }
}
```

---

## Booking Flow

### Two-Phase Booking with Temporary Hold

```javascript
class BookingService {
  constructor(inventoryService, lockService, paymentService) {
    this.inventoryService = inventoryService;
    this.lockService = lockService;
    this.paymentService = paymentService;
    this.holdDuration = 10 * 60 * 1000;  // 10 minutes
  }

  // Phase 1: Create temporary hold
  async createHold(userId, roomTypeId, checkIn, checkOut, roomsNeeded) {
    // Check availability first
    const availability = await this.inventoryService.checkAvailability(
      roomTypeId, checkIn, checkOut, roomsNeeded
    );

    if (!availability.available) {
      throw new NotAvailableError(availability.reason);
    }

    // Acquire distributed lock for this inventory
    const lockKey = `lock:${roomTypeId}:${checkIn}:${checkOut}`;
    const lock = await this.lockService.acquire(lockKey, 5000);

    try {
      // Double-check availability after acquiring lock
      const recheck = await this.inventoryService.checkAvailability(
        roomTypeId, checkIn, checkOut, roomsNeeded
      );

      if (!recheck.available) {
        throw new NotAvailableError('No longer available');
      }

      // Create reservation with hold
      const reservation = await this.createReservation({
        userId,
        roomTypeId,
        checkIn,
        checkOut,
        roomsNeeded,
        totalPrice: recheck.totalPrice,
        status: 'pending',
        heldUntil: new Date(Date.now() + this.holdDuration)
      });

      // Decrement inventory
      await this.inventoryService.decrementInventory(
        roomTypeId, checkIn, checkOut, roomsNeeded
      );

      return {
        reservationId: reservation.id,
        totalPrice: recheck.totalPrice,
        heldUntil: reservation.heldUntil,
        paymentDeadline: reservation.heldUntil
      };

    } finally {
      await this.lockService.release(lock);
    }
  }

  // Phase 2: Confirm with payment
  async confirmBooking(reservationId, paymentDetails) {
    const reservation = await this.getReservation(reservationId);

    // Check if hold is still valid
    if (reservation.status !== 'pending') {
      throw new InvalidStateError('Reservation is not in pending state');
    }

    if (new Date() > reservation.heldUntil) {
      // Hold expired - release inventory and fail
      await this.releaseHold(reservationId);
      throw new HoldExpiredError('Hold has expired');
    }

    // Process payment
    const payment = await this.paymentService.charge({
      amount: reservation.totalPrice,
      userId: reservation.userId,
      reservationId,
      ...paymentDetails
    });

    if (!payment.success) {
      throw new PaymentFailedError(payment.error);
    }

    // Confirm reservation
    await db.query(`
      UPDATE reservations
      SET status = 'confirmed',
          confirmed_at = NOW(),
          held_until = NULL
      WHERE id = $1
    `, [reservationId]);

    // Send confirmation email
    await this.notificationService.sendConfirmation(reservation);

    return {
      confirmed: true,
      confirmationNumber: this.generateConfirmationNumber(reservationId),
      reservation
    };
  }

  // Release expired holds (background job)
  async releaseExpiredHolds() {
    const expired = await db.query(`
      SELECT id, room_type_id, check_in, check_out, rooms_needed
      FROM reservations
      WHERE status = 'pending'
        AND held_until < NOW()
      FOR UPDATE SKIP LOCKED
      LIMIT 100
    `);

    for (const reservation of expired) {
      await this.releaseHold(reservation.id);
    }
  }

  async releaseHold(reservationId) {
    const reservation = await this.getReservation(reservationId);

    // Restore inventory
    await this.inventoryService.incrementInventory(
      reservation.roomTypeId,
      reservation.checkIn,
      reservation.checkOut,
      reservation.roomsNeeded
    );

    // Update reservation status
    await db.query(`
      UPDATE reservations
      SET status = 'expired',
          held_until = NULL
      WHERE id = $1
    `, [reservationId]);
  }
}
```

### Inventory Operations (Atomic)

```javascript
class InventoryService {
  async decrementInventory(roomTypeId, checkIn, checkOut, count) {
    // Atomic decrement with check
    const result = await db.query(`
      UPDATE inventory
      SET available_rooms = available_rooms - $4
      WHERE room_type_id = $1
        AND date >= $2
        AND date < $3
        AND available_rooms >= $4
      RETURNING *
    `, [roomTypeId, checkIn, checkOut, count]);

    const nights = this.getNightsBetween(checkIn, checkOut);

    if (result.rowCount !== nights) {
      // Rollback - restore what was decremented
      await this.incrementInventory(roomTypeId, checkIn, checkOut, count);
      throw new ConcurrencyError('Failed to reserve all nights');
    }

    return result.rows;
  }

  async incrementInventory(roomTypeId, checkIn, checkOut, count) {
    await db.query(`
      UPDATE inventory
      SET available_rooms = available_rooms + $4
      WHERE room_type_id = $1
        AND date >= $2
        AND date < $3
    `, [roomTypeId, checkIn, checkOut, count]);
  }
}
```

---

## Distributed Locking

```javascript
class RedisLockService {
  constructor(redis) {
    this.redis = redis;
  }

  async acquire(key, ttlMs, retryOptions = {}) {
    const lockValue = generateId();
    const maxRetries = retryOptions.maxRetries || 10;
    const retryDelay = retryOptions.retryDelay || 100;

    for (let attempt = 0; attempt < maxRetries; attempt++) {
      // Try to acquire lock
      const acquired = await this.redis.set(
        key,
        lockValue,
        'NX',
        'PX',
        ttlMs
      );

      if (acquired) {
        return { key, value: lockValue, ttl: ttlMs };
      }

      // Wait before retry
      await this.delay(retryDelay);
    }

    throw new LockAcquisitionError(`Failed to acquire lock: ${key}`);
  }

  async release(lock) {
    // Only release if we own the lock (using Lua script for atomicity)
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;

    await this.redis.eval(script, 1, lock.key, lock.value);
  }

  async extend(lock, additionalMs) {
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("pexpire", KEYS[1], ARGV[2])
      else
        return 0
      end
    `;

    const result = await this.redis.eval(
      script, 1, lock.key, lock.value, additionalMs
    );

    if (result === 0) {
      throw new LockLostError('Lock no longer held');
    }
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

---

## Handling High Concurrency

### Optimistic Locking

```javascript
async function bookWithOptimisticLock(roomTypeId, checkIn, checkOut) {
  const maxRetries = 3;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    // Read current version
    const inventory = await db.query(`
      SELECT id, available_rooms, version
      FROM inventory
      WHERE room_type_id = $1 AND date = $2
    `, [roomTypeId, checkIn]);

    if (inventory.available_rooms < 1) {
      throw new NotAvailableError();
    }

    // Try to update with version check
    const result = await db.query(`
      UPDATE inventory
      SET available_rooms = available_rooms - 1,
          version = version + 1
      WHERE id = $1 AND version = $2
      RETURNING *
    `, [inventory.id, inventory.version]);

    if (result.rowCount === 1) {
      return result.rows[0];  // Success
    }

    // Version mismatch - retry
    await this.delay(Math.random() * 100);
  }

  throw new ConcurrencyError('Failed after retries');
}
```

### Queue-Based Booking (Flash Sales)

```javascript
class FlashSaleBookingService {
  constructor(kafka, redis) {
    this.kafka = kafka;
    this.redis = redis;
  }

  // High-volume entry point - just queue the request
  async requestBooking(userId, eventId, ticketType, quantity) {
    const requestId = generateId();

    // Add to processing queue
    await this.kafka.send({
      topic: 'booking-requests',
      messages: [{
        key: eventId,  // Same event goes to same partition (ordering)
        value: JSON.stringify({
          requestId,
          userId,
          eventId,
          ticketType,
          quantity,
          timestamp: Date.now()
        })
      }]
    });

    // Return immediately with request ID
    return {
      requestId,
      status: 'queued',
      message: 'Your request is being processed'
    };
  }

  // Consumer processes requests sequentially per event
  async processBookingRequest(request) {
    const { requestId, userId, eventId, ticketType, quantity } = request;

    try {
      // Check availability
      const available = await this.checkAvailability(eventId, ticketType);

      if (available < quantity) {
        await this.updateRequestStatus(requestId, 'failed', 'SOLD_OUT');
        return;
      }

      // Reserve tickets
      await this.reserveTickets(eventId, ticketType, quantity);

      // Create booking
      const booking = await this.createBooking({
        userId,
        eventId,
        ticketType,
        quantity,
        status: 'pending_payment',
        expiresAt: Date.now() + 10 * 60 * 1000  // 10 min to pay
      });

      await this.updateRequestStatus(requestId, 'success', booking.id);

      // Notify user
      await this.notifyUser(userId, {
        type: 'BOOKING_CREATED',
        bookingId: booking.id,
        paymentDeadline: booking.expiresAt
      });

    } catch (error) {
      await this.updateRequestStatus(requestId, 'failed', error.message);
    }
  }

  async checkRequestStatus(requestId) {
    return await this.redis.hgetall(`booking_request:${requestId}`);
  }
}
```

---

## Cancellation & Refunds

```javascript
class CancellationService {
  async cancelReservation(reservationId, userId) {
    const reservation = await this.getReservation(reservationId);

    // Validate ownership
    if (reservation.userId !== userId) {
      throw new ForbiddenError();
    }

    // Check cancellation policy
    const policy = await this.getCancellationPolicy(reservation);
    const refundAmount = this.calculateRefund(reservation, policy);

    await db.transaction(async (trx) => {
      // Update reservation status
      await trx.query(`
        UPDATE reservations
        SET status = 'cancelled',
            cancelled_at = NOW()
        WHERE id = $1
      `, [reservationId]);

      // Restore inventory
      await this.inventoryService.incrementInventory(
        reservation.roomTypeId,
        reservation.checkIn,
        reservation.checkOut,
        reservation.roomsNeeded,
        trx
      );

      // Process refund if applicable
      if (refundAmount > 0) {
        await this.paymentService.refund({
          reservationId,
          amount: refundAmount,
          reason: 'CUSTOMER_CANCELLATION'
        });
      }
    });

    // Notify waitlist
    await this.notifyWaitlist(reservation);

    return {
      cancelled: true,
      refundAmount,
      policy: policy.name
    };
  }

  calculateRefund(reservation, policy) {
    const now = new Date();
    const checkIn = new Date(reservation.checkIn);
    const daysUntilCheckIn = (checkIn - now) / (24 * 60 * 60 * 1000);

    for (const rule of policy.rules) {
      if (daysUntilCheckIn >= rule.daysBeforeCheckIn) {
        return reservation.totalPrice * (rule.refundPercentage / 100);
      }
    }

    return 0;  // No refund
  }
}
```

---

## Waitlist Management

```javascript
class WaitlistService {
  async addToWaitlist(userId, roomTypeId, checkIn, checkOut, priority = 'normal') {
    const position = await this.getNextPosition(roomTypeId, checkIn, checkOut);

    const entry = await db.query(`
      INSERT INTO waitlist (id, user_id, room_type_id, check_in, check_out, priority, position)
      VALUES ($1, $2, $3, $4, $5, $6, $7)
      RETURNING *
    `, [generateId(), userId, roomTypeId, checkIn, checkOut, priority, position]);

    return {
      waitlistId: entry.id,
      position,
      estimatedWait: this.estimateWait(position)
    };
  }

  async processWaitlist(roomTypeId, checkIn, checkOut) {
    // Get next in line
    const next = await db.query(`
      SELECT * FROM waitlist
      WHERE room_type_id = $1
        AND check_in = $2
        AND check_out = $3
        AND status = 'waiting'
      ORDER BY priority DESC, position ASC
      LIMIT 1
      FOR UPDATE SKIP LOCKED
    `, [roomTypeId, checkIn, checkOut]);

    if (!next) return null;

    // Check if inventory is available
    const available = await this.inventoryService.checkAvailability(
      roomTypeId, checkIn, checkOut, 1
    );

    if (!available.available) return null;

    // Notify user and give them limited time to book
    await db.query(`
      UPDATE waitlist
      SET status = 'offered',
          offered_at = NOW(),
          expires_at = NOW() + INTERVAL '15 minutes'
      WHERE id = $1
    `, [next.id]);

    await this.notificationService.send({
      userId: next.userId,
      type: 'WAITLIST_AVAILABLE',
      roomTypeId,
      checkIn,
      checkOut,
      expiresIn: '15 minutes'
    });

    return next;
  }
}
```

---

## Interview Discussion Points

### How to prevent double-booking?

1. **Database constraints** - CHECK(available >= 0)
2. **Pessimistic locking** - SELECT FOR UPDATE
3. **Optimistic locking** - Version numbers
4. **Distributed locks** - Redis for cross-service
5. **Two-phase commit** - Hold then confirm

### How to handle flash sales?

1. **Queue requests** - Serialize processing
2. **Rate limiting** - Protect backend
3. **Pre-computed inventory** - Cache availability
4. **Lottery system** - Random selection from valid requests
5. **Virtual waiting room** - Manage traffic spikes

### How to ensure consistency across services?

1. **Saga pattern** - Compensating transactions
2. **Two-phase commit** - Distributed transaction
3. **Event sourcing** - Rebuild state from events
4. **Idempotency** - Handle retries safely
5. **Outbox pattern** - Reliable event publishing

### How to scale inventory checks?

1. **Caching** - Redis for hot inventory
2. **Read replicas** - Separate read/write
3. **Sharding** - By property/region
4. **Pre-aggregation** - Materialized availability
5. **Eventual consistency** - For availability display
