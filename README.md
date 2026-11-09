TradeX — Real-Time Order Matching Engine

TradeX is a real-time trading platform built around a price-time priority order matching engine. It provides REST APIs for order submission and management, WebSocket-based real-time updates, and an interactive browser trading terminal.

The project is designed as a progressively evolving system, starting with a simple matching engine and expanding toward event-driven architecture, risk controls, security, persistence, cloud deployment, observability, and scalability.

⸻

System Overview

                    ┌─────────────────────┐
                    │   Trading Terminal  │
                    │   Browser Interface  │
                    └──────────┬──────────┘
                               │
                         REST / WebSocket
                               │
                               ▼
                    ┌─────────────────────┐
                    │       FastAPI       │
                    │     API Layer       │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Order Validation   │
                    │   & Order Routing   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Matching Engine   │
                    │ Price-Time Priority  │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │      Order Book      │
                    │   Bid / Ask Levels   │
                    └──────────┬──────────┘
                               │
                     ┌─────────┴─────────┐
                     ▼                   ▼
              Trade Execution       Open Orders
                     │
                     ▼
              WebSocket Events
                     │
                     ▼
              Trading Terminal

⸻

Current Features

Order Management

TradeX currently supports:

* Limit orders
* Market orders
* Buy and sell orders
* Order cancellation
* Partial fills
* Remaining open quantities
* Trade execution
* Order status tracking

⸻

Matching Engine

The matching engine uses price-time priority.

For incoming orders:

1. Check whether a compatible order exists on the opposite side.
2. Match the best available price.
3. Apply time priority when multiple orders have the same price.
4. Execute the trade.
5. Reduce the remaining quantities.
6. Continue matching if quantity remains.
7. Add any remaining quantity to the order book.

Example

Incoming BUY 100 @ 105
Existing SELL orders:
105 → 40
106 → 30
107 → 50
Matching:
BUY 100 @ 105
       │
       ├── Match 40 @ 105
       ├── Match 30 @ 106
       └── Match 30 @ 107
Remaining BUY quantity = 0

This demonstrates price priority followed by execution of available liquidity.

⸻

Order Book

The order book maintains buy and sell orders by price level.

          ORDER BOOK
      SELL / ASK SIDE
Price       Quantity
107.00        50
106.00        30
105.00        40
---------------------
104.00        25
103.00        60
102.00        35
      BUY / BID SIDE

The engine maintains FIFO queues for orders at the same price level.

This allows the system to implement price-time priority.

⸻

Partial Fills

TradeX supports partial order execution.

For example:

BUY 100
Available SELL:
40
30
50
Execution:
40 → filled
30 → filled
30 → partially filled from next order
Remaining BUY = 0

If insufficient liquidity exists, the remaining quantity can remain open in the order book.

⸻

Market Orders

Market orders execute against available liquidity without specifying a fixed execution price.

The engine consumes available orders from the best available price levels until:

* The requested quantity is completely filled, or
* No compatible liquidity remains.

⸻

Order Cancellation

Open orders can be cancelled before execution.

The matching engine removes the corresponding order from the active order book and updates the order state.

⸻

Real-Time Architecture

TradeX uses WebSockets to provide live updates to connected clients.

Order Submitted
       │
       ▼
Matching Engine
       │
       ├── Order Book Updated
       ├── Trade Executed
       └── Order Status Changed
                    │
                    ▼
              WebSocket Event
                    │
                    ▼
             Trading Terminal

This allows the browser interface to update without repeatedly polling the server.

⸻

REST API

FastAPI provides the main HTTP interface.

Typical operations include:

POST   /orders
GET    /orders
GET    /orders/{id}
DELETE /orders/{id}
GET    /orderbook
GET    /trades

The API layer handles request validation and communicates with the matching engine.

⸻

WebSocket Communication

The WebSocket layer provides real-time information such as:

* Order book updates
* Trade executions
* Order status changes
* Market activity
* System events

The architecture separates request/response operations from real-time event delivery.

⸻

Interactive Trading Terminal

TradeX includes a browser-based trading terminal.

The interface provides:

* Buy / Sell order submission
* Limit / Market order selection
* Quantity and price input
* Symbol selection
* Live connection status
* Best bid
* Best ask
* Spread
* Last traded price
* Live order book
* Recent trades
* Order status
* Order cancellation
* Live activity stream
* Refresh and clear controls

The terminal communicates with the backend through REST APIs and WebSockets.

⸻

Technology Stack

Component	Technology
Programming Language	Python
API Framework	FastAPI
Real-Time Communication	WebSockets
Order Book	SortedContainers
Frontend	HTML, CSS, JavaScript
Testing	Pytest
API Server	Uvicorn

⸻

Project Structure

TradeX/
│
├── app/
│   ├── api/
│   ├── engine/
│   ├── models/
│   ├── orderbook/
│   ├── services/
│   └── main.py
│
├── tests/
│
├── frontend/
│   └── trading_terminal/
│
├── requirements.txt
├── README.md
└── ...

