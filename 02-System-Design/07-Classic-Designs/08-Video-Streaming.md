# Design: Video Streaming System (Netflix/YouTube)

## Requirements

### Functional Requirements
- Upload videos
- Stream videos (adaptive bitrate)
- Search and browse videos
- User interactions (likes, comments, watch history)
- Recommendations

### Non-Functional Requirements
- High availability (99.99%)
- Low latency playback start (<2 seconds)
- Support millions of concurrent viewers
- Global delivery
- Scale: 1B daily video views

### Capacity Estimation

```
Daily active users: 200M
Videos watched per user: 5
Daily views: 1B
Concurrent viewers (peak): 10M

Video uploads: 500K/day
Average video: 10 min, 500MB (before processing)
Daily upload storage: 250TB

Bandwidth (streaming):
- Average bitrate: 5 Mbps
- Concurrent: 10M × 5 Mbps = 50 Pbps
- (This is why CDNs are critical!)
```

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                           Clients                                     │
│              (Mobile, TV, Web, Gaming Consoles)                       │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
┌────────────────────────────────▼─────────────────────────────────────┐
│                             CDN                                       │
│           (Edge servers worldwide - CloudFront, Akamai)               │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │ Cache miss
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         Origin Servers                                │
│    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│    │ Video API    │  │ User API     │  │ Search API   │              │
│    └──────────────┘  └──────────────┘  └──────────────┘              │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         ▼                       ▼                       ▼
┌────────────────┐      ┌────────────────┐      ┌────────────────┐
│ Video Storage  │      │  Metadata DB   │      │  Search Index  │
│    (S3)        │      │  (Cassandra)   │      │ (Elasticsearch)│
└────────────────┘      └────────────────┘      └────────────────┘
```

---

## Video Upload Pipeline

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  Client  │────▶│ Upload API   │────▶│ Raw Storage  │
└──────────┘     │ (Presigned)  │     │    (S3)      │
                 └──────────────┘     └──────┬───────┘
                                             │
                                      ┌──────▼───────┐
                                      │ Transcoding  │
                                      │   Queue      │
                                      └──────┬───────┘
                                             │
         ┌───────────────────────────────────┼───────────────────────────────────┐
         ▼                                   ▼                                   ▼
┌────────────────┐                  ┌────────────────┐                  ┌────────────────┐
│  Transcoder    │                  │  Transcoder    │                  │  Transcoder    │
│  (1080p, h264) │                  │  (720p, h264)  │                  │  (480p, h264)  │
└───────┬────────┘                  └───────┬────────┘                  └───────┬────────┘
        │                                   │                                   │
        └───────────────────────────────────┼───────────────────────────────────┘
                                            ▼
                                   ┌────────────────┐
                                   │ Processed      │
                                   │ Storage (S3)   │
                                   └───────┬────────┘
                                           │
                                    ┌──────▼───────┐
                                    │  CDN Push    │
                                    └──────────────┘
```

### Upload Implementation

```javascript
// Generate presigned URL for direct upload
app.post('/api/videos/upload-url', async (req, res) => {
  const { filename, contentType, fileSize } = req.body;

  // Validate
  if (fileSize > MAX_FILE_SIZE) {
    return res.status(400).json({ error: 'File too large' });
  }

  const videoId = generateId();
  const key = `raw/${videoId}/${filename}`;

  // Generate presigned URL (valid for 1 hour)
  const uploadUrl = await s3.getSignedUrl('putObject', {
    Bucket: RAW_BUCKET,
    Key: key,
    ContentType: contentType,
    Expires: 3600
  });

  // Create video record
  await db.videos.create({
    id: videoId,
    userId: req.user.id,
    status: 'uploading',
    rawPath: key
  });

  res.json({ videoId, uploadUrl });
});

// Callback after upload complete
app.post('/api/videos/:id/uploaded', async (req, res) => {
  const { id } = req.params;

  // Trigger transcoding
  await transcodingQueue.push({
    videoId: id,
    rawPath: await db.videos.findById(id).rawPath
  });

  await db.videos.update(id, { status: 'processing' });

  res.json({ status: 'processing' });
});
```

### Transcoding

