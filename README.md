# TradeX — Real\-Time Order Matching Engine

TradeX is a real\-time multi\-symbol trading exchange simulation built around a **price\-time priority order matching engine**\. It provides REST APIs for submitting and managing orders, WebSocket\-based real\-time market events, and a browser\-based trading terminal\.

The current implementation focuses on the core exchange mechanics: order validation, order books, limit and market order matching, partial fills, cancellations, trade execution, self\-trade prevention, REST APIs, and real\-time WebSocket updates\.

> **Scope:** TradeX is currently an in-memory exchange simulation. It does not implement persistent storage, authentication, pre-trade risk controls, or a production cloud/distributed deployment.

---

## Features

### Order Management

- Buy and sell orders
- Limit orders
- Market orders
- Positive quantity validation
- Limit\-price validation
- Trader ID validation
- Unique order IDs
- Order status tracking
- Remaining quantity tracking
- Order cancellation
- Prevention of cancelling filled orders
- Prevention of cancelling an already cancelled order

Supported order states:

```text
NEW
  ↓
PARTIALLY_FILLED
  ↓
FILLED
```

An active order can also transition to:

```text
NEW / PARTIALLY_FILLED
        ↓
    CANCELLED
```

---

## Matching Engine

The matching engine implements **price\-time priority**\.

For a limit order, the engine:

1. Finds the best order on the opposite side\.
2. Checks whether the prices cross\.
3. Applies FIFO priority within the same price level\.
4. Prevents orders from the same trader from matching\.
5. Calculates the executable quantity\.
6. Creates a trade at the resting order’s price\.
7. Updates both orders’ filled quantities and statuses\.
8. Removes fully filled resting orders from the order book\.
9. Continues matching while quantity remains and compatible liquidity exists\.
10. Adds any remaining active limit quantity to the order book\.

### Example

```text
Existing SELL orders

Price   Quantity
105       40
106       30
107       50

Incoming BUY LIMIT
Price = 107
Quantity = 100

Execution

40 @ 105
30 @ 106
30 @ 107

BUY remaining quantity = 0
```

The trade price is the **maker/resting order price** in the current implementation\.

---

## Partial Fills

TradeX supports partial execution\.

Example:

```text
SELL 100 @ 105

Incoming BUY 40 @ 105

SELL filled     = 40
SELL remaining  = 60
BUY filled      = 40
BUY remaining   = 0
```

If an incoming order consumes only part of a resting order, the remaining resting quantity stays active in the order book\.

---

## Market Orders

Market orders do not specify a price\.

A market order consumes available liquidity from the best available price levels until:

- the requested quantity is completely filled, or
- no compatible liquidity remains\.

Example:

```text
ASKS

$100 → 3
$101 → 4
$102 → 5

MARKET BUY → 9

3 @ $100
4 @ $101
2 @ $102
```

The remaining `$102` liquidity stays in the order book\.

---

## Order Book

Each supported symbol has its own order book\.

Currently supported symbols:

- `BTC-USD`
- `ETH-USD`

The order book maintains separate bid and ask sides using price levels containing FIFO queues\.

```text
BIDS

Price → FIFO Orders
105   → [Order A, Order B]
103   → [Order C]
100   → [Order D]

ASKS

Price → FIFO Orders
101   → [Order E]
104   → [Order F, Order G]
110   → [Order H]
```

For bids, the highest price has priority\. For asks, the lowest price has priority\. Orders at the same price are processed FIFO\.

The order book also exposes:

- best bid
- best ask
- spread
- aggregated market depth

---

## Self\-Trade Prevention

The matching engine checks the `trader_id` of the incoming and resting orders\.

If both belong to the same trader, the match is rejected instead of creating a trade\.

```text
Trader A
  ↓
BUY BTC-USD

Trader A
  ↓
SELL BTC-USD

→ Self-trade prevented
```

---

## Real\-Time WebSocket Events

TradeX uses WebSockets for live updates to connected trading terminals\.

The matching engine publishes events including:

- `ORDER_SUBMITTED`
- `TRADE_EXECUTED`
- `ORDER_FILLED`
- `ORDER_PARTIALLY_FILLED`
- `ORDER_CANCELLED`
- `ORDER_BOOK_UPDATED`
- `MARKET_DATA`

The WebSocket endpoint is symbol\-specific, so clients subscribe to the market they want to observe\.

```text
Order Request
     ↓
FastAPI
     ↓
Matching Engine
     ↓
Order / Trade / Market Event
     ↓
WebSocket Event Queue
     ↓
Trading Terminal
```

This avoids relying on repeated client\-side polling for live market updates\.

---

## REST API

