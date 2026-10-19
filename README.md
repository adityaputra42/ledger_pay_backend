# LedgerPay

**A Go-based payment and ledger backend built with Modular Monolith,
CQRS, SQLC, PostgreSQL, and Redis.**

LedgerPay is an educational portfolio project focused on reliable
payment workflows, double-entry bookkeeping, idempotent operations,
webhook delivery, and reconciliation. The MVP focuses on simulated
top-ups, internal transfers, and manually verified payments.
Cryptocurrency payment integration is planned for a later phase.

> **Project status:** MVP in development\
> **Important:** This project is for learning and demonstration. It is
> not intended to process real money or production payments.

## Overview

LedgerPay explores how to build a payment backend where financial
records remain consistent under retries, concurrent requests, and
partial failures.

The main engineering goals are:

-   Modular Monolith architecture with explicit module boundaries.
-   CQRS-style separation between commands and queries.
-   Type-safe SQL access using SQLC.
-   Double-entry ledger with immutable posted journal entries.
-   Atomic database transactions and durable idempotency.
-   Transactional outbox and signed webhook delivery.
-   Reconciliation checks and auditable findings.
-   Secure API and infrastructure configuration, including TLS.

## Planned Features

### MVP

-   [ ] User registration and authentication.
-   [ ] Internal accounts and simulated balances.
-   [ ] Double-entry ledger and transaction history.
-   [ ] Internal account-to-account transfers.
-   [ ] Simulated top-ups.
-   [ ] Manual payment instructions and admin verification.
-   [ ] Hold, capture, release, and refund workflows.
-   [ ] Idempotency for financial write operations.
-   [ ] Transactional outbox and signed webhooks.
-   [ ] Ledger and business-state reconciliation.
-   [ ] Docker Compose development environment.
-   [ ] Integration, concurrency, and end-to-end tests.
-   [ ] OpenAPI documentation and CI pipeline.

### Out of scope for the MVP

-   Real-money processing or bank/payment gateway integrations.
-   Cryptocurrency, blockchain RPC, smart contracts, and wallet signing.
-   Multi-currency and foreign exchange.
-   Microservices, Kubernetes, or a message broker without a
    demonstrated need.
-   Custodial private-key management.

Feature checkboxes should be updated as implementation and tests are
completed.

## Architecture

``` text
Client / Flutter App
        |
      HTTPS
        |
  Reverse Proxy (Nginx)
        |
      Go API
        |
  Modular Monolith
  ├── Identity
  ├── Account
  ├── Ledger
  ├── Payment
  ├── Manual Payment
  ├── Transfer
  ├── Idempotency
  ├── Webhook
  └── Reconciliation
        |
  PostgreSQL + Redis
        |
  Transactional Outbox
        |
      Worker
        └── Webhook delivery and background tasks
```

The API and worker are components of the same modular monolith; they are
not separate business microservices. CQRS separates write operations
from read operations while using PostgreSQL as the primary data store.
The MVP does not require event sourcing, a separate read database, or
NATS.

### Core design principles

1.  **The ledger is the source of truth for internal balances.** Do not
    update a standalone balance value without recording the
    corresponding financial entries.
2.  **Posted journal entries are immutable.** Correct mistakes with
    auditable compensating entries rather than editing or deleting
    posted entries.
3.  **Every posted journal must balance.** Total debits must equal total
    credits for each ledger transaction.
4.  **Financial operations must be atomic.** Related business records,
    journal entries, idempotency results, and outbox events must commit
    or roll back together where they form one business operation.
5.  **Idempotency must be durable.** PostgreSQL is the source of truth;
    Redis is only an optimization.
6.  **Do not perform external network calls inside financial database
    transactions.** Use an outbox and background worker after commit.
7.  **Modules communicate through contracts.** A module should not
    directly depend on another module's private SQLC-generated
    repository.
8.  **Use integer-based monetary values.** Do not use `float32` or
    `float64` for money.

## Technology Stack

  -----------------------------------------------------------------------
  Technology                          Purpose
  ----------------------------------- -----------------------------------
  Go                                  API and background worker

  `chi`                               HTTP routing

  PostgreSQL                          Primary data store and transaction
                                      boundary

  `pgx/v5`                            PostgreSQL driver and transaction
                                      support

  SQLC                                Type-safe generated database access

  Redis                               Cache, rate limiting, and selected
                                      coordination optimizations

  Structured logger                   Application and audit-related
                                      operational logs

  Docker Compose                      Local development environment

  OpenAPI / Swagger                   API contract and documentation

  Go testing + PostgreSQL integration Automated verification
  tests                               
  -----------------------------------------------------------------------

Dependency versions and implementation choices should be pinned and
documented as the project evolves.

