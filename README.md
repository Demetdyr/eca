# ECA - E-Commerce Application

A microservices-based e-commerce application. Backend services are built with **.NET 8 (C#)**, frontend with **Angular 20 (SSR)**. Inter-service async communication is handled by **RabbitMQ**, authentication by **Keycloak**, and API routing by **Kong Gateway**.

## Architecture

```
                            ┌──────────────────────────┐
                            │     Kong API Gateway     │
                            │        :8000 (HTTP)      │
                            └────┬────────┬────────┬───┘
                                 │        │        │
                ┌────────────────┘        │        └────────────────┐
                ▼                         ▼                         ▼
     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
     │   Product API    │     │    Order API     │     │  Inventory API   │
     │     :5157        │     │     :5002        │     │     :5003        │
     └───────┬──────────┘     └───────┬──────────┘     └───────┬──────────┘
             │                        │                         │
             ▼                        ▼                         │
     ┌──────────────┐         ┌──────────────┐                  │
     │  Product DB  │         │   Order DB   │◄─────────────────┘
     │    :5429     │         │    :5428     │
     └──────────────┘         └──────────────┘

     ┌────────────────────────────────────────────────────────────┐
     │              RabbitMQ  :5672 (AMQP) / :15672 (UI)         │
     │     Order API ◄──────────────────────► Inventory API      │
     └────────────────────────────────────────────────────────────┘

     ┌──────────────────┐         ┌──────────────────┐
     │    Keycloak      │         │   Angular UI     │
     │     :8080        │         │     :4000        │
     │   (IDP DB :5430) │         │     (SSR)        │
     └──────────────────┘         └──────────────────┘
```

## Tech Stack

| Layer | Technology |
|---|---|
| Backend APIs | .NET 8, ASP.NET Core, Entity Framework Core 9 |
| Frontend | Angular 20 (SSR), Tailwind CSS, Express, Bun |
| Database | PostgreSQL 18 (3 separate instances) |
| Message Queue | RabbitMQ 4.2 |
| Authentication | Keycloak (OpenID Connect, JWT) |
| API Gateway | Kong (DB-less, declarative configuration) |
| Containerization | Docker & Docker Compose |

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and Docker Compose
- Bash-compatible terminal (macOS / Linux / WSL)

## Getting Started

There is a CLI tool called `eca` under the `e-commerce-app/` directory. It spins up all services with a single command.

```bash
cd e-commerce-app

# Production (builds images and starts all services)
./eca prod up

# Stop all services
./eca prod down

# Development
./eca dev up

# Stop all services
./eca dev down
```

> `prod` mode runs services with the `--build` flag, rebuilding images on every start.

## Service URLs

Once all services are up, they are accessible at the following addresses:

| Service | URL | Description |
|---|---|---|
| **UI** | http://localhost:4000 | Angular SSR frontend |
| **Kong Gateway** | http://localhost:8000 | API Gateway (single entry point for all APIs) |
| **Keycloak** | http://localhost:8080 | Identity management console |
| **Product API** | http://localhost:5157 | Product catalog API |
| **Order API** | http://localhost:5002 | Order management API |
| **Inventory API** | http://localhost:5003 | Stock management API |
| **RabbitMQ UI** | http://localhost:15672 | Message queue management dashboard |
| **Kong Admin** | http://localhost:8001 | Kong configuration API |

## API Gateway Routes (Kong)

All APIs are accessible through a single port (:8000) via Kong:

| Route | Target Service |
|---|---|
| `/auth/*` | Keycloak (:8080) |
| `/api/v1/products/*` | Product API |
| `/api/v1/categories/*` | Product API |
| `/api/v1/orders/*` | Order API |
| `/api/v1/payments/*` | Order API |
| `/api/v1/inventory/*` | Inventory API |

Rate limiting: **5 requests/second**, **10,000 requests/hour**

## Microservices

### Product API

Product catalog and category management.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/v1/products` | List all products |
| GET | `/api/v1/products/{id}` | Get product details (with categories) |
| GET | `/api/v1/products/by?sku=X&sku=Y` | Get products by SKUs |
| POST | `/api/v1/products` | Create a new product |
| PUT | `/api/v1/products/{id}` | Update a product |
| DELETE | `/api/v1/products/{id}` | Delete a product |
| GET | `/api/v1/categories` | List categories |

### Order API

Order creation and payment management.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/v1/orders` | List all orders |
| GET | `/api/v1/orders/{id}` | Get order details |
| POST | `/api/v1/orders` | Create a new order |
| DELETE | `/api/v1/orders/{id}` | Delete an order |
| GET | `/api/v1/payments` | List payments |
| POST | `/api/v1/payments` | Process a payment |

**Order Statuses:** `new` → `awaiting_stock` → `awaiting_payment` → `paid` → `fulfilled` (or `cancelled`)

### Inventory API

Stock tracking and reservation management.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/v1/inventory/stock/{sku}` | Get stock info for a product |
| POST | `/api/v1/inventory/stock` | Add/update stock |
| GET | `/api/v1/inventory/reservations` | List reservations |

## Message Flow (RabbitMQ)

Async inter-service communication works as follows:

```
Order Created:
  Order API ──[order.reserve.stock]──► Inventory API
                                            │
                              ┌─────────────┼─────────────┐
                              ▼                           ▼
                       Stock Available             Stock Unavailable
                              │                           │
   Order API ◄──[inventory.order.stock.reserved]          │
       │                                                  │
       │         Order API ◄──[inventory.order.stock.rejected]
       ▼
  Payment Confirmed:
  Order API ──[order.confirmed]──► Inventory API
                                    (Deduct stock, confirm reservation)
```

| Queue | Publisher | Consumer | Description |
|---|---|---|---|
| `order.reserve.stock` | Order API | Inventory API | Stock reservation request |
| `inventory.order.stock.reserved` | Inventory API | Order API | Stock successfully reserved |
| `inventory.order.stock.rejected` | Inventory API | Order API | Insufficient stock |
| `order.confirmed` | Order API | Inventory API | Order confirmed, deduct stock |

## Authentication (Keycloak)

- **Realm:** `eca`
- **Protocol:** OpenID Connect / OAuth2 (JWT, RS256)
- **User registration:** Enabled

### Default Users

| User | Email | Role |
|---|---|---|
| `aaa` | aaa@eca.com | Admin (all permissions) |
| `ddd` | ddd@eca.com | Regular user |

### Permission Structure

Each API endpoint requires specific Keycloak roles:

- `products:view:any`, `products:create:any`, `products:update:any`, `products:delete:any`
- `orders:view:any`, `orders:create:any`, `orders:update:any`, `orders:delete:any`
- `category:view:any`, `category:create:any`, `category:update:any`, `category:delete:any`

## Default Credentials

| Service | Username | Password |
|---|---|---|
| RabbitMQ | `admin` | `admin123` |
| Keycloak Admin | From Keycloak env file | - |
| PostgreSQL DBs | From respective `.env` files | - |

## Project Structure

```
.
├── e-commerce-app/          # Docker Compose and infrastructure config
│   ├── eca                  # CLI tool (./eca prod up, ./eca dev down, etc.)
│   ├── docker/
│   │   ├── dev.yaml         # Development environment definition
│   │   └── prod.yaml        # Production environment definition
│   └── data/
│       ├── kong/kong.yml    # API Gateway declarative configuration
│       └── keycloak/realms/ # Keycloak realm import files
│
├── eca-product-api/         # Product catalog microservice (.NET 8)
├── eca-order-api/           # Order management microservice (.NET 8)
├── eca-inventory-api/       # Stock management microservice (.NET 8)
└── eca-ui/                  # Angular 20 SSR frontend
```

## Databases

Three separate PostgreSQL 18 instances are used:

| Database | Port | Used By | Tables |
|---|---|---|---|
| `product-db` | 5429 | Product API | `products`, `categories`, `product_categories` |
| `order-db` | 5428 | Order API, Inventory API | `orders`, `order_items`, `stock_items`, `reservations` |
| `idp-db` | 5430 | Keycloak | Keycloak internal |

Each API automatically runs its own SQL migration files (`Data/V1_init.sql`, `V2_seed_data.sql`) on startup via DbUp.
