# Graphs

## Overview

Graphs are one of the most versatile data structures and appear frequently in interviews, especially for Lead-level positions. You should be comfortable with BFS, DFS, shortest paths, topological sort, and union-find.

---

## Key Concepts

### Graph Terminology

```
Vertices (V): Nodes in the graph
Edges (E): Connections between vertices
Directed: Edges have direction (A → B)
Undirected: Edges are bidirectional (A — B)
Weighted: Edges have costs/weights
Unweighted: All edges have equal weight

Degree: Number of edges connected to a vertex
  - In-degree: Incoming edges (directed)
  - Out-degree: Outgoing edges (directed)

Path: Sequence of vertices connected by edges
Cycle: Path that starts and ends at same vertex
Connected: Path exists between any two vertices
DAG: Directed Acyclic Graph
```

### Graph Representations

#### Adjacency List (Most Common)

```python
# Using dictionary of lists
graph = {
    'A': ['B', 'C'],
    'B': ['A', 'D'],
    'C': ['A', 'D'],
    'D': ['B', 'C']
}

# Using list of lists (when vertices are 0 to n-1)
graph = [
    [1, 2],      # 0 -> 1, 2
    [0, 3],      # 1 -> 0, 3
    [0, 3],      # 2 -> 0, 3
    [1, 2]       # 3 -> 1, 2
]

# Weighted graph
weighted_graph = {
    'A': [('B', 5), ('C', 3)],
    'B': [('A', 5), ('D', 2)],
    # ...
}
```

#### Adjacency Matrix

```python
# For dense graphs or when checking edge existence frequently
# matrix[i][j] = 1 if edge exists, 0 otherwise
matrix = [
    [0, 1, 1, 0],  # A connected to B, C
    [1, 0, 0, 1],  # B connected to A, D
    [1, 0, 0, 1],  # C connected to A, D
    [0, 1, 1, 0]   # D connected to B, C
]
```

#### Edge List

```python
# List of (source, destination) or (source, destination, weight)
edges = [
    (0, 1), (0, 2), (1, 3), (2, 3)
]
```

### Complexity Comparison

| Operation | Adjacency List | Adjacency Matrix |
|-----------|---------------|------------------|
| Space | O(V + E) | O(V²) |
| Add Edge | O(1) | O(1) |
| Remove Edge | O(E) | O(1) |
| Check Edge | O(degree) | O(1) |
| Get Neighbors | O(degree) | O(V) |

---

## Graph Traversals

### Breadth-First Search (BFS)

**Python:**
```python
from collections import deque

def bfs(graph, start):
    """
    Use cases:
    - Shortest path in unweighted graph
    - Level-order traversal
    - Finding all nodes at distance k
    """
    visited = {start}
    queue = deque([start])
    result = []

    while queue:
        node = queue.popleft()
        result.append(node)

        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

    return result

def bfs_shortest_path(graph, start, end):
    """Find shortest path in unweighted graph"""
    if start == end:
        return [start]

    visited = {start}
    queue = deque([(start, [start])])

    while queue:
        node, path = queue.popleft()

        for neighbor in graph[node]:
            if neighbor == end:
                return path + [neighbor]
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append((neighbor, path + [neighbor]))

    return []  # No path found

def bfs_level_order(graph, start):
    """Return nodes level by level"""
    visited = {start}
    queue = deque([start])
    levels = []

    while queue:
        level_size = len(queue)
        current_level = []

        for _ in range(level_size):
            node = queue.popleft()
            current_level.append(node)

            for neighbor in graph[node]:
                if neighbor not in visited:
                    visited.add(neighbor)
                    queue.append(neighbor)

        levels.append(current_level)

    return levels
```