```javascript
// Transcoding worker
class TranscodingWorker {
  async process(job) {
    const { videoId, rawPath } = job;

    // Download raw file
    const rawFile = await s3.getObject({ Bucket: RAW_BUCKET, Key: rawPath });

    // Define output formats
    const formats = [
      { name: '1080p', resolution: '1920x1080', bitrate: '5000k' },
      { name: '720p', resolution: '1280x720', bitrate: '2500k' },
      { name: '480p', resolution: '854x480', bitrate: '1000k' },
      { name: '360p', resolution: '640x360', bitrate: '500k' }
    ];

    // Transcode to each format
    for (const format of formats) {
      const outputPath = `processed/${videoId}/${format.name}`;

      await ffmpeg(rawFile)
        .outputOptions([
          `-vf scale=${format.resolution}`,
          `-b:v ${format.bitrate}`,
          '-c:v libx264',
          '-c:a aac',
          '-movflags +faststart'  // Enable streaming
        ])
        .save(outputPath);

      // Upload to S3
      await s3.upload({
        Bucket: PROCESSED_BUCKET,
        Key: outputPath,
        Body: fs.readFileSync(outputPath)
      });
    }

    // Generate HLS/DASH manifest
    await this.generateManifest(videoId, formats);

    // Update video status
    await db.videos.update(videoId, { status: 'ready' });

    // Push to CDN
    await this.pushToCDN(videoId);
  }

  async generateManifest(videoId, formats) {
    // HLS Master Playlist
    const manifest = `#EXTM3U
#EXT-X-VERSION:3
${formats.map(f => `
#EXT-X-STREAM-INF:BANDWIDTH=${parseInt(f.bitrate) * 1000},RESOLUTION=${f.resolution}
${f.name}/playlist.m3u8
`).join('')}`;

    await s3.putObject({
      Bucket: PROCESSED_BUCKET,
      Key: `processed/${videoId}/master.m3u8`,
      Body: manifest,
      ContentType: 'application/x-mpegURL'
    });
  }
}
```

---

## Adaptive Bitrate Streaming

### HLS (HTTP Live Streaming)

```
Master Playlist (master.m3u8):
┌─────────────────────────────────┐
│ #EXTM3U                         │
│ #EXT-X-STREAM-INF:BANDWIDTH=5M  │
│ 1080p/playlist.m3u8             │
│ #EXT-X-STREAM-INF:BANDWIDTH=2.5M│
│ 720p/playlist.m3u8              │
│ #EXT-X-STREAM-INF:BANDWIDTH=1M  │
│ 480p/playlist.m3u8              │
└─────────────────────────────────┘

Variant Playlist (1080p/playlist.m3u8):
┌─────────────────────────────────┐
│ #EXTM3U                         │
│ #EXT-X-TARGETDURATION:10        │
│ #EXTINF:10.0,                   │
│ segment_001.ts                  │
│ #EXTINF:10.0,                   │
│ segment_002.ts                  │
│ ...                             │
└─────────────────────────────────┘
```

### Client-Side ABR

```javascript
// Video player with ABR
class AdaptivePlayer {
  constructor(videoElement, manifestUrl) {
    this.video = videoElement;
    this.manifestUrl = manifestUrl;
    this.currentQuality = 'auto';
    this.bandwidthHistory = [];
  }

  async load() {
    // Parse master manifest
    const manifest = await fetch(this.manifestUrl).then(r => r.text());
    this.variants = this.parseManifest(manifest);

    // Start with lowest quality
    this.selectQuality(this.variants[this.variants.length - 1]);

    // Monitor bandwidth and switch qualities
    this.startBandwidthMonitor();
  }

  selectQuality(variant) {
    this.currentVariant = variant;
    this.loadVariantPlaylist(variant.url);
  }

  startBandwidthMonitor() {
    setInterval(() => {
      const bandwidth = this.measureBandwidth();
      this.bandwidthHistory.push(bandwidth);

      // Keep last 5 measurements
      if (this.bandwidthHistory.length > 5) {
        this.bandwidthHistory.shift();
      }

      // Average bandwidth
      const avgBandwidth = this.bandwidthHistory.reduce((a, b) => a + b) /
        this.bandwidthHistory.length;

      // Select appropriate quality
      const newVariant = this.variants.find(v =>
        v.bandwidth < avgBandwidth * 0.8  // 80% of available bandwidth
      );

      if (newVariant && newVariant !== this.currentVariant) {
        this.selectQuality(newVariant);
      }
    }, 5000);
  }
}
```

---

## CDN Architecture

```
                         ┌──────────────────┐
                         │   Origin Server  │
                         │   (Video API)    │
                         └────────┬─────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
       ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
       │  Regional    │   │  Regional    │   │  Regional    │
       │  Edge (US)   │   │  Edge (EU)   │   │  Edge (Asia) │
       └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
              │                  │                  │
    ┌─────────┼─────────┐       ...               ...
    ▼         ▼         ▼
┌────────┐┌────────┐┌────────┐
│ Edge   ││ Edge   ││ Edge   │
│ NYC    ││ LA     ││ Chicago│
└────────┘└────────┘└────────┘
    ▲         ▲         ▲
    │         │         │
  Users     Users     Users
```

### CDN Strategy

