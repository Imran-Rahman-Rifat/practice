# Day 1 — How the Web Works

**Topics:** Client-server architecture, HTTP request lifecycle, DNS, IP, TCP, TLS, backend processing, databases, latency, statelessness, and concurrency.

**Goal:** Explain what happens from entering a URL to seeing a webpage, and identify where delays and failures can happen.

## 1. What is a system?

A **system** is a collection of components that work together to achieve a goal. A shopping website, for example, typically needs a user interface, backend logic, persistent data storage, and sometimes payment or file-storage services.

System design means deciding:

- Which components the application needs.
- How data flows between components.
- How the application handles growth, failures, security, and performance.
- Which trade-offs are appropriate for its requirements.

**Key principle:** Start with the simplest architecture that meets the requirements. Add complexity only when needed.

## 2. Client-server architecture

A **client** initiates a request. A **server** receives it, performs work, and sends back a response.

```text
+-----------------+    HTTP request     +------------------+
| Browser / App   | ------------------> | Backend Server   |
| (client)        | <------------------ |                  |
+-----------------+    HTTP response    +--------+---------+
                                               |
                                               | Query / result
                                               v
                                        +--------------+
                                        | Database     |
                                        +--------------+
```

- **Frontend:** Presents the UI and handles user interaction (HTML, CSS, JavaScript, React, etc.).
- **Backend:** Handles API requests, authorization, and business rules (Python, JavaScript, Java, etc.).
- **Database:** Stores and retrieves persistent information (PostgreSQL, MySQL, etc.).

Example: When a user clicks **Add to Cart**, the client sends a request; the backend validates the product and user, updates persistent cart data if appropriate, and returns a result.

**Terminology:** A *server* can mean either a computer/virtual machine or the program serving requests. A database itself can also run as a server.

## 3. What happens when you open a website?

Example URL:

```text
https://example.com/products?id=42
```

| URL part | Meaning |
| --- | --- |
| `https` | Scheme indicating secure HTTP |
| `example.com` | Hostname |
| `/products` | Path requested |
| `id=42` | Query parameter |

A simplified lifecycle for a new HTTPS connection:

1. **Parse the URL:** The browser identifies the scheme, hostname, path, and query.
2. **DNS resolution:** The browser finds one or more network addresses for the hostname, perhaps using cached information.
3. **Connection:** For HTTP/1.1 or HTTP/2 over HTTPS, the client commonly establishes a TCP connection, then negotiates TLS. HTTP/3 instead uses QUIC over UDP.
4. **HTTP request:** The browser sends a method, target path, headers, and possibly a body.
5. **Backend processing:** The server routes the request, runs application logic, and may call a database, cache, or external service.
6. **HTTP response:** The server returns a status, headers, and optionally a response body.
7. **Rendering:** The browser handles HTML, CSS, and JavaScript, often making additional requests for assets and API data.

This is a mental model, not an exact sequence for every page: caches, CDNs, service workers, connection reuse, redirects, and modern protocols can change the flow.

### 3.1 DNS

**DNS (Domain Name System)** maps hostnames to records such as IP addresses. This lets humans use names instead of remembering addresses.

DNS lookups may involve browser/OS caches, recursive resolvers, and authoritative nameservers. DNS can support traffic management, but cached results can delay propagation of changes.

### 3.2 IP addresses and ports

An **IP address** identifies a network endpoint for routing packets. A **port** identifies the receiving service at that endpoint. HTTPS normally uses port **443**.

Useful mental model: the IP address gets traffic to the building; the port directs it to the appropriate service entrance.

### 3.3 TCP

**TCP** provides a reliable, ordered byte stream. A new TCP connection normally begins with a three-way handshake:

```text
Client                            Server
  | ------- SYN ----------------> |
  | <------ SYN + ACK ----------- |
  | ------- ACK ----------------> |
```

TCP retransmits lost data and delivers the stream in order. This matters because network delivery isn't inherently perfect.

### 3.4 TLS and HTTPS

**TLS** protects data in transit, provides integrity protection, and helps authenticate the server using certificates. HTTPS is HTTP communicated over a secure transport.

HTTP/3 is an exception to the usual *TCP + TLS* depiction: it runs over QUIC (over UDP), with integrated TLS 1.3 security.

### 3.5 HTTP

HTTP defines the semantics of web requests and responses, not how the OS routes packets.

Example request:

```http
GET /products?id=42 HTTP/1.1
Host: example.com
Accept: application/json
```

- `GET`: request a representation of a resource.
- `/products?id=42`: the target resource and parameter.
- `Host`: the destination hostname (HTTP/1.1).
- `Accept`: response formats the client can handle.

Example response:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"name":"Keyboard","price":1500}
```

Common status codes:

| Status | Meaning | Example |
| --- | --- | --- |
| 200 | OK | Successful fetch |
| 201 | Created | Resource created |
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Not authenticated |
| 403 | Forbidden | Not allowed |
| 404 | Not Found | Resource missing |
| 429 | Too Many Requests | Rate limit hit |
| 500 | Internal Server Error | Unhandled backend error |
| 503 | Service Unavailable | Temporary service failure |

### 3.6 Server logic and database access

For a simple FastAPI example:

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()
products = {42: {"name": "Keyboard", "price": 1500}}

@app.get("/products/{product_id}")
def get_product(product_id: int):
    product = products.get(product_id)
    if product is None:
        raise HTTPException(status_code=404, detail="Product not found")
    return product
```

