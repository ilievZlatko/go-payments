# Payments

A payments platform built in Go as a set of microservices. Services talk to each other asynchronously over **NATS JetStream**, use **Redis** for caching and idempotency, store their state in **PostgreSQL**, and expose metrics to **Prometheus** with dashboards in **Grafana**. A dedicated **mailing service** sends the user an email whenever an action happens on their account, such as a payment being sent or received.

Everything runs locally with **Docker Compose**.

> **Status:** early development. The wallet service skeleton and core infrastructure (Postgres, Redis, NATS) are in place. Other services and the observability stack are planned. See the [Roadmap](#roadmap).

## Architecture

```
                 ┌──────────────┐
   client ──────▶│ API Gateway  │
                 └──────┬───────┘
                        │ HTTP
          ┌─────────────┴─────────────┐
          ▼                           ▼
   ┌──────────────┐            ┌──────────────┐
   │   Wallet     │            │   Payment    │
   │   service    │            │   service    │
   └──────┬───────┘            └──────┬───────┘
          │  publish events           │
          └─────────────┬─────────────┘
                        ▼
                ┌───────────────┐
                │ NATS JetStream│
                └───────┬───────┘
                        │ subscribe
                        ▼
                ┌───────────────┐        ┌──────────┐
                │ Notification  │──SMTP─▶│  Mailbox │
                │ (mail) service│        └──────────┘
                └───────────────┘

   Shared: PostgreSQL · Redis · Prometheus ◀── /metrics ── all services
                                   │
                                   ▼
                                Grafana
```

### Services

| Service          | Responsibility                                                                 | Port   | Status      |
|------------------|--------------------------------------------------------------------------------|--------|-------------|
| **wallet**       | Wallets and balances: create a wallet, top up, read the balance                | `8081` | In progress |
| **payment**      | Moves money between wallets, records transactions, publishes payment events    | `8082` | Planned     |
| **notification** | Subscribes to events and emails the user about each action                     | `8083` | Planned     |
| **gateway**      | Single entry point: routing, authentication, rate limiting (backed by Redis)   | `8080` | Planned     |

### Infrastructure

| Component      | Purpose                                                              | Port(s)          |
|----------------|----------------------------------------------------------------------|------------------|
| **PostgreSQL** | Persistent storage for wallets and transactions                      | `5432`           |
| **Redis**      | Caching, idempotency keys for payment requests, rate limiting        | `6379`           |
| **NATS**       | Event bus with JetStream for durable, at-least-once delivery         | `4222`, `8222` (monitoring) |
| **Prometheus** | Scrapes `/metrics` from every service                                | `9090` (planned) |
| **Grafana**    | Dashboards for request rates, latency, payment volume, errors        | `3000` (planned) |
| **Mailpit**    | Local SMTP server and web inbox for testing emails                   | `1025`, `8025` (planned) |

## How a payment flows

1. The client sends `POST /payments` to the gateway with an `Idempotency-Key` header.
2. The payment service checks Redis for the key so a retried request is not charged twice.
3. In a single Postgres transaction it debits the sender's wallet, credits the receiver's and records the transaction.
4. It publishes a `payments.completed` event (or `payments.failed`) to NATS JetStream.
5. The notification service consumes the event and emails both the sender and the receiver.
6. Every step increments Prometheus metrics, which you can see in Grafana.

### Events

Events go to NATS subjects under a common prefix:

| Subject                  | Published by | Consumed by  | Triggers email                 |
|--------------------------|--------------|--------------|--------------------------------|
| `wallet.created`         | wallet       | notification | "Your wallet is ready"         |
| `wallet.topped_up`       | wallet       | notification | "Funds added to your wallet"   |
| `payments.completed`     | payment      | notification | "Payment sent" / "Payment received" |
| `payments.failed`        | payment      | notification | "Your payment failed"          |

## Getting started

### Prerequisites

- Go 1.26+
- Docker and Docker Compose

### Run the infrastructure

```sh
docker compose up -d
```

This starts Postgres, Redis and NATS. The NATS monitoring page is at <http://localhost:8222>.

### Run a service

```sh
go run ./cmd/wallet
```

Check it is up:

```sh
curl http://localhost:8081/healthz
# {"status":"ok"}
```

### Stop everything

```sh
docker compose down        # keep data
docker compose down -v     # also delete the Postgres volume
```

## Project layout

```
.
├── cmd/                 # One main package per service
│   └── wallet/
├── internal/            # Code shared between services (config, db, events, metrics)
├── db/                  # SQL migrations
├── docker-compose.yaml  # Local infrastructure
└── go.mod
```

## Configuration

Services read their configuration from environment variables. Copy `.env.example` to `.env` for local overrides; `.env` is git-ignored.

| Variable        | Default                                                        |
|-----------------|----------------------------------------------------------------|
| `DATABASE_URL`  | `postgres://payments:payments@localhost:5432/payments?sslmode=disable` |
| `REDIS_ADDR`    | `localhost:6379`                                               |
| `NATS_URL`      | `nats://localhost:4222`                                        |
| `SMTP_HOST`     | `localhost`                                                    |
| `SMTP_PORT`     | `1025`                                                         |
| `MAIL_FROM`     | `no-reply@payments.local`                                      |

## Roadmap

- [x] Docker Compose with Postgres, Redis and NATS (JetStream enabled)
- [x] Wallet service skeleton with health check and graceful shutdown
- [ ] Database migrations for wallets and transactions
- [ ] Wallet endpoints: create, top up, get balance
- [ ] Payment service with idempotency keys in Redis
- [ ] Publish domain events to NATS JetStream
- [ ] Notification service that sends emails for each event
- [ ] Mailpit in Docker Compose for local email testing
- [ ] Prometheus metrics on every service, plus Prometheus in Compose
- [ ] Grafana with provisioned dashboards
- [ ] API gateway with authentication and rate limiting
- [ ] Dockerfiles for each service and the full stack in Compose
- [ ] Tests and CI