```javascript
// CDN edge configuration
const cdnConfig = {
  // Cache video segments for 24 hours
  cacheRules: [
    { pattern: '*.ts', maxAge: 86400 },      // Segments
    { pattern: '*.m3u8', maxAge: 60 },        // Playlists (shorter for live)
    { pattern: 'thumbnail/*', maxAge: 604800 } // Thumbnails (1 week)
  ],

  // Origin shield to reduce origin load
  originShield: {
    enabled: true,
    region: 'us-east-1'
  },

  // Geographic routing
  geoRouting: {
    'NA': 'cdn-na.example.com',
    'EU': 'cdn-eu.example.com',
    'APAC': 'cdn-apac.example.com'
  }
};

// Client gets nearest CDN URL
app.get('/api/videos/:id/playback', async (req, res) => {
  const video = await db.videos.findById(req.params.id);
  const userRegion = geoip.lookup(req.ip).region;
  const cdnDomain = cdnConfig.geoRouting[userRegion] || cdnConfig.geoRouting['NA'];

  res.json({
    manifestUrl: `https://${cdnDomain}/videos/${video.id}/master.m3u8`,
    thumbnailUrl: `https://${cdnDomain}/thumbnails/${video.id}.jpg`
  });
});
```

---

## Watch Progress & Resume

```javascript
// Track watch progress
app.post('/api/videos/:id/progress', async (req, res) => {
  const { position, duration } = req.body;

  await redis.hset(`progress:${req.user.id}`, req.params.id, JSON.stringify({
    position,
    duration,
    updatedAt: Date.now()
  }));

  // If watched > 90%, mark as completed
  if (position / duration > 0.9) {
    await addToWatchHistory(req.user.id, req.params.id);
  }

  res.json({ success: true });
});

// Get resume position
app.get('/api/videos/:id/progress', async (req, res) => {
  const progress = await redis.hget(`progress:${req.user.id}`, req.params.id);

  if (progress) {
    res.json(JSON.parse(progress));
  } else {
    res.json({ position: 0 });
  }
});
```

---

## Recommendations

```javascript
// Collaborative filtering (simplified)
class RecommendationEngine {
  async getRecommendations(userId, limit = 20) {
    // Get user's watch history
    const watchHistory = await getWatchHistory(userId);

    // Find similar users (same viewing patterns)
    const similarUsers = await this.findSimilarUsers(userId, watchHistory);

    // Get videos watched by similar users but not by this user
    const candidates = await this.getCandidateVideos(
      similarUsers,
      watchHistory
    );

    // Score and rank
    const scored = candidates.map(video => ({
      video,
      score: this.calculateScore(video, userId, similarUsers)
    }));

    return scored
      .sort((a, b) => b.score - a.score)
      .slice(0, limit)
      .map(s => s.video);
  }

  calculateScore(video, userId, similarUsers) {
    let score = 0;

    // Popularity among similar users
    score += video.watchCountSimilar * 0.3;

    // Recency
    score += (1 / Math.log(daysSinceUpload(video) + 1)) * 0.2;

    // Genre match
    score += this.genreMatchScore(video, userId) * 0.3;

    // Engagement (likes, completion rate)
    score += video.engagement * 0.2;

    return score;
  }
}
```

---

## Database Schema

```sql
-- Videos
CREATE TABLE videos (
  id UUID PRIMARY KEY,
  user_id UUID,
  title VARCHAR(255),
  description TEXT,
  duration_seconds INT,
  status VARCHAR(20),  -- uploading, processing, ready, failed
  raw_path TEXT,
  thumbnail_url TEXT,
  created_at TIMESTAMP,
  published_at TIMESTAMP,
  view_count BIGINT DEFAULT 0,
  like_count BIGINT DEFAULT 0
);

-- Video formats (for each transcoded version)
CREATE TABLE video_formats (
  video_id UUID,
  quality VARCHAR(10),  -- 1080p, 720p, etc.
  path TEXT,
  bitrate INT,
  PRIMARY KEY (video_id, quality)
);

-- Watch history (Cassandra for scale)
CREATE TABLE watch_history (
  user_id UUID,
  watched_at TIMESTAMP,
  video_id UUID,
  watch_duration INT,
  PRIMARY KEY ((user_id), watched_at)
) WITH CLUSTERING ORDER BY (watched_at DESC);
```

---

## Interview Discussion Points

### How to handle live streaming?

1. **Ingest**: RTMP/WebRTC to origin
2. **Transcode**: Real-time transcoding to HLS/DASH
3. **Short segments**: 2-4 seconds for low latency
4. **Edge caching**: Short TTL, origin shield
5. **DVR**: Store recent segments for rewind

### How to reduce buffering?

1. **Adaptive bitrate** - Match quality to bandwidth
2. **Predictive buffering** - Pre-fetch next segments
3. **CDN placement** - Edge servers near users
4. **Start quality** - Begin low, increase
5. **TCP optimization** - BBR congestion control

### How to handle viral videos?

1. **CDN scaling** - Auto-scale edge capacity
2. **Origin protection** - Cache at edge aggressively
3. **Pre-positioning** - Push popular content to edges
4. **Traffic shaping** - Prioritize segments over manifests