The FastAPI application exposes the following implemented endpoints:

|Method   |Endpoint               |Purpose                             |
|---------|-----------------------|------------------------------------|
|GET      |`/health`              |Health check and supported symbols  |
|POST     |`/orders`              |Submit a new order                  |
|GET      |`/orders`              |List orders with optional filters   |
|GET      |`/orders/{order_id}`   |Retrieve one order                  |
|DELETE   |`/orders/{order_id}`   |Cancel an order                     |
|GET      |`/orderbook/{symbol}`  |Retrieve order-book depth           |
|GET      |`/market-data/{symbol}`|Retrieve current market data        |
|GET      |`/trades/{symbol}`     |Retrieve executed trades            |
|WebSocket|`/ws/{symbol}`         |Subscribe to real-time symbol events|

### Example order request

```json
{
  "symbol": "BTC-USD",
  "side": "BUY",
  "order_type": "LIMIT",
  "quantity": 2,
  "price": 100,
  "trader_id": "trader-1"
}
```

For a market order, `price` must be omitted or set to `null`\.

---

## Interactive Trading Terminal

The project includes a browser\-based trading terminal served by the FastAPI application at:

```text
/terminal
```

The terminal provides:

- Buy / Sell order submission
- Limit / Market order selection
- Quantity and price input
- Symbol selection
- Connection status
- Best bid
- Best ask
- Spread
- Last traded price
- Live order book
- Recent trades
- Order status
- Order cancellation
- Live activity stream
- Refresh and clear controls

The frontend is implemented with HTML, CSS and vanilla JavaScript\.

---

## Architecture

```text
                    Browser Trading Terminal
                              │
                       REST / WebSocket
                              │
                              ▼
                         FastAPI API
                         api/main.py
                              │
                ┌─────────────┴─────────────┐
                │                           │
                ▼                           ▼
          Order Validation             WebSocket
                │                       Manager
                ▼                           │
          Matching Engine                  │
                │                           │
                ▼                           │
            Order Book                     │
                │                           │
                ▼                           │
          Trade Execution                  │
                │                           │
                ▼                           │
             Events ────────────────────────┘
                              │
                              ▼
                         Browser Updates
```

### Main components

`api/main.py`

- FastAPI application
- REST endpoints
- WebSocket endpoint
- Request validation
- In\-memory order registry
- Per\-symbol order\-book and matching\-engine setup
- WebSocket connection/event management
- Static frontend serving

`trading/order.py`

- Order data model
- Order validation
- Order status transitions
- Fill and cancellation logic

`trading/order_book.py`

- Bid and ask price levels
- FIFO order queues
- Best bid / ask
- Spread
- Market\-depth calculation
- Order removal

`trading/matching.py`

- Matching engine
- Price crossing logic
- Limit\-order matching
- Market\-order matching
- Trade creation
- Self\-trade prevention
- Event generation

`trading/trades.py`

- Currently contains only comments describing trade storage; the active `Trade` model is implemented in `trading/matching.py`\.

`web/`

- Browser trading terminal
- HTML, CSS and JavaScript frontend

---

## Data Structures

The current implementation uses Python standard\-library data structures\.

### Order registry

```python
orders: dict[str, Order]
```

Maps an order ID to its order object and provides fast average\-case lookup by ID\.

### Price levels

```python
defaultdict(deque)
```

Maps each price to a FIFO queue of orders\.

### FIFO queues

```python
deque
```

Orders at the same price are processed in arrival order\.

### Important implementation detail

The current order book uses `max()` for the best bid and `min()` for the best ask\. Therefore, selecting the best price scans the available price keys rather than using a dedicated sorted tree or heap\.

This is suitable for the current educational/simulation scope but is an area that could be optimised for a larger price\-level set\.

---

## Technology Stack

|Component              |Technology                                  |
|-----------------------|--------------------------------------------|
|Programming Language   |Python 3.12                                 |
|API Framework          |FastAPI                                     |
|API Server             |Uvicorn                                     |
|Validation             |Pydantic / FastAPI request models           |
|Real-Time Communication|WebSockets                                  |
|Backend Data Structures|Python `dict`, `defaultdict`, `deque`, lists|
|Frontend               |HTML, CSS, JavaScript                       |
|Testing                |Pytest                                      |
|API Testing            |HTTPX / FastAPI TestClient                  |
|Containerisation       |Docker                                      |

No database, external AI/ML model, Redis, Kafka, or cloud service is currently required by the application\.

---

## Project Structure

