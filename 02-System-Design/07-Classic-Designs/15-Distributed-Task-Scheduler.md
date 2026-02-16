# Design: Distributed Task Scheduler

## Requirements

### Functional Requirements
- Schedule one-time tasks (run at specific time)
- Schedule recurring tasks (cron-like)
- Support task priorities
- Retry failed tasks with backoff
- Task dependencies (DAG execution)
- Cancel/pause scheduled tasks
- Track task execution history

### Non-Functional Requirements
- High availability (no missed executions)
- Exactly-once execution (no duplicates)
- Scalability: 10M+ scheduled tasks
- Low latency: Execute within 1 second of scheduled time
- Fault tolerance: Handle worker failures

### Capacity Estimation

```
Scheduled tasks: 10M active
Task executions: 1B/day
Peak QPS: 50K executions/second

Storage:
- Task metadata: 10M × 1KB = 10GB
- Execution history: 1B × 200 bytes = 200GB/day
- Keep 30 days: 6TB
```

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Clients / Services                              │
│              (Schedule tasks via API or SDK)                             │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────┐
│                         Scheduler API                                    │
│         (Create, Update, Delete, Query scheduled tasks)                  │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────────────────────┐
         ▼                       ▼                       ▼               │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐       │
│  Task Store     │    │  Timer Service  │    │  Queue Service  │       │
│  (Metadata)     │    │  (Trigger)      │    │  (Dispatch)     │       │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘       │
         │                      │                      │                 │
         ▼                      │                      ▼                 │
┌─────────────────┐            │             ┌─────────────────┐        │
│  PostgreSQL /   │            │             │  Kafka / Redis  │        │
│  Cassandra      │            │             │  Streams        │        │
└─────────────────┘            │             └────────┬────────┘        │
                               │                      │                 │
                               ▼                      ▼                 │
                      ┌─────────────────────────────────────────┐       │
                      │           Worker Pool                    │       │
                      │  ┌─────────┐ ┌─────────┐ ┌─────────┐   │       │
                      │  │Worker 1 │ │Worker 2 │ │Worker N │   │       │
                      │  └─────────┘ └─────────┘ └─────────┘   │       │
                      └─────────────────────────────────────────┘       │
```

---

## Core Components

### Task Model

```javascript
// Task definition
const TaskSchema = {
  id: 'uuid',
  name: 'string',
  type: 'one-time | recurring',

  // Scheduling
  scheduledAt: 'timestamp',      // For one-time tasks
  cronExpression: 'string',      // For recurring (e.g., "0 9 * * *")
  timezone: 'string',            // e.g., "America/New_York"

  // Execution
  handler: 'string',             // Handler identifier
  payload: 'json',               // Task-specific data
  priority: 'number',            // 1-10, higher = more urgent
  timeout: 'number',             // Max execution time (ms)

  // Retry policy
  maxRetries: 'number',
  retryDelay: 'number',          // Base delay for exponential backoff
  retryCount: 'number',          // Current retry count

  // State
  status: 'pending | running | completed | failed | cancelled',
  lastRunAt: 'timestamp',
  nextRunAt: 'timestamp',

  // Metadata
  createdAt: 'timestamp',
  updatedAt: 'timestamp',
  createdBy: 'string',
  tags: 'string[]'
};

// Task execution record
const ExecutionSchema = {
  id: 'uuid',
  taskId: 'uuid',
  workerId: 'string',
  status: 'running | completed | failed',
  startedAt: 'timestamp',
  completedAt: 'timestamp',
  result: 'json',
  error: 'string',
  attempt: 'number'
};
```

### Database Schema

```sql
-- Tasks table
CREATE TABLE tasks (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  type VARCHAR(20) NOT NULL,

  -- Scheduling
  scheduled_at TIMESTAMP,
  cron_expression VARCHAR(100),
  timezone VARCHAR(50) DEFAULT 'UTC',
  next_run_at TIMESTAMP NOT NULL,

  -- Execution config
  handler VARCHAR(255) NOT NULL,
  payload JSONB,
  priority INT DEFAULT 5,
  timeout_ms INT DEFAULT 300000,

  -- Retry config
  max_retries INT DEFAULT 3,
  retry_delay_ms INT DEFAULT 1000,
  retry_count INT DEFAULT 0,

  -- State
  status VARCHAR(20) DEFAULT 'pending',
  last_run_at TIMESTAMP,
  locked_by VARCHAR(255),
  locked_until TIMESTAMP,

  -- Metadata
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  created_by VARCHAR(255),
  tags TEXT[]
);

