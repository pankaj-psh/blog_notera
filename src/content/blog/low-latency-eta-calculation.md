---
title: "Architecting Low-Latency ETA Calculation: Third-Party Routing & Local Caching"
description: "A deep dive into building a high-throughput, low-latency ETA calculation architecture using self-hosted OSRM, Uber H3 spatial indexing, and ML post-processing."
pubDate: 2026-10-26
tags: ["system-design", "backend", "scalability", "architecture", "geospatial"]
draft: false
---

# Architecting Low-Latency ETA Calculation: Third-Party Routing & Local Caching

## 1. Introduction

When building real-world ride-hailing or delivery applications, calculating an accurate Estimated Time of Arrival (ETA) is one of the most critical parts of the backend architecture.

ETAs do much more than just display a number on a rider's screen. They power core backend decisions:
- Ride matching & candidate ranking
- Dispatch algorithms
- Surge pricing
- Upfront fare calculations

Scaling this to millions of users globally while relying solely on traditional shortest-path routing engines or external third-party mapping APIs introduces two major challenges:

1. **The Cost and Throughput Bottleneck:** When matching a rider with nearby drivers, evaluating multiple candidate drivers through external APIs generates thousands of requests per second. This spikes network latency and drives up infrastructure and third-party API costs.
2. **The Real-World Accuracy Gap:** Traditional routing engines calculate "naive" ETAs based on ideal graph paths. However, real-world conditions involve parking delays, traffic congestion, route deviations, and pickup friction that standard graph calculations cannot easily predict.

> Modern distributed systems often adopt a hybrid approach—combining self-hosted routing tools with machine learning post-processing, similar to architectures explored in industry research like Uber's DeeprETA.

### 1.1 What is OSRM? (Open Source Routing Machine)

Before diving into system architecture, let us clarify what OSRM is, as it serves as a foundational building block in self-hosted routing architectures.

OSRM (Open Source Routing Machine) is a high-performance, open-source C++ routing engine designed to run shortest-path queries on OpenStreetMap (OSM) data.

To understand how OSRM operates, think of a road network as a directed graph $G = (V, E)$:
- **Vertices ($V$):** Geographic intersections or road turning points.
- **Edges ($E$):** The actual road segments connecting those intersections.
- **Weights ($W$):** The distance or time required to traverse each road segment.

#### How OSRM Delivers Sub-10ms Latency

Standard graph traversal algorithms like Dijkstra's or $A^*$ become far too slow when running live across millions of global intersections.

OSRM accelerates this by pre-processing map data using techniques like **Contraction Hierarchies (CH)** or **Multi-Level Dijkstra (MLD)**:
- When planning a long-distance road trip, you do not evaluate every neighborhood street along the route.
- Instead, you get onto a highway, cover long distances quickly, and only exit back to local streets near your destination.
- OSRM pre-computes these "highway shortcuts" ahead of time, allowing it to evaluate routes in under **$10\text{ ms}$**.

#### Why Host OSRM Yourself?

Running your own containerized OSRM service in your VPC provides two major benefits:
- **Zero Per-Call API Costs:** You eliminate per-request billing from external mapping providers.
- **Low Internal Latency:** Requests stay within your internal private network, skipping public internet round-trips.

However, OSRM provides a "naive ETA"—it calculates travel times based on static or ideal graph states. To turn this into a production-grade ETA, we pair it with local caching and real-world post-processing.

---

## 2. The Core Bottleneck: Latency, Throughput, and Cost

When a rider opens the app and requests a trip, the backend matching algorithm does not calculate just a single route. To find the optimal match, it evaluates dozens of nearby candidate drivers—calculating the ETA from each driver’s current location to the pickup point.

If we rely purely on third-party mapping APIs for these real-time calculations, we hit three major constraints:

### 2.1 High Network Latency
Making external HTTP requests over the public internet to third-party routing APIs introduces an average round-trip latency of **150–400 ms** per call.

### 2.2 Throughput and Fan-Out Multiplication
The problem scales non-linearly due to request fan-out:

$$1\text{ User Request} \rightarrow 10\text{ Candidate Drivers} \rightarrow 10\text{ Route Calculations}$$

If our platform processes $5,000$ active ride requests per second during peak hours, our backend generates $50,000$ route evaluation queries per second. Managing this volume over public API endpoints leads to rate limits, network connection saturation, and degraded user experiences during high-demand windows.

### 2.3 Financial Scale (The API Cost Trap)
Third-party routing services typically charge on a per-request basis. While a fraction of a cent per API call sounds negligible for small applications, back-of-the-envelope math reveals the cost at scale:

$$50,000\text{ routing calls/second} \times 3,600\text{ seconds/hour} = 180\text{ million queries/hour}$$

