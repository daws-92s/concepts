# 3-Tier Architecture

## The Restaurant Analogy

The easiest way to see why applications split into tiers is to watch a food business grow.

**A roadside cart (1 person, does everything):**
Take the order → cook it → serve it → collect payment. One person handles the entire flow. Fine at small scale — falls apart the moment orders pile up, because the same person can't take orders and cook at the same time.

**A small hotel kitchen (a few people, one split):**
The owner hands out tokens, a cook queues them up and serves in order. One split — order-taking from cooking — already helps throughput.

**A restaurant (fully specialized roles):**

| Restaurant Role | Responsibility |
|---|---|
| **Captain** | Greets you, checks for a free table, directs you in |
| **Waiter** | Takes your order, serves the food — the interface between you and the kitchen |
| **Chef** | Only cooks — receives the order, prepares food, doesn't know or care who the customer is |
| **Food Store** | Holds all the raw ingredients, accessed only by the chef |

More roles, more specialization, more customers served at once — this is exactly why software splits into tiers as it scales: each part does one job and talks only to its neighbor.

## Mapping the Analogy to Software

| Restaurant | Software |
|---|---|
| Customer | End user |
| Captain | Load balancer — routes incoming traffic |
| Waiter | Frontend — the interface between the user and the backend |
| Chef | Backend — processes the request, applies business logic |
| Raw materials / Food store | Data / Database |

```
User (Customer)
     │
     ▼
┌─────────────────────┐
│   Load Balancer      │  ← Captain (routes you to an available table)
└─────────────────────┘
     │
     ▼
┌─────────────────────┐
│   Tier 1: Frontend   │  ← Waiter (takes the request, presents the response)
│   HTML, CSS, JS      │
└─────────────────────┘
     │
     ▼
┌─────────────────────┐
│   Tier 2: Backend    │  ← Chef (processes logic, knows the rules)
│   Java / Python /    │
│   Node.js / PHP /.NET│
└─────────────────────┘
     │
     ▼
┌─────────────────────┐
│  Tier 3: Database    │  ← Food store (holds all the data)
│  MySQL / PostgreSQL  │
│  MongoDB / Kafka     │
└─────────────────────┘
```

## The Tiers

### Tier 1 — Frontend (Web / HTTP Server)
- Delivers the UI to the browser or mobile app: HTML, CSS, JavaScript, or a mobile app
- **Stateless** — holds no user data itself; every request is handled independently
- Never talks directly to the database

### Tier 2 — Backend (Application / Middleware Server)
- Contains the business logic — validations, calculations, CRUD operations
- Accepts requests from the frontend, processes them, queries the database
- **Stateless** — this is what makes it easy to scale: add more backend servers, any of them can handle any request
- Common languages: Java, Python, Node.js, PHP, .NET, Groovy

### Tier 3 — Database Server
- Stores and retrieves persistent data
- **Stateful** — data has to survive a restart, so it can't be scaled as casually as the frontend/backend
- Relational: MySQL, PostgreSQL, MSSQL, Oracle
- Non-relational / other: MongoDB, Kafka

Day-to-day work at this tier includes installing and upgrading the database, backup/restore, creating schemas, monitoring, scaling, and clustering.

## Stateless vs Stateful

| | Stateless | Stateful |
|---|---|---|
| Tiers | Frontend, Backend | Database |
| Scaling | Easy — add more servers behind a load balancer | Hard — data has to stay consistent across nodes |
| Restart impact | None | Data can be lost without a proper persistence/backup setup |

## How Many Servers — 1-Tier vs 2-Tier vs 3-Tier

- **1-tier** — one Linux server does everything: frontend, backend, and database all on the same machine. Simple to set up, but nothing can scale or fail independently.
- **2-tier** — two servers: a client (frontend + business logic combined) talks directly to a separate database server.
- **3-tier** — three servers, one per tier: Frontend → Backend → Database. Each tier is now independent, which is what makes the "why separate tiers" reasons below possible in the first place.

## Why Separate Tiers?

| Reason | Explanation |
|---|---|
| **Security** | The database is never directly reachable from the internet — only the backend can reach it |
| **Scalability** | Each tier scales independently — add more backend servers behind a load balancer without touching the database |
| **Maintainability** | Different teams can own different tiers — frontend, backend, DBA |
| **Fault isolation** | A crashed backend doesn't take the database down with it |
| **Technology freedom** | Swap the frontend server or the database engine without rewriting the other tiers |

See also: [06-linux-admin.md](06-linux-admin.md)
