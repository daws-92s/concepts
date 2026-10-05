# Databases: SQL, NoSQL, Caching, and Message Queues

## What this is / why it matters
"Database" isn't one single thing. A real application usually combines several kinds of storage, each suited to a different job: a relational (SQL) database for data with strict relationships, a NoSQL database for data whose shape varies, a cache for speed, and a message queue for work that doesn't need an immediate answer. Knowing which tool fits which job keeps you from forcing everything into one database that's bad at most of what you're asking it to do.

## How it works

### SQL (Relational) Databases
Structure: **Database → Tables → Rows and Columns**. Every row is one record (e.g. one customer), and columns are fixed per table.

- **Primary keys** and **foreign keys** are mandatory for relating tables to each other — e.g. a `customers` table and a `payments` table are linked by a customer ID, not by stuffing payment info directly into the customer table.
- Best fit: data that needs strict structure and transaction guarantees — orders, payments, anything where "this record is definitely consistent" matters.
- Examples: MySQL, PostgreSQL, Oracle, MSSQL.

### NoSQL Databases
Structure: **Database → Collections → Documents (JSON)**. A document can hold whatever fields it needs — no fixed schema:

```json
{
  "_id": "123456",
  "name": "sivakumar",
  "mobile": "6281937079",
  "email": "info@joindevops.com",
  "address": "nellore",
  "experience": "14 years",
  "skills": "devops"
}
```

A second document in the same collection doesn't have to look the same — one record can have an `address` field and another can skip it entirely. Relationships between documents (e.g. a `student` document and a separate `student_skills` document linked by `student_id`) are *possible* but not mandatory the way foreign keys are in SQL.

| SQL | NoSQL |
|---|---|
| Table | Collection |
| Row | Document |
| Primary key / foreign key relationships: mandatory | Relationships: optional |
| Fixed columns per table | Fields can vary per document |

Best fit: data whose shape varies between records, or that needs to scale fast — product catalogs, user profiles, logs, content. Examples: MongoDB, Cassandra, DynamoDB.

### Using both in the same application
Most real applications don't pick one or the other — they use each where it fits. A typical e-commerce setup:
- **Product catalog** (name, description, ratings, reviews, specs — different per product) → NoSQL, e.g. MongoDB
- **Orders and payments** (strict structure, needs transactions) → SQL

Example — a `catalogue` service backed by MongoDB, listening on its default port `27017`. Other services (e.g. a `user` service) connect to that same MongoDB instance on `27017`, but only the specific instances that need it are allowed to — each service gets its own Security Group, and MongoDB's inbound rule only allows port `27017` from the Security Groups of the services that actually need it, not from every instance in the network. Same least-privilege networking idea as [04-linux.md](04-linux.md)'s firewall/Security Group section, just applied at the database tier.

### Redis: Caching
A **cache** is a key-value store that lives in RAM instead of on disk — the same reason RAM is faster than a hard disk is why a cache is faster than hitting the database directly.

Without a cache, every read does the full round trip: request a connection from the pool → open it → run the query → close it. That's relatively slow if it happens on every single read.

With a cache in front of the database:
```
App → Cache → User              (cache hit — fast)
App → Cache → DB → Cache → User  (cache miss — falls through to the DB, then fills the cache for next time)
```

**TTL** controls how long a cached value is trusted — e.g. cache a read for 1 hour. If the underlying data changes before that hour is up, the app has to explicitly **invalidate** (clear) that cache entry, or users keep seeing the stale value until the TTL runs out on its own.

Example: Redis, Memcached.

### Message Queues: decoupling synchronous and asynchronous work
- **Synchronous (HTTP)** — the client sends a request and waits for a response; if nothing comes back within a timeout, it's an error.
- **Asynchronous (a message queue)** — the client sends a message and moves on immediately, "fire and forget." The receiver processes it whenever it gets to it; the sender isn't blocked waiting.

Two delivery patterns:
| Pattern | Who gets the message |
|---|---|
| **Point-to-point** | Exactly one consumer |
| **Publish/subscribe** | Every subscriber interested in that message |

Examples: RabbitMQ, ActiveMQ, Kafka. (Kafka is technically more of a distributed event-streaming platform than a classic message broker like RabbitMQ/ActiveMQ, but it's commonly grouped with them since it solves the same "don't make the sender wait" problem.)

## Common problems and how to solve them
- **Stale cache reads** — forgetting to invalidate the cache after an update leaves users seeing old data until the TTL expires on its own. Fix: invalidate on write, or set a TTL short enough that temporary staleness doesn't matter for that data.
- **Forcing the wrong model onto the data** — cramming flexible, varying data (like a product catalog) into rigid SQL tables, or putting strict transactional data (like payments) into a schema-less NoSQL store, both cause more problems than they solve. Match the database type to the data's actual shape and consistency needs.
- **Expecting a queue to behave like HTTP** — a message queue intentionally doesn't guarantee an immediate response. If something genuinely needs an answer right now, it belongs over HTTP, not a queue.

## Key takeaways
- SQL enforces structure and mandatory relationships (tables, rows/columns, primary/foreign keys) — best for transactional data like orders and payments.
- NoSQL (e.g. MongoDB) stores flexible JSON documents in collections, with optional relationships — best for data whose shape varies or that needs to scale fast.
- A cache (e.g. Redis) sits in RAM in front of the database to serve fast reads, using TTL to decide how long a value stays trusted before it has to be invalidated.
- A message queue (e.g. RabbitMQ) decouples sender from receiver for asynchronous, fire-and-forget work that HTTP isn't built for.

See also: [07-3-tier.md](07-3-tier.md)