The exact directory structure may evolve as the system expands.

⸻

Performance

The current implementation has been benchmarked using a workload of approximately 10,000 orders.

Observed benchmark results:

Throughput:     ~23,000 orders/second
p95 latency:    ~72.5 μs
Test workload:  10,000 orders

These figures represent the current implementation and test environment rather than production-exchange performance guarantees.

⸻

Testing

The project includes automated tests covering core matching-engine behaviour.

Current testing includes:

* Order creation
* Limit-order matching
* Market-order matching
* Partial fills
* Order cancellation
* Order-book behaviour
* Trade execution
* API behaviour

Current test status:

11 automated tests passed

Testing will expand as additional system components are introduced.

⸻

Phase 2 — System Architecture Expansion

The next development phase expands TradeX from a core matching engine into a broader distributed trading-system architecture.

The objective is to introduce major engineering concepts through simple, understandable implementations.

⸻

Phase 2 Architecture

                         Client
                           │
                           ▼
                  Trading Terminal
                           │
                           ▼
                    API / Gateway
                           │
                           ▼
                    Authentication
                           │
                           ▼
                 Pre-Trade Risk Check
                           │
                           ▼
                    Order Routing
                           │
                           ▼
                  Matching Engine
                           │
                           ▼
                     Order Book
                           │
                           ▼
                  Trade Execution
                           │
                           ▼
                    Event Layer
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          Monitoring      Risk        AI Analysis
          Consumer      Consumer       Consumer
              │            │            │
              └────────────┼────────────┘
                           ▼
                       PostgreSQL
                           │
                    Redis / Cache

⸻

Planned Phase 2 Features

1. Event-Driven Architecture

Introduce structured events for system actions such as:

OrderSubmitted
OrderAccepted
OrderRejected
OrderMatched
TradeExecuted
OrderCancelled
RiskCheckFailed

Events will contain information such as:

* Event ID
* Event type
* Timestamp
* Order ID
* Symbol
* Quantity
* Price
* Event source

The goal is to separate the core trading engine from downstream consumers such as monitoring and risk services.

⸻

2. Pre-Trade Risk Gateway

Introduce a simple risk layer before an order reaches the matching engine.

Example checks:

Order
  │
  ▼
Risk Gateway
  │
  ├── Quantity Limit
  ├── Price Validation
  ├── Exposure Limit
  ├── Position Limit
  └── Duplicate / Invalid Order Check
          │
          ▼
      Accept / Reject

Example:

Order quantity = 10,000
Maximum allowed = 1,000
Result:
REJECTED
Reason: Quantity limit exceeded

The matching engine remains responsible for deterministic order matching, while the risk gateway controls whether an order is allowed to reach it.

⸻

3. Authentication

Introduce user authentication using:

* Login
* Password hashing
* JWT access tokens
* Protected API endpoints

Example flow:

Login
  │
  ▼
Verify Credentials
  │
  ▼
Generate JWT
  │
  ▼
Client Sends Token
  │
  ▼
Protected API

⸻

4. Authorization

Introduce role-based access control.

Example roles:

Trader
Admin

Example permissions:

Trader
 ├── Submit Order
 ├── Cancel Order
 └── View Orders
Admin
 ├── View System Events
 ├── View Risk Events
 ├── View Monitoring Data
 └── Administrative Operations

⸻

5. Security

Introduce basic application-security controls:

* Password hashing
* JWT authentication
* Role-based authorization
* Input validation
* Rate limiting
* CORS configuration
* Environment variables
* Secret management
* Secure API configuration

Secrets and credentials will not be stored directly in the repository.

⸻

6. PostgreSQL Persistence

Introduce PostgreSQL for persistent system data.

Potential data entities:

Users
Orders
Trades
Positions
Risk Events
System Events

The goal is to separate temporary in-memory matching state from persistent application data.

⸻

7. Redis

Introduce Redis for fast-access temporary data.

Potential uses:

* Caching
* Session-related data
* Temporary state
* Rate limiting
* Real-time application state

Redis will complement PostgreSQL rather than replace it.

⸻

8. Event Streaming

Introduce Redis Streams as a simple event-streaming mechanism.

Example:

Order Submitted
       │
       ▼
Redis Stream
       │
       ├──────────────► Risk Consumer
       │
       ├──────────────► Monitoring Consumer
       │
       └──────────────► Analytics Consumer

This allows different components to consume trading events independently.

⸻

9. Docker

Containerize the application using Docker.

Potential services:

Docker Compose
│
├── TradeX API
├── PostgreSQL
└── Redis

This creates a reproducible local development environment.

⸻

10. AWS Deployment

Deploy the application to AWS using a simple cloud architecture.

Initial deployment target:

GitHub
   │
   ▼
