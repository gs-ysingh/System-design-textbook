# Algorithmic System Design II — Readable Learnings Summary

## 🗺️ 1. Geo-Hashing and Location-Based Systems

### Core Problem
Efficiently find users or points within **K km radius** (e.g., Tinder, Uber, Swiggy).

### Challenges
- Naïve approach → **O(N²)** distance computations.
- Two-dimensional distance comparison is computationally expensive.
- Simple heuristics (city/state filters) are not scalable.

### Insight
- In 1-D: Use **range queries** (BST/B+ Tree/Segment Tree).
- In 2-D: Need a spatial encoding — **Geo-Hash**.

### Geo-Hash Idea
- **Divide and conquer:** Repeatedly split the world in halves.
- Assign bits:
  - Left/top → 0  
  - Right/bottom → 1  
- Nearby locations share **common prefixes** → proximity detection via prefix match.

### Implementation Details
- 32-bit precision ≈ 1 mm accuracy.
- Convert to **Base-32 alphanumeric** (like Google Plus Codes).
- Data structures:
  - **Trie:** Hard prefix match.
  - **BK-Tree:** Fuzzy prefix (edit distance ≤ 2–3).

### Edge Case
Points at region boundaries may appear distant.

### Usage Example (Redis)
```bash
GEOADD users:location 77.5946 12.9716 "user:Yogesh"
GEORADIUS users:location 77.5946 12.9716 8 km
```
Used by **Zomato, Uber, Swiggy, Google Maps**.

---

## 📍 2. Point-in-Polygon Problem (Ray Casting)

### Real-World Use
Detecting if a user is within an area — e.g. “Welcome to Airport”, geofencing, reminders.

### Algorithm: Ray Casting
1. Cast a horizontal ray from P to infinity.
2. Count intersections with polygon edges:  
   - Odd → Inside  
   - Even → Outside

### Representation
Polygon = ordered list of (x, y) points.  
Each edge = (p₁, p₂) where y₁ < y₂.

### Checks
- P above y₂ or below y₁ → No intersection  
- P right of max(x₁, x₂) → No  
- P left of min(x₁, x₂) → Yes  
- Else → Compare slopes of (A,B) and (A,P):
  - If slope(AP) > slope(AB) → Intersect

### Applications
- **Geofencing:** Uber/Airport entry  
- **Location-based reminders:** “Silent at office”  
- **AR games:** Pokémon Go-style experiences

---

## 📊 3. T-Digest Algorithm (Percentiles at Scale)

### Context
Used by **DataDog, New Relic, CloudWatch** for P50/P95/P99 computation from large data streams.

### Goal
Compute approximate percentiles **without storing all points**.

### Concept
- Maintain **cluster centroids** instead of all data.
- Cluster size:
  - Small for lower and upper extremes (more precision).
  - Large near median (less precision).

### Steps
1. Collect a data batch (e.g., 1000 samples).
2. Sort values.
3. Form clusters where (max-min) ≤ Δ.
4. For each cluster store:
   - `centroid` – representative value.
   - `weight` – number of items.
   - `cost` – sum of distances from members to centroid.
5. Merge new batches by weighted centroid update.

### Benefits
- 8 MB raw → ~5 KB stored with < 0.002% error.
- Quicker percentile computation.

### Limitation
- Spiky traffic may distort percentiles (outliers dominate).

---

## 💡 4. Related Algorithms & Further Reading

| Category | Algorithms / Topics | Usage |
|-----------|---------------------|--------|
| Probabilistic | Bloom Filter, Count–Min Sketch, HyperLogLog | Unique counts, membership |
| Collaborative Editing | Operational Transformations, Differential Sync | Google Docs-style editing |
| Geometry | Convex Hull, Quad Tree, Merkle Tree, S2 Geometry | Spatial boundaries, blockchain |
| Anomaly Detection | Isolation Forest | Detect outliers (similar logic to GeoHash) |

---

## 🔧 5. Practical System Analogies

| System | Algorithm / Concept Used |
|--------|--------------------------|
| Tinder / Swiggy / Uber | Geo-Hash + Redis GeoRadius |
| Google Maps / IFTTT | Ray Casting (point-in-polygon) |
| New Relic / DataDog | T-Digest |
| Zomato Ads | Geo proximity + ranking |
| Grofers / Amazon | Isolation Forest |

---

## 🧭 6. Key Takeaways

- Scaling and sharding ≠ solving the *real* product challenge.  
- **GeoHash** → Proximity via bitwise spatial partitioning.  
- **Ray Casting** → Inclusion test using intersection count.  
- **T-Digest** → Efficient percentile via clustering & weighted merge.  
- **Divide & Conquer** connects geolocation, graphics, and anomaly detection.  
- **Redis** is a goldmine for Geo, Bloom, Count-Min, and percentile computations.

---
