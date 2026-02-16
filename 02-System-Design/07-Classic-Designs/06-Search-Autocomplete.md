# Design: Search & Autocomplete System

## Requirements

### Functional Requirements
- Full-text search across content
- Autocomplete/typeahead suggestions
- Spelling correction
- Search ranking/relevance
- Filters and facets
- Recent searches

### Non-Functional Requirements
- Autocomplete latency <100ms
- Search latency <500ms
- Scale: 10K searches/second
- Near real-time indexing (seconds delay)
- High availability

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                            Clients                                    │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
┌────────────────────────────────▼─────────────────────────────────────┐
│                          API Gateway                                  │
└───────┬────────────────────────┬─────────────────────────────────────┘
        │                        │
        ▼                        ▼
┌───────────────┐       ┌───────────────┐
│  Autocomplete │       │    Search     │
│    Service    │       │    Service    │
└───────┬───────┘       └───────┬───────┘
        │                       │
        ▼                       ▼
┌───────────────┐       ┌───────────────┐
│   Trie/       │       │ Elasticsearch │
│   Redis       │       │    Cluster    │
└───────────────┘       └───────────────┘
                               ▲
                               │
┌──────────────────────────────┴───────────────────────────────────────┐
│                        Indexing Pipeline                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │
│  │  Data       │─▶│  Transform  │─▶│  Indexer    │                   │
│  │  Sources    │  │  (Kafka)    │  │  Workers    │                   │
│  └─────────────┘  └─────────────┘  └─────────────┘                   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Autocomplete

### Trie Data Structure

```javascript
class TrieNode {
  constructor() {
    this.children = {};
    this.isEndOfWord = false;
    this.weight = 0;        // Popularity/frequency
    this.suggestions = [];  // Top suggestions for this prefix
  }
}

class Trie {
  constructor() {
    this.root = new TrieNode();
  }

  insert(word, weight = 1) {
    let node = this.root;

    for (const char of word.toLowerCase()) {
      if (!node.children[char]) {
        node.children[char] = new TrieNode();
      }
      node = node.children[char];

      // Update top suggestions at each prefix node
      this.updateSuggestions(node, word, weight);
    }

    node.isEndOfWord = true;
    node.weight += weight;
  }

  updateSuggestions(node, word, weight) {
    // Keep top K suggestions
    const K = 10;

    // Check if word already in suggestions
    const existing = node.suggestions.find(s => s.word === word);
    if (existing) {
      existing.weight += weight;
    } else {
      node.suggestions.push({ word, weight });
    }

    // Sort by weight and keep top K
    node.suggestions.sort((a, b) => b.weight - a.weight);
    node.suggestions = node.suggestions.slice(0, K);
  }

  getSuggestions(prefix, limit = 10) {
    let node = this.root;

    for (const char of prefix.toLowerCase()) {
      if (!node.children[char]) {
        return []; // No matches
      }
      node = node.children[char];
    }

    return node.suggestions.slice(0, limit);
  }
}
```

### Redis-Based Autocomplete

```javascript
class RedisAutocomplete {
  constructor(redis) {
    this.redis = redis;
  }

  async add(text, score = 1) {
    // Add all prefixes to sorted set
    const word = text.toLowerCase();

    const pipeline = this.redis.pipeline();

    for (let i = 1; i <= word.length; i++) {
      const prefix = word.substring(0, i);
      pipeline.zincrby(`autocomplete:${prefix}`, score, word);
    }

    // Keep only top 100 per prefix
    pipeline.exec();
  }

  async getSuggestions(prefix, limit = 10) {
    const key = `autocomplete:${prefix.toLowerCase()}`;

    // Get top suggestions by score
    const results = await this.redis.zrevrange(key, 0, limit - 1, 'WITHSCORES');

    const suggestions = [];
    for (let i = 0; i < results.length; i += 2) {
      suggestions.push({
        text: results[i],
        score: parseFloat(results[i + 1])
      });
    }

    return suggestions;
  }

  // Update popularity based on selections
  async recordSelection(text) {
    await this.add(text, 1);
  }
}
```

### Autocomplete API

```javascript
app.get('/api/autocomplete', async (req, res) => {
  const { q, limit = 10 } = req.query;

  if (!q || q.length < 2) {
    return res.json({ suggestions: [] });
  }

  // Get suggestions
  let suggestions = await autocomplete.getSuggestions(q, limit);

  // Add user's recent searches
  const recent = await getRecentSearches(req.user?.id, 3);

  // Add trending searches
  const trending = await getTrendingSearches(5);

  // Merge and dedupe
  suggestions = mergeSuggestions(suggestions, recent, trending, limit);

  res.json({ suggestions });
});
```

