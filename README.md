# Marketplace Delivery Orchestration Platform

> System Design portfolio project for a marketplace delivery orchestration platform.

**Status:** In progress

## About the project

This project models the design of a delivery orchestration platform for a marketplace.

The platform coordinates order fulfillment across inventory, payment, warehouse, and external delivery providers while handling partial fulfillment, failures, retries, duplicate events, cancellations, and eventual consistency.

The goal of the project is to demonstrate the complete system analysis and system design process — from requirements discovery to API and event contracts, data modeling, reliability, security, and observability.

> This is an educational portfolio project and does not represent a production system developed for an employer.

## Business context

A marketplace works with multiple warehouses and delivery providers.

A single customer order may contain items stored in different warehouses and therefore may need to be split into several shipments.

The system must coordinate:

- inventory reservation;
- fulfillment planning;
- payment;
- shipment creation;
- delivery provider selection;
- cancellations and refunds;
- partial order fulfillment;
- failures of external systems.

One of the key challenges is maintaining consistent business states across independently operating services.

## Main design topics

The project will cover:

- Requirements analysis
- Business rules
- Functional and non-functional requirements
- C4 System Context and Container diagrams
- Domain and data modeling
- REST API and OpenAPI
- Asynchronous communication
- Kafka events
- Saga pattern
- State machines
- Idempotency
- Retry and DLQ strategies
- Transactional Outbox / Inbox
- Reconciliation
- Authentication and authorization
- Logging, metrics and distributed tracing
- Failure scenarios
- Scalability
- Architecture Decision Records (ADR)

## Planned project structure

```text
docs/
├── 01-project-brief.md
├── 02-requirements.md
├── 03-business-rules.md
├── 04-nfr.md
│
├── architecture/
├── data/
├── api/
├── events/
├── diagrams/
├── reliability/
├── security/
├── observability/
└── adr/
