---
tags:
  - r-tree
  - geo-search
  - spatial-indexing
  - postgis
  - postgresql
  - system-design
---

# R-Tree — Spatial Indexing

---

## Table of Contents

1. [What Problem Does It Solve?](#1-what-problem-does-it-solve)
2. [The Core Idea — Boxes Inside Boxes](#2-the-core-idea--boxes-inside-boxes)
3. [How a Search Works](#3-how-a-search-works)
4. [What "Overlap" Means — The Rectangle Check](#4-what-overlap-means--the-rectangle-check)
5. [How It Handles Non-Point Shapes](#5-how-it-handles-non-point-shapes)
6. [How R-Tree Is Stored in PostgreSQL](#6-how-r-tree-is-stored-in-postgresql)
7. [How R-Tree Handles Inserts and Updates](#7-how-r-tree-handles-inserts-and-updates)
8. [Geohash vs R-Tree — Side by Side](#8-geohash-vs-r-tree--side-by-side)
9. [Real Scenarios](#9-real-scenarios)
10. [The Quick Decision Guide](#10-the-quick-decision-guide)

---

## 1. What Problem Does It Solve?

R-Tree solves the same nearby-search problem as Geohash — but it's also designed to handle things Geohash cannot:

- Searching inside a **polygon** (a city boundary, a delivery zone)
- Storing **lines** (roads, rivers)
- Storing **shapes** (building footprints, country borders)
- Asking "which delivery zone does this point fall in?"

Geohash only works well with points. R-Tree works with points, lines, rectangles, and complex shapes.

---

## 2. The Core Idea — Boxes Inside Boxes

R-Tree organizes locations as a hierarchy of boxes (rectangles).

Every group of nearby things gets wrapped in a rectangle. Then groups of rectangles get wrapped in a bigger rectangle. And so on.

Think of it like nesting boxes:

```
📦 Big box: "Africa"
   📦 Medium box: "North Africa"
      📦 Small box: "Cairo downtown"
         📍 Cairo Cafe
         📍 Nile Grill
      📦 Small box: "Cairo suburbs"
         📍 Maadi Burger
         📍 New Cairo Diner
   📦 Medium box: "East Africa"
      ...
```

This is the tree. Each box in the tree is called a **node**. The smallest boxes at the bottom contain the actual locations (called **leaves**).

---

## 3. How a Search Works — Skipping Whole Branches

When you search "find restaurants near me," the R-Tree does this:

```
Step 1: Look at the top-level box ("World")
        → Does my search area overlap with this? Yes → go in

Step 2: Look at the children: "Africa", "Europe", "Asia", "Americas"
        → Does my search area overlap with "Africa"? YES → go in
        → Does my search area overlap with "Europe"?  NO  → SKIP ENTIRELY
        → Does my search area overlap with "Asia"?    NO  → SKIP ENTIRELY

Step 3: Look inside "Africa": "North Africa", "East Africa", "West Africa"
        → "North Africa"? YES → go in
        → "East Africa"?  NO  → SKIP ENTIRELY

Step 4: Look inside "North Africa": "Cairo downtown", "Cairo suburbs", "Alexandria"
        → "Cairo downtown"? YES → go in → check individual restaurants
        → "Cairo suburbs"?  YES → go in → check individual restaurants
        → "Alexandria"?      NO  → SKIP ENTIRELY
```

The database skips entire regions — Europe, Asia, East Africa — without looking at a single restaurant in them. It only does precise distance checks on the small set of candidates at the end.

---

## 4. What "Overlap" Means — The Rectangle Check

At each step the R-Tree asks: _does my search circle overlap with this rectangle?_

```
Search circle: center (31.235, 30.044), radius 5 km

Rectangle for "Cairo downtown":
  top-left  (31.20, 30.08)
  bot-right (31.30, 30.00)

Does the circle touch the rectangle? → YES → go deeper
```

This is very fast arithmetic — just comparing numbers. No complex math. And it happens on a small number of rectangles (nodes), not on millions of data points.

---

## 5. How It Handles Non-Point Shapes

This is where R-Tree really shines over Geohash.

**Storing a road:** The road is a line. The R-Tree wraps it in the smallest rectangle that fits around it:

```
Road from point A to point B:

  A ────────────────── B
  ↑                    ↑
  (31.20, 30.10)   (31.40, 30.05)

Bounding rectangle:
  ┌─────────────────────┐
  │  A ──────────────B  │
  └─────────────────────┘
  This rectangle is stored in the index
```

**Storing a delivery zone (polygon):** The polygon gets wrapped in a bounding rectangle. The search finds all rectangles that overlap your query, then does a precise point-in-polygon check only on those candidates.

---

## 6. How R-Tree Is Stored in PostgreSQL

In PostgreSQL (with PostGIS), you create a spatial index like this:

```sql
-- You have a restaurants table with a location column
CREATE INDEX ON restaurants USING GIST(location);
```

`GIST` is PostgreSQL's name for its R-Tree implementation. After you create this index, every search that uses the `location` column automatically uses the tree.

```sql
-- Find restaurants within 5 km of Cairo
SELECT name
FROM restaurants
WHERE ST_DWithin(
  location,
  ST_MakePoint(31.2357, 30.0444)::geography,
  5000  -- meters
);
```

Without the index: PostgreSQL scans every row. Slow. With the index: PostgreSQL walks the R-Tree, skips irrelevant branches. Fast.

---

## 7. How R-Tree Handles Inserts and Updates

When you add a new location:

1. The R-Tree finds the leaf node (small box) whose rectangle needs to grow the least to include the new point.
2. It inserts the point there.
3. If the leaf is too full, it splits into two leaves and the parent box gets updated.

When you delete or move a location, the tree adjusts its boxes accordingly.

This is why R-Tree is good for data that changes — adding, removing, or moving locations updates the tree without rebuilding everything.

| Thing | Description |
|---|---|
| What it is | A hierarchy of nested bounding rectangles |
| How it stores | Each node is a rectangle wrapping its children |
| How it searches | Walk the tree, skip branches that don't overlap |
| Data types | Points, lines, polygons, complex shapes |
| Speed | Very fast — O(log N) for most cases |
| Weakness | Trees can become unbalanced over heavy insert/delete loads |

---

## 8. Geohash vs R-Tree — Side by Side

### The Conceptual Difference

```
Geohash                          R-Tree
────────                         ──────
World divided into a fixed       World divided into flexible
uniform grid                     boxes that adapt to your data

Like postal codes —              Like a filing system —
same size everywhere             folders resize to fit what's inside

Simple and fast                  More powerful, handles shapes
Works well for points only       Works for points, lines, polygons
```

### Feature Comparison

| Feature | Geohash | R-Tree |
|---|---|---|
| Points | ✅ Great | ✅ Great |
| Lines (roads, rivers) | ❌ Not designed for | ✅ Yes |
| Polygons (zones, borders) | ❌ Not designed for | ✅ Yes |
| Radius search | ✅ Yes | ✅ Yes |
| Point-in-polygon | ❌ No | ✅ Yes |
| Simple to use | ✅ Very | ⚠️ Needs PostGIS setup |
| Redis support | ✅ Native | ❌ No |
| PostgreSQL support | ⚠️ Via extension | ✅ Native (GIST) |
| Good for caching layer | ✅ Yes | ❌ Not typical |
| Good for primary database | ⚠️ Limited | ✅ Yes |
| Handles shape updates | ❌ Rebuild needed | ✅ Yes, in-place |

---

## 9. Real Scenarios

### Scenario A — Talabat / Food delivery zones

**The need:** When a user enters an address, check which restaurant delivery zones cover that address. Zones are irregular polygons, not circles.

**Answer: R-Tree in PostgreSQL + PostGIS**

Why?
- Delivery zones are polygons — Geohash can't store or query polygons
- The question is "is this point INSIDE this shape?" — that's a point-in-polygon query
- PostGIS is designed exactly for this
- Zones rarely change, so write performance is not critical

```sql
-- Store zones
CREATE TABLE delivery_zones (
  restaurant_id UUID,
  zone GEOMETRY(Polygon, 4326)
);
CREATE INDEX ON delivery_zones USING GIST(zone);

-- Query: which restaurants deliver to this address?
SELECT restaurant_id
FROM delivery_zones
WHERE ST_Contains(zone, ST_MakePoint(31.2357, 30.0444));
```

---

### Scenario B — Real estate app (draw on map)

**The need:** Users draw a rectangle or polygon on a map and want to see all properties inside that area.

**Answer: R-Tree in PostgreSQL + PostGIS**

Why?
- The search area is a user-drawn polygon, not a fixed radius circle
- Geohash works with radius (circles), not arbitrary shapes
- R-Tree can intersect any shape with any other shape

```sql
-- User draws a box on the map, backend gets the 4 corner coordinates
SELECT * FROM properties
WHERE ST_Within(
  location,
  ST_MakeEnvelope(31.20, 30.00, 31.40, 30.10, 4326)
);
```

---

### Scenario C — Find gas stations along a highway

**The need:** Given a road/route (a line), find all gas stations within 2 km of that route.

**Answer: R-Tree in PostgreSQL + PostGIS**

Why?
- The search shape is a LINE, not a point
- Geohash cannot represent lines as search shapes
- PostGIS can buffer a line (expand it 2 km in all directions) and find all points inside that buffer

```sql
SELECT name FROM gas_stations
WHERE ST_DWithin(
  location,
  ST_GeomFromText('LINESTRING(31.2 30.0, 31.5 30.2, 31.8 30.4)'),
  2000  -- 2 km buffer around the road
);
```

---

### Scenario D — Store locator (McDonald's near me)

**The need:** User searches for stores near their location. Data rarely changes. Needs to be fast.

**Answer: Either works** — Geohash for simplicity + Redis caching speed; R-Tree via PostGIS if you want one database. With only a few thousand stores, performance difference is negligible.

---

## 10. The Quick Decision Guide

```
Is your data changing in real-time (drivers, active users)?
    → YES → Geohash in Redis

Do you need to store or query shapes (polygons, lines)?
    → YES → R-Tree in PostgreSQL + PostGIS

Is your search a simple radius ("find X near me")?
    → YES → Either works. Redis Geohash if you want caching speed.
             PostgreSQL R-Tree if you want one database.

Do you need point-in-polygon ("is this address inside this zone")?
    → YES → R-Tree only.

Is this a cache / speed layer in front of your database?
    → YES → Geohash in Redis.

Is this your primary persistent database?
    → YES → R-Tree in PostgreSQL.
```