## Module Responsibilities

  -----------------------------------------------------------------------
  Module                              Responsibility
  ----------------------------------- -----------------------------------
  `identity`                          Users, credentials, authentication

  `account`                           Internal accounts, ownership,
                                      account status

  `ledger`                            Ledger transactions, debit/credit
                                      entries, posting rules

  `payment`                           Payment lifecycle and business
                                      status

  `manualpayment`                     Manual payment instructions,
                                      references, and verification
                                      workflow

  `transfer`                          Internal transfer orchestration

  `idempotency`                       Idempotency keys, request
                                      fingerprints, stored results

  `webhook`                           Webhook endpoints, signatures,
                                      retries, dead-letter handling

  `reconciliation`                    Invariant checks, reconciliation
                                      runs, and findings

  `platform`                          Shared infrastructure such as
                                      database, transaction manager,
                                      HTTP, config, logger, Redis, and
                                      outbox
  -----------------------------------------------------------------------

Not every module needs the same internal folder structure. Add
abstractions only when they serve a real responsibility.

## Ledger Model

LedgerPay uses double-entry bookkeeping. Each posted ledger transaction
must have at least two entries, and the sum of debits must equal the sum
of credits.

Illustrative internal transfer of IDR 100,000:

  Account                     Debit        Credit
  ------------------- ------------- -------------
  Sender account            100,000             0
  Recipient account               0       100,000
  **Total**             **100,000**   **100,000**

This example assumes a documented debit-normal convention for the
relevant internal accounts. The exact account types and accounting
treatment must be defined in the ledger design documentation.

Important rules:

-   Only posted entries contribute to the posted balance.
-   Entries must have positive amounts and valid accounts.
-   Entries in a journal must use a consistent asset/currency.
-   Prevent duplicate journals using unique business references in
    addition to API idempotency.
-   Prevent disallowed overdrafts under concurrent requests.
-   Reconciliation must check each journal independently; a global
    debit/credit total can hide offsetting errors.
-   Reconciliation reports anomalies; it must not silently modify the
    ledger.

For the fiat MVP, use integer amounts with an explicitly documented
unit. If crypto is added later, represent token amounts in their
smallest units with a sufficiently large integer representation and
store asset/network metadata separately.

## Payment Workflows

### Simulated top-up

A simulated top-up creates a ledger posting using explicitly identified
simulation/clearing accounts. It must never be represented as
confirmation that real funds were received.

### Manual payment

1.  Create a payment with an amount, reference, instructions, and expiry
    where applicable.
2.  The user submits a reference or supporting evidence.
3.  An authorized reviewer verifies the payment according to the defined
    process.
4.  The system validates the allowed status transition and records an
    audit trail.
5.  Post ledger entries only when the business rules for recognizing the
    payment have been satisfied.

Uploading evidence alone must not automatically mark a payment as
successful.

### Transfer, hold, capture, release, and refund

These workflows must validate ownership, account status, allowed state
transitions, amounts, and idempotency. Refunds must use new compensating
journal entries rather than modifying the original posted payment
journal.

## Idempotency, Outbox, and Webhooks

### Idempotency

-   Scope keys by actor/user, operation, and key.
-   Store a normalized request fingerprint.
-   Same key + same request: return the previous operation result.
-   Same key + different request: reject the request.
-   Use database constraints to handle concurrent requests safely.
-   Persist the result with the relevant financial operation in the
    appropriate database transaction.
-   Redis must not be the only storage for idempotency state.

### Transactional outbox

Write an outbox event in the same database transaction as the business
change that produced it. A worker delivers the event after commit.

### Webhook delivery

-   Use HTTPS and verify TLS certificates.
-   Sign payloads using HMAC-SHA256 with a documented canonical format.
-   Include a stable event ID and timestamp.
-   Retry transient failures with exponential backoff and jitter.
-   Track attempts and move exhausted deliveries to a dead-letter state.
-   Assume **at-least-once delivery**; consumers should deduplicate
    events.
-   Protect outbound webhook requests against SSRF, unsafe redirects,
    and unbounded response sizes.

## Security

-   Require HTTPS for public API traffic.
-   Use TLS with certificate verification for connections crossing
    untrusted networks.
-   Do not expose PostgreSQL or Redis directly to the public internet.
-   Hash passwords using an appropriate password-hashing algorithm.
-   Enforce authentication and resource-level authorization on the
    server.
-   Validate input and apply rate limits, timeouts, and request-size
    limits.
-   Keep secrets out of source control; provide placeholders in
    `.env.example`.
-   Redact passwords, access tokens, webhook secrets, and sensitive data
    from logs.
-   Protect administrative actions with explicit authorization and audit
    trails.
-   Never trust a client-supplied balance, payment status, or amount
    without server-side validation.

See [`docs/security.md`](docs/security.md) when the security
documentation is implemented.

## Repository Structure

The planned structure is:

``` text
ledgerpay/
├── cmd/
│   ├── api/
│   ├── worker/
│   └── reconcile/
├── internal/
│   ├── modules/
│   │   ├── identity/
│   │   ├── account/
│   │   ├── ledger/
│   │   ├── payment/
│   │   ├── manualpayment/
│   │   ├── transfer/
│   │   ├── idempotency/
│   │   ├── webhook/
│   │   └── reconciliation/
│   └── platform/
├── migrations/
├── sqlc/
├── tests/
│   ├── integration/
│   ├── concurrency/
│   └── e2e/
├── docs/
│   └── adr/
├── compose.yaml
├── Dockerfile
├── Makefile
├── go.mod
└── README.md
```