---

## Full-Text Search

### Elasticsearch Setup

```javascript
// Index mapping
const indexMapping = {
  mappings: {
    properties: {
      title: {
        type: 'text',
        analyzer: 'english',
        fields: {
          keyword: { type: 'keyword' },
          autocomplete: {
            type: 'text',
            analyzer: 'autocomplete_analyzer'
          }
        }
      },
      description: {
        type: 'text',
        analyzer: 'english'
      },
      category: { type: 'keyword' },
      tags: { type: 'keyword' },
      price: { type: 'float' },
      rating: { type: 'float' },
      created_at: { type: 'date' },
      popularity: { type: 'integer' }
    }
  },
  settings: {
    analysis: {
      analyzer: {
        autocomplete_analyzer: {
          type: 'custom',
          tokenizer: 'standard',
          filter: ['lowercase', 'autocomplete_filter']
        }
      },
      filter: {
        autocomplete_filter: {
          type: 'edge_ngram',
          min_gram: 2,
          max_gram: 20
        }
      }
    }
  }
};
```

### Search Query

```javascript
async function search(query, filters = {}, options = {}) {
  const { page = 1, limit = 20, sort = 'relevance' } = options;

  const searchBody = {
    from: (page - 1) * limit,
    size: limit,
    query: {
      bool: {
        must: [
          {
            multi_match: {
              query,
              fields: ['title^3', 'description', 'tags^2'],
              type: 'best_fields',
              fuzziness: 'AUTO'  // Spelling correction
            }
          }
        ],
        filter: []
      }
    },
    highlight: {
      fields: {
        title: {},
        description: { fragment_size: 150 }
      }
    },
    aggs: {
      categories: { terms: { field: 'category' } },
      price_ranges: {
        range: {
          field: 'price',
          ranges: [
            { to: 50 },
            { from: 50, to: 100 },
            { from: 100, to: 500 },
            { from: 500 }
          ]
        }
      }
    }
  };

  // Add filters
  if (filters.category) {
    searchBody.query.bool.filter.push({
      term: { category: filters.category }
    });
  }

  if (filters.priceMin || filters.priceMax) {
    searchBody.query.bool.filter.push({
      range: {
        price: {
          gte: filters.priceMin,
          lte: filters.priceMax
        }
      }
    });
  }

  // Sorting
  if (sort === 'price_asc') {
    searchBody.sort = [{ price: 'asc' }];
  } else if (sort === 'price_desc') {
    searchBody.sort = [{ price: 'desc' }];
  } else if (sort === 'newest') {
    searchBody.sort = [{ created_at: 'desc' }];
  } else {
    // Relevance - combine text score with popularity
    searchBody.query = {
      function_score: {
        query: searchBody.query,
        functions: [
          {
            field_value_factor: {
              field: 'popularity',
              factor: 1.2,
              modifier: 'log1p'
            }
          }
        ],
        boost_mode: 'multiply'
      }
    };
  }

  const result = await elasticsearch.search({
    index: 'products',
    body: searchBody
  });

  return {
    total: result.hits.total.value,
    hits: result.hits.hits.map(hit => ({
      ...hit._source,
      score: hit._score,
      highlights: hit.highlight
    })),
    facets: {
      categories: result.aggregations.categories.buckets,
      priceRanges: result.aggregations.price_ranges.buckets
    }
  };
}
```

---

## Indexing Pipeline

```javascript
// CDC from database to Kafka to Elasticsearch
class IndexingPipeline {
  constructor() {
    this.kafka = new Kafka({ brokers: ['...'] });
    this.consumer = this.kafka.consumer({ groupId: 'indexer' });
    this.es = new Client({ node: 'http://elasticsearch:9200' });
  }

  async start() {
    await this.consumer.connect();
    await this.consumer.subscribe({ topic: 'product-changes' });

    await this.consumer.run({
      eachBatch: async ({ batch }) => {
        const operations = [];

        for (const message of batch.messages) {
          const event = JSON.parse(message.value.toString());

          if (event.operation === 'DELETE') {
            operations.push(
              { delete: { _index: 'products', _id: event.id } }
            );
          } else {
            operations.push(
              { index: { _index: 'products', _id: event.id } },
              this.transformDocument(event.data)
            );
          }
        }

        // Bulk index
        if (operations.length > 0) {
          await this.es.bulk({ body: operations });
        }
      }
    });
  }

  transformDocument(data) {
    return {
      title: data.name,
      description: data.description,
      category: data.category,
      tags: data.tags || [],
      price: data.price,
      rating: data.rating || 0,
      popularity: data.viewCount || 0,
      created_at: data.createdAt
    };
  }
}
```

