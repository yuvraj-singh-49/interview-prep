# Design: Distributed File Storage (Dropbox/Google Drive)

## Requirements

### Functional Requirements
- Upload and download files
- File sync across devices
- File sharing with permissions
- File versioning and history
- Offline support
- Conflict resolution

### Non-Functional Requirements
- High availability (99.9%)
- Strong consistency for metadata
- Eventual consistency for file content (acceptable)
- Low latency for small files (<1 second)
- Support large files (up to 50GB)
- Scale: 500M users, 10B files

### Capacity Estimation

```
Users: 500M total, 100M DAU
Files per user: 200 average
Total files: 100B files
Average file size: 500KB
Total storage: 50PB

Daily uploads: 2 files/user × 100M = 200M files/day
Upload throughput: 200M × 500KB = 100TB/day
QPS: 200M / 86400 = 2300 uploads/second

Sync operations: Much higher (10x uploads)
Metadata QPS: 50K/second
```

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Client Apps (Desktop/Mobile/Web)                     │
│                    ┌─────────────────────────────────┐                  │
│                    │  Local File System Watcher      │                  │
│                    │  Chunking & Deduplication       │                  │
│                    │  Local Database (SQLite)        │                  │
│                    └─────────────────────────────────┘                  │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────┐
│                            API Gateway                                   │
│                    (Auth, Rate Limiting, Routing)                        │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────────────────────┐
         ▼                       ▼                       ▼               │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐       │
│  Metadata       │    │  Block Service  │    │  Sync Service   │       │
│  Service        │    │  (Upload/DL)    │    │  (Real-time)    │       │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘       │
         │                      │                      │                 │
         ▼                      ▼                      ▼                 │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐       │
│  Metadata DB    │    │  Block Storage  │    │  Notification   │       │
│  (PostgreSQL)   │    │  (S3/HDFS)      │    │  Service        │       │
└─────────────────┘    └─────────────────┘    └─────────────────┘       │
```

---

## File Chunking & Deduplication

### Why Chunking?

```
Benefits:
1. Resume interrupted uploads/downloads
2. Parallel transfer (faster)
3. Deduplication (save storage)
4. Efficient sync (only changed chunks)
5. Large file support (50GB+)

Chunk size: 4MB (typical)
- Too small: More metadata overhead
- Too large: Less deduplication benefit
```

### Content-Defined Chunking (CDC)

```javascript
// Rolling hash for content-defined chunk boundaries
class ContentDefinedChunker {
  constructor(options = {}) {
    this.minChunkSize = options.minChunkSize || 256 * 1024;   // 256KB
    this.maxChunkSize = options.maxChunkSize || 8 * 1024 * 1024; // 8MB
    this.avgChunkSize = options.avgChunkSize || 4 * 1024 * 1024; // 4MB
    this.mask = this.calculateMask(this.avgChunkSize);
  }

  calculateMask(avgSize) {
    // Mask determines average chunk size
    // P(boundary) = 1/avgSize when (hash & mask) === 0
    const bits = Math.log2(avgSize);
    return (1 << bits) - 1;
  }

  *chunk(data) {
    let start = 0;
    let hash = 0;

    for (let i = 0; i < data.length; i++) {
      // Rolling hash (Rabin fingerprint simplified)
      hash = ((hash << 1) + data[i]) & 0xFFFFFFFF;

      const chunkSize = i - start + 1;

      // Check for chunk boundary
      const isBoundary = (hash & this.mask) === 0;
      const isMinSize = chunkSize >= this.minChunkSize;
      const isMaxSize = chunkSize >= this.maxChunkSize;

      if ((isBoundary && isMinSize) || isMaxSize || i === data.length - 1) {
        yield {
          data: data.slice(start, i + 1),
          start,
          end: i + 1,
          hash: this.computeHash(data.slice(start, i + 1))
        };
        start = i + 1;
        hash = 0;
      }
    }
  }

  computeHash(chunk) {
    const crypto = require('crypto');
    return crypto.createHash('sha256').update(chunk).digest('hex');
  }
}
```

### Deduplication

```javascript
class DeduplicationService {
  constructor(blockStorage, metadataDB) {
    this.blockStorage = blockStorage;
    this.metadataDB = metadataDB;
  }