This is the intended target structure; directories and commands should
be added as their implementation begins.

## Development Roadmap

  --------------------------------------------------------------------------------
  Phase                         Work                       Completion criteria
  ----------------------------- -------------------------- -----------------------
  0\. Foundation                Go module, config,         App starts locally and
                                logging, Docker Compose,   CI passes
                                health checks, CI          

  1\. Database                  Migrations, PostgreSQL     Migrations and code
                                pool, transaction manager, generation are
                                SQLC                       reproducible

  2\. Identity & account        Authentication,            Users can access only
                                authorization, account     permitted accounts
                                ownership                  

  3\. Ledger core               Journal model, atomic      Balanced journals,
                                posting, balance/history   rollback, and
                                queries                    constraints are tested

  4\. Transfers                 Transfer command,          Concurrent transfers
                                concurrency control,       cannot cause disallowed
                                idempotency, outbox        overdrafts

  5\. Top-up & manual payment   Simulation flow, manual    Invalid transitions and
                                verification, audit trail  duplicate credits are
                                                           rejected

  6\.                           State machine and          Amount limits and
  Hold/capture/release/refund   compensating journals      repeated requests are
                                                           tested

  7\. Idempotency hardening     Fingerprints, concurrent   Retries do not
                                key claims, result replay  duplicate financial
                                                           operations

  8\. Outbox & webhook          Worker, HMAC, retry,       Delivery failures are
                                dead-letter                tracked and recoverable

  9\. Reconciliation            Journal/business invariant Anomalies are detected
                                checks                     and recorded

  10\. Security & API docs      TLS, API validation,       Deployment
                                OpenAPI, audit logging     configuration and API
                                                           docs are reviewed

  11\. Portfolio readiness      Concurrency/failure/load   Results and known
                                tests, diagrams, demo      limitations are
                                                           documented
  --------------------------------------------------------------------------------

Implement one phase at a time. Do not mark a feature complete until its
implementation and relevant tests are in place.

## Testing Strategy

### Unit tests

Test ledger rules, request validation, payment state transitions,
idempotency fingerprints, webhook signature generation, and retry
policies.

### Integration tests

Use a real PostgreSQL test database or container to test migrations,
constraints, commit/rollback, SQLC queries, idempotency, and outbox
consistency. Mocks alone are insufficient for database concurrency
guarantees.

### Concurrency tests

Example scenario: an account has IDR 100,000, and two concurrent
requests each attempt to transfer IDR 80,000. If overdrafts are
disallowed, at most one transfer should succeed.

Also test:

-   Concurrent requests with the same idempotency key.
-   Duplicate payment approvals.
-   Repeated capture, release, or refund requests.
-   Two workers attempting to claim the same outbox event.
-   Client retry after the server commits but before the client receives
    the response.

### Failure tests

Test database errors before commit, worker restarts, webhook
timeouts/5xx responses, and uncertain external operation outcomes.

## Getting Started

The exact setup commands will be documented once the corresponding
Makefile targets and Compose services are implemented.

Planned commands:

``` bash
# Start local dependencies
make dev-up

# Apply database migrations
make migrate-up

# Generate SQLC code
make sqlc

# Run tests
make test

# Run the API
make run-api

# Run the worker
make run-worker
```

These commands are planned targets, not guaranteed to exist in the
current codebase. Check the actual `Makefile` and project configuration
before running them.

## Documentation

Planned project documentation:

-   `docs/architecture.md` --- system architecture and deployment.
-   `docs/module-dependencies.md` --- module boundaries and allowed
    dependencies.
-   `docs/ledger-design.md` --- account types, debit/credit conventions,
    and posting rules.
-   `docs/transaction-flows.md` --- transfer, manual payment,
    hold/capture/release, and refund flows.
-   `docs/security.md` --- TLS, authentication, secrets, threat model,
    and webhook security.
-   `docs/adr/` --- Architecture Decision Records for significant
    technical choices.
-   OpenAPI specification --- endpoint contracts and request/response
    schemas.
-   Test reports --- integration, concurrency, failure, and load-test
    results.

## Future: Cryptocurrency Payments

Crypto is **not part of the MVP**. After the manual payment and ledger
flows are stable, future work may include:

1.  Wallet connection and ownership verification.
2.  An adapter for one blockchain network.
3.  Deposit monitoring and transaction/finality verification.
4.  Idempotent on-chain deposit crediting.
5.  Reconciliation between on-chain observations and internal ledger
    records.
6.  Withdrawal workflows after signing security and operational controls
    are designed.

Crypto integration will require explicit decisions about custody,
networks, tokens, finality, fees, signing, and accounting. On-chain
balances must not be assumed to equal internal user balances.

## Disclaimer

LedgerPay is an educational portfolio project. MVP top-ups are
simulations, and manual payments follow a demonstration verification
process. This software is not a production payment system and must not
be used to process real funds without appropriate security reviews,
operational controls, legal/regulatory review, and professional
auditing.
