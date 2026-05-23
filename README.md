# Payment Processing & Reconciliation Service

- A production-style payment backend that processes charges through a mock gateway, maintains a double-entry ledger, reconciles internal records against settlement reports, and uses an LLM to explain discrepancies in plain English for the operations team.

![Java](https://img.shields.io/badge/Java-17-007396?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2-6DB33F?logo=springboot&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white)
![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black)
![License](https://img.shields.io/badge/license-MIT-blue)
![Build](https://img.shields.io/badge/build-in%20progress-yellow)

---

## The Problem

When a payment is charged through a gateway like Stripe or Adyen, three things can go wrong that lose companies real money:

1. **Duplicate charges** — the user double-clicks, the network retries, and the same card gets charged twice.
2. **Lost transactions** — the gateway succeeds but your database write fails, so the customer paid and your system has no record.
3. **Silent drift** — over thousands of transactions, internal records and the gateway's settlement reports stop matching, and nobody notices until the monthly finance review.

This service is designed around solving those three problems.

## How It Solves Them

| Problem | Solution in this project |
|---|---|
| Duplicate charges | **Idempotency-key support** on the charge endpoint — the same key always returns the same result, even on retry |
| Lost transactions | **Double-entry ledger** — every charge writes balanced debit/credit entries in the same DB transaction as the payment record |
| Silent drift | **Scheduled reconciliation job** — nightly comparison of internal ledger against gateway settlement reports, with mismatches flagged |
| Hard-to-debug mismatches | **LLM-generated explanations** — when reconciliation flags a discrepancy, an LLM produces a plain-English summary for the ops team ("Charge of $50 appears in our ledger but not in the gateway report; likely a refund processed after settlement window closed") |

## Architecture

```
┌──────────────┐        ┌─────────────────────┐        ┌─────────────────┐
│  React UI    │──────▶ │  Spring Boot API    │ ─────▶ │  Mock Gateway   │
│  Dashboard   │        │                     │        │                 │
└──────────────┘        │  • Payment Intents  │        └─────────────────┘
                        │  • Idempotency      │
                        │  • Double-Entry     │        ┌─────────────────┐
                        │    Ledger           │ ─────▶ │  Webhook        │
                        │  • Reconciliation   │        │  Consumer       │
                        │    Scheduler        │        │  (with retry)   │
                        │  • LLM Explainer    │        └─────────────────┘
                        └──────────┬──────────┘
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │  PostgreSQL         │
                        │  payments / ledger  │
                        │  webhook_events     │
                        └─────────────────────┘
```

## Tech Stack

**Backend** — Java 17 · Spring Boot 3.2 · Spring Data JPA · Bean Validation · Spring Scheduling
**Database** — PostgreSQL 16 (prod) · H2 (dev / tests)
**Frontend** — React 18 · Vite · Tailwind
**Testing** — JUnit 5 · Mockito · Spring Boot Test
**Infrastructure** — Docker · Docker Compose · GitHub Actions CI
**LLM Integration** — Anthropic Claude API (for discrepancy explanations)

## Key Design Decisions

- **Double-entry bookkeeping over single-row updates.** Every money movement writes two ledger rows (debit + credit) that must sum to zero. This is how real financial systems work — it makes drift mathematically detectable.
- **Idempotency keys stored, not just checked.** The full response is cached against the key, so a retry returns the *exact* original response — not just "already processed."
- **Reconciliation is a scheduled batch job, not real-time.** Gateway settlement reports arrive on a delay (T+1, T+2), so reconciliation runs nightly against the previous day's window.
- **Webhook delivery uses exponential backoff with a dead-letter queue**, because consumers go down and we can't lose events.
- **LLM is used for explanation, not decision-making.** The system decides what's a discrepancy using deterministic rules; the LLM only generates human-readable summaries so ops doesn't have to read raw JSON diffs.

## Running Locally

```bash
# Clone and run — uses H2 in-memory, no setup needed
git clone https://github.com/pohekarabhilasha/payment-reconciliation.git
cd payment-reconciliation
mvn spring-boot:run
```

The API is live at `http://localhost:8080`. The H2 console is at `http://localhost:8080/h2-console` (JDBC URL: `jdbc:h2:mem:payments`).

### Running with PostgreSQL (production-like)

```bash
docker compose up -d           # starts Postgres
mvn spring-boot:run -Dspring-boot.run.profiles=postgres
```

### Running tests

```bash
mvn test
```

## Project Status

This project is being built incrementally — each step is a self-contained commit so the engineering thought process is visible in the Git history.

-  **Step 1** — Project skeleton, Maven setup, smoke test, CI-ready structure
-  **Step 2** — Core domain entities (`Payment`, `PaymentIntent`, `LedgerEntry`)
-  **Step 3** — REST API for creating payment intents
-  **Step 4** — Mock payment gateway integration + charge processing
-  **Step 5** — Idempotency-key support for exactly-once charges
-  **Step 6** — Double-entry ledger writes on every charge
-  **Step 7** — Webhook emission with retry + exponential backoff
-  **Step 8** — Scheduled reconciliation job against settlement reports
-  **Step 9** — LLM integration for discrepancy explanations
-  **Step 10** — React dashboard for transactions and reconciliation results
-  **Step 11** — Dockerfile + docker-compose
-  **Step 12** — GitHub Actions CI pipeline


## License

MIT