**JavaScript:**
```javascript
function bfs(graph, start) {
  // Use cases: Shortest path in unweighted graph, Level-order traversal
  const visited = new Set([start]);
  const queue = [start];
  const result = [];

  while (queue.length > 0) {
    const node = queue.shift();
    result.push(node);

    for (const neighbor of graph[node] || []) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push(neighbor);
      }
    }
  }

  return result;
}

function bfsShortestPath(graph, start, end) {
  // Find shortest path in unweighted graph
  if (start === end) return [start];

  const visited = new Set([start]);
  const queue = [[start, [start]]]; // [node, path]

  while (queue.length > 0) {
    const [node, path] = queue.shift();

    for (const neighbor of graph[node] || []) {
      if (neighbor === end) {
        return [...path, neighbor];
      }
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push([neighbor, [...path, neighbor]]);
      }
    }
  }

  return []; // No path found
}

function bfsLevelOrder(graph, start) {
  // Return nodes level by level
  const visited = new Set([start]);
  const queue = [start];
  const levels = [];

  while (queue.length > 0) {
    const levelSize = queue.length;
    const currentLevel = [];

    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift();
      currentLevel.push(node);

      for (const neighbor of graph[node] || []) {
        if (!visited.has(neighbor)) {
          visited.add(neighbor);
          queue.push(neighbor);
        }
      }
    }

    levels.push(currentLevel);
  }

  return levels;
}
```

### Depth-First Search (DFS)

**Python:**
```python
def dfs_recursive(graph, start, visited=None):
    """
    Use cases:
    - Detecting cycles
    - Topological sort
    - Finding connected components
    - Path finding (not necessarily shortest)
    """
    if visited is None:
        visited = set()

    visited.add(start)
    result = [start]

    for neighbor in graph[start]:
        if neighbor not in visited:
            result.extend(dfs_recursive(graph, neighbor, visited))

    return result

def dfs_iterative(graph, start):
    visited = set()
    stack = [start]
    result = []

    while stack:
        node = stack.pop()
        if node not in visited:
            visited.add(node)
            result.append(node)
            # Add neighbors in reverse order for same order as recursive
            for neighbor in reversed(graph[node]):
                if neighbor not in visited:
                    stack.append(neighbor)

    return result
```

**JavaScript:**
```javascript
function dfsRecursive(graph, start, visited = new Set()) {
  // Use cases: Detecting cycles, Topological sort, Connected components
  visited.add(start);
  const result = [start];

  for (const neighbor of graph[start] || []) {
    if (!visited.has(neighbor)) {
      result.push(...dfsRecursive(graph, neighbor, visited));
    }
  }

  return result;
}

function dfsIterative(graph, start) {
  const visited = new Set();
  const stack = [start];
  const result = [];

  while (stack.length > 0) {
    const node = stack.pop();
    if (!visited.has(node)) {
      visited.add(node);
      result.push(node);
      // Add neighbors in reverse order for same order as recursive
      const neighbors = graph[node] || [];
      for (let i = neighbors.length - 1; i >= 0; i--) {
        if (!visited.has(neighbors[i])) {
          stack.push(neighbors[i]);
        }
      }
    }
  }

  return result;
}
```

---

## Cycle Detection

### Undirected Graph

**Python:**
```python
def has_cycle_undirected(graph, n):
    """Using DFS"""
    visited = set()

    def dfs(node, parent):
        visited.add(node)

        for neighbor in graph[node]:
            if neighbor not in visited:
                if dfs(neighbor, node):
                    return True
            elif neighbor != parent:
                return True

        return False

    for node in range(n):
        if node not in visited:
            if dfs(node, -1):
                return True

    return False
```

**JavaScript:**
```javascript
function hasCycleUndirected(graph, n) {
  // Using DFS
  const visited = new Set();

  function dfs(node, parent) {
    visited.add(node);

    for (const neighbor of graph[node] || []) {
      if (!visited.has(neighbor)) {
        if (dfs(neighbor, node)) return true;
      } else if (neighbor !== parent) {
        return true;
      }
    }

    return false;
  }

  for (let node = 0; node < n; node++) {
    if (!visited.has(node)) {
      if (dfs(node, -1)) return true;
    }
  }

  return false;
}
```

### Directed Graph

**Python:**
```python
def has_cycle_directed(graph, n):
    """Using DFS with three colors (white/gray/black)"""
    WHITE, GRAY, BLACK = 0, 1, 2
    color = [WHITE] * n

    def dfs(node):
        color[node] = GRAY

        for neighbor in graph[node]:
            if color[neighbor] == GRAY:  # Back edge
                return True
            if color[neighbor] == WHITE:
                if dfs(neighbor):
                    return True

        color[node] = BLACK
        return False

    for node in range(n):
        if color[node] == WHITE:
            if dfs(node):
                return True

    return False
```

