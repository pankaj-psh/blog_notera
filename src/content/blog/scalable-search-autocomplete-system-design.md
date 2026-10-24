---
title: "Design and Architecture of a Scalable Search Autocomplete System"
description: "A practical guide to designing a high-throughput, low-latency search autocomplete system that handles billions of queries."
pubDate: 2026-10-25
tags: ["system-design", "backend", "scalability", "architecture"]
draft: false
---

# Design and Architecture of a Scalable Search Autocomplete System

## 1. Introduction

- Have you ever used Google Search? When you start typing a letter, like `h`, a list of popular searches pops up right away. You might see suggestions like:
  - `how to make coffee`
  - `how long to boil eggs`
- This feature is called **autocomplete** or **typeahead**. It saves people millions of typing hours every year.
- Our goal is to design this system from scratch: a backend service that takes what a user types and instantly shows the top ten most popular searches. It has to be very fast, handle billions of searches every day, and never lag behind the user's typing speed.
- An average person types about four characters per second. That means our system must return suggestions in less than **240 ms**. Suggestions do not need to be updated every second; it is acceptable for the list to be up to one hour old.
- The frontend talks to the backend using simple web requests. Every time the user types a new letter, the app asks the backend for suggestions. Once the user finishes typing and hits Search, it sends the final query to the actual search engine.

## 2. System API Design

The frontend search box needs two simple API calls to communicate with the backend.

### 2.1 Live autocomplete requests — `/complete`

Every time the user types a new letter or presses Backspace, the browser sends a request to the server.

```http
GET /complete?q=ho
```

Here, `q` is the text typed so far. The server receives the prefix `ho` and replies with a small JSON list of the top ten suggestions:

```json
{
  "prefix": "ho",
  "suggestions": [
    "how to make coffee",
    "how long to boil eggs",
    "hot dog recipe",
    "hotel deals",
    "home depot",
    "hollyhock flowers",
    "how to learn python",
    "how much is bitcoin",
    "hollow knight guide",
    "house for sale"
  ]
}
```

Because this runs on every keypress, it has to be extremely fast. We do not search or sort on the fly while the user types; we look up the answer and send it back immediately.

### 2.2 Final search requests — `/search`

When the user sees a suggestion they like and clicks it—or presses Enter—the browser makes a second request:

```http
GET /search?q=how+to+make+coffee
```

This request does two important things:

1. Returns the actual search results web pages.
2. Silently saves a note in the background that someone searched for `how to make coffee` at a particular time.

## 3. Initial Approach and Limitations

When developers first think about building autocomplete, a common idea is a tree data structure called a **trie** (pronounced “try”).

### 3.1 What is a trie?

Imagine a tree where each branch is a single letter. As you walk down the branches, the letters combine to spell words. If you type `h`, the system navigates to the `h` node. From there, it looks at sub-branches such as `ho`, `how`, and `hotel`, and gathers the words with the highest frequency counts stored at the relevant nodes.

A simplified example:

```text
(root)
  |
  h
 / \
e   o
|  / \
l  t   w
|  |   |
p  el  to make coffee
```

At first glance, a trie seems ideal. Finding a prefix in a trie is fast because lookup speed depends on the length of the typed word, not on how many words exist in the database.

### 3.2 Why do tries fail at web scale?

A trie can work well for a small app, but it becomes difficult to scale to millions of global users for three main reasons:

1. **Severe memory waste**
   - Trees are built from many small linked nodes connected by pointers. Every new letter creates nodes and pointers, which use extra memory. Storing hundreds of millions of search phrases in a large trie can consume a huge amount of RAM.
2. **Slow sorting on every keystroke**
   - Finding the node for `ho` is fast, but the system must still search its descendants to determine the ten most popular words. Traversing many branches and sorting them on every keystroke can cause CPU spikes and threaten the 240 ms target.
3. **Concurrent write locks**
   - A search engine constantly receives new queries. Updating frequency counts directly in the trie while millions of users read from it can cause contention. Reads may be blocked while writes finish, increasing latency.

## 4. Architecture Pivot: CQRS and Batch Processing

Since tries and real-time database updates can become bottlenecks at scale, the strategy changes to solve two problems:

- Stop calculating popular searches on the fly.
- Stop allowing writes to slow down reads.

Two key ideas help address these problems: **CQRS** and **batch processing**.

### 4.1 CQRS — separating reads from writes

CQRS stands for **Command Query Responsibility Segregation**. The idea is to split the system into two separate paths:

- **Write path:** Records what people are searching for. It collects raw logs in the background.
- **Read path:** Answers keystrokes. It does not write anything; it looks up precomputed answers as quickly as possible.

Because reads and writes are separated, users typing in the search bar do not have to wait for background logs to be saved.

### 4.2 Batch processing — the prep-kitchen analogy

Imagine a busy restaurant kitchen. If the kitchen prepared every ingredient only after each customer ordered, food would take a long time to arrive. Instead, the kitchen prepares ingredients ahead of time so they are ready when needed.

The autocomplete system works similarly:

- It does not sort millions of search phrases while the user is typing.
- It collects search logs over time.
- Once every hour, a background job processes the logs to identify the top searches and saves the results in ready-to-use lists.
- When a user types `ho`, the server retrieves the precomputed list instead of calculating it from scratch.

## 5. The Write Path: Batch Pipeline

The write path processes large volumes of raw search logs and turns them into precomputed lists.

### Step 1: Data collection and 1% sampling

Whenever someone submits a search, the system writes a line to a raw analytics log, for example:

