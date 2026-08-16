# API Gateway vs Load Balancer

*Frontend Team Lead's notes — how requests actually reach a microservice*

## The Core Confusion This Clears Up
Both sound like "traffic routers," but they solve **different problems at different layers**:
- **API Gateway** decides *which service* should handle a request
- **Load Balancer** decides *which instance* of that service should handle it

*Frontend analogy:* API Gateway is like your app's router (React Router) deciding which page/component handles a URL. Load Balancer is like deciding which of several identical running copies of that component's server should render it — the "which" is about capacity/health, not routing logic.

## API Gateway
Acts as a **single entry point** for all client requests, sitting in front of your microservices and routing each request to the right one.

Key responsibilities:
- **API Composition** — tailoring/aggregating responses differently depending on the client (e.g., a mobile app might get a lighter payload than a desktop web client, combined from multiple services in one response)
- **Authentication/Authorization** — centralizing security checks here instead of duplicating auth logic in every single microservice
- **Service Discovery** — knowing which service instances are currently healthy/available to actually handle a request

*Frontend analogy:* like a BFF (Backend-for-Frontend) layer — one place that shapes and secures responses before they reach the client, instead of every downstream service having to handle that itself.

## Load Balancer
Once the API Gateway has decided *which microservice* should handle a request, the **Load Balancer** decides *which running instance* of that service actually gets it — since in production, a service usually isn't just one server, it's many identical instances running for redundancy and scale.

- Distributes traffic across those instances to avoid overloading any single one
- Helps ensure uptime — if one instance is unhealthy, traffic goes to the healthy ones instead

## Putting It Together: Full Request Flow
For large-scale systems (millions of requests), the real flow has more layers than just "Gateway then Load Balancer":

1. **DNS-based Load Balancing** — first routes the user to the nearest **geographical region** (reduces latency by not sending an Indian user's request to a US server, for example)
2. **Regional API Gateway** — once routed to a region, the request hits that region's API Gateway, which routes it to the correct microservice
3. **Load Balancer (per service)** — distributes the request across the healthy instances of that specific microservice

This layered structure is also what gives the system **high availability** — if an entire availability zone or region goes down, traffic is rerouted to a healthy one automatically, rather than the whole system failing.

*Frontend analogy:* this is conceptually similar to how a CDN routes users to the nearest edge server before ever reaching your actual origin server — geography-aware routing happens before your application-level routing even begins.

## Interview Questions & Answers

**Q: What's the actual difference in responsibility between an API Gateway and a Load Balancer?**
API Gateway answers "which *service* should handle this?" — it understands your application's routes/domains (e.g. `/orders` → Order Service). Load Balancer answers "which *instance* of that already-chosen service should handle it?" — it doesn't know or care about business logic, just which running copies are healthy and how to spread load evenly across them.

**Q: Why centralize auth at the API Gateway instead of in each microservice?**
It avoids duplicating auth logic (and auth bugs) across every service, and gives you one place to update security policy. The trade-off: the Gateway becomes a critical dependency — if it goes down or has a bug, it can block access to every service behind it, and it needs to be built defensively (high availability, well-tested) since so much now depends on it.

**Q: Why use DNS-based load balancing before requests even reach a regional API Gateway?**
Latency and fault isolation. If every user worldwide hit one central API Gateway, users far from that region would see high latency, and a regional outage would take down everyone. Routing by geography first means users hit a nearby, low-latency entry point, and a regional failure only affects that region's users — other regions keep working.

**Q: What happens to in-flight requests if an entire region goes down?**
Requests already in-flight to the failed region typically time out and fail on the client side (this is why clients need retry logic). New requests get rerouted by the DNS layer to a healthy region once health checks detect the failure — but there's usually a brief detection window (health check interval) where some requests may still fail before rerouting kicks in. This is why idempotent operations and client-side retries matter — worth digging into further as a follow-up topic.

---
*Source: Ultimate System Design playlist, Video 2 (API Gateway vs Load Balancer) — https://youtu.be/m48dMDGZk1w*

> **Note:** This video is from an independent creator, not an official/verified source. Concepts here have been cross-checked for accuracy against standard system design fundamentals rather than taken at face value.