**JavaScript:**
```javascript
function hasCycleDirected(graph, n) {
  // Using DFS with three colors (white/gray/black)
  const WHITE = 0, GRAY = 1, BLACK = 2;
  const color = new Array(n).fill(WHITE);

  function dfs(node) {
    color[node] = GRAY;

    for (const neighbor of graph[node] || []) {
      if (color[neighbor] === GRAY) { // Back edge
        return true;
      }
      if (color[neighbor] === WHITE) {
        if (dfs(neighbor)) return true;
      }
    }

    color[node] = BLACK;
    return false;
  }

  for (let node = 0; node < n; node++) {
    if (color[node] === WHITE) {
      if (dfs(node)) return true;
    }
  }

  return false;
}
```

---

## Topological Sort

**Python:**
```python
def topological_sort_dfs(graph, n):
    """
    DFS-based topological sort
    Time: O(V + E)
    """
    visited = set()
    stack = []

    def dfs(node):
        visited.add(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                dfs(neighbor)
        stack.append(node)

    for node in range(n):
        if node not in visited:
            dfs(node)

    return stack[::-1]

def topological_sort_kahn(graph, n):
    """
    Kahn's algorithm (BFS-based)
    Also detects cycles - returns empty if cycle exists
    """
    # Calculate in-degrees
    in_degree = [0] * n
    for node in range(n):
        for neighbor in graph[node]:
            in_degree[neighbor] += 1

    # Start with nodes having no dependencies
    queue = deque([node for node in range(n) if in_degree[node] == 0])
    result = []

    while queue:
        node = queue.popleft()
        result.append(node)

        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    # Check for cycle
    if len(result) != n:
        return []  # Cycle exists

    return result
```

**JavaScript:**
```javascript
function topologicalSortDFS(graph, n) {
  // DFS-based topological sort - Time: O(V + E)
  const visited = new Set();
  const stack = [];

  function dfs(node) {
    visited.add(node);
    for (const neighbor of graph[node] || []) {
      if (!visited.has(neighbor)) {
        dfs(neighbor);
      }
    }
    stack.push(node);
  }

  for (let node = 0; node < n; node++) {
    if (!visited.has(node)) {
      dfs(node);
    }
  }

  return stack.reverse();
}

function topologicalSortKahn(graph, n) {
  // Kahn's algorithm (BFS-based) - Also detects cycles
  const inDegree = new Array(n).fill(0);

  for (let node = 0; node < n; node++) {
    for (const neighbor of graph[node] || []) {
      inDegree[neighbor]++;
    }
  }

  // Start with nodes having no dependencies
  const queue = [];
  for (let node = 0; node < n; node++) {
    if (inDegree[node] === 0) queue.push(node);
  }

  const result = [];

  while (queue.length > 0) {
    const node = queue.shift();
    result.push(node);

    for (const neighbor of graph[node] || []) {
      inDegree[neighbor]--;
      if (inDegree[neighbor] === 0) {
        queue.push(neighbor);
      }
    }
  }

  // Check for cycle
  if (result.length !== n) return []; // Cycle exists

  return result;
}
```

### Course Schedule Problem

**Python:**
```python
def can_finish(num_courses, prerequisites):
    """Classic topological sort application"""
    graph = [[] for _ in range(num_courses)]
    in_degree = [0] * num_courses

    for course, prereq in prerequisites:
        graph[prereq].append(course)
        in_degree[course] += 1

    queue = deque([c for c in range(num_courses) if in_degree[c] == 0])
    completed = 0

    while queue:
        course = queue.popleft()
        completed += 1

        for next_course in graph[course]:
            in_degree[next_course] -= 1
            if in_degree[next_course] == 0:
                queue.append(next_course)

    return completed == num_courses

def find_order(num_courses, prerequisites):
    """Return one valid ordering (or empty if impossible)"""
    graph = [[] for _ in range(num_courses)]
    in_degree = [0] * num_courses

    for course, prereq in prerequisites:
        graph[prereq].append(course)
        in_degree[course] += 1

    queue = deque([c for c in range(num_courses) if in_degree[c] == 0])
    order = []

    while queue:
        course = queue.popleft()
        order.append(course)

        for next_course in graph[course]:
            in_degree[next_course] -= 1
            if in_degree[next_course] == 0:
                queue.append(next_course)

    return order if len(order) == num_courses else []
```