Even at negotiated enterprise volume rates (e.g., $\$0.50$ per $1,000$ calls), running peak dispatch continuously would result in thousands of dollars per hour in vendor bills.

### The Architectural Problem Statement
This brings us to our primary design goal:
> **How can we serve high-throughput ETA calculations with sub-50 ms internal latency and minimal operational cost without sacrificing prediction accuracy?**

To solve this, we cannot treat route calculation as an external black-box API dependency. We need a multi-tiered architecture that combines localized spatial indexing, fast internal graph lookups, and intelligent local caching.

---

## 3. Graph Modeling & Spatial Indexing Basics

To solve the latency and throughput bottleneck, we first look at how road networks are stored and queried in memory.

At a fundamental level, mapping data is represented as a directed graph $G = (V, E)$, where:
- **Vertices ($V$):** Represent geographic intersections or turning points.
- **Edges ($E$):** Represent road segments connecting those intersections (one-way streets, highways, alleys).
- **Weights ($W$):** Represent the traversal cost, typically measured in distance or estimated travel time based on speed limits.

### The Challenge of Spatial Coordinates vs. Graph Nodes
GPS coordinates from a mobile phone arrive as continuous latitude and longitude pairs—for example, `(37.7749, -122.4194)`.

However, graph algorithms cannot directly query raw floating-point latitude and longitude values. Before we can compute a path, we must perform a **map matching** step: snapping the raw GPS coordinates to the nearest valid edge or vertex in our graph network.

```text
Raw GPS Point (Lat/Lng)
          X
          |
          |  (Spatial Query / Snap)
          v
=================== Nearest Road Edge ===================
Node A (Intersection) -------------> Node B (Intersection)
```

Running a spatial distance check (like the haversine formula) across every road segment in a city to find the nearest point would be computationally prohibitive—$O(N)$ for every incoming request. To make coordinate lookups instantaneous, we rely on **Spatial Indexing**.

### Discrete Spatial Indexing: Uber H3 & Google S2
Instead of treating the world as a continuous map of infinite lat/lng points, spatial indexing frameworks divide the surface of the Earth into discrete, standardized grid cells.

Two of the most widely used systems in production are:
- **Google S2:** Divides the Earth using hierarchical quadrilateral (square-like) projections.
- **Uber H3:** Divides the Earth into a hexagonal hierarchical grid.

#### Why Hexagons (H3)?
In logistics and ride-hailing systems, Uber H3 is preferred over square grids because all neighboring hexagonal cells are equidistant from their center points. This uniform adjacency makes neighborhood radius searches, geospatial aggregations, and spatial caching significantly cleaner to model.

```text
       / \     / \
      /   \___/   \
      \   /   \   /
       \ /  H  \ /
       / \     / \
      /   \___/   \
      \   /   \   /
       \ /     \ /
```

By converting raw coordinates into an H3 Index (represented as a 64-bit integer), we can:
- Instantly group nearby drivers and riders into the same hexagonal cell.
- Perform fast $O(1)$ spatial hash lookups in local memory or Redis.
- Avoid expensive geometric calculations on every single incoming ETA request.

---

## 4. Multi-Tiered Routing Architecture

To eliminate third-party API costs while maintaining sub-50 ms latencies, we adopt a **Multi-Tiered Routing Architecture**. Rather than treating route calculation as a single monolith, we split the workflow into distinct layers:
1. Local caching
2. Self-hosted graph routing
3. Lightweight ML post-processing

```text
+-------------------+         1. Check Cache          +---------------------------+
| Dispatch Engine / | ------------------------------> | Local Tier-1 Cache        |
|  Matching API     | <------------------------------ | (H3 Spatial Grid / Redis) |
+-------------------+         Cache Hit (<5ms)        +---------------------------+
          |
          | 2. Cache Miss: Query Routing Engine
          v
+---------------------------------------------------------------------------------+
| Multi-Tiered Routing Pipeline                                                   |
|                                                                                 |
|  +--------------------------------+       Naive ETA       +------------------+  |
|  | In-House OSRM Cluster          | --------------------> | DeeprETA ML      |  |
|  | (CH / MLD Graph Pre-processing)|  (Ideal Graph Path)   | Post-Processor   |  |
|  +--------------------------------+                       +------------------+  |
|                                                                     |           |
+---------------------------------------------------------------------+-----------+
                                                                      |
                                                            Final Adjusted ETA
                                                             (Sub-50ms Total)
                                                                      v
                                                             +------------------+
                                                             |  Caller / User   |
                                                             +------------------+
```

### 4.1 Spatial Cache Lookup (Tier-1)
Before sending a query to any routing engine, the backend checks a distributed spatial cache (such as Redis).
- Using the Uber H3 grid system, we map origin and destination coordinates to discrete cell IDs at a specific resolution (e.g., H3 resolution 8, which covers roughly $\sim 0.73\text{ km}^2$).
- If another driver in the exact same cell recently requested an ETA to the same destination cell, we serve the cached estimate instantly.
- This layer achieves **sub-$5\text{ ms}$** response times and intercepts a large fraction of redundant requests during peak dispatch windows.

