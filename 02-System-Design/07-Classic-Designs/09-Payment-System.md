# Design: Payment System

## Requirements

### Functional Requirements
- Process payments (credit card, debit, digital wallets)
- Handle refunds and chargebacks
- Support multiple currencies
- Payment status tracking
- Recurring payments/subscriptions
- Split payments (marketplace)
- Fraud detection

### Non-Functional Requirements
- High availability (99.99%)
- Strong consistency (no double charges)
- PCI DSS compliance
- Idempotency (exactly-once processing)
- Low latency (<2 seconds for payment)
- Audit trail for all transactions

### Capacity Estimation

```
Transactions: 10M/day = ~115 TPS
Peak: 3x = ~350 TPS
Transaction record: ~1KB
Daily storage: 10GB
Monthly storage: 300GB

Revenue processed: $1B/day
Average transaction: $100
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
│                (Rate limiting, Auth, SSL termination)                    │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────┐
│                         Payment Service                                  │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│   │  Payment    │  │  Validation │  │  Risk       │  │  Routing    │   │
│   │  API        │→ │  Service    │→ │  Engine     │→ │  Service    │   │
│   └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Payment         │    │ Stripe          │    │ PayPal          │
│ Processor       │    │ Gateway         │    │ Gateway         │
│ (Primary)       │    │ (Backup)        │    │ (Alternative)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Data Layer                                       │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│   │ Transactions│  │  Ledger     │  │  Wallet     │  │  Audit Log  │   │
│   │    DB       │  │    DB       │  │    DB       │  │  (Append)   │   │
│   └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Payment Flow

### Standard Payment Flow

```
┌────────┐     ┌─────────────┐     ┌───────────┐     ┌──────────────┐
│ Client │────▶│ Payment API │────▶│ Validation│────▶│ Risk Engine  │
└────────┘     └─────────────┘     └───────────┘     └──────┬───────┘
                                                            │
    ┌───────────────────────────────────────────────────────┘
    ▼
┌───────────────┐     ┌──────────────┐     ┌──────────────────┐
│ Create Pending│────▶│ Call Payment │────▶│ Update Status    │
│ Transaction   │     │ Gateway      │     │ (Success/Failed) │
└───────────────┘     └──────────────┘     └──────────────────┘
                                                     │
                            ┌────────────────────────┤
                            ▼                        ▼
                    ┌──────────────┐        ┌──────────────┐
                    │ Notify       │        │ Update       │
                    │ Merchant     │        │ Ledger       │
                    └──────────────┘        └──────────────┘
```

### Implementation

```javascript
class PaymentService {
  async processPayment(paymentRequest) {
    const { amount, currency, paymentMethod, merchantId, idempotencyKey } = paymentRequest;

    // 1. Idempotency check
    const existing = await this.checkIdempotency(idempotencyKey);
    if (existing) {
      return existing;
    }

    // 2. Validate payment request
    await this.validatePayment(paymentRequest);

    // 3. Risk assessment
    const riskScore = await this.riskEngine.assess(paymentRequest);
    if (riskScore > RISK_THRESHOLD) {
      return this.createDeclinedPayment(paymentRequest, 'HIGH_RISK');
    }

    // 4. Create pending transaction
    const transaction = await this.createTransaction({
      id: generateId(),
      amount,
      currency,
      merchantId,
      status: 'PENDING',
      idempotencyKey,
      createdAt: new Date()
    });

    try {
      // 5. Select payment gateway
      const gateway = this.selectGateway(paymentMethod, amount);

      // 6. Process with gateway
      const gatewayResponse = await this.processWithGateway(
        gateway,
        paymentRequest,
        transaction.id
      );

      // 7. Update transaction status
      const updatedTransaction = await this.updateTransaction(transaction.id, {
        status: gatewayResponse.success ? 'COMPLETED' : 'FAILED',
        gatewayReference: gatewayResponse.reference,
        processedAt: new Date()
      });

      // 8. Update ledger
      if (gatewayResponse.success) {
        await this.ledger.credit(merchantId, amount, currency, transaction.id);
      }

      // 9. Store idempotency result
      await this.storeIdempotencyResult(idempotencyKey, updatedTransaction);

      // 10. Emit event
      await this.eventBus.emit('payment.processed', updatedTransaction);

      return updatedTransaction;

    } catch (error) {
      // Handle failure
      await this.updateTransaction(transaction.id, {
        status: 'FAILED',
        errorCode: error.code,
        errorMessage: error.message
      });

      throw error;
    }
  }