**JavaScript:**
```javascript
function canFinish(numCourses, prerequisites) {
  // Classic topological sort application
  const graph = Array.from({ length: numCourses }, () => []);
  const inDegree = new Array(numCourses).fill(0);

  for (const [course, prereq] of prerequisites) {
    graph[prereq].push(course);
    inDegree[course]++;
  }

  const queue = [];
  for (let c = 0; c < numCourses; c++) {
    if (inDegree[c] === 0) queue.push(c);
  }

  let completed = 0;

  while (queue.length > 0) {
    const course = queue.shift();
    completed++;

    for (const nextCourse of graph[course]) {
      inDegree[nextCourse]--;
      if (inDegree[nextCourse] === 0) {
        queue.push(nextCourse);
      }
    }
  }

  return completed === numCourses;
}

function findOrder(numCourses, prerequisites) {
  // Return one valid ordering (or empty if impossible)
  const graph = Array.from({ length: numCourses }, () => []);
  const inDegree = new Array(numCourses).fill(0);

  for (const [course, prereq] of prerequisites) {
    graph[prereq].push(course);
    inDegree[course]++;
  }

  const queue = [];
  for (let c = 0; c < numCourses; c++) {
    if (inDegree[c] === 0) queue.push(c);
  }

  const order = [];

  while (queue.length > 0) {
    const course = queue.shift();
    order.push(course);

    for (const nextCourse of graph[course]) {
      inDegree[nextCourse]--;
      if (inDegree[nextCourse] === 0) {
        queue.push(nextCourse);
      }
    }
  }

  return order.length === numCourses ? order : [];
}
```

---

## Shortest Path Algorithms

### Dijkstra's Algorithm

**Python:**
```python
import heapq

def dijkstra(graph, start, n):
    """
    Single source shortest path for non-negative weights
    Time: O((V + E) log V) with binary heap
    """
    distances = [float('inf')] * n
    distances[start] = 0
    heap = [(0, start)]  # (distance, node)

    while heap:
        dist, node = heapq.heappop(heap)

        if dist > distances[node]:
            continue

        for neighbor, weight in graph[node]:
            new_dist = dist + weight
            if new_dist < distances[neighbor]:
                distances[neighbor] = new_dist
                heapq.heappush(heap, (new_dist, neighbor))

    return distances

def dijkstra_with_path(graph, start, end, n):
    """Return shortest distance and path"""
    distances = [float('inf')] * n
    distances[start] = 0
    parent = [-1] * n
    heap = [(0, start)]

    while heap:
        dist, node = heapq.heappop(heap)

        if node == end:
            break

        if dist > distances[node]:
            continue

        for neighbor, weight in graph[node]:
            new_dist = dist + weight
            if new_dist < distances[neighbor]:
                distances[neighbor] = new_dist
                parent[neighbor] = node
                heapq.heappush(heap, (new_dist, neighbor))

    # Reconstruct path
    path = []
    curr = end
    while curr != -1:
        path.append(curr)
        curr = parent[curr]

    return distances[end], path[::-1]
```

