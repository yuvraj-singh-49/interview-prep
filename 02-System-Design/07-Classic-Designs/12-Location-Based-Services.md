# Design: Location-Based Services (Uber/Yelp)

## Requirements

### Functional Requirements
- Find nearby places/drivers within radius
- Real-time location tracking
- ETA calculation
- Search with filters (rating, cuisine, price)
- Location history

### Non-Functional Requirements
- Low latency (<100ms for nearby search)
- Real-time updates (driver locations)
- High availability (99.99%)
- Scale: 100M searches/day, 1M concurrent drivers

### Capacity Estimation

```
Places (Yelp-like): 200M businesses
Drivers (Uber-like): 1M concurrent active
Location updates: 1M drivers × 1 update/4 sec = 250K updates/sec
Nearby searches: 100M/day = 1200/sec
Data per location: 50 bytes

Storage:
- Places: 200M × 1KB = 200GB
- Driver locations: 1M × 50 bytes = 50MB (in-memory)
```

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Mobile Clients                                  │
│                   (GPS updates, Nearby searches)                         │
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
│  Location       │    │  Search         │    │  Matching       │       │
│  Service        │    │  Service        │    │  Service        │       │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘       │
         │                      │                      │                 │
         ▼                      ▼                      │                 │
┌─────────────────┐    ┌─────────────────┐            │                 │
│  Location       │    │  Elasticsearch  │            │                 │
│  Cache (Redis)  │    │  (Places)       │            │                 │
└─────────────────┘    └─────────────────┘            │                 │
         │                                             │                 │
         ▼                                             │                 │
┌─────────────────┐                                   │                 │
│  Geospatial     │◄──────────────────────────────────┘                 │
│  Index          │                                                     │
│  (QuadTree/     │                                                     │
│   Geohash)      │                                                     │
└─────────────────┘                                                     │
```

---

## Geospatial Indexing

### Geohash

```
Geohash divides Earth into grid cells with hierarchical precision:
- 1 char: 5,000km × 5,000km
- 4 chars: 39km × 20km
- 6 chars: 1.2km × 0.6km
- 8 chars: 38m × 19m
- 12 chars: 3.7cm × 1.8cm

Example: "9q8yyk" = San Francisco area

Nearby cells share prefix:
9q8yy[k,m,j,h] are all adjacent
```

```javascript
class Geohash {
  static BASE32 = '0123456789bcdefghjkmnpqrstuvwxyz';

  static encode(lat, lon, precision = 12) {
    let minLat = -90, maxLat = 90;
    let minLon = -180, maxLon = 180;
    let hash = '';
    let bit = 0;
    let ch = 0;
    let isLon = true;

    while (hash.length < precision) {
      if (isLon) {
        const mid = (minLon + maxLon) / 2;
        if (lon >= mid) {
          ch |= (1 << (4 - bit));
          minLon = mid;
        } else {
          maxLon = mid;
        }
      } else {
        const mid = (minLat + maxLat) / 2;
        if (lat >= mid) {
          ch |= (1 << (4 - bit));
          minLat = mid;
        } else {
          maxLat = mid;
        }
      }

      isLon = !isLon;
      bit++;

      if (bit === 5) {
        hash += this.BASE32[ch];
        bit = 0;
        ch = 0;
      }
    }

    return hash;
  }

  static decode(hash) {
    let minLat = -90, maxLat = 90;
    let minLon = -180, maxLon = 180;
    let isLon = true;

    for (const char of hash) {
      const bits = this.BASE32.indexOf(char);

      for (let i = 4; i >= 0; i--) {
        const bit = (bits >> i) & 1;

        if (isLon) {
          const mid = (minLon + maxLon) / 2;
          if (bit) {
            minLon = mid;
          } else {
            maxLon = mid;
          }
        } else {
          const mid = (minLat + maxLat) / 2;
          if (bit) {
            minLat = mid;
          } else {
            maxLat = mid;
          }
        }
        isLon = !isLon;
      }
    }

    return {
      lat: (minLat + maxLat) / 2,
      lon: (minLon + maxLon) / 2,
      bounds: { minLat, maxLat, minLon, maxLon }
    };
  }