-- Indexes for efficient queries
CREATE INDEX idx_tasks_next_run ON tasks(next_run_at)
  WHERE status = 'pending';
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_handler ON tasks(handler);
CREATE INDEX idx_tasks_locked ON tasks(locked_until)
  WHERE locked_by IS NOT NULL;

-- Execution history
CREATE TABLE task_executions (
  id UUID PRIMARY KEY,
  task_id UUID REFERENCES tasks(id),
  worker_id VARCHAR(255),
  status VARCHAR(20),
  attempt INT,
  started_at TIMESTAMP,
  completed_at TIMESTAMP,
  result JSONB,
  error TEXT
);

CREATE INDEX idx_executions_task ON task_executions(task_id, started_at DESC);
```

---

## Timer Service

### Time-Based Task Polling

```javascript
class TimerService {
  constructor(taskStore, queueService, options = {}) {
    this.taskStore = taskStore;
    this.queueService = queueService;
    this.pollInterval = options.pollInterval || 1000;  // 1 second
    this.batchSize = options.batchSize || 100;
    this.lookAheadMs = options.lookAheadMs || 5000;    // 5 seconds
  }

  async start() {
    console.log('Timer service started');

    while (true) {
      try {
        await this.pollAndDispatch();
      } catch (error) {
        console.error('Poll error:', error);
      }

      await this.sleep(this.pollInterval);
    }
  }