```text
[10:30:01 AM] search_query: "how to make coffee"
[10:30:01 AM] search_query: "hotel deals"
[10:30:02 AM] search_query: "how to make coffee"
```

Saving every log when billions of searches occur daily can overwhelm storage. For autocomplete, recording a random **1% sample** may be enough to identify popular terms while greatly reducing data volume.

### Step 2: Streaming logs through message brokers

Instead of writing logs directly to a slow database, the server sends sampled logs to a streaming system such as **Apache Kafka**.

Kafka acts like a fast conveyor belt: it receives large volumes of logs, holds them, and passes them to the processing engine.

### Step 3: MapReduce aggregation job

Once every hour, a background job—such as one built with **Apache Hadoop** or **Spark**—processes the logs in two phases.

#### 3.1 Map phase: generating prefixes

Workers take each full search query and break it into all its starting prefixes. For example, for the search term `hotel`, they create pairs like these:

```text
h     → hotel
ho    → hotel
hot   → hotel
hote  → hotel
hotel → hotel
```

#### 3.2 Reduce phase: sorting the top ten

Another group of workers groups the pairs by prefix. For the prefix `ho`, the system counts how often each full query appeared during the hour:

```text
how to make coffee: 50,000 counts
hotel deals:        35,000 counts
home depot:         20,000 counts
hollow knight guide: 5,000 counts
```

Finally, the reducer sorts the queries by count, keeps the top ten, and discards the rest.

## 6. The Read Path: Serving Layer

Once the batch pipeline has calculated and saved the top ten list for each prefix, the read path serves users' requests in real time.

### 6.1 Flat key-value storage — O(1) lookups

Instead of using a complex tree such as a trie, the serving layer uses a key-value store such as **Redis** or **Cassandra**. The data is stored in a flat table:

| Key (prefix) | Value (precomputed top-ten list) |
|---|---|
| `h` | `["how to make coffee", "how long to boil eggs", ...]` |
| `ho` | `["how to make coffee", "hotel deals", "home depot", ...]` |
| `hot` | `["hot dog recipe", "hotel deals", "hotmail login", ...]` |

When the user types `ho`, the server does not search, calculate, or sort. It looks up the `ho` key and returns the stored array. A key-value lookup is commonly treated as **O(1)** on average, so the lookup itself can be very fast.

### 6.2 Multi-level caching

To meet the sub-240 ms latency goal globally, caches are placed at several layers between the user and the primary key-value database:

1. **Browser cache:** After a user receives suggestions for `ho`, the browser can keep the JSON response in local memory. If the same prefix is requested again, it may display the cached list without making another network request.
2. **CDN / edge cache:** Edge servers around the world, such as Cloudflare or AWS CloudFront, cache popular prefixes like `a`, `in`, or `ho` close to users. This can answer requests without contacting the main database.
3. **In-memory Redis cache:** If a request misses the CDN cache, the backend can read from Redis before falling back to persistent storage.

### 6.3 Graceful fallbacks

If someone types a rare prefix such as `ho79x` that does not exist in the precomputed table, the server returns an empty list (`[]`) immediately. The user experiences no typing lag and can continue entering a custom query.

## 7. Scaling and Performance Optimizations

Even with key-value lookups and caching, handling billions of requests per day worldwide requires infrastructure that scales.

### 7.1 Database sharding and partitioning

A single database server cannot necessarily store every prefix or handle all incoming traffic. The key-value database can be divided across multiple servers using **sharding**.

Rather than assigning prefixes alphabetically—which could overload a server holding popular prefixes—the design uses consistent hashing. A simplified hash-based assignment is:

\[
\text{Server ID} = \text{Hash( Prefix )} \bmod N
\]

Hashing spreads prefixes across database nodes to balance storage and traffic.

### 7.2 Handling hotspot prefixes

Single-letter and two-letter prefixes such as `g` or `ho` can receive far more requests than longer prefixes such as `hollyhock flowers`. This creates a hotspot if one server receives too much traffic.

Two possible solutions are:

- **Prefix frequency replication:** Copy extremely popular prefixes to multiple replica servers. Requests for `ho` can then be load-balanced across replicas rather than sent to one machine.
- **Aggressive edge caching:** Short prefixes change relatively infrequently. Longer cache lifetimes can allow a large share of requests for prefixes such as `h` or `ho` to be answered at the network edge.

### 7.3 Multi-region deployment

To keep responses fast for users around the world, deploy read-path servers in multiple data centers, such as in the US, Europe, and Asia-Pacific. **Anycast DNS** can help route a user's request to a nearby data center.

## 8. Conclusion and Summary

Building a web-scale autocomplete system can seem complicated, but the architecture becomes easier to understand when divided into separate components. Instead of relying on complex tries or forcing a database to calculate suggestions in real time, the system uses precomputed results and fast lookups.

### Key takeaways

- **Decouple reads and writes (CQRS):** Separating live user requests from background logging helps keep typing responses fast and independent of heavy database traffic.
- **Precompute everything offline:** Batch processing with MapReduce performs the expensive work in the background. By calculating the top ten results every hour, the live API can use simple lookups.
- **Optimize the read path:** Flat key-value stores and multi-layer caching—browser, CDN, and Redis—help deliver fast responses at high request volumes.
- **Scale smartly:** Database sharding, replication of hot prefixes, and multi-region deployments help keep the system available and responsive worldwide.

By combining batch processing with fast key-value serving, a large real-time computation problem becomes a straightforward lookup system.