### 4.2 Self-Hosted OSRM Cluster (Naive Routing)
If the request misses the Tier-1 cache, it routes internally to our containerized OSRM cluster deployed within our VPC or private data center.
- **Role:** Calculates topological route, distance, and baseline travel time based on OpenStreetMap graph data.
- **Performance:** Because OSRM leverages Contraction Hierarchies (CH) or Multi-Level Dijkstra (MLD) shortcuts, path computation finishes in **$< 10\text{ ms}$**.
- **Output:** Produces a **Naive Routing Engine ETA ($Y_0$)**, representing theoretical travel time under ideal driving conditions and static road speed limits.

### 4.3 ML Residual Post-Processing (DeeprETA Model)
While OSRM is extremely fast, its naive ETA cannot account for drivers spending minutes looking for parking or localized rush-hour bottlenecks.

Instead of injecting every real-world complexity directly into the heavy graph traversal engine, we pass the naive output $Y_0$ into a lightweight machine learning post-processor (inspired by Uber's DeeprETA):
- **Residual Formulation:** Rather than predicting travel time from scratch, the model predicts the residual difference $r$:

$$Y_{\text{actual}} = Y_0 + r$$

- **Feature Representation:** The model takes discrete spatial embeddings (origin and destination H3 cells), temporal features (time of day, day of week), trip types (ride pickup vs. food delivery), and baseline $Y_0$.
- **Inference Speed:** By using pre-computed $O(1)$ embedding lookups and a shallow architecture (such as a linear transformer), inference completes in **$< 10\text{ ms}$**.

### Summary of the Pipeline
- **Internal Graph Engine:** Handles raw distance and turn-by-turn geometry at zero per-call vendor cost.
- **ML Post-Processor:** Adjusts for real-world driving behavior, parking delays, and dynamic traffic.
- **End-to-End Latency:** The combined workflow completes reliably within a **sub-$50\text{ ms}$** budget.

---

## 5. Smart Caching Strategies

When operating at scale, sending every candidate driver lookup directly to the OSRM cluster—even with sub-10ms graph traversals—can quickly saturate CPU cores during peak hours. A well-designed caching strategy reduces graph engine load, smooths out latency spikes, and keeps infrastructure costs low.

Caching ETAs is trickier than caching static user profiles because routes change, traffic fluctuates, and GPS locations are continuous floats. Here is how we make spatial caching work reliably in production:

### 5.1 Spatial Binning (The Key Quantization Trick)
The biggest pitfall in geospatial caching is using raw latitude and longitude floats as cache keys. A query at `(37.774901, -122.419402)` will never hit a cache key stored at `(37.774908, -122.419415)`, even though those points are just a few feet apart.

To achieve high cache hit rates, we quantize origin and destination coordinates into discrete Uber H3 spatial cells:

```text
Raw Origin Coordinate               Raw Destination Coordinate
(37.7749, -122.4194)                (37.7833, -122.4167)
        |                                   |
H3 Res 8: 8828308281fffff           H3 Res 8: 8828308285fffff
        \                                   /
         \                                 /
          +-------------------------------+
                          |
                          v 
            Redis Key: "eta:8828308281fffff:8828308285fffff"
```

- **Resolution Selection:** Using H3 Resolution 8 ($\sim 0.73\text{ km}^2$) or Resolution 9 ($\sim 0.1\text{ km}^2$) strikes an optimal balance between spatial granularity and cache hit efficiency.
- **Cell-to-Cell Keys:** We compose cache keys using the origin H3 index and destination H3 index combined with temporal buckets (e.g., 15-minute time windows).

### 5.2 Time-to-Live (TTL) Decay and Dynamic Expiry
Traffic conditions change rapidly during rush hours, so cached ETAs cannot remain valid indefinitely. We implement dynamic TTL strategies based on traffic volatility:
- **Static Off-Peak Windows:** 10 to 15-minute TTLs when traffic speeds are stable.
- **Dynamic Peak Windows:** 2 to 5-minute TTLs during active rush hours or high-congestion events.
- **Distance-Based TTL Scaling:** Short trips within dense downtown grids expire quickly (2–3 minutes) because localized bottlenecks form fast. Highway segments covering long distances decay more slowly.

### 5.3 Spatial Blur & Origin/Destination Approximation
In high-density dispatch scenarios—such as evaluating 20 nearby drivers for a single pickup location—the destination is identical for all 20 checks, while candidate drivers are clustered within a tight radius around the origin.

Instead of running 20 distinct graph calculations:
1. We approximate the origin cluster to a single representative H3 centroid.
2. We fetch the baseline ETA for that spatial centroid once.
3. We apply a lightweight Euclidean straight-line distance adjustment ($\Delta t = \Delta d / \text{avg\_speed}$) to compensate for individual driver offsets.

This **Spatial Blur** technique converts $N$ expensive graph traversals into $1$ graph query and $N - 1$ constant-time math adjustments, cutting candidate evaluation graph calls by up to 80%.

| Strategy | Primary Mechanism | Benefit |
| :--- | :--- | :--- |
| **Spatial Binning** | H3 Cell Quantization | Turns continuous GPS points into high-hit-rate cache keys. |
| **Dynamic TTLs** | Traffic-aware expiration windows | Prevents stale routing estimates during traffic transitions. |
| **Spatial Blur** | Centroid-based origin approximation | Reduces fan-out graph calls by up to 80% during candidate matching. |

---

## 6. System Observability & Trade-Offs

Deploying a multi-tiered routing pipeline in production requires robust observability to catch regressions early, alongside a clear understanding of the architectural trade-offs.

### 6.1 Core Observability & Metrics

#### Accuracy Metrics
- **Mean Absolute Error (MAE):** The average absolute difference between predicted ETA ($\hat{Y}$) and Actual Time of Arrival ($Y$).
- **P50 & P95 Error Quantiles:** Tracking median and tail error distributions is critical. A system with an acceptable P50 MAE can still degrade user trust if P95 error spikes to 15+ minutes during adverse weather or sudden traffic spikes.
- **Asymmetric Residual Tracking:** Overestimating arrival times (making a user wait less than expected) is generally less harmful than underestimating arrival times (causing a user to be late). We monitor under-prediction vs. over-prediction bias separately using loss functions like asymmetric Huber loss.

#### Infrastructure & Pipeline Metrics
- **Cache Hit Ratios (Tier-1):** Monitors what percentage of requests are resolved by our spatial H3 cache versus falling back to OSRM.
- **Graph Engine Latency (OSRM):** P99 tail latency of local CH/MLD graph traversals.
- **Post-Processing Inference Latency:** Tracking embedding lookup and model execution times to ensure ML inference stays strictly below $10\text{ ms}$.

### 6.2 Key Architectural Trade-Offs

No architecture is without compromises. Designing a low-latency, hybrid ETA platform requires balancing competing engineering goals:

```text
                  Accuracy (Complex ML Models)
                              /\
                             /  \
                            /    \
                           /      \
                          /        \
   Latency & Throughput  <----------> Operational Complexity
   (Simple Graph / Cache)            (Pre-processing / Custom Map Graphs)
```

1. **Map Freshness vs. Pre-Processing Time:**
   - **Contraction Hierarchies (CH):** Provides blazingly fast queries ($< 5\text{ ms}$), but pre-processing speed weights across a massive graph can take hours. Updating graph weights for sudden road closures or live traffic events becomes difficult.
   - **Multi-Level Dijkstra (MLD):** Slightly higher query latency than CH, but allows sub-minute graph weight updates. Choosing between CH and MLD depends on how quickly your system needs to react to live traffic changes.

2. **Model Complexity vs. Inference Latency:**
   - Deep neural networks and graph convolutional networks (GCNs) can yield higher accuracy by modeling every road segment. However, their execution overhead easily breaches our sub-$50\text{ ms}$ latency budget.
   - By using shallow residual models with pre-computed $O(1)$ spatial embeddings (like DeeprETA), we deliberately trade a fraction of theoretical deep-model precision for microsecond-level serving speeds.

3. **Spatial Cache Granularity vs. Hit Rate:**
   - Using fine-grained H3 cells (Resolution 9, $\sim 0.1\text{ km}^2$) ensures highly precise cache lookups, but lowers overall cache hit ratios.
   - Using coarser H3 cells (Resolution 7, $\sim 5\text{ km}^2$) boosts cache hits significantly, but introduces approximation noise that relies heavier on our post-processing ML layer to correct.

---

## 7. Conclusion and Summary

Architecting a low-latency, production-grade ETA system is fundamentally an exercise in balancing latency, cost, and real-world accuracy.

- Relying solely on third-party mapping APIs leads to prohibitive costs and high latency during high-density dispatch matching.
- Relying purely on raw shortest-path graph calculations fails to capture the messy realities of human driving behavior, parking delays, and traffic fluctuations.

By adopting a hybrid, multi-tiered architecture, we decouple topology calculations from real-world adjustments:
1. **Spatial Indexing & Caching (Uber H3 + Redis)**
2. **In-House Routing (OSRM)**
3. **Residual Post-Processing (DeeprETA Model)**

This architecture reliably keeps end-to-end latency below **$50\text{ ms}$** while scaling effortlessly to thousands of candidate evaluations per second.