  async processWithGateway(gateway, request, transactionId) {
    const startTime = Date.now();

    try {
      const response = await withTimeout(
        gateway.charge({
          amount: request.amount,
          currency: request.currency,
          paymentMethod: request.paymentMethod,
          metadata: { transactionId }
        }),
        GATEWAY_TIMEOUT_MS
      );

      // Log gateway interaction
      await this.logGatewayCall({
        gateway: gateway.name,
        transactionId,
        duration: Date.now() - startTime,
        success: true,
        response
      });

      return response;

    } catch (error) {
      await this.logGatewayCall({
        gateway: gateway.name,
        transactionId,
        duration: Date.now() - startTime,
        success: false,
        error: error.message
      });

      // Try backup gateway if available
      if (this.shouldFailover(error)) {
        return this.processWithGateway(
          this.getBackupGateway(gateway),
          request,
          transactionId
        );
      }

      throw error;
    }
  }
}
```

---

## Idempotency

### Preventing Double Charges

```javascript
class IdempotencyService {
  constructor(redis, db) {
    this.redis = redis;
    this.db = db;
  }

  async checkAndLock(idempotencyKey, ttlSeconds = 86400) {
    // Atomic check and lock
    const lockKey = `idempotency:lock:${idempotencyKey}`;
    const resultKey = `idempotency:result:${idempotencyKey}`;

    // Check for existing result
    const existingResult = await this.redis.get(resultKey);
    if (existingResult) {
      return { exists: true, result: JSON.parse(existingResult) };
    }

    // Try to acquire lock
    const acquired = await this.redis.set(
      lockKey,
      Date.now(),
      'NX',
      'EX',
      60  // Lock expires in 60 seconds
    );

    if (!acquired) {
      // Another request is processing
      throw new ConflictError('Payment is being processed');
    }

    return { exists: false, lockKey };
  }

  async storeResult(idempotencyKey, result, ttlSeconds = 86400) {
    const resultKey = `idempotency:result:${idempotencyKey}`;
    const lockKey = `idempotency:lock:${idempotencyKey}`;

    // Store result and release lock atomically
    const multi = this.redis.multi();
    multi.set(resultKey, JSON.stringify(result), 'EX', ttlSeconds);
    multi.del(lockKey);
    await multi.exec();

    // Also persist to database for durability
    await this.db.idempotencyKeys.upsert({
      key: idempotencyKey,
      result: result,
      createdAt: new Date()
    });
  }

  async releaseLock(lockKey) {
    await this.redis.del(lockKey);
  }
}

// Usage in payment handler
app.post('/api/payments', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'];

  if (!idempotencyKey) {
    return res.status(400).json({ error: 'Idempotency-Key header required' });
  }

  const { exists, result, lockKey } = await idempotencyService.checkAndLock(idempotencyKey);

  if (exists) {
    // Return cached result
    return res.status(200).json(result);
  }

  try {
    const payment = await paymentService.processPayment(req.body);
    await idempotencyService.storeResult(idempotencyKey, payment);
    return res.status(201).json(payment);
  } catch (error) {
    await idempotencyService.releaseLock(lockKey);
    throw error;
  }
});
```

---

## Double-Entry Ledger

### Ledger Design

```javascript
// Every transaction creates balanced entries (debits = credits)
class Ledger {
  async recordTransaction(transaction) {
    const entries = this.createEntries(transaction);

    // Validate balance
    const totalDebits = entries.filter(e => e.type === 'DEBIT').reduce((sum, e) => sum + e.amount, 0);
    const totalCredits = entries.filter(e => e.type === 'CREDIT').reduce((sum, e) => sum + e.amount, 0);

    if (totalDebits !== totalCredits) {
      throw new Error('Unbalanced transaction');
    }

    // Insert all entries atomically
    await this.db.transaction(async (trx) => {
      for (const entry of entries) {
        await trx.insert('ledger_entries', entry);
      }

      // Update account balances
      for (const entry of entries) {
        const delta = entry.type === 'DEBIT' ? -entry.amount : entry.amount;
        await trx.raw(`
          UPDATE accounts
          SET balance = balance + ?
          WHERE id = ?
        `, [delta, entry.accountId]);
      }
    });

    return entries;
  }