Docker Build
   │
   ▼
AWS EC2
   │
   ├── TradeX API
   ├── WebSocket Server
   └── Supporting Services

The initial objective is to understand:

* Linux server deployment
* Networking
* Ports
* Environment variables
* Security groups
* Docker deployment
* Application monitoring

More advanced cloud infrastructure can be explored later.

⸻

11. Monitoring and Observability

Introduce basic system monitoring.

Metrics may include:

Request latency
Order latency
Orders submitted
Orders rejected
Trades executed
WebSocket connections
Error count
Event processing time
Throughput

The system will expose enough operational information to understand how the platform behaves during different workloads.

⸻

12. Scalability

The architecture will explore how different components can scale independently.

For example:

                    Load Balancer
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
          API #1       API #2       API #3
             │            │            │
             └────────────┼────────────┘
                          │
                    Redis / DB
                          │
                    Trading Layer

Stateless API services can be replicated horizontally.

The matching engine is stateful, so scaling requires careful handling of order-book ownership.

⸻

Symbol-Based Partitioning

Trading symbols can be distributed across independent matching-engine instances.

Example:

BTC-USD  ──► Matching Engine A
ETH-USD  ──► Matching Engine B
AAPL     ──► Matching Engine C

This allows different symbols to be processed independently.

Multiple independent matching engines should not process the same symbol simultaneously unless state coordination and ordering guarantees are explicitly designed.

⸻

13. AI-Assisted Analysis

AI will be introduced around the deterministic trading system rather than inside the core matching algorithm.

Potential architecture:

Trading Events
      │
      ▼
Risk / Monitoring Data
      │
      ▼
AI Analysis Layer
      │
      ├── Risk Explanation
      ├── Anomaly Detection
      ├── Event Summarisation
      └── Operational Insights

The matching engine remains deterministic and responsible for the actual trading logic.

AI provides analysis, explanations, and operational assistance rather than deciding whether trades should execute.

⸻

14. CI/CD

Introduce GitHub Actions for automated development workflows.

Potential pipeline:

Git Push
   │
   ▼
GitHub Actions
   │
   ├── Run Tests
   ├── Validate Code
   ├── Build Docker Image
   └── Deploy

Deployment automation will be introduced after the core containerized application is stable.

⸻

Development Roadmap

Phase 1
│
├── Matching Engine
├── Order Book
├── Price-Time Priority
├── Limit Orders
├── Market Orders
├── Partial Fills
├── Cancellation
├── REST API
├── WebSockets
├── Trading Terminal
└── Performance Testing
        │
        ▼
Phase 2
│
├── Event-Driven Architecture
├── Pre-Trade Risk
├── Authentication
├── Authorization
├── PostgreSQL
├── Redis
├── Event Streaming
├── Docker
├── AWS
├── Monitoring
├── Scalability
├── AI Analysis
└── CI/CD

⸻

Design Principles

Deterministic Matching

The matching engine follows explicit price-time priority rules.

Separation of Responsibilities

Different components have separate responsibilities:

API
 │
 ├── Request Handling
 │
 ├── Authentication
 │
 └── Validation
Risk Gateway
 │
 └── Pre-Trade Controls
Matching Engine
 │
 └── Order Matching
Order Book
 │
 └── Market State
Event Layer
 │
 └── Event Distribution
Monitoring / AI
 │
 └── Analysis

Incremental Architecture

The system is developed progressively.

Each Phase 2 component is introduced independently so its purpose and interaction with the existing system can be understood before adding further complexity.

⸻

Current Project Status

Component	Status
Matching Engine	Implemented
Order Book	Implemented
Price-Time Priority	Implemented
Limit Orders	Implemented
Market Orders	Implemented
Partial Fills	Implemented
Order Cancellation	Implemented
REST API	Implemented
WebSockets	Implemented
Trading Terminal	Implemented
Automated Tests	Implemented
Performance Benchmarking	Implemented
Event-Driven Architecture	Planned
Pre-Trade Risk	Planned
Authentication	Planned
Authorization	Planned
PostgreSQL	Planned
Redis	Planned
Event Streaming	Planned
Docker	Planned
AWS Deployment	Planned
Monitoring	Planned
Scalability	Planned
AI Analysis	Planned
CI/CD	Planned

⸻

Running the Current Version

Install dependencies:

pip install -r requirements.txt

Start the FastAPI server:

uvicorn app.main:app --reload

Open the trading terminal in a browser and connect it to the running API.

The exact startup commands may evolve as Docker and cloud deployment are introduced.

⸻

Project Scope

TradeX focuses on understanding the architecture and engineering principles behind a real-time trading system.

The current implementation intentionally uses simpler components than a production financial exchange. Future phases progressively introduce persistence, security, event-driven processing, cloud infrastructure, observability, and scalability.

TradeX is not intended for real financial trading or production financial infrastructure.