  static getNeighbors(hash) {
    // Return all 8 adjacent geohash cells
    const { lat, lon, bounds } = this.decode(hash);
    const latDiff = bounds.maxLat - bounds.minLat;
    const lonDiff = bounds.maxLon - bounds.minLon;

    const neighbors = [];
    for (const dLat of [-latDiff, 0, latDiff]) {
      for (const dLon of [-lonDiff, 0, lonDiff]) {
        if (dLat === 0 && dLon === 0) continue;
        neighbors.push(this.encode(lat + dLat, lon + dLon, hash.length));
      }
    }
    return neighbors;
  }
}
```

### QuadTree

```javascript
class QuadTree {
  constructor(boundary, capacity = 4) {
    this.boundary = boundary;  // { x, y, width, height }
    this.capacity = capacity;
    this.points = [];
    this.divided = false;
    this.northwest = null;
    this.northeast = null;
    this.southwest = null;
    this.southeast = null;
  }

  insert(point) {
    // Point: { x (lon), y (lat), data }
    if (!this.contains(point)) {
      return false;
    }

    if (this.points.length < this.capacity && !this.divided) {
      this.points.push(point);
      return true;
    }

    if (!this.divided) {
      this.subdivide();
    }

    return (
      this.northwest.insert(point) ||
      this.northeast.insert(point) ||
      this.southwest.insert(point) ||
      this.southeast.insert(point)
    );
  }

  subdivide() {
    const { x, y, width, height } = this.boundary;
    const halfW = width / 2;
    const halfH = height / 2;

    this.northwest = new QuadTree({ x, y, width: halfW, height: halfH }, this.capacity);
    this.northeast = new QuadTree({ x: x + halfW, y, width: halfW, height: halfH }, this.capacity);
    this.southwest = new QuadTree({ x, y: y + halfH, width: halfW, height: halfH }, this.capacity);
    this.southeast = new QuadTree({ x: x + halfW, y: y + halfH, width: halfW, height: halfH }, this.capacity);

    this.divided = true;

    // Redistribute existing points
    for (const point of this.points) {
      this.northwest.insert(point) ||
      this.northeast.insert(point) ||
      this.southwest.insert(point) ||
      this.southeast.insert(point);
    }
    this.points = [];
  }

  queryRange(range) {
    const found = [];

    if (!this.intersects(range)) {
      return found;
    }

    for (const point of this.points) {
      if (this.rangeContains(range, point)) {
        found.push(point);
      }
    }

    if (this.divided) {
      found.push(...this.northwest.queryRange(range));
      found.push(...this.northeast.queryRange(range));
      found.push(...this.southwest.queryRange(range));
      found.push(...this.southeast.queryRange(range));
    }

    return found;
  }

  // Query within radius (circular range)
  queryRadius(centerLat, centerLon, radiusKm) {
    // Convert radius to approximate bounding box
    const latDelta = radiusKm / 111;  // ~111km per degree latitude
    const lonDelta = radiusKm / (111 * Math.cos(centerLat * Math.PI / 180));

    const range = {
      x: centerLon - lonDelta,
      y: centerLat - latDelta,
      width: lonDelta * 2,
      height: latDelta * 2
    };

    // Get candidates from bounding box
    const candidates = this.queryRange(range);

    // Filter by actual distance
    return candidates.filter(point => {
      const distance = this.haversineDistance(
        centerLat, centerLon,
        point.y, point.x
      );
      return distance <= radiusKm;
    });
  }

  haversineDistance(lat1, lon1, lat2, lon2) {
    const R = 6371;  // Earth radius in km
    const dLat = (lat2 - lat1) * Math.PI / 180;
    const dLon = (lon2 - lon1) * Math.PI / 180;

    const a = Math.sin(dLat / 2) ** 2 +
              Math.cos(lat1 * Math.PI / 180) *
              Math.cos(lat2 * Math.PI / 180) *
              Math.sin(dLon / 2) ** 2;

    return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  }
}
```

---

## Redis Geospatial (Production Solution)

```javascript
class LocationService {
  constructor(redis) {
    this.redis = redis;
  }

  // Add/Update driver location
  async updateDriverLocation(driverId, lat, lon) {
    await this.redis.geoadd('driver_locations', lon, lat, driverId);

    // Store additional driver info
    await this.redis.hset(`driver:${driverId}`, {
      lat,
      lon,
      updatedAt: Date.now(),
      status: 'available'
    });

    // Set expiry for stale locations
    await this.redis.expire(`driver:${driverId}`, 60);  // 1 minute
  }