  async uploadFile(userId, filePath, fileData) {
    const chunker = new ContentDefinedChunker();
    const chunks = [];
    const newChunks = [];

    // Chunk the file
    for (const chunk of chunker.chunk(fileData)) {
      chunks.push({
        hash: chunk.hash,
        size: chunk.data.length,
        index: chunks.length
      });

      // Check if chunk already exists (deduplication)
      const exists = await this.metadataDB.chunkExists(chunk.hash);

      if (!exists) {
        newChunks.push({
          hash: chunk.hash,
          data: chunk.data
        });
      }
    }

    // Upload only new chunks
    await Promise.all(
      newChunks.map(chunk =>
        this.blockStorage.upload(chunk.hash, chunk.data)
      )
    );

    // Increment reference count for all chunks
    await this.metadataDB.incrementChunkRefs(chunks.map(c => c.hash));

    // Create file metadata
    const fileId = generateId();
    await this.metadataDB.createFile({
      id: fileId,
      userId,
      path: filePath,
      chunks: chunks.map(c => c.hash),
      size: fileData.length,
      version: 1,
      createdAt: new Date()
    });

    return {
      fileId,
      totalChunks: chunks.length,
      newChunks: newChunks.length,
      savedBytes: (chunks.length - newChunks.length) * 4 * 1024 * 1024
    };
  }
}
```

---

## Sync Protocol

### Delta Sync

```javascript
class SyncService {
  async syncFile(userId, fileId, localVersion, localChunks) {
    // Get server state
    const serverFile = await this.metadataDB.getFile(fileId);

    if (!serverFile) {
      return { action: 'DELETED_ON_SERVER' };
    }

    if (serverFile.version === localVersion) {
      return { action: 'UP_TO_DATE' };
    }

    // Calculate diff
    const serverChunks = new Set(serverFile.chunks);
    const localChunkSet = new Set(localChunks);

    const toDownload = serverFile.chunks.filter(h => !localChunkSet.has(h));
    const toUpload = localChunks.filter(h => !serverChunks.has(h));

    if (serverFile.version > localVersion) {
      // Server is newer - download changes
      return {
        action: 'DOWNLOAD',
        chunks: toDownload,
        newVersion: serverFile.version
      };
    } else {
      // Local is newer - upload changes
      return {
        action: 'UPLOAD',
        chunks: toUpload,
        baseVersion: serverFile.version
      };
    }
  }

  // Real-time sync notification
  async notifyClients(userId, fileId, change) {
    // Get all connected clients for this user
    const clients = await this.getConnectedClients(userId);

    for (const client of clients) {
      await this.sendNotification(client, {
        type: 'FILE_CHANGED',
        fileId,
        change
      });
    }
  }
}
```

### Conflict Resolution

```javascript
class ConflictResolver {
  async resolveConflict(fileId, localVersion, serverVersion) {
    // Strategy 1: Last writer wins (simplest)
    // Strategy 2: Keep both versions (what Dropbox does)
    // Strategy 3: Three-way merge (for text files)

    const serverFile = await this.metadataDB.getFile(fileId);

    // Create conflict copy
    const conflictFile = {
      ...serverFile,
      id: generateId(),
      path: this.generateConflictPath(serverFile.path),
      isConflict: true,
      conflictOf: fileId,
      conflictCreatedAt: new Date()
    };

    await this.metadataDB.createFile(conflictFile);

    // Notify user
    await this.notificationService.send({
      userId: serverFile.userId,
      type: 'CONFLICT_DETECTED',
      originalFile: serverFile.path,
      conflictFile: conflictFile.path
    });

    return {
      resolved: true,
      conflictFilePath: conflictFile.path
    };
  }

  generateConflictPath(originalPath) {
    const ext = path.extname(originalPath);
    const base = path.basename(originalPath, ext);
    const dir = path.dirname(originalPath);
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const hostname = os.hostname();

    return `${dir}/${base} (${hostname}'s conflicted copy ${timestamp})${ext}`;
  }
}
```

---

## Metadata Service

### Database Schema

```sql
-- Users
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email VARCHAR(255) UNIQUE,
  storage_quota BIGINT DEFAULT 15000000000,  -- 15GB
  storage_used BIGINT DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Files (current version)