---

## Spelling Correction

```javascript
class SpellChecker {
  constructor(elasticsearch) {
    this.es = elasticsearch;
  }

  async suggest(query) {
    const result = await this.es.search({
      index: 'products',
      body: {
        suggest: {
          text: query,
          spelling: {
            phrase: {
              field: 'title',
              size: 1,
              gram_size: 3,
              direct_generator: [{
                field: 'title',
                suggest_mode: 'popular'
              }],
              highlight: {
                pre_tag: '<em>',
                post_tag: '</em>'
              }
            }
          }
        }
      }
    });

    const suggestions = result.suggest.spelling[0].options;

    if (suggestions.length > 0) {
      return {
        corrected: suggestions[0].text,
        highlighted: suggestions[0].highlighted
      };
    }

    return null;
  }
}

// Usage in search
async function searchWithCorrection(query) {
  const results = await search(query);

  if (results.total < 5) {
    const correction = await spellChecker.suggest(query);

    if (correction) {
      const correctedResults = await search(correction.corrected);

      return {
        ...correctedResults,
        correction: {
          original: query,
          corrected: correction.corrected,
          message: `Showing results for "${correction.corrected}". Search for "${query}" instead?`
        }
      };
    }
  }

  return results;
}
```

---

## Recent & Trending Searches

```javascript
// Recent searches per user
class RecentSearches {
  async record(userId, query) {
    const key = `recent_searches:${userId}`;

    await redis.pipeline()
      .lpush(key, JSON.stringify({ query, timestamp: Date.now() }))
      .ltrim(key, 0, 49)  // Keep last 50
      .expire(key, 86400 * 30)  // 30 days
      .exec();
  }

  async get(userId, limit = 10) {
    const key = `recent_searches:${userId}`;
    const results = await redis.lrange(key, 0, limit - 1);
    return results.map(r => JSON.parse(r));
  }
}

// Trending searches (global)
class TrendingSearches {
  async record(query) {
    const hour = Math.floor(Date.now() / 3600000);
    const key = `trending:${hour}`;

    await redis.zincrby(key, 1, query.toLowerCase());
    await redis.expire(key, 7200);  // 2 hours
  }

  async get(limit = 10) {
    const hour = Math.floor(Date.now() / 3600000);
    const keys = [
      `trending:${hour}`,
      `trending:${hour - 1}`
    ];

    // Merge last 2 hours
    const tmpKey = `trending:merged:${hour}`;
    await redis.zunionstore(tmpKey, keys.length, ...keys);
    await redis.expire(tmpKey, 60);

    const results = await redis.zrevrange(tmpKey, 0, limit - 1, 'WITHSCORES');

    const trending = [];
    for (let i = 0; i < results.length; i += 2) {
      trending.push({
        query: results[i],
        count: parseFloat(results[i + 1])
      });
    }

    return trending;
  }
}
```

---

## Scaling Considerations

### Elasticsearch Cluster

```
Cluster sizing:
- 3 master nodes (dedicated)
- 6-10 data nodes
- Shards: 5-10 per index
- Replicas: 1-2

Index strategies:
- Time-based indices for logs
- Aliases for zero-downtime reindex
- Hot-warm architecture for cost
```

### Caching

```javascript
// Cache search results
async function searchWithCache(query, filters, options) {
  const cacheKey = `search:${hash(JSON.stringify({ query, filters, options }))}`;

  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  const results = await search(query, filters, options);

  // Cache for 5 minutes
  await redis.setex(cacheKey, 300, JSON.stringify(results));

  return results;
}
```

---

## Interview Discussion Points

### How to rank search results?

1. **Text relevance** (TF-IDF, BM25)
2. **Popularity signals** (views, sales, ratings)
3. **Freshness** (for time-sensitive content)
4. **Personalization** (user history)
5. **Business rules** (promoted items)

### How to handle misspellings?

1. **Fuzzy matching** (edit distance)
2. **Phonetic matching** (Soundex)
3. **n-gram similarity**
4. **Machine learning** (trained on search logs)
5. **"Did you mean?"** suggestions

### How to make autocomplete fast?

1. **Precomputed suggestions** per prefix
2. **In-memory data structures** (Trie, Redis)
3. **Edge caching** (CDN for popular prefixes)
4. **Debounce on client** (wait 100-200ms)
5. **Limit prefix length** (start at 2-3 chars)