**JavaScript:**
```javascript
// MinHeap implementation for Dijkstra
class MinHeap {
  constructor() {
    this.heap = [];
  }

  push(val) {
    this.heap.push(val);
    this._bubbleUp(this.heap.length - 1);
  }

  pop() {
    if (this.heap.length === 0) return null;
    const min = this.heap[0];
    const last = this.heap.pop();
    if (this.heap.length > 0) {
      this.heap[0] = last;
      this._bubbleDown(0);
    }
    return min;
  }

  _bubbleUp(i) {
    while (i > 0) {
      const parent = Math.floor((i - 1) / 2);
      if (this.heap[parent][0] <= this.heap[i][0]) break;
      [this.heap[parent], this.heap[i]] = [this.heap[i], this.heap[parent]];
      i = parent;
    }
  }

  _bubbleDown(i) {
    const n = this.heap.length;
    while (true) {
      let smallest = i;
      const left = 2 * i + 1, right = 2 * i + 2;
      if (left < n && this.heap[left][0] < this.heap[smallest][0]) smallest = left;
      if (right < n && this.heap[right][0] < this.heap[smallest][0]) smallest = right;
      if (smallest === i) break;
      [this.heap[smallest], this.heap[i]] = [this.heap[i], this.heap[smallest]];
      i = smallest;
    }
  }

  get length() { return this.heap.length; }
}

function dijkstra(graph, start, n) {
  // Single source shortest path for non-negative weights
  // Time: O((V + E) log V) with binary heap
  const distances = new Array(n).fill(Infinity);
  distances[start] = 0;
  const heap = new MinHeap();
  heap.push([0, start]); // [distance, node]

  while (heap.length > 0) {
    const [dist, node] = heap.pop();

    if (dist > distances[node]) continue;

    for (const [neighbor, weight] of graph[node] || []) {
      const newDist = dist + weight;
      if (newDist < distances[neighbor]) {
        distances[neighbor] = newDist;
        heap.push([newDist, neighbor]);
      }
    }
  }

  return distances;
}

function dijkstraWithPath(graph, start, end, n) {
  // Return shortest distance and path
  const distances = new Array(n).fill(Infinity);
  distances[start] = 0;
  const parent = new Array(n).fill(-1);
  const heap = new MinHeap();
  heap.push([0, start]);

  while (heap.length > 0) {
    const [dist, node] = heap.pop();

    if (node === end) break;
    if (dist > distances[node]) continue;

    for (const [neighbor, weight] of graph[node] || []) {
      const newDist = dist + weight;
      if (newDist < distances[neighbor]) {
        distances[neighbor] = newDist;
        parent[neighbor] = node;
        heap.push([newDist, neighbor]);
      }
    }
  }

  // Reconstruct path
  const path = [];
  let curr = end;
  while (curr !== -1) {
    path.push(curr);
    curr = parent[curr];
  }

  return [distances[end], path.reverse()];
}
```

### Bellman-Ford Algorithm

**Python:**
```python
def bellman_ford(edges, n, start):
    """
    Handles negative weights, detects negative cycles
    Time: O(V * E)
    """
    distances = [float('inf')] * n
    distances[start] = 0

    # Relax all edges n-1 times
    for _ in range(n - 1):
        for u, v, weight in edges:
            if distances[u] != float('inf') and distances[u] + weight < distances[v]:
                distances[v] = distances[u] + weight

    # Check for negative cycles
    for u, v, weight in edges:
        if distances[u] != float('inf') and distances[u] + weight < distances[v]:
            return None  # Negative cycle detected

    return distances
```

**JavaScript:**
```javascript
function bellmanFord(edges, n, start) {
  // Handles negative weights, detects negative cycles
  // Time: O(V * E)
  const distances = new Array(n).fill(Infinity);
  distances[start] = 0;

  // Relax all edges n-1 times
  for (let i = 0; i < n - 1; i++) {
    for (const [u, v, weight] of edges) {
      if (distances[u] !== Infinity && distances[u] + weight < distances[v]) {
        distances[v] = distances[u] + weight;
      }
    }
  }

  // Check for negative cycles
  for (const [u, v, weight] of edges) {
    if (distances[u] !== Infinity && distances[u] + weight < distances[v]) {
      return null; // Negative cycle detected
    }
  }

  return distances;
}
```

### Floyd-Warshall Algorithm

**Python:**
```python
def floyd_warshall(graph, n):
    """
    All pairs shortest path
    Time: O(V³)
    """
    dist = [[float('inf')] * n for _ in range(n)]

    # Initialize with direct edges
    for i in range(n):
        dist[i][i] = 0
        for j, weight in graph[i]:
            dist[i][j] = weight

    # Try all intermediate vertices
    for k in range(n):
        for i in range(n):
            for j in range(n):
                if dist[i][k] + dist[k][j] < dist[i][j]:
                    dist[i][j] = dist[i][k] + dist[k][j]

    return dist
```

**JavaScript:**
```javascript
function floydWarshall(graph, n) {
  // All pairs shortest path - Time: O(V³)
  const dist = Array.from({ length: n }, () => new Array(n).fill(Infinity));

  // Initialize with direct edges
  for (let i = 0; i < n; i++) {
    dist[i][i] = 0;
    for (const [j, weight] of graph[i] || []) {
      dist[i][j] = weight;
    }
  }

  // Try all intermediate vertices
  for (let k = 0; k < n; k++) {
    for (let i = 0; i < n; i++) {
      for (let j = 0; j < n; j++) {
        if (dist[i][k] + dist[k][j] < dist[i][j]) {
          dist[i][j] = dist[i][k] + dist[k][j];
        }
      }
    }
  }

  return dist;
}
```