  // Find drivers within radius
  async findNearbyDrivers(lat, lon, radiusKm, limit = 20) {
    // GEORADIUS returns drivers sorted by distance
    const results = await this.redis.georadius(
      'driver_locations',
      lon, lat,
      radiusKm, 'km',
      'WITHDIST',
      'WITHCOORD',
      'COUNT', limit,
      'ASC'  // Nearest first
    );

    // Enrich with driver details
    const drivers = await Promise.all(
      results.map(async ([driverId, distance, [lon, lat]]) => {
        const info = await this.redis.hgetall(`driver:${driverId}`);
        return {
          driverId,
          distance: parseFloat(distance),
          lat: parseFloat(lat),
          lon: parseFloat(lon),
          ...info
        };
      })
    );

    // Filter only available drivers
    return drivers.filter(d => d.status === 'available');
  }

  // Remove driver (went offline)
  async removeDriver(driverId) {
    await this.redis.zrem('driver_locations', driverId);
    await this.redis.del(`driver:${driverId}`);
  }
}
```

---

## Real-Time Location Tracking

### WebSocket Location Updates

```javascript
class LocationTrackingService {
  constructor(wss, redis) {
    this.wss = wss;
    this.redis = redis;
    this.subscriptions = new Map();  // rideId -> Set of WebSocket
  }

  // Driver sends location update
  async handleDriverUpdate(driverId, lat, lon, rideId) {
    // Update location in Redis
    await this.redis.geoadd('driver_locations', lon, lat, driverId);

    // If driver is on a ride, notify rider
    if (rideId) {
      const message = JSON.stringify({
        type: 'DRIVER_LOCATION',
        driverId,
        lat,
        lon,
        timestamp: Date.now()
      });

      // Send to all subscribers of this ride
      const subscribers = this.subscriptions.get(rideId);
      if (subscribers) {
        for (const ws of subscribers) {
          if (ws.readyState === WebSocket.OPEN) {
            ws.send(message);
          }
        }
      }

      // Update ETA
      await this.updateETA(rideId, lat, lon);
    }
  }

  // Rider subscribes to driver location
  subscribeToRide(rideId, ws) {
    if (!this.subscriptions.has(rideId)) {
      this.subscriptions.set(rideId, new Set());
    }
    this.subscriptions.get(rideId).add(ws);

    ws.on('close', () => {
      this.subscriptions.get(rideId)?.delete(ws);
    });
  }

  async updateETA(rideId, driverLat, driverLon) {
    const ride = await this.getRide(rideId);

    // Calculate ETA using routing service
    const eta = await this.routingService.calculateETA(
      driverLat, driverLon,
      ride.pickupLat, ride.pickupLon
    );

    // Notify rider
    const subscribers = this.subscriptions.get(rideId);
    if (subscribers) {
      const message = JSON.stringify({
        type: 'ETA_UPDATE',
        eta,
        timestamp: Date.now()
      });

      for (const ws of subscribers) {
        if (ws.readyState === WebSocket.OPEN) {
          ws.send(message);
        }
      }
    }
  }
}
```

### Location Update Batching

```javascript
class LocationBatcher {
  constructor(locationService) {
    this.locationService = locationService;
    this.batch = [];
    this.batchSize = 1000;
    this.flushInterval = 100;  // ms

    setInterval(() => this.flush(), this.flushInterval);
  }

  add(driverId, lat, lon) {
    this.batch.push({ driverId, lat, lon });

    if (this.batch.length >= this.batchSize) {
      this.flush();
    }
  }

  async flush() {
    if (this.batch.length === 0) return;

    const toProcess = this.batch;
    this.batch = [];

    // Batch GEOADD
    const pipeline = this.locationService.redis.pipeline();

    for (const { driverId, lat, lon } of toProcess) {
      pipeline.geoadd('driver_locations', lon, lat, driverId);
    }

    await pipeline.exec();
  }
}
```

---

## Places Search (Yelp-like)

### Elasticsearch Geospatial

```javascript
// Index mapping
const placesMapping = {
  mappings: {
    properties: {
      name: { type: 'text', analyzer: 'english' },
      location: { type: 'geo_point' },
      address: { type: 'text' },
      categories: { type: 'keyword' },
      rating: { type: 'float' },
      reviewCount: { type: 'integer' },
      priceLevel: { type: 'integer' },
      hours: { type: 'object' },
      attributes: { type: 'keyword' }
    }
  }
};

