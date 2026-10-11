# Nginx: Reverse Proxy for a Multi-Service Backend

## What this is / why it matters
[07-3-tier.md](07-3-tier.md) introduced Nginx's roles — web server, load balancer, reverse proxy, SSL termination, caching — using a single backend. Real applications usually split into several backend services instead of one (e.g. a `catalogue` service, a `user` service, a `cart` service), and Nginx is what sits in front of all of them, routing each request to the right one while giving every service consistent tracing, security headers, and error handling it wouldn't otherwise have.

## How it works

### Routing multiple services behind one entry point
Each service gets its own `location` block, matched by a path prefix. The trailing slash on both the path and the `proxy_pass` target matters — it's what strips the `/api/<service>` prefix before forwarding:
```nginx
# /api/catalogue/products -> http://<CATALOGUE-IP>:8080/products
location /api/catalogue/ { proxy_pass http://<CATALOGUE-IP>:8080/; }
location /api/user/      { proxy_pass http://<USER-IP>:8080/; }
location /api/cart/      { proxy_pass http://<CART-IP>:8080/; }
location /api/shipping/  { proxy_pass http://<SHIPPING-IP>:8080/; }
location /api/payment/   { proxy_pass http://<PAYMENT-IP>:8080/; }
```
The browser only ever calls `http://daws92s.store/api/catalogue/product/STAN-1` — it never needs to know that the catalogue service actually lives at its own private IP on port 8080.

### Common settings applied to every backend call
A handful of settings get reused across every `location` block that proxies to a service:
```nginx
proxy_http_version 1.1;
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_connect_timeout 5s;
proxy_read_timeout    30s;   # returns 504 if a service takes longer than this
```
The `X-Forwarded-*` headers matter because, from each service's point of view, every request now looks like it's coming from Nginx — these headers preserve the real client's IP and protocol so the service (and its logs) can still see who actually made the request.

### Tracing one request across every service
```nginx
add_header X-Request-Id $request_id always;
proxy_set_header X-Request-Id $request_id;
proxy_hide_header X-Request-Id;  # a service's own X-Request-Id is hidden; Nginx's is authoritative
```
Nginx generates a unique `$request_id` for every incoming request and passes it to whichever service handles it, and back to the browser in the response. If a request touches multiple services (e.g. catalogue calls cart), that same ID threads through all of them — so one slow or failed click in a browser can be traced across every service's logs using a single ID, instead of guessing which log lines belong together by timestamp.

### Security headers and hiding server details
```nginx
server_tokens off;   # don't reveal the Nginx version to clients
add_header X-Content-Type-Options nosniff always;
add_header X-Frame-Options SAMEORIGIN always;
add_header Referrer-Policy strict-origin-when-cross-origin always;
```
`server_tokens off` is a small but easy win — there's no reason to tell every visitor exactly which Nginx version is running, since that's free information for anyone looking for a known vulnerability to try.

### Compression
```nginx
gzip on;
gzip_types text/css application/javascript application/json image/svg+xml;
gzip_min_length 1024;
```
Nginx compresses responses above a minimum size before sending them, which cuts transfer time for text-heavy responses like CSS, JS, and JSON — images and already-compressed formats don't need this and aren't listed.

### Giving API clients consistent error responses
By default, if Nginx can't reach a service at all, or the service doesn't respond in time, Nginx returns its own plain HTML error page — which is useless to a JSON API client. Mapping those cases to a custom, JSON-shaped response fixes that:
```nginx
error_page 502 = @bad_gateway;
error_page 504 = @gateway_timeout;
location @bad_gateway {
    default_type application/json;
    return 502 '{"message":"service not reachable (502 Bad Gateway from nginx)"}';
}
location @gateway_timeout {
    default_type application/json;
    return 504 '{"message":"service did not answer in time (504 Gateway Timeout from nginx)"}';
}
```
This only covers errors from Nginx itself (it couldn't reach the service, or the service was too slow) — an error a service returns on its own (its own `404`, `503`, etc.) passes straight through untouched, since that's the service correctly reporting its own problem.

### Health checks
```nginx
location = /health {
    stub_status on;
    access_log off;
}
```
`stub_status` exposes Nginx's own connection counters — this is a check on Nginx itself, separate from whether any backend service is healthy.

### Caching static assets
```nginx
# product images — fall back to a placeholder if missing
location /images/ {
    expires 1h;
    try_files $uri /images/placeholder.jpg;
}

# filenames in /assets contain a content hash, so they never change once built — cache forever
location /assets/ {
    expires max;
    try_files $uri =404;
}
```
The difference in cache duration is deliberate: images can change without their filename changing, so they get a short cache window; hashed asset filenames are guaranteed to change if their content does, so caching them forever is safe.

### Serving a single-page app (React, etc.)
```nginx
location / {
    expires -1;
    try_files $uri $uri/ /index.html;
}
```
React (or any client-side router) handles paths like `/product/STAN-1` or `/cart` entirely in the browser — the server never actually has a file at that path. Falling back to `index.html` for any unmatched path lets React's own router take over and render the right page client-side.

## Common problems and how to solve them
- **Forgetting the `X-Forwarded-*` headers** — every service's logs then show Nginx's own IP as the client, making it impossible to trace real traffic sources or debug per-client issues.
- **Leaving Nginx's default error pages in place for a JSON API** — clients expecting JSON get an HTML error page instead, which breaks anything trying to parse the response. Fix with custom `error_page` blocks that return JSON.
- **Caching hashed assets too conservatively, or un-hashed images too aggressively** — match the cache duration to whether the filename actually changes when the content does.

## Key takeaways
- One Nginx server can front many backend services, routing by path prefix (`/api/<service>/`) so the browser never needs to know any service's real address.
- `X-Request-Id` lets a single request be traced across every service it touches — essential once there's more than one service involved.
- Nginx's own errors (502/504) and a service's own errors are different things — only the former should be reshaped into API-friendly JSON.
- Cache duration should match how the asset actually changes: forever for hashed, immutable filenames; short for anything that can change without a filename change.

See also: [07-3-tier.md](07-3-tier.md), [10-backend.md](10-backend.md)