### 0-1 BFS

**Python:**
```python
def bfs_01(graph, start, n):
    """
    Shortest path when edge weights are only 0 or 1
    Time: O(V + E) using deque
    """
    distances = [float('inf')] * n
    distances[start] = 0
    deque_bfs = deque([start])

    while deque_bfs:
        node = deque_bfs.popleft()

        for neighbor, weight in graph[node]:
            new_dist = distances[node] + weight
            if new_dist < distances[neighbor]:
                distances[neighbor] = new_dist
                if weight == 0:
                    deque_bfs.appendleft(neighbor)  # Priority
                else:
                    deque_bfs.append(neighbor)

    return distances
```

**JavaScript:**
```javascript
function bfs01(graph, start, n) {
  // Shortest path when edge weights are only 0 or 1
  // Time: O(V + E) using deque
  const distances = new Array(n).fill(Infinity);
  distances[start] = 0;
  const deque = [start];

  while (deque.length > 0) {
    const node = deque.shift();

    for (const [neighbor, weight] of graph[node] || []) {
      const newDist = distances[node] + weight;
      if (newDist < distances[neighbor]) {
        distances[neighbor] = newDist;
        if (weight === 0) {
          deque.unshift(neighbor); // Priority - add to front
        } else {
          deque.push(neighbor);
        }
      }
    }
  }

  return distances;
}
```

---

## Union-Find (Disjoint Set Union)

**Python:**
```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n
        self.components = n

    def find(self, x):
        """Find with path compression"""
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]

    def union(self, x, y):
        """Union by rank"""
        root_x, root_y = self.find(x), self.find(y)

        if root_x == root_y:
            return False

        if self.rank[root_x] < self.rank[root_y]:
            root_x, root_y = root_y, root_x

        self.parent[root_y] = root_x
        if self.rank[root_x] == self.rank[root_y]:
            self.rank[root_x] += 1

        self.components -= 1
        return True

    def connected(self, x, y):
        return self.find(x) == self.find(y)
```

**JavaScript:**
```javascript
class UnionFind {
  constructor(n) {
    this.parent = Array.from({ length: n }, (_, i) => i);
    this.rank = new Array(n).fill(0);
    this.components = n;
  }

  find(x) {
    // Find with path compression
    if (this.parent[x] !== x) {
      this.parent[x] = this.find(this.parent[x]);
    }
    return this.parent[x];
  }

  union(x, y) {
    // Union by rank
    let rootX = this.find(x);
    let rootY = this.find(y);

    if (rootX === rootY) return false;

    if (this.rank[rootX] < this.rank[rootY]) {
      [rootX, rootY] = [rootY, rootX];
    }

    this.parent[rootY] = rootX;
    if (this.rank[rootX] === this.rank[rootY]) {
      this.rank[rootX]++;
    }

    this.components--;
    return true;
  }

  connected(x, y) {
    return this.find(x) === this.find(y);
  }
}
```

### Applications

**Python:**
```python
def count_components(n, edges):
    """Count connected components"""
    uf = UnionFind(n)
    for u, v in edges:
        uf.union(u, v)
    return uf.components

def valid_tree(n, edges):
    """Check if edges form a valid tree (connected, no cycles)"""
    if len(edges) != n - 1:
        return False

    uf = UnionFind(n)
    for u, v in edges:
        if not uf.union(u, v):  # Cycle detected
            return False

    return uf.components == 1

def kruskal_mst(n, edges):
    """Minimum Spanning Tree using Kruskal's algorithm"""
    # Sort edges by weight
    edges.sort(key=lambda x: x[2])

    uf = UnionFind(n)
    mst = []
    total_weight = 0

    for u, v, weight in edges:
        if uf.union(u, v):
            mst.append((u, v, weight))
            total_weight += weight
            if len(mst) == n - 1:
                break

    return mst, total_weight
```