  createEntries(transaction) {
    switch (transaction.type) {
      case 'PAYMENT':
        return [
          // Debit customer's payment source (external)
          { accountId: 'EXTERNAL_PAYMENTS', type: 'DEBIT', amount: transaction.amount },
          // Credit merchant's account
          { accountId: transaction.merchantAccountId, type: 'CREDIT', amount: transaction.netAmount },
          // Credit platform fee account
          { accountId: 'PLATFORM_FEES', type: 'CREDIT', amount: transaction.platformFee }
        ];

      case 'REFUND':
        return [
          // Debit merchant's account
          { accountId: transaction.merchantAccountId, type: 'DEBIT', amount: transaction.amount },
          // Credit customer (external)
          { accountId: 'EXTERNAL_REFUNDS', type: 'CREDIT', amount: transaction.amount }
        ];

      case 'PAYOUT':
        return [
          // Debit merchant's platform balance
          { accountId: transaction.merchantAccountId, type: 'DEBIT', amount: transaction.amount },
          // Credit external bank transfer
          { accountId: 'EXTERNAL_PAYOUTS', type: 'CREDIT', amount: transaction.amount }
        ];

      default:
        throw new Error(`Unknown transaction type: ${transaction.type}`);
    }
  }

  async getBalance(accountId) {
    const result = await this.db.query(`
      SELECT
        SUM(CASE WHEN type = 'CREDIT' THEN amount ELSE 0 END) -
        SUM(CASE WHEN type = 'DEBIT' THEN amount ELSE 0 END) as balance
      FROM ledger_entries
      WHERE account_id = ?
    `, [accountId]);

    return result.balance;
  }
}
```

### Database Schema

```sql
-- Accounts
CREATE TABLE accounts (
  id UUID PRIMARY KEY,
  type VARCHAR(20) NOT NULL,  -- MERCHANT, PLATFORM, EXTERNAL
  currency VARCHAR(3) NOT NULL,
  balance DECIMAL(20, 2) DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Ledger entries (immutable)
CREATE TABLE ledger_entries (
  id UUID PRIMARY KEY,
  transaction_id UUID NOT NULL,
  account_id UUID NOT NULL REFERENCES accounts(id),
  type VARCHAR(10) NOT NULL,  -- DEBIT, CREDIT
  amount DECIMAL(20, 2) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_ledger_account ON ledger_entries(account_id, created_at);
CREATE INDEX idx_ledger_transaction ON ledger_entries(transaction_id);

-- Transactions
CREATE TABLE transactions (
  id UUID PRIMARY KEY,
  idempotency_key VARCHAR(255) UNIQUE,
  merchant_id UUID NOT NULL,
  amount DECIMAL(20, 2) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  status VARCHAR(20) NOT NULL,
  payment_method_type VARCHAR(20),
  gateway VARCHAR(50),
  gateway_reference VARCHAR(255),
  error_code VARCHAR(50),
  error_message TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  processed_at TIMESTAMP
);

CREATE INDEX idx_transactions_merchant ON transactions(merchant_id, created_at DESC);
CREATE INDEX idx_transactions_status ON transactions(status) WHERE status = 'PENDING';
```

---

## Refunds

```javascript
class RefundService {
  async processRefund(refundRequest) {
    const { transactionId, amount, reason, idempotencyKey } = refundRequest;

    // Idempotency check
    const existing = await this.checkIdempotency(idempotencyKey);
    if (existing) return existing;

    // Get original transaction
    const originalTransaction = await this.getTransaction(transactionId);

    // Validate refund
    await this.validateRefund(originalTransaction, amount);

    // Calculate refundable amount
    const previousRefunds = await this.getRefundsForTransaction(transactionId);
    const totalRefunded = previousRefunds.reduce((sum, r) => sum + r.amount, 0);
    const refundableAmount = originalTransaction.amount - totalRefunded;

    if (amount > refundableAmount) {
      throw new ValidationError('Refund amount exceeds refundable balance');
    }

    // Create refund record
    const refund = await this.createRefund({
      id: generateId(),
      transactionId,
      amount,
      currency: originalTransaction.currency,
      reason,
      status: 'PENDING'
    });

    try {
      // Process with gateway
      const gatewayResponse = await this.gateway.refund({
        originalReference: originalTransaction.gatewayReference,
        amount,
        currency: originalTransaction.currency
      });

      // Update refund status
      await this.updateRefund(refund.id, {
        status: 'COMPLETED',
        gatewayReference: gatewayResponse.reference
      });

      // Update ledger
      await this.ledger.recordTransaction({
        type: 'REFUND',
        merchantAccountId: originalTransaction.merchantAccountId,
        amount,
        refundId: refund.id
      });

      // Update original transaction
      if (totalRefunded + amount === originalTransaction.amount) {
        await this.updateTransaction(transactionId, { status: 'FULLY_REFUNDED' });
      } else {
        await this.updateTransaction(transactionId, { status: 'PARTIALLY_REFUNDED' });
      }

      return refund;

    } catch (error) {
      await this.updateRefund(refund.id, {
        status: 'FAILED',
        errorMessage: error.message
      });
      throw error;
    }
  }
}
```

---

## Fraud Detection

```javascript
class FraudDetectionEngine {
  async assess(paymentRequest) {
    const signals = await Promise.all([
      this.checkVelocity(paymentRequest),
      this.checkDeviceFingerprint(paymentRequest),
      this.checkGeoLocation(paymentRequest),
      this.checkCardBehavior(paymentRequest),
      this.checkMLModel(paymentRequest)
    ]);

    return this.calculateRiskScore(signals);
  }

  async checkVelocity(request) {
    const { customerId, amount } = request;
    const timeWindow = 3600;  // 1 hour

    // Count recent transactions
    const recentTxCount = await redis.get(`velocity:count:${customerId}`);
    const recentTxAmount = await redis.get(`velocity:amount:${customerId}`);

    // Update counters
    await redis.multi()
      .incr(`velocity:count:${customerId}`)
      .incrbyfloat(`velocity:amount:${customerId}`, amount)
      .expire(`velocity:count:${customerId}`, timeWindow)
      .expire(`velocity:amount:${customerId}`, timeWindow)
      .exec();

    return {
      signal: 'velocity',
      score: this.calculateVelocityScore(recentTxCount, recentTxAmount, amount)
    };
  }

  async checkGeoLocation(request) {
    const { customerId, ipAddress, billingCountry } = request;

    const geoInfo = await this.geoIPService.lookup(ipAddress);

    // Check for impossible travel
    const lastTransaction = await this.getLastTransaction(customerId);
    if (lastTransaction) {
      const timeDiff = Date.now() - lastTransaction.timestamp;
      const distance = this.calculateDistance(
        lastTransaction.location,
        geoInfo.location
      );

      // If distance is too far for time elapsed
      const maxPossibleDistance = (timeDiff / 3600000) * 900;  // 900 km/h max
      if (distance > maxPossibleDistance) {
        return { signal: 'geo', score: 0.9 };  // High risk
      }
    }

    // Check billing/IP country mismatch
    if (geoInfo.country !== billingCountry) {
      return { signal: 'geo', score: 0.5 };
    }

    return { signal: 'geo', score: 0.1 };
  }

  async checkMLModel(request) {
    // Call ML model for advanced fraud detection
    const features = await this.extractFeatures(request);
    const prediction = await this.mlModel.predict(features);

    return {
      signal: 'ml',
      score: prediction.fraudProbability
    };
  }

  calculateRiskScore(signals) {
    // Weighted average of signals
    const weights = {
      velocity: 0.2,
      device: 0.2,
      geo: 0.25,
      card: 0.15,
      ml: 0.2
    };

    let totalScore = 0;
    for (const signal of signals) {
      totalScore += signal.score * weights[signal.signal];
    }

    return totalScore;
  }
}
```

---

## Reconciliation

```javascript
class ReconciliationService {
  // Run daily to reconcile with payment gateway
  async reconcileDay(date) {
    const gatewayReport = await this.gateway.getSettlementReport(date);
    const ourTransactions = await this.getTransactionsForDate(date);

    const discrepancies = [];

    // Check each gateway transaction
    for (const gwTx of gatewayReport.transactions) {
      const ourTx = ourTransactions.find(t => t.gatewayReference === gwTx.reference);

      if (!ourTx) {
        discrepancies.push({
          type: 'MISSING_IN_OUR_SYSTEM',
          gatewayReference: gwTx.reference,
          amount: gwTx.amount
        });
        continue;
      }

      if (Math.abs(ourTx.amount - gwTx.amount) > 0.01) {
        discrepancies.push({
          type: 'AMOUNT_MISMATCH',
          transactionId: ourTx.id,
          ourAmount: ourTx.amount,
          gatewayAmount: gwTx.amount
        });
      }

      if (ourTx.status !== this.mapGatewayStatus(gwTx.status)) {
        discrepancies.push({
          type: 'STATUS_MISMATCH',
          transactionId: ourTx.id,
          ourStatus: ourTx.status,
          gatewayStatus: gwTx.status
        });
      }
    }

    // Check for transactions we have but gateway doesn't
    for (const ourTx of ourTransactions) {
      const gwTx = gatewayReport.transactions.find(t => t.reference === ourTx.gatewayReference);
      if (!gwTx && ourTx.status === 'COMPLETED') {
        discrepancies.push({
          type: 'MISSING_IN_GATEWAY',
          transactionId: ourTx.id,
          amount: ourTx.amount
        });
      }
    }

    // Store reconciliation result
    await this.storeReconciliationResult(date, {
      totalTransactions: ourTransactions.length,
      discrepancies,
      status: discrepancies.length === 0 ? 'MATCHED' : 'NEEDS_REVIEW'
    });

    // Alert if discrepancies found
    if (discrepancies.length > 0) {
      await this.alertService.sendReconciliationAlert(date, discrepancies);
    }

    return discrepancies;
  }
}
```

---

## Webhook Handling

```javascript
// Receive payment gateway webhooks
app.post('/webhooks/stripe', async (req, res) => {
  const sig = req.headers['stripe-signature'];

  // Verify webhook signature
  let event;
  try {
    event = stripe.webhooks.constructEvent(req.body, sig, WEBHOOK_SECRET);
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  // Idempotency - check if we've processed this event
  const processed = await redis.get(`webhook:${event.id}`);
  if (processed) {
    return res.status(200).json({ received: true, duplicate: true });
  }

  // Process webhook
  try {
    switch (event.type) {
      case 'payment_intent.succeeded':
        await handlePaymentSucceeded(event.data.object);
        break;
      case 'payment_intent.payment_failed':
        await handlePaymentFailed(event.data.object);
        break;
      case 'charge.refunded':
        await handleRefund(event.data.object);
        break;
      case 'charge.dispute.created':
        await handleDispute(event.data.object);
        break;
      default:
        console.log(`Unhandled event type: ${event.type}`);
    }

    // Mark as processed
    await redis.setex(`webhook:${event.id}`, 86400 * 7, '1');

    res.status(200).json({ received: true });
  } catch (error) {
    console.error('Webhook processing error:', error);
    res.status(500).json({ error: error.message });
  }
});

async function handlePaymentSucceeded(paymentIntent) {
  const transactionId = paymentIntent.metadata.transactionId;

  await db.transactions.update(transactionId, {
    status: 'COMPLETED',
    gatewayReference: paymentIntent.id,
    processedAt: new Date()
  });

  // Update ledger
  const transaction = await db.transactions.findById(transactionId);
  await ledger.credit(transaction.merchantId, transaction.amount);

  // Notify merchant
  await notificationService.send({
    type: 'PAYMENT_RECEIVED',
    merchantId: transaction.merchantId,
    amount: transaction.amount
  });
}
```

---

## PCI Compliance

### Tokenization

```javascript
// Never store raw card numbers
class TokenizationService {
  async tokenize(cardData) {
    // Use payment gateway's tokenization
    const token = await stripe.paymentMethods.create({
      type: 'card',
      card: {
        number: cardData.number,
        exp_month: cardData.expMonth,
        exp_year: cardData.expYear,
        cvc: cardData.cvc
      }
    });

    // Store only token reference
    await db.paymentMethods.create({
      userId: cardData.userId,
      token: token.id,
      last4: token.card.last4,
      brand: token.card.brand,
      expMonth: token.card.exp_month,
      expYear: token.card.exp_year
    });

    return {
      id: token.id,
      last4: token.card.last4,
      brand: token.card.brand
    };
  }
}

// Client-side tokenization (recommended)
// Card data goes directly to payment gateway, never touches our servers
```

---

## Interview Discussion Points

### How to handle payment failures?

1. **Retry logic** - Exponential backoff for transient failures
2. **Fallback gateways** - Switch to backup payment processor
3. **Idempotency** - Ensure retries don't cause double charges
4. **Status reconciliation** - Verify final state with gateway
5. **User notification** - Inform user of failure and next steps

### How to ensure exactly-once processing?

1. **Idempotency keys** - Client-provided unique identifiers
2. **Transaction state machine** - PENDING → PROCESSING → COMPLETED/FAILED
3. **Distributed locks** - Prevent concurrent processing
4. **Webhook deduplication** - Store processed webhook IDs

### How to handle chargebacks?

1. **Automated detection** - Webhook from payment gateway
2. **Evidence collection** - Gather transaction proof automatically
3. **Dispute response** - Submit evidence within deadline
4. **Reserve funds** - Hold merchant funds for dispute period
5. **Fraud prevention** - Update risk models with chargeback data

### How to scale payment processing?

1. **Horizontal scaling** - Stateless payment services
2. **Queue-based processing** - Decouple API from processing
3. **Gateway distribution** - Multiple payment processors
4. **Database sharding** - Shard by merchant or region
5. **Caching** - Cache merchant configs, rate limits