class PlacesSearchService {
  async searchNearby(params) {
    const {
      lat, lon, radiusKm = 5,
      query, categories, priceLevel,
      minRating, sortBy = 'distance',
      page = 1, limit = 20
    } = params;

    const must = [];
    const filter = [];

    // Text search
    if (query) {
      must.push({
        multi_match: {
          query,
          fields: ['name^3', 'categories^2', 'address']
        }
      });
    }

    // Geo filter
    filter.push({
      geo_distance: {
        distance: `${radiusKm}km`,
        location: { lat, lon }
      }
    });

    // Category filter
    if (categories?.length) {
      filter.push({ terms: { categories } });
    }

    // Price level filter
    if (priceLevel) {
      filter.push({ term: { priceLevel } });
    }

    // Rating filter
    if (minRating) {
      filter.push({ range: { rating: { gte: minRating } } });
    }

    // Build query
    const searchBody = {
      from: (page - 1) * limit,
      size: limit,
      query: {
        bool: {
          must: must.length ? must : [{ match_all: {} }],
          filter
        }
      }
    };

    // Sorting
    if (sortBy === 'distance') {
      searchBody.sort = [{
        _geo_distance: {
          location: { lat, lon },
          order: 'asc',
          unit: 'km'
        }
      }];
    } else if (sortBy === 'rating') {
      searchBody.sort = [{ rating: 'desc' }];
    } else if (sortBy === 'reviewCount') {
      searchBody.sort = [{ reviewCount: 'desc' }];
    }

    // Execute search
    const result = await this.es.search({
      index: 'places',
      body: searchBody
    });

    return {
      total: result.hits.total.value,
      places: result.hits.hits.map(hit => ({
        ...hit._source,
        distance: hit.sort?.[0]
      }))
    };
  }
}
```

---

## Driver Matching (Uber-like)

```javascript
class MatchingService {
  constructor(locationService, redis) {
    this.locationService = locationService;
    this.redis = redis;
  }

  async findAndMatchDriver(rideRequest) {
    const { pickupLat, pickupLon, riderId } = rideRequest;

    // Find nearby available drivers
    const nearbyDrivers = await this.locationService.findNearbyDrivers(
      pickupLat, pickupLon, 5  // 5km radius
    );

    if (nearbyDrivers.length === 0) {
      return { matched: false, reason: 'NO_DRIVERS' };
    }

    // Score and rank drivers
    const scoredDrivers = await this.scoreDrivers(nearbyDrivers, rideRequest);

    // Try to match with best driver
    for (const driver of scoredDrivers) {
      const accepted = await this.offerRideToDriver(driver, rideRequest);

      if (accepted) {
        await this.createRide(rideRequest, driver);
        return { matched: true, driver };
      }
    }

    return { matched: false, reason: 'NO_ACCEPTANCE' };
  }

  async scoreDrivers(drivers, rideRequest) {
    const scored = await Promise.all(
      drivers.map(async driver => {
        let score = 0;

        // Distance (closer = better)
        score += (5 - driver.distance) * 20;

        // Rating
        score += driver.rating * 10;

        // Acceptance rate
        const stats = await this.getDriverStats(driver.driverId);
        score += stats.acceptanceRate * 10;

        // Vehicle match (if rider requested specific type)
        if (rideRequest.vehicleType === driver.vehicleType) {
          score += 15;
        }

        // ETA to pickup
        const eta = await this.calculateETA(driver, rideRequest);
        score += Math.max(0, 15 - eta);  // Penalty for longer ETA

        return { ...driver, score, eta };
      })
    );

    return scored.sort((a, b) => b.score - a.score);
  }

  async offerRideToDriver(driver, rideRequest) {
    // Create ride offer
    const offer = {
      offerId: generateId(),
      rideRequest,
      driverId: driver.driverId,
      expiresAt: Date.now() + 15000  // 15 seconds to accept
    };

    // Store offer
    await this.redis.setex(
      `offer:${offer.offerId}`,
      20,
      JSON.stringify(offer)
    );

    // Send to driver
    await this.notifyDriver(driver.driverId, {
      type: 'RIDE_OFFER',
      offer
    });

    // Wait for response
    return new Promise((resolve) => {
      const checkResponse = async () => {
        const response = await this.redis.get(`offer_response:${offer.offerId}`);

        if (response === 'ACCEPTED') {
          resolve(true);
        } else if (response === 'DECLINED' || Date.now() > offer.expiresAt) {
          resolve(false);
        } else {
          setTimeout(checkResponse, 500);
        }
      };

      checkResponse();
    });
  }
}
```

---

## Sharding Strategy

### Location-Based Sharding

```javascript
// Shard by geohash prefix
class GeoShardRouter {
  constructor(shardCount = 256) {
    this.shardCount = shardCount;
  }