**JavaScript:**
```javascript
function countComponents(n, edges) {
  // Count connected components
  const uf = new UnionFind(n);
  for (const [u, v] of edges) {
    uf.union(u, v);
  }
  return uf.components;
}

function validTree(n, edges) {
  // Check if edges form a valid tree (connected, no cycles)
  if (edges.length !== n - 1) return false;

  const uf = new UnionFind(n);
  for (const [u, v] of edges) {
    if (!uf.union(u, v)) { // Cycle detected
      return false;
    }
  }

  return uf.components === 1;
}

function kruskalMST(n, edges) {
  // Minimum Spanning Tree using Kruskal's algorithm
  // Sort edges by weight
  edges.sort((a, b) => a[2] - b[2]);

  const uf = new UnionFind(n);
  const mst = [];
  let totalWeight = 0;

  for (const [u, v, weight] of edges) {
    if (uf.union(u, v)) {
      mst.push([u, v, weight]);
      totalWeight += weight;
      if (mst.length === n - 1) break;
    }
  }

  return [mst, totalWeight];
}
```

---

## Connected Components

**Python:**
```python
def count_connected_components(graph, n):
    """Using DFS"""
    visited = set()
    count = 0

    def dfs(node):
        visited.add(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                dfs(neighbor)

    for node in range(n):
        if node not in visited:
            dfs(node)
            count += 1

    return count
```

**JavaScript:**
```javascript
function countConnectedComponents(graph, n) {
  // Using DFS
  const visited = new Set();
  let count = 0;

  function dfs(node) {
    visited.add(node);
    for (const neighbor of graph[node] || []) {
      if (!visited.has(neighbor)) {
        dfs(neighbor);
      }
    }
  }

  for (let node = 0; node < n; node++) {
    if (!visited.has(node)) {
      dfs(node);
      count++;
    }
  }

  return count;
}
```

---

## Grid-Based Graph Problems

### Number of Islands

**Python:**
```python
def num_islands(grid):
    if not grid:
        return 0

    rows, cols = len(grid), len(grid[0])
    count = 0

    def dfs(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
            return
        grid[r][c] = '0'  # Mark visited
        dfs(r + 1, c)
        dfs(r - 1, c)
        dfs(r, c + 1)
        dfs(r, c - 1)

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                dfs(r, c)
                count += 1

    return count
```

**JavaScript:**
```javascript
function numIslands(grid) {
  if (!grid || grid.length === 0) return 0;

  const rows = grid.length;
  const cols = grid[0].length;
  let count = 0;

  function dfs(r, c) {
    if (r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] !== '1') {
      return;
    }
    grid[r][c] = '0'; // Mark visited
    dfs(r + 1, c);
    dfs(r - 1, c);
    dfs(r, c + 1);
    dfs(r, c - 1);
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] === '1') {
        dfs(r, c);
        count++;
      }
    }
  }

  return count;
}
```

### Rotting Oranges (Multi-source BFS)

**Python:**
```python
def oranges_rotting(grid):
    rows, cols = len(grid), len(grid[0])
    queue = deque()
    fresh = 0

    # Find all rotten oranges and count fresh
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == 2:
                queue.append((r, c))
            elif grid[r][c] == 1:
                fresh += 1

    if fresh == 0:
        return 0

    minutes = 0
    directions = [(0, 1), (0, -1), (1, 0), (-1, 0)]

    while queue:
        minutes += 1
        for _ in range(len(queue)):
            r, c = queue.popleft()
            for dr, dc in directions:
                nr, nc = r + dr, c + dc
                if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == 1:
                    grid[nr][nc] = 2
                    fresh -= 1
                    queue.append((nr, nc))

    return minutes - 1 if fresh == 0 else -1
```

**JavaScript:**
```javascript
function orangesRotting(grid) {
  const rows = grid.length;
  const cols = grid[0].length;
  const queue = [];
  let fresh = 0;

  // Find all rotten oranges and count fresh
  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] === 2) {
        queue.push([r, c]);
      } else if (grid[r][c] === 1) {
        fresh++;
      }
    }
  }

  if (fresh === 0) return 0;

  let minutes = 0;
  const directions = [[0, 1], [0, -1], [1, 0], [-1, 0]];

  while (queue.length > 0) {
    minutes++;
    const size = queue.length;
    for (let i = 0; i < size; i++) {
      const [r, c] = queue.shift();
      for (const [dr, dc] of directions) {
        const nr = r + dr, nc = c + dc;
        if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] === 1) {
          grid[nr][nc] = 2;
          fresh--;
          queue.push([nr, nc]);
        }
      }
    }
  }

  return fresh === 0 ? minutes - 1 : -1;
}
```

