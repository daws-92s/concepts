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

## Hands-On Practice Setup

For a lab where you build a 3-tier deployment yourself (e.g. an Expense Tracker app across three EC2 instances), a ready-made practice AMI has the tools preinstalled so you're not stuck on environment setup instead of the architecture itself:

| Item | Value |
|---|---|
| AMI name | Redhat-9-DevOps-Practice |
| AMI ID | `ami-0220d79f3f480ecf5` |
| Default OS login | `ec2-user` / `DevOps321` |
| Default DB login | `root` / `ExpenseApp@1` |

These are default credentials baked into a public training AMI, not production secrets — change them (or don't reuse the AMI) for anything beyond a throwaway practice environment.

## Deploying the Backend

The backend deployment steps, in order:
1. Install the programming language runtime (e.g. Node.js)
2. Create a directory to hold the application (e.g. `/app`)
3. Create a dedicated system user to run it
4. Download the application code
5. Install its dependencies
6. Create a systemd service file so the OS manages it as a proper service

**Why a system user, not a human or root user?** Running an application under a real person's login is a real risk, not just a convention:
- That human account has more privileges than the app needs — if the server is compromised, the blast radius is bigger than it needs to be
- Files end up owned by a specific person's name instead of the application
- What happens when that person leaves the company? Their account (and anything tied to it) has to be dealt with
- It breaks auditing — you can't tell "the app did this" from "a person did this" in the logs

A **system user** solves this: no interactive login, no password, no shell — it exists purely to own and run the process, with only the access it actually needs (least privilege).
```
useradd --system --home /app --shell /sbin/nologin --comment "expense system user" expense
```

**Build tools** — every language has a standard way to declare dependencies and package the app into a deployable artifact (`.zip`, `.tar.gz`, `.jar`, `.war`):

| Language | Build tool | Build file | Source extension |
|---|---|---|---|
| Java | Maven | `pom.xml` | `.java` |
| Node.js | npm | `package.json` | `.js` |
| Python | pip | `requirements.txt` | `.py` |

For Node.js specifically: `npm install` reads `package.json` and downloads everything into a `node_modules` folder; `package-lock.json` pins the exact versions actually installed (including sub-dependencies), so a install today matches an install next month.

Downloading a prebuilt artifact and extracting it:
```
curl -o /tmp/backend.tar.gz <artifact-url>
```

**Running it as a real service**, instead of a script someone has to remember to start — a systemd unit file at `/etc/systemd/system/backend.service`:
```ini
[Unit]
Description=Expense Backend Service
After=network.target

[Service]
User=expense
Environment=DB_HOST=<backend-private-ip>
Environment=DB_USER=expense
Environment=DB_PWD=ExpenseApp@1
Environment=DB_DATABASE=transactions
ExecStart=/bin/node /app/index.js
SyslogIdentifier=backend
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
This answers the three questions any service needs answered: **who** runs it (`User=`), **how** to start it (`ExecStart=`), and **what environment** it needs (DB host, credentials, database name) — plus what to do if it crashes (`Restart=on-failure`).

`systemctl start backend` then: looks up `backend.service` in `/etc/systemd/system`, runs the `ExecStart` command as the specified `User`, and injects the declared environment variables.

## Public vs Private IP
IPv4 has about 4 billion addresses (2³²). A **public IP** is reachable from the internet; a **private IP** (like `172.31.0.54`) only works inside its own network (e.g. a VPC). The backend and database only need private IPs — nothing outside the VPC should be able to reach them directly; only the frontend needs a public IP, since it's the only tier the internet is allowed to talk to.

## Deploying the Frontend (Nginx)
Modern backend frameworks increasingly ship with their own lightweight built-in server, so a separate, heavyweight application server often isn't needed anymore. On the frontend side, **Nginx** is the popular choice, and it does more than just serve static files:
- HTTP server (serves HTML/CSS/JS)
- Load balancer
- Reverse proxy
- SSL/TLS termination
- Caching

Useful defaults to know:
- `/usr/share/nginx/html/` — where Nginx serves static files from by default (`index.html` lives here)
- `/etc/nginx/nginx.conf` — the main config file
- `/var/log/nginx/` — access and error logs

A domain name is just a friendlier way to reach the same public IP — `http://daws92s.online` and `http://<frontend-public-ip>` point to the same server. Standard ports: HTTP defaults to 80, HTTPS defaults to 443; a non-standard port (e.g. `:81`) has to be specified explicitly in the URL.

An access log line breaks down like this:
```
203.0.113.42 - - [30/Sep/2026:02:24:50 +0000] "GET / HTTP/1.1" 200 9466 "-" "Mozilla/5.0 ..."
```
client IP → timestamp → request line (method + path) → status code → response size → user agent.

## Forward Proxy vs Reverse Proxy
Both act "on behalf of" someone, but on opposite sides of the request:

| | Forward Proxy | Reverse Proxy |
|---|---|---|
| Acts on behalf of | The client | The server |
| Hides | The client's identity from the server | The server's identity from the client |
| Typical uses | VPNs, changing geo-location, traffic monitoring, content restriction | SSL/TLS termination, caching, load balancing |

The frontend Nginx server acts as a **reverse proxy** for the backend — the browser only ever talks to Nginx, never directly to the backend server:
```nginx
location /api/ {
    proxy_http_version 1.1;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_pass http://<backend-private-ip>:8080/;
}
```
The `X-Forwarded-*` headers matter because, from the backend's point of view, every request now looks like it's coming from Nginx — these headers preserve the original client's IP and protocol so the backend (or its logs) can still see who actually made the request.

A small health-check endpoint is common alongside this:
```nginx
location /health {
    stub_status on;
    access_log off;
}
```

## REST API & HTTP Methods
An **API** (Application Programming Interface) is how the frontend and backend talk to each other. A REST API maps CRUD operations onto HTTP methods, all against the same URL:

| Operation | HTTP Method | Example |
|---|---|---|
| Read | `GET` | `GET /api/transaction` — list transactions |
| Create | `POST` | `POST /api/transaction` with a JSON body — add a transaction |
| Update | `PUT` | `PUT /api/transaction` with a JSON body (including the `id`) — update a transaction |
| Delete | `DELETE` | `DELETE /api/transaction/32` — delete transaction 32 |

A typical response is JSON — structured, key-value data that's easy for both humans and code to read:
```json
{
  "result": [
    { "id": 26, "amount": 1000, "description": "food", "category": "Food" },
    { "id": 25, "amount": 500, "description": "shopping in mall", "category": "Shopping" }
  ]
}
```

## HTTP Status Codes
Computers work in numbers; status codes are how a server tells the client what happened without needing a full sentence:

| Range | Meaning | Common codes |
|---|---|---|
| 1XX | Informational | — |
| 2XX | Success | `200` OK, `201` Created, `204` No Content (e.g. after a delete) |
| 3XX | Redirection | — |
| 4XX | Client-side error | `400` Bad Request, `401` wrong/missing credentials, `403` Forbidden (no authorization), `404` Not Found |
| 5XX | Server-side error | `500` Internal Server Error, `501` Not Implemented, `502` Bad Gateway (frontend can't reach backend), `503` Service Unavailable, `504` Gateway Timeout (backend didn't respond in time) |

`502` and `504` are worth knowing well in a 3-tier setup specifically — they usually mean the problem isn't Nginx itself, it's that the backend tier behind it is down or too slow.

## Why Separate Tiers?

| Reason | Explanation |
|---|---|
| **Security** | The database is never directly reachable from the internet — only the backend can reach it |
| **Scalability** | Each tier scales independently — add more backend servers behind a load balancer without touching the database |
| **Maintainability** | Different teams can own different tiers — frontend, backend, DBA |
| **Fault isolation** | A crashed backend doesn't take the database down with it |
| **Technology freedom** | Swap the frontend server or the database engine without rewriting the other tiers |

See also: [06-linux-admin.md](06-linux-admin.md)