The client requests `/products/42`. FastAPI routes the request to the matching function and serializes the returned dictionary as JSON. This example stores data in memory; a real app could query PostgreSQL.

**Security reminder:** Validate access in the backend. A hidden frontend button is not an authorization system.

### 3.7 Browser rendering

After receiving HTML, the browser parses markup, processes CSS, executes JavaScript, lays out elements, and paints the page. JavaScript applications may make additional API requests after the initial document loads.

## 4. How networking layers fit together

For a common HTTPS-over-TCP connection:

```text
HTTP   -> request/response semantics
TLS    -> encrypted, authenticated transport
TCP    -> reliable, ordered bytes
IP     -> addressing and routing packets
```

Remember that HTTP/3 uses QUIC over UDP, so this stack is not universal.

## 5. Latency and throughput

- **Latency:** Time to complete an operation or receive a response.
- **Throughput:** Amount of completed work per unit time (for example, requests per second).
- **P95 latency:** 95% of measured requests complete within this time.
- **P99 latency:** 99% of measured requests complete within this time.

Example of an *illustrative* 2,000 ms sequential user-visible delay:

| Component | Time |
| --- | ---: |
| Network | 150 ms |
| Backend logic | 250 ms |
| Database query | 1,300 ms |
| Browser processing | 300 ms |
| **Total** | **2,000 ms** |

The database dominates in this example. Rather than optimizing CSS or immediately adding servers, investigate the database query first.

**Engineering rule:** Measure before optimizing. Real workloads can overlap operations, so do not assume all timings simply add together.

## 6. Stateless vs stateful servers

A **stateless application server** doesn't need to remember user-specific request history in its own process memory. It can use request credentials and shared stores to retrieve necessary state.

This makes it easier for a load balancer to send consecutive requests to different healthy instances.

A **stateful server** holds relevant state locally, such as active connections or game sessions. Stateful designs can be appropriate but require careful handling of failover and coordination.

**Important:** Stateless servers still use state. The state simply does not need to be stored exclusively inside one server instance.

## 7. Concurrency and race conditions

Suppose inventory is `1` and two customers both try to buy the last item:

```text
Customer A: reads stock = 1
Customer B: reads stock = 1
Customer A: decides purchase is allowed
Customer B: decides purchase is allowed
```

Without appropriate concurrency control, both purchases might be accepted. That's a **race condition**: correctness depends on timing and interleaving of operations.

Solutions include transactions, locks, and atomic conditional updates; these will be covered in the database lessons.

A faster server does not automatically make concurrent operations correct.

## 8. Practical exercises

### Exercise A — Browser Developer Tools

1. Open Chrome or Edge and press `F12`.
2. Select the **Network** tab.
3. Visit `https://example.com` and reload with the Network tab open.
4. Select the document request.
5. Inspect **Headers**, **Response**, and **Timing**.
6. Record the request method, status code, content type, and timing breakdown.

### Exercise B — HTTP with curl

Show response headers and content:

```bash
curl -i https://example.com
```

Show connection and TLS details:

```bash
curl -v https://example.com
```

Show total request time:

```bash
curl -w '\nTotal: %{time_total}s\n' -o /dev/null -s https://example.com
```

Show cumulative timing milestones:

```bash
curl -w '\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n' -o /dev/null -s https://example.com
```

Run the last command five times. Compare the timings. These fields are cumulative milestones, **not** separate durations to add together.

### Exercise C — Student Management System

Design the simplest architecture that supports registering students, viewing profiles, editing information, and enrolling in courses.

Answer:

1. What acts as the client?
2. What are the responsibilities of the backend?
3. Which data belongs in the database?
4. What happens when a user clicks **View Profile**?
5. What response should the user see if the database is unavailable?
6. What might become a bottleneck if many students request profiles concurrently?

A reasonable starting point:

```text
Student Web App <--HTTP--> Backend API <--SQL--> PostgreSQL
```

Do not add microservices or queues without a requirement that justifies them.

## 9. Check your understanding

Try to answer without looking at the notes:

1. What's the difference between a client, an application server, and a database?
2. Why is DNS necessary, and can a DNS answer be cached?
3. How do TCP, TLS, and HTTP differ?
4. What is inside an HTTP request and response?
5. Why might a webpage make dozens of HTTP requests?
6. What is the difference between latency and throughput?
7. Why are stateless application servers often easier to scale horizontally?
8. How can concurrent operations cause incorrect inventory counts?
9. Name at least four stages that could make loading a website slow.
10. Explain the request lifecycle from entering `https://example.com` to seeing a page.

## 10. Day 1 quick revision

| Concept | One-line meaning |
| --- | --- |
| Client | Initiates network requests |
| Server | Responds to requests |
| Backend | Runs application logic |
| Database | Stores and retrieves data |
| DNS | Resolves hostnames to network addresses |
| IP | Addresses and routes packets |
| TCP | Reliable ordered byte stream |
| TLS | Protects data in transit |
| HTTP | Defines web requests and responses |
| Latency | Duration of an operation |
| Throughput | Work completed per unit time |
| Stateless server | Doesn't require per-user state exclusively in local memory |
| Concurrency | Multiple operations progress over overlapping time |

**Completion criterion:** Explain, in your own words, how a browser finds the server, establishes a secure connection, makes an HTTP request, receives a response, and displays content. Mention where backend logic and the database fit, and what may slow down the process.

**Next:** Day 2 — HTTP Methods, REST API Design, Headers, Cookies, Sessions, and Authentication Basics.