  async pollAndDispatch() {
    const now = Date.now();
    const cutoff = new Date(now + this.lookAheadMs);

    // Find tasks due for execution
    const dueTasks = await this.taskStore.findDueTasks({
      nextRunBefore: cutoff,
      status: 'pending',
      limit: this.batchSize
    });

    for (const task of dueTasks) {
      // Try to acquire lock (prevent duplicate dispatch)
      const locked = await this.taskStore.tryLock(task.id, {
        lockDuration: task.timeout + 60000,  // timeout + buffer
        workerId: this.workerId
      });

      if (locked) {
        // Dispatch to queue
        await this.queueService.enqueue({
          taskId: task.id,
          handler: task.handler,
          payload: task.payload,
          priority: task.priority,
          timeout: task.timeout,
          attempt: task.retryCount + 1
        });

        // Update task status
        await this.taskStore.updateStatus(task.id, 'queued');
      }
    }
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### Hierarchical Timing Wheel

```javascript
// Efficient timer for millions of scheduled tasks
class TimingWheel {
  constructor(tickMs = 1, wheelSize = 60) {
    this.tickMs = tickMs;
    this.wheelSize = wheelSize;
    this.interval = tickMs * wheelSize;

    this.buckets = Array.from({ length: wheelSize }, () => new Set());
    this.currentTick = 0;
    this.currentTime = Date.now();

    // Overflow wheel for far-future tasks
    this.overflowWheel = null;
  }

  add(task, executeAt) {
    const delay = executeAt - this.currentTime;

    if (delay < this.tickMs) {
      // Execute immediately
      return { immediate: true, task };
    }

    if (delay < this.interval) {
      // Fits in this wheel
      const ticks = Math.floor(delay / this.tickMs);
      const bucketIndex = (this.currentTick + ticks) % this.wheelSize;
      this.buckets[bucketIndex].add(task);
      return { added: true, bucket: bucketIndex };
    }

    // Overflow to higher-level wheel
    if (!this.overflowWheel) {
      this.overflowWheel = new TimingWheel(this.interval, this.wheelSize);
    }
    return this.overflowWheel.add(task, executeAt);
  }

  advance() {
    this.currentTick = (this.currentTick + 1) % this.wheelSize;
    this.currentTime += this.tickMs;

    // Get tasks from current bucket
    const bucket = this.buckets[this.currentTick];
    const tasks = Array.from(bucket);
    bucket.clear();

    // Cascade from overflow wheel
    if (this.currentTick === 0 && this.overflowWheel) {
      const overflowTasks = this.overflowWheel.advance();
      for (const task of overflowTasks) {
        this.add(task, task.executeAt);
      }
    }

    return tasks;
  }

  remove(taskId) {
    for (const bucket of this.buckets) {
      for (const task of bucket) {
        if (task.id === taskId) {
          bucket.delete(task);
          return true;
        }
      }
    }
    return this.overflowWheel?.remove(taskId) || false;
  }
}
```

---

## Queue Service

### Priority Queue with Redis

```javascript
class TaskQueueService {
  constructor(redis) {
    this.redis = redis;
    this.queuePrefix = 'task_queue';
  }

  async enqueue(task) {
    const queueKey = `${this.queuePrefix}:${task.priority}`;
    const taskData = JSON.stringify({
      ...task,
      enqueuedAt: Date.now()
    });

    // Add to priority-specific sorted set (score = timestamp)
    await this.redis.zadd(queueKey, Date.now(), taskData);

    // Notify workers
    await this.redis.publish('task_available', task.priority);
  }

  async dequeue(priorities = [10, 9, 8, 7, 6, 5, 4, 3, 2, 1]) {
    // Try each priority level (highest first)
    for (const priority of priorities) {
      const queueKey = `${this.queuePrefix}:${priority}`;

      // Atomic pop from sorted set
      const result = await this.redis.zpopmin(queueKey);

      if (result && result.length > 0) {
        return JSON.parse(result[0]);
      }
    }

    return null;
  }

  async dequeueBlocking(timeout = 5000) {
    // Block until task available
    const keys = [10, 9, 8, 7, 6, 5, 4, 3, 2, 1]
      .map(p => `${this.queuePrefix}:${p}`);

    const result = await this.redis.bzpopmin(keys, timeout / 1000);

    if (result) {
      return JSON.parse(result[1]);
    }
    return null;
  }

  async getQueueStats() {
    const stats = {};

    for (let priority = 1; priority <= 10; priority++) {
      const queueKey = `${this.queuePrefix}:${priority}`;
      stats[priority] = await this.redis.zcard(queueKey);
    }

    return stats;
  }
}
```

### Kafka-Based Queue (High Throughput)

```javascript
class KafkaTaskQueue {
  constructor(kafka) {
    this.kafka = kafka;
    this.producer = kafka.producer();
    this.consumer = kafka.consumer({ groupId: 'task-workers' });
  }

  async enqueue(task) {
    await this.producer.send({
      topic: 'scheduled-tasks',
      messages: [{
        key: task.id,
        value: JSON.stringify(task),
        headers: {
          priority: task.priority.toString(),
          handler: task.handler
        }
      }],
      // Partition by handler for locality
      partition: this.getPartition(task.handler)
    });
  }

  async startConsuming(handler) {
    await this.consumer.connect();
    await this.consumer.subscribe({
      topic: 'scheduled-tasks',
      fromBeginning: false
    });

    await this.consumer.run({
      eachMessage: async ({ message, partition, heartbeat }) => {
        const task = JSON.parse(message.value.toString());

        try {
          await handler(task);
          // Commit offset on success
        } catch (error) {
          // Handle failure (DLQ, retry, etc.)
          await this.handleFailure(task, error);
        }
      }
    });
  }
}
```

---

## Worker Service

### Task Worker

```javascript
class TaskWorker {
  constructor(queueService, taskStore, options = {}) {
    this.queueService = queueService;
    this.taskStore = taskStore;
    this.workerId = options.workerId || `worker-${os.hostname()}-${process.pid}`;
    this.handlers = new Map();
    this.running = false;
  }

  registerHandler(name, handler) {
    this.handlers.set(name, handler);
  }

  async start() {
    this.running = true;
    console.log(`Worker ${this.workerId} started`);

    while (this.running) {
      try {
        const task = await this.queueService.dequeueBlocking(5000);

        if (task) {
          await this.executeTask(task);
        }
      } catch (error) {
        console.error('Worker error:', error);
        await this.sleep(1000);
      }
    }
  }

  async executeTask(task) {
    const execution = await this.taskStore.createExecution({
      taskId: task.taskId,
      workerId: this.workerId,
      attempt: task.attempt,
      startedAt: new Date()
    });

    const handler = this.handlers.get(task.handler);

    if (!handler) {
      await this.handleFailure(task, execution, new Error(`Unknown handler: ${task.handler}`));
      return;
    }

    try {
      // Execute with timeout
      const result = await this.executeWithTimeout(
        () => handler(task.payload),
        task.timeout
      );

      await this.handleSuccess(task, execution, result);

    } catch (error) {
      await this.handleFailure(task, execution, error);
    }
  }

  async executeWithTimeout(fn, timeoutMs) {
    return Promise.race([
      fn(),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Task timeout')), timeoutMs)
      )
    ]);
  }

  async handleSuccess(task, execution, result) {
    await this.taskStore.updateExecution(execution.id, {
      status: 'completed',
      completedAt: new Date(),
      result
    });

    const taskRecord = await this.taskStore.getTask(task.taskId);

    if (taskRecord.type === 'recurring') {
      // Schedule next run
      const nextRun = this.calculateNextRun(taskRecord.cronExpression, taskRecord.timezone);
      await this.taskStore.updateTask(task.taskId, {
        status: 'pending',
        nextRunAt: nextRun,
        lastRunAt: new Date(),
        retryCount: 0,
        lockedBy: null,
        lockedUntil: null
      });
    } else {
      // One-time task completed
      await this.taskStore.updateTask(task.taskId, {
        status: 'completed',
        lastRunAt: new Date(),
        lockedBy: null,
        lockedUntil: null
      });
    }
  }

  async handleFailure(task, execution, error) {
    await this.taskStore.updateExecution(execution.id, {
      status: 'failed',
      completedAt: new Date(),
      error: error.message
    });

    const taskRecord = await this.taskStore.getTask(task.taskId);

    if (task.attempt < taskRecord.maxRetries) {
      // Schedule retry with exponential backoff
      const delay = taskRecord.retryDelay * Math.pow(2, task.attempt - 1);
      const nextRetry = new Date(Date.now() + delay);

      await this.taskStore.updateTask(task.taskId, {
        status: 'pending',
        nextRunAt: nextRetry,
        retryCount: task.attempt,
        lockedBy: null,
        lockedUntil: null
      });
    } else {
      // Max retries exceeded
      await this.taskStore.updateTask(task.taskId, {
        status: 'failed',
        lockedBy: null,
        lockedUntil: null
      });

      // Send to dead letter queue
      await this.sendToDeadLetterQueue(task, error);
    }
  }

  calculateNextRun(cronExpression, timezone) {
    const parser = require('cron-parser');
    const interval = parser.parseExpression(cronExpression, { tz: timezone });
    return interval.next().toDate();
  }
}
```

---

## Distributed Coordination

### Leader Election for Timer Service

```javascript
class DistributedTimerService {
  constructor(zk, taskStore, queueService) {
    this.zk = zk;
    this.taskStore = taskStore;
    this.queueService = queueService;
    this.isLeader = false;
    this.timerService = null;
  }

  async start() {
    // Participate in leader election
    const election = new LeaderElection(this.zk, '/scheduler/timer-leader');

    election.onBecomeLeader = () => {
      console.log('Became timer leader');
      this.isLeader = true;
      this.startTimerService();
    };

    election.onLoseLeadership = () => {
      console.log('Lost timer leadership');
      this.isLeader = false;
      this.stopTimerService();
    };

    await election.start();
  }

  startTimerService() {
    this.timerService = new TimerService(this.taskStore, this.queueService);
    this.timerService.start();
  }

  stopTimerService() {
    if (this.timerService) {
      this.timerService.stop();
      this.timerService = null;
    }
  }
}
```

### Sharded Timer Service

```javascript
// Partition tasks across multiple timer instances
class ShardedTimerService {
  constructor(options) {
    this.shardId = options.shardId;
    this.totalShards = options.totalShards;
    this.taskStore = options.taskStore;
    this.queueService = options.queueService;
  }

  async pollAndDispatch() {
    // Only poll tasks belonging to this shard
    const dueTasks = await this.taskStore.findDueTasks({
      nextRunBefore: new Date(Date.now() + 5000),
      status: 'pending',
      shardKey: this.shardId,
      totalShards: this.totalShards,
      limit: 100
    });

    // Process tasks...
  }
}

// Task store query with sharding
async function findDueTasks({ shardKey, totalShards, ...filters }) {
  return db.query(`
    SELECT * FROM tasks
    WHERE next_run_at <= $1
      AND status = 'pending'
      AND MOD(ABS(HASHTEXT(id::text)), $2) = $3
    ORDER BY priority DESC, next_run_at ASC
    LIMIT $4
    FOR UPDATE SKIP LOCKED
  `, [filters.nextRunBefore, totalShards, shardKey, filters.limit]);
}
```

---

## Task Dependencies (DAG)

### DAG Execution

```javascript
class DAGScheduler {
  constructor(taskStore, queueService) {
    this.taskStore = taskStore;
    this.queueService = queueService;
  }

  async createDAG(dagDefinition) {
    const { name, tasks, edges } = dagDefinition;

    // Create DAG record
    const dag = await this.taskStore.createDAG({
      id: generateId(),
      name,
      status: 'pending',
      createdAt: new Date()
    });

    // Create task nodes
    const taskMap = new Map();
    for (const taskDef of tasks) {
      const task = await this.taskStore.createTask({
        ...taskDef,
        dagId: dag.id,
        status: 'blocked'  // Waiting for dependencies
      });
      taskMap.set(taskDef.name, task);
    }

    // Create edges (dependencies)
    for (const [from, to] of edges) {
      await this.taskStore.createDependency({
        dagId: dag.id,
        fromTaskId: taskMap.get(from).id,
        toTaskId: taskMap.get(to).id
      });
    }

    // Find and queue root tasks (no dependencies)
    const rootTasks = await this.findRootTasks(dag.id);
    for (const task of rootTasks) {
      await this.taskStore.updateTask(task.id, { status: 'pending' });
      await this.queueService.enqueue(task);
    }

    return dag;
  }

  async onTaskComplete(taskId) {
    const task = await this.taskStore.getTask(taskId);

    if (!task.dagId) return;  // Not part of a DAG

    // Find dependent tasks
    const dependents = await this.taskStore.getDependents(taskId);

    for (const dependent of dependents) {
      // Check if all dependencies are complete
      const allDepsComplete = await this.checkAllDependencies(dependent.id);

      if (allDepsComplete) {
        // Unblock and queue
        await this.taskStore.updateTask(dependent.id, { status: 'pending' });
        await this.queueService.enqueue(dependent);
      }
    }

    // Check if DAG is complete
    await this.checkDAGComplete(task.dagId);
  }

  async checkAllDependencies(taskId) {
    const dependencies = await this.taskStore.getDependencies(taskId);

    for (const dep of dependencies) {
      const depTask = await this.taskStore.getTask(dep.fromTaskId);
      if (depTask.status !== 'completed') {
        return false;
      }
    }

    return true;
  }

  async checkDAGComplete(dagId) {
    const incompleteTasks = await this.taskStore.query(`
      SELECT COUNT(*) FROM tasks
      WHERE dag_id = $1 AND status != 'completed'
    `, [dagId]);

    if (incompleteTasks === 0) {
      await this.taskStore.updateDAG(dagId, {
        status: 'completed',
        completedAt: new Date()
      });
    }
  }
}

// DAG Definition Example
const dagDefinition = {
  name: 'ETL Pipeline',
  tasks: [
    { name: 'extract', handler: 'etl.extract', payload: { source: 's3://...' } },
    { name: 'transform', handler: 'etl.transform' },
    { name: 'validate', handler: 'etl.validate' },
    { name: 'load', handler: 'etl.load', payload: { destination: 'warehouse' } },
    { name: 'notify', handler: 'notifications.send' }
  ],
  edges: [
    ['extract', 'transform'],
    ['transform', 'validate'],
    ['validate', 'load'],
    ['load', 'notify']
  ]
};
```

---

## Cron Expression Parser

```javascript
class CronParser {
  // Cron format: minute hour day-of-month month day-of-week
  // Example: "0 9 * * 1-5" = 9 AM on weekdays

  static parse(expression) {
    const parts = expression.split(' ');
    if (parts.length !== 5) {
      throw new Error('Invalid cron expression');
    }

    return {
      minute: this.parseField(parts[0], 0, 59),
      hour: this.parseField(parts[1], 0, 23),
      dayOfMonth: this.parseField(parts[2], 1, 31),
      month: this.parseField(parts[3], 1, 12),
      dayOfWeek: this.parseField(parts[4], 0, 6)
    };
  }

  static parseField(field, min, max) {
    if (field === '*') {
      return { type: 'any' };
    }

    if (field.includes('/')) {
      const [range, step] = field.split('/');
      return { type: 'step', step: parseInt(step), range: this.parseField(range, min, max) };
    }

    if (field.includes('-')) {
      const [start, end] = field.split('-').map(Number);
      return { type: 'range', start, end };
    }

    if (field.includes(',')) {
      const values = field.split(',').map(Number);
      return { type: 'list', values };
    }

    return { type: 'value', value: parseInt(field) };
  }

  static getNextRun(expression, after = new Date()) {
    const cron = this.parse(expression);
    const next = new Date(after);
    next.setSeconds(0);
    next.setMilliseconds(0);
    next.setMinutes(next.getMinutes() + 1);

    for (let i = 0; i < 366 * 24 * 60; i++) {  // Max 1 year search
      if (this.matches(cron, next)) {
        return next;
      }
      next.setMinutes(next.getMinutes() + 1);
    }

    throw new Error('No next run found within a year');
  }

  static matches(cron, date) {
    return (
      this.fieldMatches(cron.minute, date.getMinutes()) &&
      this.fieldMatches(cron.hour, date.getHours()) &&
      this.fieldMatches(cron.dayOfMonth, date.getDate()) &&
      this.fieldMatches(cron.month, date.getMonth() + 1) &&
      this.fieldMatches(cron.dayOfWeek, date.getDay())
    );
  }

  static fieldMatches(field, value) {
    switch (field.type) {
      case 'any': return true;
      case 'value': return field.value === value;
      case 'range': return value >= field.start && value <= field.end;
      case 'list': return field.values.includes(value);
      case 'step':
        if (field.range.type === 'any') {
          return value % field.step === 0;
        }
        return this.fieldMatches(field.range, value) && value % field.step === 0;
      default: return false;
    }
  }
}
```

---

## Monitoring & Observability

```javascript
class SchedulerMetrics {
  constructor(prometheus) {
    this.tasksScheduled = new prometheus.Counter({
      name: 'scheduler_tasks_scheduled_total',
      help: 'Total number of tasks scheduled',
      labelNames: ['handler', 'type']
    });

    this.tasksExecuted = new prometheus.Counter({
      name: 'scheduler_tasks_executed_total',
      help: 'Total number of tasks executed',
      labelNames: ['handler', 'status']
    });

    this.taskDuration = new prometheus.Histogram({
      name: 'scheduler_task_duration_seconds',
      help: 'Task execution duration',
      labelNames: ['handler'],
      buckets: [0.1, 0.5, 1, 5, 10, 30, 60, 300]
    });

    this.taskDelay = new prometheus.Histogram({
      name: 'scheduler_task_delay_seconds',
      help: 'Delay between scheduled time and execution',
      buckets: [0.1, 0.5, 1, 5, 10, 30, 60]
    });

    this.queueSize = new prometheus.Gauge({
      name: 'scheduler_queue_size',
      help: 'Number of tasks in queue',
      labelNames: ['priority']
    });

    this.activeWorkers = new prometheus.Gauge({
      name: 'scheduler_active_workers',
      help: 'Number of active workers'
    });
  }

  recordExecution(handler, status, durationMs, delayMs) {
    this.tasksExecuted.inc({ handler, status });
    this.taskDuration.observe({ handler }, durationMs / 1000);
    this.taskDelay.observe(delayMs / 1000);
  }
}
```

---

## Interview Discussion Points

### How to ensure exactly-once execution?

1. **Distributed locks** - Lock task before dispatch
2. **Idempotency keys** - Handlers check before executing
3. **Database constraints** - Unique execution per schedule
4. **Fencing tokens** - Reject stale executions
5. **Two-phase commit** - Coordinate dispatch and execution

### How to handle missed schedules?

1. **Catch-up execution** - Run all missed schedules
2. **Skip and continue** - Only run next scheduled time
3. **Coalesce** - Run once for all missed
4. **Alert** - Notify and let operator decide
5. **Configurable policy** - Per-task setting

### How to scale to millions of tasks?

1. **Sharding** - Partition by task ID or handler
2. **Timing wheels** - Efficient timer management
3. **Batch processing** - Poll multiple tasks at once
4. **Separate hot/cold** - Different stores for near/far tasks
5. **Caching** - Cache frequently accessed tasks

### How to handle worker failures?

1. **Heartbeats** - Detect dead workers
2. **Lock timeouts** - Auto-release stuck tasks
3. **Execution tracking** - Know which tasks were running
4. **Automatic reassignment** - Requeue orphaned tasks
5. **Graceful shutdown** - Complete current task before exit
