# Bank Stack

**A Production-Oriented Digital Banking Platform built with Java, Spring
Boot Microservices, Spring Security, Apache Kafka, and PostgreSQL.**

Bank Stack is a learning and portfolio project exploring how a digital
banking backend can be designed around secure service boundaries,
reliable payment workflows, ledger-based transaction records, and
event-driven communication.

> **Project status:** In development. Features and architecture
> described here are intended scope unless explicitly marked as
> implemented. This repository is an educational project, not a real
> banking service.

## Overview

Bank Stack aims to model core digital-banking workflows while applying
backend engineering practices used in distributed systems:

-   Domain-driven service boundaries
-   Secure REST APIs and role-based authorization
-   JWT-based authentication and service-to-service security
-   Asynchronous communication through Kafka
-   Payment orchestration and saga-style workflows
-   Idempotent command handling and failure recovery
-   Ledger-oriented transaction history and auditability
-   PostgreSQL persistence and database-per-service boundaries where
    appropriate
-   Automated testing, observability, and containerized local
    development

The goal is to understand not only how to implement backend features,
but also why architectural choices are made and how systems behave when
dependencies fail.

## Goals

-   Design a clear functional MVP and define service responsibilities.
-   Build maintainable Spring Boot services with validation and
    consistent error handling.
-   Model account, payment, and transaction lifecycles explicitly.
-   Use events to decouple services where asynchronous processing is
    appropriate.
-   Handle retries, duplicate messages, and partial failures safely.
-   Document trade-offs, API contracts, event schemas, and operational
    concerns.

## Planned technology stack

  -----------------------------------------------------------------------
  Area                                Technology
  ----------------------------------- -----------------------------------
  Language                            Java

  Backend framework                   Spring Boot 3

  Security                            Spring Security, OAuth 2.0 / JWT

  Identity provider                   Auth0 or a configurable OIDC
                                      provider

  Messaging                           Apache Kafka

  Database                            PostgreSQL

  Persistence                         Spring Data JPA / Hibernate

  API                                 REST, JSON, OpenAPI

  Build                               Maven

  Testing                             JUnit 5, Mockito, Testcontainers

  Local environment                   Docker Compose

  Observability                       Actuator, structured logging,
                                      metrics and tracing (planned)

  CI                                  GitHub Actions (planned)
  -----------------------------------------------------------------------

> The exact versions and tools may evolve as implementation progresses.
> Update this section to match the dependencies actually used by the
> repository.

## Architecture

The following is a target architecture, not a claim that every service
is already implemented.


``` mermaid
flowchart LR
    Client[Web / Mobile Client] --> Gateway[API Gateway]
    Gateway --> Identity[Identity Provider]
    Gateway --> Account[Account Service]
    Gateway --> Payment[Payment Service]
    Payment --> Kafka[(Apache Kafka)]
    Kafka --> Ledger[Ledger Service]
    Kafka --> Notification[Notification Service]
    Account --> AccountDB[(Account PostgreSQL)]
    Payment --> PaymentDB[(Payment PostgreSQL)]
    Ledger --> LedgerDB[(Ledger PostgreSQL)]
```


### Candidate service boundaries

  -----------------------------------------------------------------------
  Service                             Responsibility
  ----------------------------------- -----------------------------------
  API Gateway                         Entry point, routing, and
                                      cross-cutting request controls

  Account Service                     Customer account metadata and
                                      account lifecycle

  Payment Service                     Payment initiation, validation,
                                      orchestration, and status

  Ledger Service                      Append-oriented debit/credit
                                      entries and transaction history

  Notification Service                Asynchronous customer notifications
                                      from domain events

  Identity Provider                   Authentication, token issuance, and
                                      identity management
  -----------------------------------------------------------------------

Service boundaries should follow domain ownership. Avoid sharing a
single database schema across services as a shortcut; use explicit APIs
or events to exchange information.

## Example payment workflow

A proposed payment flow:

1.  The client submits a payment request with a unique idempotency key.
2.  The Payment Service authenticates and authorizes the request,
    validates the command, and records the payment state.
3.  The service publishes a payment event using a reliable
    event-publishing strategy, such as the transactional outbox pattern.
4.  The Ledger Service consumes the event and records the appropriate
    ledger entries idempotently.
5.  Domain events update payment status and trigger downstream actions
    such as notifications.
6.  Retries and compensating actions handle recoverable failures;
    unresolved cases move to a review or dead-letter workflow.

**Important:** Kafka delivery and database updates are not automatically
one atomic transaction. The implementation must account for duplicate
delivery, retries, ordering, and the possibility that a consumer
processes a message more than once.

## Security approach

Planned security principles:

-   Authenticate users through OAuth 2.0 / OpenID Connect and validate
    JWTs.
-   Enforce authorization at the API and domain-operation layers.
-   Use least-privilege roles and scopes.
-   Validate all input and avoid exposing internal error details.
-   Protect service-to-service communication with an explicit trust
    model.
-   Keep secrets out of source control and use environment-based
    configuration.
-   Record security-relevant audit events without logging credentials,
    tokens, or sensitive payment data.

## Reliability and distributed-systems patterns

The project will explore these patterns where they fit the use case:

-   **Idempotency:** prevent repeated client requests or message
    deliveries from creating duplicate effects.
-   **Transactional outbox:** coordinate database state changes with
    event publication.
