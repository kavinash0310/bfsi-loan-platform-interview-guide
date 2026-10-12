# BFSI Loan Platform Interview Guide

An interview preparation guide for explaining a Java/Spring Boot loan-processing platform: its service boundaries, loan creation, disbursement, repayments, Kafka workflows, integrations, and production trade-offs.

The guide was prepared from a supplied code snapshot and the experience details shared by the author. It is written in generalized terms so it can be used for interview practice without publishing employer source code or customer information. It is not an official system design document.

## Start here

1. [Architecture and end-to-end flows](docs/architecture.md)
2. [Project interview Q&A](docs/project-interview-qna.md)
3. [Personal answer and rehearsal checklist](docs/rehearsal-checklist.md)

## Services represented in the snapshot

| Service area | Responsibility shown by the code |
|---|---|
| Loan management | Loan records and processing state, product-specific loan operations, disbursement, repayment, and status handling. |
| CRM integration | Inbound and outbound CRM data mapping, lead-stage events, callbacks, and failed-request records. |
| Admin | Administrative profiles, roles, employee hierarchy, and cache support. |
| Location orchestration | Coordinates geo-tagging and address/coordinate lookups; uses Redis-backed records and service clients. |
| Geocoding | Converts addresses and coordinates through an external geocoding provider. |
| Reference data | Shared lookup/master data exposed through REST and GraphQL endpoints. |

The ZIP also references other backend services through client interfaces. They are described by role in the guide; their implementations were not included in the snapshot.

## Privacy and accuracy

This repository contains explanatory notes only. It deliberately excludes source code, Git history, configuration, internal URLs, company and provider names, realistic test fixtures, customer identifiers, and credentials from the supplied archive. The architecture is anonymized and simplified. Product flags and downstream systems can change the path taken by a request.

The Q&A distinguishes what the snapshot shows from design recommendations. Before an interview, confirm which changes you personally made, which system owns each business record, and which deployment or security details you are allowed to discuss. Replace the prompts in the rehearsal checklist with truthful examples from your own work.

## Suggested study order

Understand the service map first, then rehearse the loan-creation, disbursement, and repayment flows aloud. Finally, practice the tricky questions about duplicate requests, asynchronous processing, partial failure, and what the code snapshot does not prove.