CREATE TABLE files (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  parent_folder_id UUID REFERENCES files(id),
  name VARCHAR(255) NOT NULL,
  path TEXT NOT NULL,
  is_folder BOOLEAN DEFAULT FALSE,
  size BIGINT,
  checksum VARCHAR(64),
  version INT DEFAULT 1,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  deleted_at TIMESTAMP,  -- Soft delete for trash
  UNIQUE(user_id, path)
);

CREATE INDEX idx_files_user_path ON files(user_id, path);
CREATE INDEX idx_files_parent ON files(parent_folder_id);

-- File versions (history)
CREATE TABLE file_versions (
  id UUID PRIMARY KEY,
  file_id UUID REFERENCES files(id),
  version INT NOT NULL,
  chunks TEXT[] NOT NULL,  -- Array of chunk hashes
  size BIGINT,
  checksum VARCHAR(64),
  created_at TIMESTAMP DEFAULT NOW(),
  created_by UUID REFERENCES users(id),
  UNIQUE(file_id, version)
);

-- Chunks (deduplicated blocks)
CREATE TABLE chunks (
  hash VARCHAR(64) PRIMARY KEY,
  size INT NOT NULL,
  reference_count INT DEFAULT 1,
  storage_location TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- File-chunk mapping
CREATE TABLE file_chunks (
  file_id UUID REFERENCES files(id),
  version INT,
  chunk_index INT,
  chunk_hash VARCHAR(64) REFERENCES chunks(hash),
  PRIMARY KEY (file_id, version, chunk_index)
);

-- Sharing
CREATE TABLE shares (
  id UUID PRIMARY KEY,
  file_id UUID REFERENCES files(id),
  shared_by UUID REFERENCES users(id),
  shared_with UUID REFERENCES users(id),  -- NULL for link shares
  permission VARCHAR(20),  -- 'view', 'edit'
  link_token VARCHAR(64) UNIQUE,  -- For public links
  password_hash VARCHAR(255),
  expires_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### File Operations

```javascript
class MetadataService {
  async createFile(userId, path, chunks, size) {
    return await this.db.transaction(async (trx) => {
      // Check quota
      const user = await trx.query('SELECT * FROM users WHERE id = $1', [userId]);
      if (user.storage_used + size > user.storage_quota) {
        throw new QuotaExceededError();
      }

      // Create file record
      const file = await trx.query(`
        INSERT INTO files (id, user_id, path, name, size, version)
        VALUES ($1, $2, $3, $4, $5, 1)
        ON CONFLICT (user_id, path)
        DO UPDATE SET
          version = files.version + 1,
          size = $5,
          updated_at = NOW()
        RETURNING *
      `, [generateId(), userId, path, this.extractName(path), size]);

      // Create version record
      await trx.query(`
        INSERT INTO file_versions (id, file_id, version, chunks, size)
        VALUES ($1, $2, $3, $4, $5)
      `, [generateId(), file.id, file.version, chunks, size]);

      // Update user storage
      await trx.query(`
        UPDATE users SET storage_used = storage_used + $1 WHERE id = $2
      `, [size, userId]);

      return file;
    });
  }

  async moveFile(userId, fileId, newPath) {
    return await this.db.transaction(async (trx) => {
      // Check if destination exists
      const existing = await trx.query(
        'SELECT id FROM files WHERE user_id = $1 AND path = $2',
        [userId, newPath]
      );

      if (existing) {
        throw new FileExistsError();
      }

      // Move file (and all children if folder)
      await trx.query(`
        UPDATE files
        SET path = $3 || substring(path FROM length($2) + 1),
            updated_at = NOW()
        WHERE user_id = $1 AND path LIKE $2 || '%'
      `, [userId, this.getOldPath(fileId), newPath]);
    });
  }

  async deleteFile(userId, fileId, permanent = false) {
    if (permanent) {
      // Actually delete
      return await this.db.transaction(async (trx) => {
        // Get file and all versions
        const versions = await trx.query(
          'SELECT chunks FROM file_versions WHERE file_id = $1',
          [fileId]
        );

        // Decrement chunk references
        const allChunks = versions.flatMap(v => v.chunks);
        for (const chunk of allChunks) {
          await trx.query(`
            UPDATE chunks SET reference_count = reference_count - 1
            WHERE hash = $1
          `, [chunk]);
        }

        // Delete file
        await trx.query('DELETE FROM files WHERE id = $1', [fileId]);
      });
    } else {
      // Soft delete (move to trash)
      await this.db.query(`
        UPDATE files SET deleted_at = NOW() WHERE id = $1
      `, [fileId]);
    }
  }
}
```

---

## Block Storage

### Upload/Download with Multipart

```javascript
class BlockStorageService {
  constructor(s3Client) {
    this.s3 = s3Client;
    this.bucket = process.env.BLOCK_STORAGE_BUCKET;
  }

  async uploadChunk(hash, data) {
    // Check if already exists (dedup)
    try {
      await this.s3.headObject({
        Bucket: this.bucket,
        Key: hash
      });
      return { status: 'exists' };
    } catch (e) {
      if (e.code !== 'NotFound') throw e;
    }

    // Compress before storing
    const compressed = await this.compress(data);

    // Upload
    await this.s3.putObject({
      Bucket: this.bucket,
      Key: hash,
      Body: compressed,
      ContentType: 'application/octet-stream',
      Metadata: {
        'original-size': data.length.toString(),
        'compressed-size': compressed.length.toString()
      }
    });

    return { status: 'uploaded' };
  }

  async downloadChunk(hash) {
    const response = await this.s3.getObject({
      Bucket: this.bucket,
      Key: hash
    });

    const compressed = await this.streamToBuffer(response.Body);
    return this.decompress(compressed);
  }

  // Parallel download for file reconstruction
  async downloadFile(chunks, concurrency = 10) {
    const results = new Array(chunks.length);

    // Download in parallel with limited concurrency
    const semaphore = new Semaphore(concurrency);

    await Promise.all(
      chunks.map(async (hash, index) => {
        await semaphore.acquire();
        try {
          results[index] = await this.downloadChunk(hash);
        } finally {
          semaphore.release();
        }
      })
    );

    return Buffer.concat(results);
  }

  // Presigned URLs for direct client upload/download
  async getUploadUrl(hash, expiresIn = 3600) {
    return this.s3.getSignedUrl('putObject', {
      Bucket: this.bucket,
      Key: hash,
      Expires: expiresIn,
      ContentType: 'application/octet-stream'
    });
  }

  async getDownloadUrl(hash, expiresIn = 3600) {
    return this.s3.getSignedUrl('getObject', {
      Bucket: this.bucket,
      Key: hash,
      Expires: expiresIn
    });
  }
}
```

---

## Real-Time Sync

### Long Polling / WebSocket Notifications

```javascript
class NotificationService {
  constructor(redis) {
    this.redis = redis;
    this.connections = new Map();  // userId -> Set of WebSocket connections
  }

  async subscribe(userId, ws) {
    if (!this.connections.has(userId)) {
      this.connections.set(userId, new Set());
    }
    this.connections.get(userId).add(ws);

    // Send any pending notifications
    const pending = await this.redis.lrange(`notifications:${userId}`, 0, -1);
    for (const notification of pending) {
      ws.send(notification);
    }
    await this.redis.del(`notifications:${userId}`);

    ws.on('close', () => {
      this.connections.get(userId)?.delete(ws);
    });
  }

  async notify(userId, event) {
    const message = JSON.stringify(event);
    const connections = this.connections.get(userId);

    if (connections && connections.size > 0) {
      // User is online - send directly
      for (const ws of connections) {
        if (ws.readyState === WebSocket.OPEN) {
          ws.send(message);
        }
      }
    } else {
      // User offline - queue notification
      await this.redis.rpush(`notifications:${userId}`, message);
      await this.redis.expire(`notifications:${userId}`, 86400);  // 24 hours
    }

    // Also publish for other server instances
    await this.redis.publish(`user:${userId}:events`, message);
  }

  async notifyFileChange(file, changeType) {
    // Notify file owner
    await this.notify(file.userId, {
      type: 'FILE_CHANGED',
      fileId: file.id,
      path: file.path,
      changeType,  // 'created', 'modified', 'deleted', 'moved'
      version: file.version,
      timestamp: Date.now()
    });

    // Notify shared users
    const shares = await this.getFileShares(file.id);
    for (const share of shares) {
      await this.notify(share.sharedWith, {
        type: 'SHARED_FILE_CHANGED',
        fileId: file.id,
        path: file.path,
        changeType,
        sharedBy: file.userId
      });
    }
  }
}
```

---

## Client Architecture

### Desktop Client

```javascript
class DesktopSyncClient {
  constructor(syncFolder) {
    this.syncFolder = syncFolder;
    this.localDB = new SQLite('sync.db');
    this.watcher = null;
  }

  async start() {
    // Initialize local database
    await this.initLocalDB();

    // Start file system watcher
    this.startWatcher();

    // Connect to sync service
    await this.connectWebSocket();

    // Initial sync
    await this.fullSync();
  }

  startWatcher() {
    this.watcher = chokidar.watch(this.syncFolder, {
      ignored: /(^|[\/\\])\../,  // Ignore dotfiles
      persistent: true,
      ignoreInitial: true,
      awaitWriteFinish: {
        stabilityThreshold: 2000,
        pollInterval: 100
      }
    });

    this.watcher
      .on('add', path => this.handleLocalChange('add', path))
      .on('change', path => this.handleLocalChange('change', path))
      .on('unlink', path => this.handleLocalChange('delete', path));
  }

  async handleLocalChange(type, filePath) {
    // Debounce rapid changes
    this.pendingChanges.set(filePath, { type, timestamp: Date.now() });

    clearTimeout(this.debounceTimer);
    this.debounceTimer = setTimeout(() => this.processPendingChanges(), 500);
  }

  async processPendingChanges() {
    for (const [path, change] of this.pendingChanges) {
      try {
        if (change.type === 'delete') {
          await this.syncDelete(path);
        } else {
          await this.syncUpload(path);
        }
      } catch (error) {
        console.error(`Sync failed for ${path}:`, error);
        this.retryQueue.add(path);
      }
    }
    this.pendingChanges.clear();
  }

  async syncUpload(filePath) {
    const relativePath = path.relative(this.syncFolder, filePath);
    const fileData = await fs.readFile(filePath);

    // Chunk the file
    const chunker = new ContentDefinedChunker();
    const chunks = [];

    for (const chunk of chunker.chunk(fileData)) {
      chunks.push({
        hash: chunk.hash,
        data: chunk.data
      });
    }

    // Check which chunks need upload
    const chunkHashes = chunks.map(c => c.hash);
    const existingChunks = await this.api.checkChunks(chunkHashes);
    const newChunks = chunks.filter(c => !existingChunks.has(c.hash));

    // Upload new chunks
    for (const chunk of newChunks) {
      const uploadUrl = await this.api.getUploadUrl(chunk.hash);
      await this.uploadToUrl(uploadUrl, chunk.data);
    }

    // Update file metadata
    await this.api.updateFile(relativePath, chunkHashes);

    // Update local database
    await this.localDB.updateFile(relativePath, {
      chunks: chunkHashes,
      syncedAt: Date.now()
    });
  }
}
```

---

## Interview Discussion Points

### How to handle large files (50GB+)?

1. **Chunking** - Split into 4MB chunks
2. **Parallel upload** - Upload multiple chunks simultaneously
3. **Resumable uploads** - Track uploaded chunks, resume from failure
4. **Streaming** - Don't load entire file in memory
5. **Progress tracking** - Show upload progress per chunk

### How to minimize sync time?

1. **Delta sync** - Only transfer changed chunks
2. **Content-defined chunking** - Stable boundaries despite insertions
3. **Deduplication** - Don't re-upload existing chunks
4. **Compression** - Compress chunks before transfer
5. **Parallel transfers** - Concurrent chunk upload/download

### How to handle conflicts?

1. **Vector clocks** - Track causality
2. **Last-write-wins** - Simple but loses data
3. **Keep both** - Create conflict copies (Dropbox approach)
4. **Three-way merge** - For text files
5. **User resolution** - Present conflicts to user

### How to ensure consistency?

1. **Atomic file updates** - Version numbers, checksums
2. **Two-phase commit** - For critical operations
3. **Optimistic locking** - Check version before update
4. **Eventual consistency** - Accept for file content
5. **Strong consistency** - Required for metadata
