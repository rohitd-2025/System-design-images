# Flash Sale System Design Workshop

## Overview
Design a high-traffic flash sale system for an e-commerce platform based on the .NET eShop reference architecture. This workshop will test your understanding of scalability, consistency, performance, and system design principles.


**Scenario:** Your e-commerce platform wants to launch flash sales where limited quantities of products are sold at discounted prices for a short duration (e.g., 1000 units of iPhone at 50% off for 2 hours). The system must handle:
- 100,000+ concurrent users
- High read traffic during sale announcements
- Burst write traffic when sale starts
- Inventory consistency (no overselling)
- Fair ordering (first-come, first-served)

## Reference Architecture
Study the [.NET eShop](https://github.com/dotnet/eShop) microservices architecture and adapt it for flash sale requirements.

## Team Details
**Team Name:** Scalability Squad

**Team Members:**
- Rohit Vikas Dadgal (Devops Specialist)
- Harsh Shivam (Backend Developer)
- Ankisha Srivastava (System Architect)
- Jashandeep Kaur (Performance Engineer)


---

## Exercise 1: System Architecture Design

### Task 1.1: High-Level Architecture Diagram
**Deliverable:** Create a system architecture diagram showing all components needed for flash sales.

**Requirements to Address:**
- Microservices involved (Catalog, Inventory, Ordering, Payment, etc.)
- External systems (CDN, Load Balancers, Caching layers)
- Data storage solutions
- Message queues/Event buses
- API Gateway configuration

**Architecture Diagram:**

### C1: Context Diagram

![Context Diagram](https://raw.githubusercontent.com/rohitd-2025/System-design-images/866e9320c3158364e7bc8da8a13ee13b18a7ba65/C1.png)

### C2: Container Diagram

![Container Diagram](https://raw.githubusercontent.com/rohitd-2025/System-design-images/main/C2.png)

### C3: Component Diagram

![Component Diagram](https://raw.githubusercontent.com/rohitd-2025/System-design-images/main/C3.png)

### C4: Overall Diagram

![Overall Diagram](https://raw.githubusercontent.com/rohitd-2025/System-design-images/main/C4.png)

### Task 1.2: Service Breakdown
**Deliverable:** List and describe each microservice with its responsibilities.

| Service Name | Primary Responsibility | Key APIs | .NET Components | Data Store |
| :--- | :--- | :--- | :--- | :--- |
| **Flash Sale Service** | Manages flash sale lifecycle. Handles purchase requests & user queueing. | `GET /sales`, `POST /sales/{id}/purchase` | ASP.NET Core Web API, EF Core, StackExchange.Redis, RabbitMQ/Kafka Client | Redis (Hot Path), **SQL Server** (Cold Path) |
| **Inventory Service** | Manages master stock. Provides atomic operations for flash sale stock. | `POST /stock/decrement`, `GET /stock/{id}` | ASP.NET Core Web API, EF Core, StackExchange.Redis (for `DECR`) | Redis (Sale Stock), **SQL Server** (Master Stock) |
| **Order Service** | Creates and manages orders async. Processes queued purchase requests. | `POST /orders`, `GET /orders/{id}` | ASP.NET Core Web API, .NET Worker Service, EF Core | **SQL Server** |
| **Catalog Service** | Provides product information (details, price). | `GET /products/{id}` | ASP.NET Core Web API, EF Core, StackExchange.Redis (for caching) | **SQL Server**, Redis Cache |
| **Notification Service** | Sends real-time and email/SMS notifications. | `POST /notify` (Event-driven) | ASP.NET Core (SignalR Hub), .NET Worker Service, SendGrid/Twilio SDK | N/A (Uses SignalR) |
| **User Service** | Handles user authentication and authorization. | `POST /token`, `GET /user/me` | ASP.NET Core Web API, **ASP.NET Core Identity**, EF Core | **SQL Server** (Identity) |
| **Payment Service** | Integrates with payment gateways (Stripe, Braintree). | `POST /payment` | ASP.NET Core Web API, EF Core, Stripe.NET / Braintree SDK | **SQL Server**, Azure Key Vault (for keys) |

**Additional Services (if any):**

**Waiting Room Service:**
- **Purpose**: A virtual bouncer that manages the 100k+ user flood. It creates a fair "first-come, first-served" line to prevent the site from crashing. Dependencies: User Service, Flash Sale Service, API Gateway

**Order Processing Service (Worker):**
- **Purpose**: A background worker that handles the 'buy' clicks. It grabs requests from a queue to check stock, take payment, and create the order, keeping the website fast. Dependencies: Message Queue, Inventory Service, Payment Service, Order Service, Notification Service

---

## Exercise 2: Data Design

### Task 2.1: Database Schema Design
**Deliverable:** Design database schemas for flash sale entities.

**Flash Sale Service Schema:**
```sql
-- Design your flash sale tables here
-- Consider: flash_sales, flash_sale_items, flash_sale_participants, etc.

CREATE TABLE flash_sales (
    sale_id INT PRIMARY KEY IDENTITY(1,1),
    name NVARCHAR(255) NOT NULL,
    start_time DATETIME2 NOT NULL,
    end_time DATETIME2 NOT NULL,
    status NVARCHAR(50) NOT NULL
);

CREATE TABLE flash_sale_inventory (
    flash_sale_item_id INT PRIMARY KEY IDENTITY(1,1),
    sale_id INT NOT NULL,
    product_id INT NOT NULL,
    sale_price DECIMAL(18, 2) NOT NULL,
    total_sale_quantity INT NOT NULL,
    available_sale_quantity INT NOT NULL,
    
    CONSTRAINT FK_flash_inv_sale FOREIGN KEY (sale_id) REFERENCES flash_sales(sale_id),
    CONSTRAINT FK_flash_inv_product FOREIGN KEY (product_id) REFERENCES products(product_id)
);
CREATE TABLE flash_sale_items (
    flash_sale_item_id INT PRIMARY KEY IDENTITY(1,1),
    sale_id INT NOT NULL,
    product_id INT NOT NULL,
    sale_price DECIMAL(18, 2) NOT NULL,
    total_sale_quantity INT NOT NULL,
    available_sale_quantity INT NOT NULL,
    
    CONSTRAINT FK_flash_item_sale FOREIGN KEY (sale_id) REFERENCES flash_sales(sale_id),
    CONSTRAINT FK_flash_item_product FOREIGN KEY (product_id) REFERENCES products(product_id)
);



CREATE TABLE users (
    user_id INT PRIMARY KEY IDENTITY(1,1),
    email NVARCHAR(255) NOT NULL UNIQUE,
    password_hash NVARCHAR(MAX) NOT NULL,
    created_at DATETIME2 NOT NULL DEFAULT GETDATE()
);

CREATE TABLE products (
    product_id INT PRIMARY KEY IDENTITY(1,1),
    name NVARCHAR(255) NOT NULL,
    description NVARCHAR(MAX),
    regular_price DECIMAL(18, 2) NOT NULL
);



CREATE TABLE orders (
    order_id INT PRIMARY KEY IDENTITY(1,1),
    user_id INT NOT NULL,
    order_date DATETIME2 NOT NULL DEFAULT GETDATE(),
    order_total DECIMAL(18, 2) NOT NULL,
    status NVARCHAR(50) NOT NULL,
    
    CONSTRAINT FK_order_user FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY IDENTITY(1,1),
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    price_paid DECIMAL(18, 2) NOT NULL,
    
    CONSTRAINT FK_order_item_order FOREIGN KEY (order_id) REFERENCES orders(order_id),
    CONSTRAINT FK_order_item_product FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```




**Inventory Service Schema:**
```sql
-- Design inventory management tables
-- Consider: stock levels, reservations, locks

CREATE TABLE inventory (
    product_id INT PRIMARY KEY,
    stock_quantity INT NOT NULL,
    
    CONSTRAINT FK_inventory_product FOREIGN KEY (product_id) REFERENCES products(product_id)
);

```
## ER Schema

![Schema](https://raw.githubusercontent.com/rohitd-2025/System-design-images/main/ER_1.png)

### Task 2.2: Caching Strategy
**Deliverable:** Define your caching strategy and data structures.
 
**Redis Cache Design:**
```
Cache Layer 1 - Application Cache (Redis):
 
Key Pattern: flash_sale:{sale_id}
 
Value: { "sale_info": { "name": "...", "startTime": "...", "endTime": "..." }, "status": "Active" }
 
TTL: 3600 seconds (1 hour) or until sale status changes (e.g., ends)
(Stores the main details of the flash sale (name, times, status) that are read by many users but change infrequently.)
 
Key Pattern: flash_sale_inventory:{sale_id}:{product_id}
 
Value: 980 (An integer representing the available quantity)
 
TTL: 60 seconds
(This is the "hot" key; an atomic counter for the actual available stock, used to handle high-velocity purchase requests and prevent overselling.)
 
Key Pattern: user_session:{user_id}
 
Value: { "jwt": "...", "roles": ["user"], "auth_status": "authenticated" }
 
TTL: 1800 seconds (30 minutes), extended on activity
(Caches the user)
```
 
**CDN Cache Strategy:**
```
Static Content Caching:
- Product images: Cache duration 7 days (604,800 seconds)
- Sale banners: Cache duration 1 day (86,400 seconds)
- JavaScript/CSS: Cache duration 30 days (2,592,000 seconds)
 
Dynamic Content:
- Product availability: Cache duration 0 seconds (No-Cache)
- Sale countdown: Cache strategy (Short TTL (5-10 seconds) or Streaming)
A very short TTL (5-10s) can slightly reduce origin load, but the critical time value should be driven by client-side JavaScript connecting to a highly available time source or a real-time service (e.g., SignalR, WebSockets) to ensure absolute precision.
```
---

## Exercise 3: API Design

### Task 3.1: RESTful API Specification
**Deliverable:** Design the main APIs for flash sale operations.

**Flash Sale Management APIs:**
API: GET /sales

Purpose: **To retrieve a list of all currently active and upcoming flash sales for display on the main sales page.**

Response: **A JSON array of sale objects, e.g., `[ { "saleId": "...", "productId": "...", "productName": "...", "startTime": "...", "endTime": "...", "salePrice": 499, "remainingStock": 150 } ]`**

Caching: **Aggressive. CDN cache (15-30s TTL) and/or a distributed cache (Redis) (5-10s TTL) to handle high read load.**

API: POST /sales/{id}/purchase

Purpose: **To attempt to purchase the flash sale item. This is the main "buy button" API. It should be lightweight, validate the request, and create a reservation/pending order.**

Request Body: **`{ "userId": "...", "quantity": 1 }` (Quantity likely locked to 1)**

Response: **`202 Accepted` with an `orderId` or `reservationId` for status tracking (e.g., `{ "orderId": "...", "status": "Pending" }`). Errors: `429 Too Many Requests`, `409 Conflict` (Out of stock).**

Rate Limiting: **Essential. Per-user (e.g., 1 request per 5 seconds) and global (e.g., 10,000 requests/min).**

API: POST /stock/decrement
Method: **POST**

Purpose: **(Internal Service-to-Service) Called by the Order Service/SAGA to commit the stock reduction *after* a purchase is fully confirmed (e.g., payment cleared). Not for client use.**

Request/Response: **Request: `{ "productId": "...", "quantity": 1 }`. Response: `200 OK` or `409 Conflict` (if stock is already 0).**

API: GET /stock/{id}

Method: **GET**

Purpose: **To get the current stock level for a specific product. This is likely used for internal reporting or general product pages, *not* for the high-traffic flash sale status (which should come from a Redis cache).**

Request/Response: **Response: `{ "productId": "...", "stockLevel": 1200 }`. Caching: Short TTL (e.g., 60 seconds).**

API: POST /orders

Method: **POST**

Purpose: **(Internal Service-to-Service) Creates the final order record, typically called by the Order SAGA orchestrator once inventory is reserved and payment is successful.**

Request/Response: **Request: `{ "userId": "...", "productId": "...", "quantity": 1, "finalPrice": 499, "paymentId": "..." }`. Response: `201 Created` with `{ "orderId": "..." }`.**

API: GET /orders/{id}

Method: **GET**

Purpose: **Allows a client (user) to poll for the status of their purchase attempt after receiving a `202 Accepted` from the purchase API, or to view order history.**

Request/Response: **Response: `{ "orderId": "...", "status": "Pending" | "Confirmed" | "Failed", "productId": "...", "createdTime": "..." }`**

API: GET /products/{id}

Method: **GET**

Purpose: **To get the static, detailed information for a product (e.g., description, full-size images, specifications, original price).**

Request/Response: **Response: `{ "productId": "...", "name": "...", "description": "...", "images": [...] }`. Caching: Very high TTL (e.g., 24 hours) on CDN.**

API: POST /notify (Event-driven)

Method: **POST**

Purpose: **(Internal Webhook) An endpoint for the Notification Service. It is triggered by events from a message bus (e.g., 'OrderConfirmed', 'SaleStarted') to send emails, SMS, or push notifications.**

Request/Response: **Request: `{ "event": "OrderConfirmed", "userId": "...", "details": {...} }`. Response: `202 Accepted`.**

API: POST /token

Method: **POST**

Purpose: **User authentication (Auth Service). Exchanges user credentials (username/password) for a JWT access token.**

Request/Response: **Request: `{ "username": "user@example.com", "password": "..." }`. Response: `{ "accessToken": "...", "expiresIn": 3600 }`.**

API: GET /user/me

Method: **GET**

Purpose: **To get the profile information for the currently authenticated user (using the JWT from `/token`).**

Request/Response: **Request: (Requires `Authorization: Bearer ...` header). Response: `{ "userId": "...", "email": "user@example.com", "name": "..." }`.**

API: POST /payment

Method: **POST**

Purpose: **(Internal Service-to-Service) Called by the Order SAGA to process a payment with a third-party payment gateway.**

Request/Response: **Request: `{ "orderId": "...", "amount": 499.00, "paymentToken": "tok_123..." }`. Response: `{ "paymentId": "...", "status": "Success" | "Failed" }`.**


### Task 3.2: Event-Driven Architecture
**Deliverable:** Define events and event handlers for flash sale operations.

**Domain Events:**

Event: FlashSaleStarted

Payload: { "saleId": "...", "productId": "...", "startTime": "...", "duration": "...", "totalStock": 1000, "salePrice": 499.99 }

Publishers: **Sales Service** (or a Scheduler)

Subscribers: **Notification Service** 
(triggers POST /notify for "Sale is live!" alerts), **Caching Service** (to warm the cache for GET /sales).

Event: PurchaseAttempted

Payload: { "orderId": "...", "saleId": "...", "userId": "...", "productId": "...", "quantity": 1, "timestamp": "..." }

Publishers: **Sales Service** (after POST /sales/{id}/purchase is validated)

Subscribers: **Order Service** (initiates the order SAGA), **Inventory Service** (to attempt reservation).

Event: InventoryReserved

Payload: { "orderId": "...", "userId": "...", "productId": "...", "quantity": 1, "expiresAt": "..." }

Publishers: **Inventory Service**

Subscribers: **Order Service** (to proceed to the payment step), **Notification Service** (triggers POST /notify for "Item reserved").

Event: PaymentProcessed

Payload: { "orderId": "...", "paymentId": "...", "status": "Success" | "Failed", "amount": 499.99 }

Publishers: **Payment Service** (after POST /payment completes)

Subscribers: **Order Service** (to continue or compensate the SAGA).

Event: OrderConfirmed

Payload: { "orderId": "...", "userId": "...", "productId": "...", "finalPrice": "...", "purchaseTime": "..." }

Publishers: **Order Service** (after POST /orders is successful)

Subscribers: **Inventory Service** (triggers POST /stock/decrement to commit stock), **Notification Service** (triggers POST /notify for "Order Confirmed" email).


---

## Exercise 4: Scalability & Performance

### Task 4.1: Load Distribution Strategy
**Deliverable:** Design your load balancing and traffic management approach.

**Load Balancing Configuration:**

API Gateway (YARP) Configuration:
- Routing strategy: Path-based routing
- Load balancing algorithm: Least Connections
- Health check configuration: _______________
- Circuit breaker settings:
  *Break after 5 consecutive failures
  *Reset timeout: 30 seconds
  *Fallback: Return cached response or “Service temporarily unavailable”
 
Database Load Distribution:
- Read replicas strategy: Use SQL server always on    availability groups
- Write scaling approach:
  *Vertical scaling for primary DB during sale
  *Queue-based write operations for orders
- Sharding strategy (if applicable): Horizontal sharding by sale_id for flash sale tables
 
### Task 4.2: Performance Optimization
**Deliverable:** Identify performance bottlenecks and optimization strategies.
 
**Bottleneck Analysis:**
| Component | Potential Bottleneck | Optimization Strategy | Expected Improvement |
|-----------|---------------------|----------------------|---------------------|
| Database Writes | High contention during order creation | Use async processing via RabbitMQ, batch inserts | Reduce DB write latency by 60–70% |
| Inventory Checks | Frequent stock validation requests | Use Redis atomic counters (DECR) and cache hot inventory data | Improve response time from 100ms → <5ms |
| User Authentication | Surge in login/token requests | Implement JWT tokens, enable distributed cache for sessions, and use rate limiting | Handle 10x more concurrent logins |
| Payment Processing | External gateway latency | Use async payment confirmation, fallback queue, and circuit breaker | Reduce timeout failures by 50% |
 
**Caching Optimization:**

Cache Hit Ratio Targets:
- Product catalog: 95%
   (Most product details should come from cache to avoid DB hits.)
- Flash sale status: 90%
   (Sale metadata like start/end time and status should be cached for quick reads.)
- User sessions: 85%
   (Session tokens and user info should mostly be served from cache.)
 
Cache Warming Strategy:
- Preload flash sale details (sale info, inventory counters) into Redis before the sale starts.
- Preload user session tokens for logged-in users to avoid authentication delays.
- Preload product catalog cache by loading top-selling and flash sale products.
_______________
 
Cache Invalidation Strategy:
- Flash sale keys: Invalidate immediately when sale ends or inventory reaches zero.
- User sessions: Invalidate on logout or token expiry.
- Product catalog: Use TTL (e.g., 1 hour) or event-driven invalidation on price/stock changes.
_______________

---

## Exercise 5: Consistency & Reliability

### Task 5.1: Data Consistency Strategy
**Deliverable:** Design your approach to handle data consistency in distributed system.

**ACID vs BASE Trade-offs:**

Strong Consistency Requirements:
- Inventory management: Inventory management requires strong consistency (ACID) to guarantee that no more items are sold than are actually available (preventing overselling/phantom stock).
- Payment processing: Payment processing requires strong consistency (ACID) during the final transaction with the external gateway (e.g., Stripe, PayPal). It is critical to ensure that money is either deducted exactly once or not at all (no double charging or silent failures).
 
Eventual Consistency Acceptable:
- User notifications: The core business transaction (the order and payment) is already complete and durable. A small delay (a few seconds or minutes) in sending the confirmation email, SMS, or push notification does not impact the integrity of the sale itself. The system prioritizes finishing the transaction over the immediate delivery of a status message.
- Analytics/reporting: Business decisions based on sales data (e.g., "how many units were sold in the last hour") are typically made on historical, aggregated data and do not require real-time, second-by-second accuracy. A slight lag in data being reflected in dashboards or reports is acceptable, allowing for asynchronous data aggregation and warehousing, which reduces load on the operational databases.

 
**Distributed Transaction Handling:**

Saga Pattern Implementation: The flash sale system uses a Saga orchestrated by asynchronous messages to handle the multi-step transaction, prioritizing speed and decoupling over two-phase commit (2PC).
 
Step 1: Reserve Inventory
- Success: Publish FlashSaleStockReserved event to the Message Bus. This triggers Step 2 (Payment).
- Failure/Compensation: Publish FlashSalePurchaseFailed event (e.g., stock sold out, user banned). Send real-time notification to the user.
 
Step 2: Process Payment
- Success: On successful payment: Publish PaymentConfirmed event. This triggers Step 3 (Order Confirmation).
- Failure/Compensation: Publish PaymentFailed event. Compensation: Flash Sale Service consumes this event and performs an atomic stock increment ($INCR$) in Redis to release the reserved unit back to the pool.
 
Step 3: Confirm Order
- Success: On successful order creation: Publish OrderCreated event. Final Actions: Order Processing Worker notifies the Notification Service (email/SMS) and updates the master Inventory Service (optional, as master stock is reconciled later).
- Failure/Compensation: Publish OrderCreationFailed event (e.g., database error). Compensation: This is a critical failure. Order Processing Worker must trigger a PaymentRefundRequest to the Payment Service, and the Flash Sale Service must perform the atomic stock increment ($INCR$) compensation (as in Payment Failure).

 
### Task 5.2: Failure Handling

**Deliverable:** Design fault tolerance and recovery mechanisms.
 
**Circuit Breaker Configuration:**

Service: Payment Service
Failure Threshold: 5 failures (within 30 seconds)
Timeout: 5 seconds
Fallback Strategy: Log Failure, Initiate Refund Saga: If the circuit is open, immediately log the error, skip the payment attempt, and initiate the Compensation/Refund step of the Order Processing Saga. The reserved stock is returned.
 
Service: Inventory Service
Failure Threshold: 7 failures (within 60 seconds)
Timeout: 10 seconds
Fallback Strategy: Serve Stale Data (for read-only): If the circuit is open for a GET /stock/{id} request, the system should serve the cached master stock from Redis if available. For write/update operations (non-flash sale related): Fail fast and rely on monitoring/alerting.

 
**Retry Policies:**

Operation: Database Write
Retry Count: 3
Backoff Strategy: Exponential Backoff (e.g., wait 0.5s, then 1s, then 2s) with Jitter.
Circuit Breaker Integration: Applied before the Circuit Breaker. Retries are attempted first. If the retries still exceed the threshold, the circuit breaker opens.

---

## Exercise 6: Security & Compliance

### Task 6.1: Security Design
**Deliverable:** Design security measures for flash sale system.

**Authentication & Authorization:**

User Authentication:

- Method: **OAuth 2.0 / OIDC (JWT). The POST /token endpoint issues a short-lived JWT access token (e.g., 15 minutes) and a long-lived refresh token.**
- Token expiry: **Access Token: 15 minutes. Refresh Token: 7 days. Tokens must be validated at the API Gateway (YARP).**
- Rate limiting per user: **Apply per-user rate limiting on critical endpoints like POST /sales/{id}/purchase to prevent abuse by a single authenticated user.**

API Security:
- Authentication scheme: **All sensitive endpoints (e.g., /purchase, /orders) must require a valid `Authorization: Bearer <JWT>` header.**
- Rate limiting strategy: **1. Global (IP-based) limiting on all endpoints to block basic DoS. 2. Per-user limiting on /purchase. 3. Aggressive limiting on POST /token to prevent credential stuffing.**
- DDoS protection: **Use a CDN/WAF (e.g., Cloudflare, Azure Front Door) at the edge to absorb L3/L4 attacks and filter L7 (application) attacks.**

**Data Protection:**

PII Data Handling:
- Encryption at rest: **All databases (SQL Server, Postgres) and object storage (e.g., logs) must use platform-managed encryption (e.g., TDE, AES-256).**
- Encryption in transit: **TLS 1.2+ for all client-to-server and service-to-service communication. Use mTLS for internal service mesh if high security is required.**
- Data retention policy: **PII in logs anonymized after 30 days. Order history retained for 5 years. User accounts purged after 2 years of inactivity.**

Payment Data:

- PCI DSS compliance approach: **Outsource. The client's browser/app will send payment details directly to a PCI-compliant gateway (e.g., Stripe, Braintree).**

- Tokenization strategy: **The payment gateway returns a non-sensitive "payment token" (e.g., `tok_123...`). Only this token is ever passed to our backend (e.g., in the POST /payment request). We never store raw credit card numbers.**

### Task 6.2: Fraud Prevention

**Deliverable:** Design fraud detection and prevention measures.

**Bot Detection:**

Detection Methods:

- Rate limiting: **IP-based and user-agent-based rate limiting to catch simple "dumb" bots.**
- CAPTCHA integration: **Challenge users with a CAPTCHA (e.g., hCaptcha, reCAPTCHA v3) *before* the POST /purchase request is allowed, especially if they exhibit bot-like behavior (e.g., instant clicks).**
- Behavioral analysis: **Use client-side fingerprinting to detect headless browsers, VMs, or inhuman request speeds (e.g., adding to cart and checking out in < 500ms).**

Prevention Measures:

- Account verification: **Require a verified email or phone number to be eligible for the flash sale.**
- Purchase limits: **Hard limit of one item per user/account/address/payment method for the flash sale product.**
- Suspicious activity handling: **Automatically flag or (in a "shadow ban") allow but fail orders from users who fail bot detection checks. Place them in a low-priority queue.**

---

## Exercise 7: Monitoring & Observability

### Task 7.1: Metrics & Monitoring
**Deliverable:** Define key metrics and monitoring strategy.

**Business Metrics:**
| Metric | Purpose | Target Value | Alert Threshold |
|--------|---------|--------------|----------------|
| Sale conversion rate | **% of users who successfully purchase after visiting the sale page.** | **> 5%** | **< 1% (investigate)** |
| Average response time | **User-perceived latency for the `POST /purchase` endpoint.** | **< 200ms (P95)** | **> 500ms (P95) for 1 min** |
| Inventory accuracy | **Discrepancy between Redis stock and final DB orders (oversell count).** | **0 (Zero oversell)** | **> 0 (CRITICAL)** |
| Payment success rate | **% of payment attempts (POST /payment) that succeed.** | **> 95%** | **< 90% (investigate gateway)** |

**Technical Metrics:**

Application Performance:
- Request latency P99: **< 800ms for /purchase, < 100ms for /status**
- Throughput (RPS): **Target: 1,000 RPS on /purchase, 10,000 RPS on /status. Alert if /purchase > 2,000 RPS (potential bot activity).**
- Error rate: **< 0.1% (excluding 4xx errors like 429). Alert on any spike in 5xx errors > 1%.**

Infrastructure Metrics:
- CPU utilization: **< 80% (individual pod/VM). Alert at > 90% for 5 mins.**
- Memory usage: **< 85% (individual pod/VM). Alert at > 95% for 5 mins.**
- Database connections: **< 75% of max pool size. Alert at > 90% of pool.**


### Task 7.2: Logging & Tracing
**Deliverable:** Design logging and distributed tracing strategy.

**Logging Strategy:**

Log Levels and Content:

ERROR: **Unhandled exceptions, 5xx errors, service-to-service failures (e.g., circuit breaker open), SAGA compensation failures.**
WARN: **Handled exceptions (e.g., 429 Too Many Requests, 409 Conflict - out of stock), long-running queries (> 500ms), retry attempts.**
INFO: **Key business events: SaleStarted, PurchaseAttempted (with OrderId), OrderConfirmed (with OrderId), SAGA steps initiated.**
DEBUG: **(Disabled in Production) Detailed request/response bodies, verbose event payloads.**

Structured Logging Format:
{
  "timestamp": "...",
  "level": "INFO",
  "service": "OrderService",
  "traceId": "abc-123",
  "spanId": "def-456",
  "userId": "user-guid-789",
  "operation": "ConfirmOrder",
  "duration": 55,
  "result": "Success",
  "orderId": "order-guid-111"
}

**Distributed Tracing:**
Trace Correlation:
- Trace ID generation: **At the API Gateway (YARP) or the first service hit (Sales Service). Use OpenTelemetry standard.**
- Cross-service propagation: **Propagate `traceId` and `spanId` via HTTP headers (W3C Trace Context) and message bus headers (e.g., RabbitMQ headers).**
- Sampling strategy: **100% sampling for POST /purchase (critical path). 10% sampling for high-volume GET /status. 5% sampling for all other endpoints.**
---

## Exercise 8: Deployment & DevOps

### Overview:
- Packaging: Every microservice is packaged as a lightweight Docker container using multi-stage builds.

- Orchestration: We're using Kubernetes (AKS) to run all our containers.

- Scaling: We use two auto-scaling methods:

- HPA (Horizontal Pod Autoscaler): Scales our web-facing services based on CPU and memory.

- KEDA(Kubernetes-based Event-Driven Autoscaling): Scales our background workers (like the Order Processing Service) based on the length of the message queue. If 5,000 orders flood the queue, KEDA automatically scales up the workers to process them, then scales them back down to zero to save money.

### Task 8.1: Infrastructure as Code
**Deliverable:** Design your deployment architecture.


**Orchestration:**

- Platform choice: **Kubernetes (e.g., Azure Kubernetes Service - AKS) for its robust auto-scaling and service discovery.**
- Scaling strategy: **Horizontal Pod Autoscaler (HPA) based on CPU and Memory usage (> 75%). Use KEDA (Kubernetes-based Event-Driven Autoscaling) for queue-based services (e.g., Payment, Notification) based on RabbitMQ queue length.**
- Rolling update strategy: **`MaxSurge: 25%`, `MaxUnavailable: 10%`. This ensures zero downtime by gradually rolling out new pods while keeping the service available.**

## **Environment Strategy:**

**Development Environment:**
- Local development setup: **Docker Compose to run all essential services (Redis, RabbitMQ, SQL Server, API Gateway) and the specific microservice being worked on.**
- Database seeding: **Use `dotnet ef database update` with a specific `seed` migration on startup for local/dev environments.**
- Service mocking: **Developers can run services they don't need in a "mock" mode or rely on default responses from the API Gateway.**

**Production Environment:**
- Auto-scaling triggers: **HPA on CPU/Memory > 75% for 3 minutes. KEDA scaler on RabbitMQ `Order.Processing` queue > 100 messages.**
- Resource allocation: **Use Kubernetes resource `requests` (guaranteed) and `limits` (max) for all pods. Use dedicated node pools for critical (DB, Cache) vs. stateless (API) workloads.**
- Backup strategy: **Managed Database (e.g., Azure SQL) with Point-in-Time Recovery (PITR) enabled. Daily snapshots of persistent volumes.**

### Task 8.2: CI/CD Pipeline

**Deliverable:** Design continuous integration and deployment pipeline.



**Production Deployment (on Git Tag / Manual Approval)**
   - Deployment strategy: **Blue/Green or Canary. 1. Deploy new version (Green) to K8s. 2. Run smoke tests. 3. Shift 10% of live traffic (Canary). 4. Monitor error rates for 15 mins. 5. If stable, shift 100% traffic. 6. Tear down old version (Blue).**
   - Rollback plan: **Automated. If error rates spike > 2% during canary, immediately shift 100% of traffic back to the stable (Blue) deployment.**
---

## Exercise 9: Capacity Planning

### Task 9.1: Load Estimation
**Deliverable:** Calculate system capacity requirements.

**Traffic Estimation:**
Peak Load Calculations:

- Expected concurrent users: **100,000**
- Requests per second: **API (Read): 20,000 RPS (GET /status, GET /active). API (Write): 5,000 RPS (POST /purchase at T=0).**

- Database transactions per second: 

**Targeting < 100 TPS (most writes are async). Peak writes will hit Redis, not the DB.**

- Cache operations per second:

 **Reads: 20,000+ OPS (stock checks). Writes: 5,000+ OPS (atomic `DECRBY` operations).**

Growth Planning:

- 6-month projection: **Support for 150,000 concurrent users (50% growth).**
- 1-year projection: **Support for 250,000 concurrent users (150% growth).**
- Scaling triggers: **Proactively scale up infrastructure (e.g., K8s nodes, DB/Cache replicas) 1 hour before a major planned sale.**

**Resource Requirements:**

| Component | CPU | Memory | Storage | Network |
 |---|---|---|---|---|
  | API Gateway | 4 Cores | 8 GB RAM | N/A | High (10 Gbps) | 
  | Flash Sale Service | 8 Cores | 16 GB RAM | N/A | High (10 Gbps) | 
  | Inventory Service | 8 Cores | 16 GB RAM | N/A | Medium |
| Database | 16 Cores | 128 GB RAM | High (SSD IOPS) | Medium |
 | Cache (Redis) | 8 Cores | 64 GB RAM | N/A (In-memory) | Very High (20 Gbps+) |

### Task 9.2: Cost Optimization
**Deliverable:** Analyze cost implications and optimization strategies.

**Cost Analysis:**
Infrastructure Costs:
- Compute resources: **$5,000 /month (Baseline)**
- Storage costs: **$1,000 /month (DB + Logs)**
- Network costs: **$2,000 /month (Egress + CDN)**
- Third-party services: **$1,500 /month (Monitoring, Auth, Payment Gateway fees)**
- **Sale Day Burst Cost:** **+$2,000 (for 24h burst capacity)**

Cost Optimization Strategies:
- Reserved instances: **Use 1-year or 3-year reserved instances (RIs) for baseline compute/DB (e.g., 70% of capacity).**
- Auto-scaling policies: **Aggressively scale *down* pods and cluster nodes after the 2-hour sale window to avoid paying for idle burst capacity.**
- Resource right-sizing: **Use ARM-based VMs (e.g., Azure Dpsv5/Epsv5 series) for better price-performance on .NET workloads. Use `dotnet-monitor` to find and fix memory leaks.**

---

## Exercise 10: Testing Strategy

### Task 10.1: Test Plan Design
**Deliverable:** Create comprehensive testing strategy.

**Testing Pyramid:**

Unit Tests:
- Coverage target: **> 85% for business logic (e.g., in Domain models, Services).**
- Test categories: **SAGA logic, validation rules (e.g., sale start/end times), price calculation, state transitions.**
- Mock strategies: **Mock all external dependencies: `IInventoryRepository`, `IPaymentGateway`, `IMessageBusPublisher`, and other microservices.**

Integration Tests:
- Service-to-service: **Verify API contracts using Pact. Test critical SAGA flows (e.g., Order Service -> Inventory Service -> Payment Service).**
- Database integration: **Test repository logic against a testcontainer (e.g., `testcontainers-dotnet`) running a real SQL Server/Postgres image.**
- External API integration: **Test Redis integration by checking atomic `DECRBY` logic. Test RabbitMQ by publishing/consuming real messages.**

End-to-End Tests:
- Critical user journeys: **1. User sees sale. 2. User clicks purchase. 3. User polls for status. 4. User sees "Confirmed" page. 5. Admin sees order in backend.**
- Cross-browser testing: **(Not applicable for this backend-focused workshop, but would be critical for the UI).**
- Mobile responsiveness: **(N/A for backend).**


### Task 10.2: Performance Testing
**Deliverable:** Design load and stress testing approach.

**Load Testing Scenarios:**
Scenario 1: Normal Flash Sale Load (Warm-up)
- Virtual users: **10,000 (simulating pre-sale browsing)**
- Ramp-up time: **10 minutes**
- Test duration: **30 minutes**
- Success criteria: **P95 response time < 200ms for GET /sales. Error rate < 0.01%.**

Scenario 2: Peak Load (Sale Start "Thundering Herd")
- Virtual users: **100,000**
- Ramp-up time: **1 minute (all users hit at once)**
- Test duration: **5 minutes (peak burst)**
- Success criteria: **P95 on POST /purchase < 500ms. P99 < 1000ms. System processes 5,000 RPS on /purchase. Zero (0) oversells. Error rate (5xx) < 0.1%.**

Scenario 3: Stress Testing (Find the breaking point)
- Load increase strategy: **Start at 100,000 users and increase by 20,000 users every 5 minutes until failure.**
- Breaking point identification: **Identify which component fails first (e.g., Redis connections maxed, DB CPU at 100%, API Gateway queue overflows).**
- Recovery testing: **After the test, reduce load to normal (10,000 VUs) and measure time-to-recovery (TTR). Services must recover automatically (e.g., circuit breakers close) within 5 minutes.**

---

## Submission Guidelines

### Required Deliverables:

1. **Architecture Document** (Exercises 1-2)
   - High-level architecture diagram
   - Service specifications
   - Database design
   - Caching strategy

2. **API Specification** (Exercise 3)
   - RESTful API documentation
   - Event schema definitions
   - Integration patterns

3. **Technical Design** (Exercises 4-6)
   - Scalability solutions
   - Consistency guarantees
   - Security measures
   - Performance optimizations

4. **Operations Plan** (Exercises 7-8)
   - Monitoring strategy
   - Deployment architecture
   - CI/CD pipeline design

5. **Capacity & Testing Plan** (Exercises 9-10)
   - Resource requirements
   - Cost analysis
   - Testing strategy

### Evaluation Criteria:


- **Completeness** (25%): All sections addressed with thoughtful responses
- **Technical Accuracy** (25%): Solutions demonstrate understanding of distributed systems
- **Scalability** (20%): Design handles specified load requirements
- **Trade-off Analysis** (15%): Clear reasoning for design decisions
- **Real-world Applicability** (15%): Practical solutions that can be implemented

Will not publish result but I will declare which team is winner

### Submission Format:
- Complete this worksheet with your responses
- Include any additional diagrams or code snippets
- Provide references to eShop architecture components where applicable
- Submit as a markdown file with embedded diagrams

---

**Estimated Time:** 8-12 hours
**Difficulty:** Advanced
**Prerequisites:** Understanding of microservices, docker container, distributed systems, and .NET ecosystem

Good luck

- By Divyang Panchasara