```text
tradex-main/
│
├── api/
│   ├── main.py
│   ├── order_routes.py
│   └── schemas.py
│
├── benchmarks/
│   ├── benchmark.py
│   └── benchmark_exchange.py
│
├── docs/
│   ├── architecture.md
│   └── order_execution.md
│
├── tests/
│   ├── test_api.py
│   ├── test_events.py
│   ├── test_integration.py
│   ├── test_matching.py
│   ├── test_order.py
│   ├── test_order_book.py
│   ├── test_order_history.py
│   ├── test_self_trade.py
│   └── test_websocket.py
│
├── trading/
│   ├── __init__.py
│   ├── matching.py
│   ├── order.py
│   ├── order_book.py
│   └── trades.py
│
├── web/
│   ├── app.js
│   ├── index.html
│   └── style.css
│
├── Dockerfile
├── pytest.ini
├── requirements.txt
└── README.md
```

Note: `api/order_routes.py` and `api/schemas.py` are currently placeholder modules; the active FastAPI routes and `OrderRequest` model are implemented in `api/main.py`\.

---

## Testing

The repository contains automated tests covering:

- order creation and validation
- limit\-order matching
- market\-order matching
- partial fills
- order cancellation
- order\-book behaviour
- trade execution
- self\-trade prevention
- order history and filtering
- REST API behaviour
- integration workflows
- event publishing
- WebSocket behaviour
- multi\-symbol behaviour

The current repository test suite passes:

```text
68 passed
```

Run the tests with:

```bash
pytest -q
```

---

## Benchmark

The repository includes `benchmarks/benchmark_exchange.py`, which runs a matching\-engine workload using **10,000 orders**\.

Run it with:

```bash
python benchmarks/benchmark_exchange.py
```

The benchmark reports:

- orders processed
- trades generated
- quantity matched
- elapsed time
- calculated throughput in orders/second

The benchmark result depends on the machine and runtime environment\. The repository does not contain a fixed production\-performance guarantee\.

---

## Running Locally

### 1\. Create a virtual environment

Windows PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

macOS / Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 2\. Install dependencies

```bash
pip install -r requirements.txt
```

### 3\. Start the API

```bash
uvicorn api.main:app --reload
```

The application runs on:

```text
http://127.0.0.1:8000
```

The trading terminal is available at:

```text
http://127.0.0.1:8000/terminal
```

API documentation is available through FastAPI’s generated documentation at:

```text
http://127.0.0.1:8000/docs
```

---

## Running with Docker

Build the image:

```bash
docker build -t tradex .
```

Run the container:

```bash
docker run -p 8000:8000 tradex
```

Then open:

```text
http://localhost:8000/terminal
```

The current Docker setup packages the FastAPI application, trading engine and web frontend into a single container\. There is no Docker Compose configuration or multi\-container infrastructure in the current repository\.

---

## Current Scope and Limitations

TradeX currently focuses on demonstrating exchange and real\-time backend mechanics\. The following are **not implemented** in the current codebase:

- persistent database storage
- user authentication
- role\-based authorization
- pre\-trade risk controls
- position management
- account balances
- production\-grade order persistence
- distributed matching engines
- Redis caching
- Kafka or other external event brokers
- load balancing
- Kubernetes deployment
- cloud infrastructure configuration
- production observability/monitoring
- rate limiting
- production secret management

The current exchange state is held in application memory\. Restarting the application clears active orders, order history and in\-memory trade/event state\.

The system is therefore best described as a **real\-time exchange simulation and backend engineering project**, rather than a production trading exchange\.

---

## Future Improvements

Possible future engineering work includes:

- persistent storage for orders and trades
- authentication and authorization
- pre\-trade risk validation
- account and position management
- stronger concurrency controls around matching
- dedicated market\-data/event infrastructure
- improved price\-level data structures
- distributed deployment
- observability and metrics
- rate limiting and production security controls
- automated CI/CD
- cloud deployment

These are future improvements, not currently implemented features\.

---

## Interview Focus

The strongest technical areas demonstrated by the current implementation are:

1. **Matching\-engine design** — price\-time priority, maker/taker matching and partial fills\.
2. **Data structures** — dictionaries, price levels and FIFO queues\.
3. **Backend APIs** — FastAPI request validation and REST endpoint design\.
4. **Real\-time systems** — WebSocket event delivery and per\-symbol subscriptions\.
5. **State management** — in\-memory order and trade state\.
6. **Testing** — unit, API, integration and WebSocket tests\.
7. **System limitations** — persistence, concurrency, scalability and production hardening\.
8. **Containerisation** — packaging and running the application with Docker\.

TradeX is intentionally scoped so that the core matching and real\-time behaviour can be understood, tested and extended without introducing unnecessary distributed\-system infrastructure\.
