# Lecture 7: Proximity Search Algorithms

> **Lecture:** 07  
> **Topic:** System Design Concept  
> **Application:** Proximity Search (Finding Nearby Locations)  
> **Use Cases:** Uber, Zomato, Hotel Booking, Restaurant Search  
> **Difficulty:** Medium  
> **Previous:** [[Lecture-06-Food-Delivery-System-Design|Lecture 6: Food Delivery System]]

---

## 📋 Table of Contents

1. [Brief Introduction](#1-brief-introduction)
2. [Four Proximity Search Approaches](#2-four-proximity-search-approaches)
3. [Approach 1: Quad Tree](#3-approach-1-quad-tree)
4. [Approach 2: Geohash](#4-approach-2-geohash)
5. [Approach 3: PostgreSQL + GIS Extension](#5-approach-3-postgresql--gis-extension)
6. [Approach 4: Elasticsearch](#6-approach-4-elasticsearch)
7. [PostgreSQL + Fuzzy Search (Bonus)](#7-postgresql--fuzzy-search-bonus)
8. [Comparison Table](#8-comparison-table)
9. [Decision Tree: Which Approach to Use?](#9-decision-tree-which-approach-to-use)
10. [Interview Q&A](#10-interview-qa)
11. [Key Takeaways](#11-key-takeaways)

---

<br>

## 1. Brief Introduction

### What is Proximity Search?

**Proximity search** is the ability to find all entities (restaurants, hotels, drivers) within a certain radius of a given location.

**Examples:**
- **Uber:** Find drivers within 5 km
- **Zomato:** Find restaurants within 10 km
- **Airbnb:** Find hotels within 20 km
- **Google Maps:** Find gas stations within 2 km

### Interview Context

⚠️ **Important:**

This is **NOT a standalone interview question!**

You'll need proximity search **while designing:**
- 🚗 Uber/Ola (find nearby drivers)
- 🍕 Zomato/Swiggy (find nearby restaurants)
- 🏨 Airbnb/Booking.com (find nearby hotels)

**Your interviewer expects you to:**
- ✅ Know multiple approaches
- ✅ Compare pros/cons
- ✅ Choose the right one for the use case

---

## 2. Four Proximity Search Approaches

### Overview

| Approach | Complexity | Performance | Use Case |
|----------|------------|-------------|----------|
| **1. Quad Tree** | High | Fast (in-memory) | Static data, rare updates |
| **2. Geohash** | Medium | Very Fast | Real-time location updates |
| **3. PostgreSQL + GIS** | Low | Fast | Location + text search |
| **4. Elasticsearch** | Low | Very Fast | Location + text search |

---

## 3. Approach 1: Quad Tree

### 3.1 What is a Quad Tree?

A **Quad Tree** is a tree data structure where each node has **exactly 4 children**.

**Concept:**
- Divide the world map into 4 quadrants
- Recursively subdivide each quadrant into 4 sub-quadrants
- Stop when quadrant has ≤ N entities (e.g., 100 restaurants)

### 3.2 Visual Representation

```
┌─────────────────────────────────┐
│ NW          │          NE       │
│             │                   │
│      ┌──────┼──────┐            │
│      │ NW   │  NE  │            │
│──────┼──────┼──────┼────────────│
│      │ SW   │  SE  │            │
│      └──────┼──────┘            │
│             │                   │
│ SW          │          SE       │
└─────────────────────────────────┘
```

**Tree Structure:**

```
                    Root (World)
                        │
        ┌───────────────┼───────────────┐
        │               │               │
       NW              NE              SW              SE
        │
    ┌───┼───┐
    │   │   │
   NW  NE  SW  SE
```

### 3.3 How It Works

**Step 1: Build the Tree**

```
1. Start with world map (root node)
2. Divide into 4 quadrants (NW, NE, SW, SE)
3. For each quadrant:
   - If entities > threshold (e.g., 100):
     - Subdivide into 4 sub-quadrants
   - Else:
     - Mark as leaf node
4. Repeat until all quadrants are leaf nodes
```

**Example:**

```
World Map (10,000 restaurants)
    ↓
Divide into 4 quadrants
    ├─ NW (3,000 restaurants) → Subdivide
    ├─ NE (2,000 restaurants) → Subdivide
    ├─ SW (500 restaurants) → Subdivide
    └─ SE (4,500 restaurants) → Subdivide
```

---

**Step 2: Search for Nearby Entities**

**Problem:** Find all restaurants within 5 km of user

**Algorithm:**

```python
def find_nearby(user_lat, user_lon, radius_km):
    # 1. Find the quadrant where user is located
    current_quadrant = find_quadrant(user_lat, user_lon)
    
    # 2. Get all restaurants in current quadrant
    results = current_quadrant.get_entities()
    
    # 3. Check if radius extends beyond current quadrant
    if radius_extends_beyond(current_quadrant, radius_km):
        # 4. Move to parent quadrant
        parent = current_quadrant.parent
        
        # 5. Search all sibling quadrants
        for sibling in parent.children:
            results += sibling.get_entities()
        
        # 6. If still not enough, move to grandparent
        if radius_extends_beyond(parent, radius_km):
            grandparent = parent.parent
            for child in grandparent.children:
                results += child.get_entities()
    
    # 7. Filter by actual distance
    nearby = []
    for entity in results:
        distance = calculate_distance(user_lat, user_lon, entity.lat, entity.lon)
        if distance <= radius_km:
            nearby.append(entity)
    
    return nearby
```

---

### 3.4 The Problem: Unbalanced Tree

**Issue:** Data is **NOT uniformly distributed**

**Example:**

```
Antarctica: 0 restaurants → Leaf node (depth 1)
Manhattan: 10,000 restaurants → Subdivided 10 times (depth 10)
```

**Result:** **Unbalanced tree!**

```
                    Root
                     │
        ┌────────────┼────────────┐
        │            │            │
   Antarctica     Manhattan     Ocean
   (depth 1)      (depth 10)   (depth 1)
```

**Why this is bad:**
- ❌ Search time varies drastically (O(1) vs O(10))
- ❌ Complex to traverse (backtrack to parent, check siblings)
- ❌ Difficult to maintain

---

### 3.5 Example: Finding Nearby Restaurants

**Scenario:**
- User at location **U**
- Find all restaurants within **5 km**

**Visual:**

```
┌─────────────────────────────────┐
│         │         │         │   │
│    ●    │         │    ●    │   │  ● = Restaurant
│         │         │         │   │  U = User
│─────────┼─────────┼─────────┼───│
│         │    U    │         │   │
│         │    ●    │    ●    │   │
│         │         │         │   │
│─────────┼─────────┼─────────┼───│
│    ●    │         │         │   │
│         │         │    ●    │   │
└─────────────────────────────────┘
```

**Steps:**

1. **User is in quadrant (2, 2)**
   - Contains 1 restaurant ✅

2. **5 km radius extends to neighboring quadrants**
   - Need to check quadrants: (1,1), (1,2), (1,3), (2,1), (2,3), (3,1), (3,2), (3,3)

3. **Traverse tree:**
   - Current quadrant → Parent → All siblings
   - Parent → Grandparent → All siblings
   - **Complex traversal!** ❌

---

### 3.6 Pros and Cons

**Pros:**
- ✅ **Fast search** (if balanced): O(log N)
- ✅ **In-memory:** No database queries
- ✅ **Efficient for static data**

**Cons:**
- ❌ **Unbalanced tree:** Real-world data is not uniform
- ❌ **Complex traversal:** Need to check parent + siblings
- ❌ **High memory usage:** Entire tree in memory
- ❌ **Difficult to update:** Adding new entity requires rebuilding tree
- ❌ **Requires replicas:** Need read + write copies

**When to use:**
- ✅ Static data (hotels, landmarks)
- ✅ Rare updates
- ❌ **Avoid for real-time location updates** (Uber drivers)

---

## 4. Approach 2: Geohash

### 4.1 What is Geohash?

**Geohash** is a geocoding system that encodes latitude/longitude into a short string.

**Key Idea:**
- Divide world into grid
- Assign each grid cell a unique string
- **Nearby locations have similar geohashes!**

### 4.2 How Geohash Works

**Step 1: Divide World into 4 Quadrants**

```
┌─────────────────────────────────┐
│  0 (NW)     │     1 (NE)        │
│             │                   │
│─────────────┼───────────────────│
│             │                   │
│  2 (SW)     │     3 (SE)        │
└─────────────────────────────────┘
```

**Step 2: Recursively Subdivide**

```
Quadrant 0 (NW):
┌─────────────────┐
│  00  │  01      │
│──────┼──────────│
│  02  │  03      │
└─────────────────┘

Quadrant 00:
┌─────────────────┐
│ 000  │ 001      │
│──────┼──────────│
│ 002  │ 003      │
└─────────────────┘
```

**Step 3: Base32 Encoding**

Convert numeric string to Base32:

```
0000 → "0"
0001 → "1"
0010 → "2"
...
tdr1y → Specific location in Bangalore
```

---

### 4.3 Real-World Example

**Location:** McDonald's, Koramangala, Bangalore

**Geohash:** `tdr1y9fu`

**Breakdown:**

| Level | Geohash | Area Covered | Precision |
|-------|---------|--------------|-----------|
| 1 | `t` | India | ~5,000 km |
| 2 | `td` | South India | ~1,250 km |
| 3 | `tdr` | Karnataka | ~156 km |
| 4 | `tdr1` | Bangalore | ~39 km |
| 5 | `tdr1y` | Koramangala | ~5 km |
| 6 | `tdr1y9` | Specific area | ~1 km |
| 7 | `tdr1y9f` | Street | ~150 m |
| 8 | `tdr1y9fu` | Building | ~38 m |

---

### 4.4 Finding Nearby Locations

**Key Property:** **Nearby locations share geohash prefix!**

**Example:**

```
McDonald's:  tdr1y9fu
Starbucks:   tdr1y9fv  ← Only last character differs!
Subway:      tdr1y9fw
```

**Search Algorithm:**

```sql
-- Find all restaurants with geohash starting with "tdr1y9f"
SELECT * FROM restaurants
WHERE geohash LIKE 'tdr1y9f%';
```

**Result:** All restaurants within ~150m!

---

### 4.5 Precision Levels

| Precision | Geohash Length | Lat/Lon Bits | Area Covered |
|-----------|----------------|--------------|--------------|
| 1 | 1 char | 5 bits | ±2,500 km |
| 2 | 2 chars | 10 bits | ±630 km |
| 3 | 3 chars | 15 bits | ±78 km |
| 4 | 4 chars | 20 bits | ±20 km |
| 5 | 5 chars | 25 bits | ±2.4 km |
| 6 | 6 chars | 30 bits | ±610 m |
| 7 | 7 chars | 35 bits | ±76 m |
| 8 | 8 chars | 40 bits | ±19 m |
| 9 | 9 chars | 45 bits | ±2 m |

**Choosing Precision:**

```
5 km radius → Precision 5 (5 characters)
1 km radius → Precision 6 (6 characters)
500 m radius → Precision 6 (6 characters)
100 m radius → Precision 7 (7 characters)
```

---

### 4.6 Database Schema

**Table: Restaurants**

```sql
CREATE TABLE restaurants (
    restaurant_id UUID PRIMARY KEY,
    name VARCHAR(255),
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    geohash VARCHAR(12),  -- Precomputed geohash
    created_at TIMESTAMP
);

-- Index for fast search
CREATE INDEX idx_geohash ON restaurants(geohash);
```

**Example Data:**

| restaurant_id | name | latitude | longitude | geohash |
|---------------|------|----------|-----------|---------|
| rest_1 | McDonald's | 12.9352 | 77.6245 | tdr1y9fu |
| rest_2 | Starbucks | 12.9355 | 77.6248 | tdr1y9fv |
| rest_3 | Subway | 12.9358 | 77.6251 | tdr1y9fw |

---

### 4.7 Search Query

**Find all restaurants within 5 km:**

```sql
-- User location: 12.9350, 77.6240
-- Geohash: tdr1y9

SELECT * FROM restaurants
WHERE geohash LIKE 'tdr1y%'
ORDER BY geohash;
```

**Result:** All restaurants with geohash starting with `tdr1y` (~5 km radius)

---

### 4.8 Generating Geohash

**Python Example:**

```python
import geohash

# Encode lat/lon to geohash
lat, lon = 12.9352, 77.6245
hash_value = geohash.encode(lat, lon, precision=8)
print(hash_value)  # Output: tdr1y9fu

# Decode geohash to lat/lon
decoded_lat, decoded_lon = geohash.decode('tdr1y9fu')
print(decoded_lat, decoded_lon)  # Output: 12.9352, 77.6245
```

**Java Example:**

```java
import ch.hsr.geohash.GeoHash;

// Encode
GeoHash hash = GeoHash.withCharacterPrecision(12.9352, 77.6245, 8);
System.out.println(hash.toBase32());  // Output: tdr1y9fu

// Decode
GeoHash decoded = GeoHash.fromGeohashString("tdr1y9fu");
System.out.println(decoded.getPoint());  // Output: 12.9352, 77.6245
```

---

### 4.9 Pros and Cons

**Pros:**
- ✅ **Simple to implement:** Just string prefix matching
- ✅ **Fast search:** O(1) with index
- ✅ **Easy to update:** Just update geohash column
- ✅ **Low memory:** No in-memory tree
- ✅ **Perfect for real-time updates** (Uber drivers)

**Cons:**
- ❌ **Edge cases:** Locations near quadrant boundaries
- ❌ **No text search:** Only location-based

**When to use:**
- ✅ **Real-time location updates** (Uber, Ola, Rapido)
- ✅ **High-frequency updates** (driver location every 5 seconds)
- ✅ **Location-only search** (no text search needed)

---

## 5. Approach 3: PostgreSQL + GIS Extension

### 5.1 What is PostGIS?

**PostGIS** is a spatial database extension for PostgreSQL that adds support for geographic objects.

**Features:**
- ✅ Geospatial data types (POINT, POLYGON, etc.)
- ✅ Distance calculations
- ✅ Proximity search
- ✅ Built-in indexing (R-Tree)

### 5.2 Installation

```sql
-- Enable PostGIS extension
CREATE EXTENSION postgis;
```

---

### 5.3 Database Schema

**Table: Restaurants**

```sql
CREATE TABLE restaurants (
    restaurant_id UUID PRIMARY KEY,
    name VARCHAR(255),
    location GEOGRAPHY(POINT, 4326),  -- Stores lat/lon
    created_at TIMESTAMP
);

-- Create spatial index
CREATE INDEX idx_location ON restaurants USING GIST(location);
```

**Insert Data:**

```sql
INSERT INTO restaurants (restaurant_id, name, location)
VALUES (
    'rest_1',
    'McDonald''s',
    ST_GeogFromText('POINT(77.6245 12.9352)')  -- lon, lat (note order!)
);
```

⚠️ **Important:** PostGIS uses **(longitude, latitude)** order!

---

### 5.4 Search Query

**Find all restaurants within 5 km:**

```sql
-- User location: 12.9350, 77.6240
SELECT 
    restaurant_id,
    name,
    ST_Distance(
        location,
        ST_GeogFromText('POINT(77.6240 12.9350)')
    ) AS distance_meters
FROM restaurants
WHERE ST_DWithin(
    location,
    ST_GeogFromText('POINT(77.6240 12.9350)'),
    5000  -- 5 km in meters
)
ORDER BY distance_meters;
```

**Explanation:**

- `ST_GeogFromText('POINT(lon lat)')`: Create point from lon/lat
- `ST_DWithin(location, point, distance)`: Check if within distance
- `ST_Distance(location, point)`: Calculate actual distance

---

### 5.5 Pros and Cons

**Pros:**
- ✅ **Built-in functionality:** No external libraries
- ✅ **Accurate distance calculations:** Uses spherical geometry
- ✅ **Spatial indexing:** Fast queries with GIST index
- ✅ **SQL-based:** Easy to integrate

**Cons:**
- ❌ **PostgreSQL only:** Not portable to other databases
- ❌ **Slower than Elasticsearch** for text search
- ❌ **Requires extension:** Extra setup

**When to use:**
- ✅ Already using PostgreSQL
- ✅ Need accurate distance calculations
- ✅ Location + text search (with fuzzy extension)

---

## 6. Approach 4: Elasticsearch

### 6.1 What is Elasticsearch?

**Elasticsearch** is a distributed search engine with built-in geospatial support.

**Features:**
- ✅ Full-text search
- ✅ Geospatial queries
- ✅ Fast (sub-second queries)
- ✅ Scalable

### 6.2 Index Mapping

**Create Index:**

```json
PUT /restaurants
{
  "mappings": {
    "properties": {
      "restaurant_id": { "type": "keyword" },
      "name": { "type": "text" },
      "location": { "type": "geo_point" },
      "cuisine": { "type": "keyword" },
      "rating": { "type": "float" }
    }
  }
}
```

**Index Document:**

```json
POST /restaurants/_doc/rest_1
{
  "restaurant_id": "rest_1",
  "name": "McDonald's",
  "location": {
    "lat": 12.9352,
    "lon": 77.6245
  },
  "cuisine": "Fast Food",
  "rating": 4.2
}
```

---

### 6.3 Search Query

**Find all restaurants within 5 km:**

```json
GET /restaurants/_search
{
  "query": {
    "bool": {
      "filter": {
        "geo_distance": {
          "distance": "5km",
          "location": {
            "lat": 12.9350,
            "lon": 77.6240
          }
        }
      }
    }
  },
  "sort": [
    {
      "_geo_distance": {
        "location": {
          "lat": 12.9350,
          "lon": 77.6240
        },
        "order": "asc",
        "unit": "km"
      }
    }
  ]
}
```

---

### 6.4 Combined Search (Location + Text)

**Find "Pizza" restaurants within 5 km:**

```json
GET /restaurants/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "Pizza"
          }
        }
      ],
      "filter": {
        "geo_distance": {
          "distance": "5km",
          "location": {
            "lat": 12.9350,
            "lon": 77.6240
          }
        }
      }
    }
  }
}
```

**Result:** All pizza restaurants within 5 km!

---

### 6.5 Pros and Cons

**Pros:**
- ✅ **Full-text search:** Best for text + location
- ✅ **Very fast:** Sub-second queries
- ✅ **Scalable:** Distributed architecture
- ✅ **Built-in geospatial:** No extensions needed

**Cons:**
- ❌ **Extra infrastructure:** Need to run Elasticsearch cluster
- ❌ **Eventual consistency:** Not real-time (CDC pipeline)
- ❌ **Higher cost:** More resources

**When to use:**
- ✅ **Text + location search** (restaurant name + proximity)
- ✅ **High search volume**
- ✅ **Already using Elasticsearch**

---

## 7. PostgreSQL + Fuzzy Search (Bonus)

### 7.1 What is Fuzzy Search?

**Fuzzy search** allows matching similar strings (typos, misspellings).

**Example:**
- User searches: "Mcdonald"
- Matches: "McDonald's" ✅

### 7.2 Installation

```sql
-- Enable fuzzy string match extension
CREATE EXTENSION fuzzystrmatch;
CREATE EXTENSION pg_trgm;
```

---

### 7.3 Fuzzy Search Query

**Find restaurants with similar names:**

```sql
SELECT 
    name,
    similarity(name, 'Mcdonald') AS similarity_score
FROM restaurants
WHERE name % 'Mcdonald'  -- % operator for similarity
ORDER BY similarity_score DESC;
```

**Result:**

| name | similarity_score |
|------|------------------|
| McDonald's | 0.85 |
| MacDonald Cafe | 0.72 |

---

### 7.4 Combined: Fuzzy + Proximity

**Find "Pizza" restaurants within 5 km (with fuzzy matching):**

```sql
SELECT 
    restaurant_id,
    name,
    ST_Distance(
        location,
        ST_GeogFromText('POINT(77.6240 12.9350)')
    ) AS distance_meters,
    similarity(name, 'Piza') AS name_similarity
FROM restaurants
WHERE 
    name % 'Piza'  -- Fuzzy match
    AND ST_DWithin(
        location,
        ST_GeogFromText('POINT(77.6240 12.9350)'),
        5000
    )
ORDER BY name_similarity DESC, distance_meters ASC;
```

**Result:** Matches "Pizza Palace" even if user typed "Piza"!

---

## 8. Comparison Table

| Feature | Quad Tree | Geohash | PostgreSQL GIS | Elasticsearch |
|---------|-----------|---------|----------------|---------------|
| **Complexity** | High | Medium | Low | Low |
| **Search Speed** | Fast (in-memory) | Very Fast | Fast | Very Fast |
| **Text Search** | ❌ No | ❌ No | ✅ Yes (with extension) | ✅ Yes |
| **Real-time Updates** | ❌ Difficult | ✅ Easy | ✅ Easy | ✅ Easy |
| **Memory Usage** | High | Low | Low | Medium |
| **Scalability** | ❌ Limited | ✅ High | ✅ High | ✅ Very High |
| **Maintenance** | ❌ Complex | ✅ Simple | ✅ Simple | ✅ Simple |
| **Use Case** | Static data | Real-time location | Location + text | Location + text |

---

## 9. Decision Tree: Which Approach to Use?

```
Do you need TEXT search (restaurant name, dish name)?
    │
    ├─ NO → Use GEOHASH
    │        ✅ Best for: Uber (find drivers), Rapido
    │        ✅ Real-time location updates
    │
    └─ YES → Do you need FULL-TEXT search (fuzzy, autocomplete)?
             │
             ├─ YES → Use ELASTICSEARCH
             │         ✅ Best for: Zomato, Swiggy, Airbnb
             │         ✅ Text + location search
             │
             └─ NO → Use POSTGRESQL + GIS
                      ✅ Best for: Simple location + exact text match
                      ✅ Already using PostgreSQL
```

---

## 10. Interview Q&A

### Q1: Why avoid Quad Tree in production?

**A:**
- ❌ **Unbalanced tree:** Real-world data is not uniform
- ❌ **Complex traversal:** Need to check parent + siblings
- ❌ **High memory:** Entire tree in memory
- ❌ **Difficult updates:** Rebuilding tree is expensive
- ❌ **Requires replicas:** Need read + write copies

**Better alternatives:** Geohash, PostgreSQL GIS, Elasticsearch

### Q2: When to use Geohash?

**A:**
**Use Geohash when:**
- ✅ **Real-time location updates** (Uber drivers)
- ✅ **High-frequency updates** (every 5-10 seconds)
- ✅ **Location-only search** (no text search needed)
- ✅ **Simple implementation**

**Example:** Uber driver location updates

### Q3: When to use Elasticsearch?

**A:**
**Use Elasticsearch when:**
- ✅ **Text + location search** (restaurant name + proximity)
- ✅ **Fuzzy search** (typos, misspellings)
- ✅ **High search volume**
- ✅ **Already using Elasticsearch**

**Example:** Zomato restaurant search

### Q4: When to use PostgreSQL + GIS?

**A:**
**Use PostgreSQL + GIS when:**
- ✅ **Already using PostgreSQL**
- ✅ **Need accurate distance calculations**
- ✅ **Location + exact text match**
- ✅ **Don't want extra infrastructure**

**Example:** Hotel booking with exact name search

### Q5: How does Geohash handle edge cases?

**A:**
**Problem:** Locations near quadrant boundaries

**Example:**
```
Location A: geohash = "tdr1y9"
Location B: geohash = "tdr1z0"  ← Different prefix, but nearby!
```

**Solution:** Search **multiple geohash prefixes**

```sql
-- Search current + neighboring geohashes
SELECT * FROM restaurants
WHERE geohash IN ('tdr1y9', 'tdr1z0', 'tdr1y8', ...)
```

**Better:** Use **geohash neighbors** library

```python
import geohash

neighbors = geohash.neighbors('tdr1y9')
# Returns: ['tdr1y8', 'tdr1yb', 'tdr1yc', 'tdr1z0', ...]
```

### Q6: Can you combine Geohash + Elasticsearch?

**A:**
**Yes!** Elasticsearch internally uses **Geohash** for geospatial indexing.

**How it works:**
1. Elasticsearch encodes lat/lon as geohash
2. Stores in inverted index
3. Searches using geohash prefix matching

**Best of both worlds:**
- ✅ Fast geospatial search (Geohash)
- ✅ Full-text search (Elasticsearch)

### Q7: How to choose geohash precision?

**A:**

| Radius | Precision | Geohash Length |
|--------|-----------|----------------|
| 5 km | 5 | 5 characters |
| 1 km | 6 | 6 characters |
| 500 m | 6 | 6 characters |
| 100 m | 7 | 7 characters |
| 20 m | 8 | 8 characters |

**Rule of thumb:** Higher precision = smaller area

### Q8: Performance comparison?

**A:**

| Approach | Search Time | Update Time | Memory |
|----------|-------------|-------------|--------|
| Quad Tree | 10-50ms | 100ms (rebuild) | High |
| Geohash | 1-5ms | 1ms | Low |
| PostgreSQL GIS | 10-20ms | 5ms | Low |
| Elasticsearch | 5-10ms | 10ms (CDC) | Medium |

**Winner:** Geohash (for location-only search)

### Q9: How to handle millions of entities?

**A:**

**Strategies:**

1. **Sharding by geohash:**
   - Shard 1: geohash starts with "t"
   - Shard 2: geohash starts with "u"
   - Distribute load

2. **Caching:**
   - Cache popular searches in Redis
   - TTL: 5 minutes

3. **Indexing:**
   - Create index on geohash column
   - B-Tree index for prefix matching

### Q10: Real-world example?

**A:**

**Uber:**
- **Problem:** Find drivers within 5 km
- **Solution:** Geohash
- **Why:** Real-time location updates (every 5 seconds)

**Zomato:**
- **Problem:** Find "Pizza" restaurants within 10 km
- **Solution:** Elasticsearch
- **Why:** Text + location search

**Airbnb:**
- **Problem:** Find hotels within 20 km
- **Solution:** PostgreSQL + GIS
- **Why:** Static data, accurate distance calculations

---

## 11. Key Takeaways

✅ **Avoid Quad Tree** in production (complex, unbalanced)  
✅ **Use Geohash** for real-time location updates (Uber)  
✅ **Use Elasticsearch** for text + location search (Zomato)  
✅ **Use PostgreSQL GIS** for simple location + text (Airbnb)  
✅ **Geohash precision:** 5 chars = 5 km, 6 chars = 1 km  
✅ **Nearby locations share geohash prefix**  
✅ **Elasticsearch uses Geohash internally**  
✅ **PostgreSQL GIS uses (lon, lat) order** (not lat, lon!)  

---

## Summary

**Four Approaches:**
1. **Quad Tree:** Avoid (complex, unbalanced)
2. **Geohash:** Best for real-time location updates
3. **PostgreSQL + GIS:** Best for location + exact text
4. **Elasticsearch:** Best for location + full-text search

**Decision Matrix:**

| Use Case | Best Approach |
|----------|---------------|
| Uber (find drivers) | Geohash |
| Zomato (find restaurants) | Elasticsearch |
| Airbnb (find hotels) | PostgreSQL GIS or Elasticsearch |
| Google Maps (find gas stations) | Elasticsearch |

**Interview Strategy:**
- Mention all 4 approaches
- Compare pros/cons
- Choose based on use case
- Justify your decision

**End of Lecture 7**