  getShardKey(lat, lon) {
    // Use first 2 chars of geohash (32 × 32 = 1024 possible cells)
    const geohash = Geohash.encode(lat, lon, 2);
    return this.hashToShard(geohash);
  }

  hashToShard(geohash) {
    let hash = 0;
    for (const char of geohash) {
      hash = ((hash << 5) - hash + char.charCodeAt(0)) | 0;
    }
    return Math.abs(hash) % this.shardCount;
  }

  // Get all shards that might contain results for a radius query
  getShardsForRadius(centerLat, centerLon, radiusKm) {
    const shards = new Set();

    // Get geohash of center
    const centerHash = Geohash.encode(centerLat, centerLon, 2);
    shards.add(this.hashToShard(centerHash));

    // Add neighboring cells
    const neighbors = Geohash.getNeighbors(centerHash);
    for (const neighbor of neighbors) {
      shards.add(this.hashToShard(neighbor));
    }

    return Array.from(shards);
  }
}
```

---

## Database Schema

```sql
-- Places
CREATE TABLE places (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  location GEOGRAPHY(POINT, 4326) NOT NULL,
  address TEXT,
  city VARCHAR(100),
  country VARCHAR(2),
  categories TEXT[],
  rating DECIMAL(2,1),
  review_count INT DEFAULT 0,
  price_level INT,
  hours JSONB,
  attributes JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

-- PostGIS spatial index
CREATE INDEX idx_places_location ON places USING GIST(location);
CREATE INDEX idx_places_city ON places(city);

-- Drivers
CREATE TABLE drivers (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  vehicle_type VARCHAR(20),
  license_plate VARCHAR(20),
  rating DECIMAL(2,1),
  total_rides INT DEFAULT 0,
  status VARCHAR(20) DEFAULT 'offline',
  current_location GEOGRAPHY(POINT, 4326)
);

-- Driver location history (time-series)
CREATE TABLE driver_locations (
  driver_id UUID,
  recorded_at TIMESTAMP,
  location GEOGRAPHY(POINT, 4326),
  PRIMARY KEY (driver_id, recorded_at)
);

-- Rides
CREATE TABLE rides (
  id UUID PRIMARY KEY,
  rider_id UUID,
  driver_id UUID,
  pickup_location GEOGRAPHY(POINT, 4326),
  dropoff_location GEOGRAPHY(POINT, 4326),
  pickup_address TEXT,
  dropoff_address TEXT,
  status VARCHAR(20),
  requested_at TIMESTAMP,
  accepted_at TIMESTAMP,
  pickup_at TIMESTAMP,
  completed_at TIMESTAMP,
  fare DECIMAL(10,2),
  distance_km DECIMAL(6,2)
);
```

---

## Interview Discussion Points

### How to handle high-frequency location updates?

1. **Batching** - Aggregate updates, flush periodically
2. **In-memory store** - Redis for hot data
3. **Sampling** - Update every N seconds, not continuously
4. **Delta compression** - Only send if moved significantly
5. **Regional sharding** - Distribute by geography

### How to find nearest efficiently?

1. **Geohash indexing** - O(1) cell lookup + neighbors
2. **QuadTree** - O(log n) for range queries
3. **R-Tree** - Optimized for rectangles
4. **Redis GEO** - Built-in O(log n) + k
5. **PostGIS** - Spatial indexes for SQL

### How to handle cross-shard queries?

1. **Geohash routing** - Query only relevant shards
2. **Scatter-gather** - Query all, merge results
3. **Hierarchical caching** - Regional aggregations
4. **Approximate results** - Expand radius if few results
5. **Smart routing** - Pre-compute hot areas

### How to ensure consistency for matching?

1. **Distributed locks** - Prevent double-booking
2. **Optimistic locking** - Check driver availability
3. **Reservation pattern** - Temporary hold
4. **Eventual consistency** - Acceptable with compensations
5. **Single-writer per driver** - Partition by driver ID