### Walls and Gates (Multi-source BFS)

**Python:**
```python
def walls_and_gates(rooms):
    """Fill each empty room with distance to nearest gate"""
    if not rooms:
        return

    rows, cols = len(rooms), len(rooms[0])
    INF = 2147483647

    # Start BFS from all gates
    queue = deque()
    for r in range(rows):
        for c in range(cols):
            if rooms[r][c] == 0:
                queue.append((r, c))

    directions = [(0, 1), (0, -1), (1, 0), (-1, 0)]

    while queue:
        r, c = queue.popleft()
        for dr, dc in directions:
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and rooms[nr][nc] == INF:
                rooms[nr][nc] = rooms[r][c] + 1
                queue.append((nr, nc))
```

**JavaScript:**
```javascript
function wallsAndGates(rooms) {
  // Fill each empty room with distance to nearest gate
  if (!rooms || rooms.length === 0) return;

  const rows = rooms.length;
  const cols = rooms[0].length;
  const INF = 2147483647;

  // Start BFS from all gates
  const queue = [];
  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (rooms[r][c] === 0) {
        queue.push([r, c]);
      }
    }
  }

  const directions = [[0, 1], [0, -1], [1, 0], [-1, 0]];

  while (queue.length > 0) {
    const [r, c] = queue.shift();
    for (const [dr, dc] of directions) {
      const nr = r + dr, nc = c + dc;
      if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && rooms[nr][nc] === INF) {
        rooms[nr][nc] = rooms[r][c] + 1;
        queue.push([nr, nc]);
      }
    }
  }
}
```

---

## Bipartite Graph

**Python:**
```python
def is_bipartite(graph):
    """Check if graph can be 2-colored"""
    n = len(graph)
    color = [-1] * n

    def bfs(start):
        queue = deque([start])
        color[start] = 0

        while queue:
            node = queue.popleft()
            for neighbor in graph[node]:
                if color[neighbor] == -1:
                    color[neighbor] = 1 - color[node]
                    queue.append(neighbor)
                elif color[neighbor] == color[node]:
                    return False

        return True

    for node in range(n):
        if color[node] == -1:
            if not bfs(node):
                return False

    return True
```

**JavaScript:**
```javascript
function isBipartite(graph) {
  // Check if graph can be 2-colored
  const n = graph.length;
  const color = new Array(n).fill(-1);

  function bfs(start) {
    const queue = [start];
    color[start] = 0;

    while (queue.length > 0) {
      const node = queue.shift();
      for (const neighbor of graph[node] || []) {
        if (color[neighbor] === -1) {
          color[neighbor] = 1 - color[node];
          queue.push(neighbor);
        } else if (color[neighbor] === color[node]) {
          return false;
        }
      }
    }

    return true;
  }

  for (let node = 0; node < n; node++) {
    if (color[node] === -1) {
      if (!bfs(node)) return false;
    }
  }

  return true;
}
```

---

## Must-Know Problems

### Easy
1. Number of Islands
2. Flood Fill
3. Find if Path Exists in Graph

### Medium
1. Clone Graph
2. Course Schedule I & II
3. Pacific Atlantic Water Flow
4. Rotting Oranges
5. Number of Provinces
6. Graph Valid Tree
7. Word Ladder
8. Evaluate Division
9. Redundant Connection
10. Network Delay Time

### Hard
1. Word Ladder II
2. Alien Dictionary
3. Minimum Cost to Connect All Points
4. Reconstruct Itinerary
5. Critical Connections in a Network
6. Shortest Path in a Grid with Obstacles

---

## Practice Checklist

- [ ] Can implement BFS and DFS fluently
- [ ] Know when to use BFS vs DFS
- [ ] Can detect cycles in directed and undirected graphs
- [ ] Understand topological sort and its applications
- [ ] Can implement Dijkstra's algorithm
- [ ] Know Union-Find and when to use it
- [ ] Comfortable with grid-based graph problems