-   **Saga / process manager:** coordinate multi-step workflows without
    assuming a distributed ACID transaction.
-   **Retries with backoff:** retry transient failures while limiting
    retry storms.
-   **Dead-letter handling:** isolate messages that cannot be processed
    and support controlled replay.
-   **Optimistic locking:** protect concurrent updates where
    appropriate.
-   **Timeouts and circuit breakers:** limit the impact of slow or
    unavailable dependencies.
-   **Audit trail:** retain traceable records of important state
    changes.
-   **Observability:** correlate logs, metrics, and traces across
    service boundaries.

## Data and ledger principles

A banking-style ledger should not be treated as an ordinary mutable
balance field.

The intended design separates:

-   **Account data:** ownership, account status, and account metadata.
-   **Payment state:** requested amount, participants, status,
    idempotency key, and lifecycle timestamps.
-   **Ledger entries:** immutable debit and credit records representing
    financial movements.
-   **Balance view:** a derived or maintained projection whose
    consistency rules are documented.

The design should define currency handling, decimal precision, posting
rules, reversals, reconciliation, and concurrency behavior before
implementing money movement. A production banking system requires
substantially more controls, testing, compliance work, and operational
safeguards than this educational project.

## API sketch

These are example endpoint ideas and may change as the domain model is
implemented.

  ---------------------------------------------------------------------------------------------
  Method                  Endpoint                                      Purpose
  ----------------------- --------------------------------------------- -----------------------
  `GET`                   `/api/v1/accounts`                            List accounts available
                                                                        to the authenticated
                                                                        user

  `GET`                   `/api/v1/accounts/{accountId}`                Retrieve account
                                                                        details after
                                                                        authorization

  `POST`                  `/api/v1/payments`                            Initiate a payment with
                                                                        an idempotency key

  `GET`                   `/api/v1/payments/{paymentId}`                Retrieve payment status

  `GET`                   `/api/v1/accounts/{accountId}/transactions`   Read transaction
                                                                        history
  ---------------------------------------------------------------------------------------------

Document request schemas, response schemas, error codes, authorization
rules, and idempotency behavior alongside each implemented endpoint.

## Event sketch

Potential Kafka events include:

-   `PaymentRequested`
-   `PaymentAccepted`
-   `PaymentRejected`
-   `LedgerEntryPosted`
-   `PaymentCompleted`
-   `PaymentFailed`

Event contracts should be versioned and include an event ID, event type,
schema version, timestamp, correlation ID, and relevant aggregate
identifier. Avoid placing secrets or unnecessary personal information in
event payloads.

## Repository structure

One possible starting layout:

``` text
bank-stack/
├── services/
│   ├── account-service/
│   ├── payment-service/
│   ├── ledger-service/
│   └── notification-service/
├── infrastructure/
│   ├── docker/
│   └── kafka/
├── docs/
│   ├── architecture.md
│   ├── adr/
│   ├── api/
│   └── events/
├── docker-compose.yml
├── .env.example
├── .gitignore
└── README.md
```

Adapt this structure to the actual repository. A modular monolith can
also be a sensible first milestone before splitting domains into
independently deployable services.

## Getting started

Setup instructions will be added once the initial services and local
infrastructure are implemented.

A typical local workflow will be:

1.  Install a compatible JDK and Maven.
2.  Configure local environment variables using `.env.example`.
3.  Start PostgreSQL and Kafka using Docker Compose.
4.  Configure the identity provider and local test credentials.
5.  Build and test the services.
6.  Start the required services and inspect the API documentation.

Never commit real credentials, access tokens, database passwords, or
production configuration.

## Testing strategy

Planned test coverage:

-   Unit tests for domain rules and payment state transitions
-   API tests for validation, authentication, and authorization
-   Repository tests for persistence constraints
-   Integration tests with PostgreSQL and Kafka using Testcontainers
-   Consumer tests for duplicate events, retries, and invalid payloads
-   Failure-path tests for timeouts, unavailable dependencies, and
    partial workflows
-   End-to-end tests for successful and rejected payment scenarios

## Roadmap

-   [ ] Define the functional MVP and domain glossary
-   [ ] Write architecture decisions and service contracts
-   [ ] Set up the Spring Boot build and code-quality checks
-   [ ] Configure PostgreSQL and local Kafka infrastructure
-   [ ] Implement identity integration and authorization rules
-   [ ] Implement account management
-   [ ] Implement payment initiation and state transitions
-   [ ] Add idempotency and reliable event publishing
-   [ ] Implement ledger posting and transaction history
-   [ ] Add retries, dead-letter handling, and reconciliation scenarios
-   [ ] Add integration and end-to-end tests
-   [ ] Add metrics, health checks, structured logs, and tracing
-   [ ] Document deployment and operational runbooks

Update the checklist as work is completed; only mark features
implemented when they exist and have been tested.

## Contributing

Contributions, issue reports, and architecture discussions are welcome.
For substantial changes, open an issue first to explain the proposed
behavior or design trade-off. Keep changes focused, add tests for
behavior, and update documentation when API or event contracts change.

## Disclaimer

Bank Stack is an educational software engineering project. It is not
affiliated with a bank and must not be used to hold, transfer, or
process real funds. The architecture is illustrative and does not claim
regulatory compliance or production banking readiness.

## Author

Built as a backend engineering portfolio project to explore Java, Spring
Boot, microservices, security, Kafka, and distributed-system design.
