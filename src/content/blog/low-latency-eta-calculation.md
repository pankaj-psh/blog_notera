Architecting Low-Latency ETA Calculation: Third-Party Routing & Local Caching


1. Introduction
	-> As the title suggests, when we build real-world ride-hailing or delivery applications, calculating an accurate Estimated Time of Arrival (ETA) is one of the most critical parts of our backend architecture.
	
	-> Let me explain one thing here: ETAs do more than just display a number on a rider's screen. They power the core logic behind the scenes—including ride matching, dispatch algorithms, surge pricing, and upfront fare calculations.
	
	-> Now, if we scale this to millions of users globally, relying solely on traditional shortest-path routing engines or external third-party mapping APIs introduces two major challenges:
		1. The Cost and Throughput Bottleneck: When matching a rider with nearby drivers, evaluating multiple candidate drivers through external APIs generates thousands of requests per second. This increases network latency and significantly drives up cloud infrastructure and API costs.
		2. The Real-World Accuracy Gap: Traditional routing engines calculate "naive" ETAs based on ideal graph paths. However, real-world conditions involve variables like parking delays, route deviations, and pickup friction that standard graph calculations cannot easily predict.
		
			`To address these challenges, modern distributed systems often adopt a hybrid approach—combining self-hosted routing tools with machine learning post-processing, similar to architectures explored in industry research like Uber's DeeprETA.`
	
	-> 1.1. What is OSRM? (Open Source Routing Machine)
		-> Before we move ahead into system designs, let's clarify what OSRM actually is, as it serves as a foundational block in self-hosted routing architectures.
		-> OSRM stands for Open Source Routing Machine. It is a high-performance, open-source C++ routing engine designed to run shortest-path queries on OpenStreetMap (OSM) data.
	-> To understand how OSRM operates, think of a road network as a simple graph:
		-> Nodes: Intersections or turning points on a map.
		-> Edges: The actual road segments connecting those intersections.
		-> Weights: The distance or time required to traverse each road segment.
	-> How OSRM Delivers Sub-10ms Latency:
		-> Standard graph traversal algorithms like Dijkstra's or $A^*$ become far too slow when running live across millions of global intersections.
		-> OSRM speeds this up by pre-processing map data using techniques like Contraction Hierarchies (CH) or Multi-Level Dijkstra (MLD).
		-> Imagine planning a long-distance road trip:
			-> you don't evaluate every neighborhood street along the route.
			-> Instead, you get onto a highway, cover long distances quickly, and only exit back to local streets near your destination.
			-> OSRM pre-computes these "highway shortcuts" ahead of time, allowing it to evaluate routes in under $10\text{ ms}$.
	-> Why Host OSRM Yourself?
		-> Running your own containerized OSRM service in your VPC provides two major benefits:
			-> Zero Per-Call API Costs: You eliminate per-request billing from external mapping providers.
			-> Low Internal Latency: Requests stay within your internal network, skipping public internet round-trips.
	
		However, OSRM provides a "naive ETA"—it calculates travel times based on static or ideal graph states. To turn this into a production-grade ETA, we still need to pair it with local caching and real-world post-processing.

2. The Core Bottleneck: Latency, Throughput, and Cost

3. Graph Modeling & Spatial Indexing Basics

4. Multi-Tiered Routing Architecture

5. Smart Caching Strategies

6. System Observability & Trade-Offs

7. Conclusion and Summary
