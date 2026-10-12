# Project interview Q&A

Answers are written as speaking notes. Use the ones that match what you actually worked on, and be ready to name your own change, test, or production example. “Snapshot shows” means the behavior is visible in the supplied code; it does not mean the author personally implemented it.

## Easy — understand the project

### 1. What problem did the platform solve?

It supported loan-related backend workflows such as creating loan records, coordinating disbursement and repayment operations, and integrating with customer, lead, CRM, payment, and reference-data systems. The services separated those responsibilities so each integration or domain area could evolve without putting the entire workflow into one application.

### 2. How would you explain the project in one minute?

I worked on Java/Spring Boot backend services for a loan-processing platform. The loan-management service coordinated product-specific loan operations and external integrations. It used persistent processing records for workflow state and Kafka for selected asynchronous work and status events. Supporting services handled CRM synchronization, admin data, location lookup, and shared reference data.

### 3. What was the main service you worked with?

The loan-management service is the central workflow service in the supplied snapshot. It exposes loan and payment-related APIs, selects product-specific implementations, keeps processing and status records, and calls downstream systems. I would describe my own work more narrowly: backend integrations and concurrent payout/disbursement workflows.

### 4. What other services are part of the supplied project bundle?

The bundle includes loan management, CRM integration, admin, location orchestration, geocoding, and reference-data services. They are related backend services, but they do not all sit in one request path. I would explain the ones relevant to the interviewer’s question and identify which ones I personally changed.

### 5. What does the CRM integration service do?

It maps and exchanges data between the loan/origination side and the CRM. The snapshot includes inbound and outbound operations, composite/batch updates, stage events, and records for failed requests. The mapper and adapter layers keep external CRM payloads from becoming the internal domain model.

### 6. What does the reference-data service do?

It serves shared, relatively stable lookup data that multiple services need. Keeping that data behind a service avoids each consumer maintaining a separate copy of the same reference values. The snapshot exposes lookup functionality through REST and GraphQL.

### 7. Why is there an admin service?

It handles operational user data such as admin profiles, role mappings, employee hierarchy, and related updates. That is a separate responsibility from creating or servicing a customer loan, so it can have its own data model and access rules.

### 8. What do the location services do?

The location orchestrator handles geo-tagging and lookup flows and stores location-related results in Redis. A separate geocoding service calls an external provider to translate addresses and coordinates. Redis can avoid repeating expensive lookups, but it should be treated as a cache rather than the authoritative source unless the business defines it otherwise.

### 9. Why use both REST and Kafka?

REST is useful when a caller needs a direct response or must validate a request immediately. Kafka is useful when work can continue after the API responds, when services need event-based handoff, or when consumers need to process events independently. The choice depends on the business flow, latency needs, and failure behavior.

### 10. What build and packaging evidence appears in the snapshot?

The services include Maven build files, Dockerfiles, and GitLab CI configuration. That tells me how the code is set up to be built and packaged, but it does not prove a pipeline passed or identify the production deployment platform. The complete technology map and questions about how each component works are in the [tech stack guide](tech-stack-and-how-it-works.md).

## Medium — explain the request and data flow

### 11. How does the loan API choose an implementation?

The controller reads the product identifier and selects a registered orchestrator or provider-specific service. That keeps product differences behind an interface instead of filling the controller with product-specific branches. It also means an unknown product should be validated carefully so it cannot silently take the wrong default path.

### 12. What happens before loan creation starts?

The controller checks required loan, customer, disbursement, and lead data, runs domain validation, and scrubs the request. After that, it selects the synchronous or configured asynchronous flow. Validation should reject bad input before external side effects begin.

### 13. Why keep a loan-processing record?

Loan creation spans multiple calls and may take longer than a single database operation. A processing record keeps the request and progress that the service needs to resume or investigate work after a downstream failure. It is a workflow record, not proof that every downstream system completed successfully.

### 14. How does the synchronous path work?

The orchestrator creates or loads processing state, creates the customer and related parties as needed, handles collateral when applicable, and calls the core lending integration to create the loan. It then persists the loan record and starts follow-up work such as mandate or disbursement processing. The exact path can vary by product.

### 15. How does the asynchronous loan-creation path work?

The service validates the request, persists processing state, obtains or reuses a loan reference, stores an event, and publishes it to Kafka. A consumer later uses the saved state to create the customer and loan, update progress, and trigger follow-up events. A recovery scheduler retries eligible failed work.

### 16. How do you explain loan-processing statuses?

I describe them as checkpoints in the workflow: received, customer created, loan created, and event processed or failed. A status should move forward only after the corresponding operation has succeeded and the result has been persisted. I would confirm the exact enum/state transition rules before listing every status in an interview.

### 17. Why use a product-specific factory or strategy?

Different products may use different core systems, validation, payment rules, or response mappings. A factory/strategy routes a product to the right implementation while keeping the controller and common workflow readable. The cost is more configuration and a need for tests that verify each product is mapped correctly.

### 18. Where does the loan account number come from?

The asynchronous path checks for an already generated reference and otherwise obtains one through a sequence service or uses a configured lead reference. Persisting the chosen value before downstream processing gives retries a stable identifier. The business must ensure that its uniqueness rules are enforced at the correct boundary.

### 19. How is concurrency used in the project?

The code uses `CompletableFuture` for independent related-party work, and the experience details I shared also mention concurrent payout/disbursement workflows. Parallel work can reduce latency when calls are independent, but it needs bounded executors, clear timeouts, exception handling, and idempotent side effects. I would explain only the concurrency change I personally made.

### 20. Why is there a lock around some loan-creation requests?

Two requests for the same lead could otherwise create duplicate customers or loans. The snapshot requests a try-once lock with a time-to-live around a lead-based operation. A lock reduces concurrent duplication, but the database and downstream systems still need idempotency because locks can expire, fail, or be bypassed by another entry point.

### 21. How is a disbursement initiated?

The service validates the loan and payout details, checks loan eligibility and amount constraints, creates a disbursement record, then asks the payment integration to initiate the payout. A later payment status event updates the record and loan state. This separates the request to pay from the eventual confirmation that money moved.

### 22. How does the service reduce duplicate disbursements?

It uses stable disbursement record identifiers, duplicate-key handling, existing-status checks, and a check that active/completed disbursement amounts do not exceed the loan amount. The important interview point is that these controls help but are not a universal exactly-once guarantee; concurrent requests and provider timeouts still need reconciliation and downstream idempotency.

### 23. What happens when a payment status arrives later?

The service maps the payment response to a disbursement status, updates only from acceptable prior states, derives the loan state, and sends a status update to the appropriate source when allowed. This makes callbacks part of the workflow rather than treating the original HTTP response as the final result.

### 24. How does the asynchronous repayment path avoid duplicates?

The snapshot builds a duplicate-check identifier from the payment transaction reference and loan account, checks whether it already exists, persists the request, and publishes an event keyed by the loan account. A duplicate is rejected instead of being submitted again. A production system should also enforce uniqueness in storage and make the downstream repayment operation idempotent.

### 25. Why key repayment events by loan account?

Kafka preserves ordering within a partition, not across the whole topic. Using the loan account as the key sends that account’s events to the same partition, which helps preserve their relative order. It can also create a hot partition if a small number of accounts dominate traffic, so key distribution should be measured.

### 26. Why use a dead-letter-style record in CRM integration?

If an external CRM operation fails, retaining a sanitized request/response reference helps support investigate and replay it instead of losing the business update. A DLQ needs access controls, retention, redaction, replay procedures, and alerting; it should not become an unmonitored storage bucket for sensitive payloads.

### 27. What is the role of Redis here?

The snapshot shows Redis used for cache/location-related data and some service support. It can reduce repeated lookups or provide fast access to short-lived values. I would not describe it as the system of record for loan data without verifying that specific design.

### 28. How do the services communicate with downstream systems?

The code uses Feign or HTTP clients for request/response integrations, and Kafka listeners/publishers for selected event-driven workflows. A client adapter can centralize authentication, timeouts, request mapping, and error translation. The calling service still needs a clear policy for retries and duplicate side effects.

### 29. What database is used for loan workflow state?

The loan-processing, loan, disbursement, and repayment records in the snapshot are MongoDB documents. The admin module uses JPA-based relational persistence, and other services have their own stores. That reflects service-owned persistence; it does not mean every service shares one database or one transaction.

### 30. How did you observe system behavior?

The snapshot includes Actuator and metrics libraries, and code paths increment counters around selected workflow outcomes. In production I would correlate a request/lead/loan reference across logs and traces, monitor latency and failure rates per downstream dependency, and avoid putting sensitive data or bearer tokens in logs.

## Hard — explain trade-offs and failure handling

### 31. Why can’t `@Transactional` make loan creation atomic?

Spring transactions cover the configured local transaction manager and data store. They cannot atomically include a remote core-lending call, a payment provider, and a Kafka broker. The service therefore needs persisted workflow state, idempotent operations, retries, and, where needed, compensating actions or reconciliation.

### 32. How would you close the database-to-Kafka dual-write gap?

I would use a transactional outbox: save the business/workflow update and an outbox event in one local database transaction, then publish the outbox asynchronously and mark it delivered. Consumers still need idempotency because a publisher can resend after an uncertain acknowledgement. The snapshot shows persisted state plus Kafka publishing, so I would present outbox as a design improvement unless I verified it was implemented.

### 33. What if the core lending system succeeds but saving the local result fails?

The next retry must not blindly create another loan. Use a stable business reference or idempotency key with the downstream system, query for the existing loan when the response is uncertain, and reconcile the local processing record. Alert on records stuck between “request sent” and “result persisted.”

### 34. What if the payment provider accepts a payout but the request times out?

Treat the result as unknown, not as a definite failure. Query the provider using the stable payout reference, wait for a callback, or reconcile before retrying. Retrying immediately can create a second payout even when the first one succeeded.

### 35. How do you handle an out-of-order callback?

Validate the callback’s identifier and current state, then allow only legal state transitions. A late “requested” update should not overwrite a terminal “completed” state. Persist the event ID/status for deduplication and send contradictory or impossible transitions to investigation rather than silently applying them.

### 36. What happens if a Kafka consumer crashes during processing?

The broker can redeliver a record depending on offset-commit settings. The consumer should be idempotent and commit only after durable processing or after safely handing work to another durable queue. If it starts an asynchronous task and returns early, I would verify that the offset cannot be committed before the task finishes; otherwise a crash can lose the work.

### 37. Does Kafka provide exactly-once business processing?

No, not by itself. Kafka transactions can provide specific broker-level guarantees, but an external database or payment provider is outside that transaction. Business-level idempotency, deduplication, and reconciliation are still required.

### 38. How would you prevent two scheduler instances from retrying the same loan?

Use a distributed lease/lock or an atomic claim operation on the record, with an expiry and a compare-and-set style state change. The snapshot’s scheduler acquires a record lock and uses rate limiting, but the deployment configuration should be checked to confirm the lock is shared across instances. Metrics should show claimed, retried, failed, and exhausted work.

### 39. How would you size the executor for concurrent downstream calls?

Start from the downstream capacity, expected latency, request rate, and application resource limits. Use a named bounded executor with explicit queue capacity, timeout, rejection policy, and metrics; do not let unbounded work accumulate on the common pool. Protect each dependency with timeouts and a circuit breaker where appropriate.

### 40. A single product’s async flow is failing. How would you isolate it?

Compare success/error metrics by product and workflow stage, inspect the saved processing record, and correlate the event with its partition/offset and downstream call. Check product configuration, mapping/validation, feature flags, and provider response codes. Replay one sanitized record in a controlled path after confirming that downstream operations are idempotent.

### 41. How would you investigate a slow API when the database query is fast?

Break total latency into request parsing, validation, each downstream call, queue wait, thread-pool wait, serialization, and response writing. Compare p50/p95/p99 and dependency latency, inspect traces, and check connection/executor pool saturation. A fast database query does not rule out slow network calls or blocked worker threads.

### 42. How would you secure sensitive loan and identity data?

Use least-privilege service identities, TLS, short-lived tokens, encrypted storage where required, and strict authorization at the API boundary. Mask or omit account, identity, contact, and token fields from logs; restrict test data and DLQ access; and define retention. Only describe the exact authentication scheme you personally verified.

### 43. How would you scale the loan-management service?

Keep instances stateless where possible and store workflow state in the database. Scale API instances horizontally, scale Kafka consumers within partition limits, tune connection pools against the database’s capacity, and isolate slow dependencies with bounded concurrency and timeouts. Watch hot keys, retry storms, and downstream quotas while scaling.

### 44. How would you test the critical loan lifecycle?

Unit-test validation, mapping, state transitions, and idempotency. Use integration tests for persistence and Kafka serialization/consumer behavior, contract tests for downstream adapters, and scenario tests for timeouts, duplicate messages, and out-of-order callbacks. A payout test should use a fake provider and assert that a retry does not create a second payment.

## Tricky and scenario-based questions

### 45. The async API returned HTTP 200. Is the loan fully created?

Not necessarily. In the asynchronous path, the response can mean the request passed validation, processing state was saved, and the event was submitted. I would check the consumer result and persisted terminal status before saying the downstream loan was created. I would also confirm the producer acknowledgement policy.

### 46. Two identical disbursement requests arrive at the same time. What can still go wrong?

Both requests might pass an aggregate amount check before either record is visible. A unique record key protects the same disbursement ID, but different IDs for the same business payout may still race. I would enforce a stable idempotency key and atomic amount reservation/conditional update, then reconcile provider status.

### 47. A customer-creation call succeeds, then loan creation fails. What is the recovery plan?

Keep the workflow record and mark the failed stage with a retryable or terminal reason. On retry, reuse the customer reference or look it up rather than creating a second customer. Resume from the last safe checkpoint and alert if the record exceeds its expected processing time.

### 48. A Kafka event is published, but the producer times out before receiving an acknowledgement. Should it publish again?

The result is ambiguous: the broker may have stored the first copy. The producer or recovery job may need to retry, so the consumer must be idempotent. Use a stable event/business ID and track publish/consume status rather than relying on the timeout as proof that nothing was delivered.

### 49. The consumer launches a `CompletableFuture` and immediately returns. What should you inspect?

Check the Kafka listener’s acknowledgement mode and offset commit timing. If the listener returns before the future completes, the broker may consider the record consumed while business work is still running. Either keep processing inside the acknowledgement boundary or persist a durable handoff before returning.

### 50. The event is delivered twice after a consumer rebalance. Is that a Kafka defect?

No. Consumers can see a record again when processing completed but the offset was not committed, or when commit timing is uncertain. The application should deduplicate by event/business ID and make state transitions safe to repeat.

### 51. A retry keeps failing on one malformed message. What do you do?

Classify the error as transient or permanent. Retry transient failures with bounded exponential backoff and jitter; route permanently invalid or exhausted messages to a controlled DLQ with a reason and alert. Fixing the payload or code should be followed by an audited replay, not an infinite retry loop.

### 52. Redis is unavailable during a location lookup. What should happen?

If Redis is only a cache, fall back to the geocoding provider when safe, with a timeout and rate limit, then repopulate the cache. If Redis contains required workflow state, fail safely and do not pretend the operation succeeded. Confirm which data is cache versus authoritative before choosing the fallback.

### 53. A downstream service takes ten seconds to respond. How do you stop worker exhaustion?

Set a realistic connection/read timeout, cap concurrent calls, and fail fast when the dependency is unhealthy. Use a circuit breaker and bulkhead where they fit, and return a clear retryable response or accept work asynchronously if the business flow permits. Do not hold a request thread for ten seconds without a bounded capacity plan.

### 54. Duplicate repayments appear even though the request was deduplicated locally. Why?

Local deduplication may not cover a different transaction reference, a race before the record insert, an external retry, or a provider-side duplicate. Enforce a unique business key in the database and pass the same idempotency reference downstream. Reconcile the ledger before replaying.

### 55. A late payout failure callback arrives after a success callback. Which status wins?

Do not use arrival time alone. Validate provider event sequence/version or query the provider’s authoritative status, then apply a monotonic transition rule. Preserve both events for audit and alert on contradictory terminal states.

### 56. Can you call the loan process a Saga?

Only if the workflow has explicit local transactions and defined compensating actions for failures across services. The snapshot shows orchestration, persisted progress, retries, and callbacks; that alone does not prove a complete Saga implementation. I would describe it as a multi-step orchestrated workflow unless I can name the compensation for each step.

### 57. Was an API Gateway part of this project?

The supplied archive does not establish an API Gateway in front of these services. It includes service clients and, for some outbound integrations, access-token handling through a gateway-like provider. I would describe an inbound gateway only if I verified it in the deployment architecture.

### 58. Where did JWT/OAuth 2.0 fit?

The code snapshot includes authentication helpers and outbound bearer-token handling, but it does not show enough to claim the full token issuer, validation filters, or every service’s grant type. I would explain the exact flow I implemented or supported—who issues the token, how the service validates it, what scopes/claims matter, and how secrets are protected—and avoid guessing beyond that.

### 59. What was your individual contribution?

My experience was mainly in Java/Spring Boot backend integrations and concurrent payout/disbursement workflows. I would name one service, one specific change, the tests I added, and the production or business result. I would separate my work from the broader platform capabilities shown in the archive.

### 60. Tell me about a production issue you fixed.

Use a real example: “A [specific symptom] affected [scope]. I checked [logs/metrics/traces], found [root cause], and changed [code/config/process]. I verified it with [test/metric/business confirmation], and added [prevention].” Do not invent impact numbers or reuse customer data from test fixtures.

### 61. How would you handle two critical production issues at once?

Compare customer and financial impact, scope, data integrity, and whether either issue is worsening. Contain the highest-risk issue first, assign owners where possible, communicate status, and protect evidence for later root-cause analysis. For payment or balance uncertainty, prioritize preventing duplicate or incorrect money movement.

### 62. What would you do if the database CPU reached 100% during peak traffic?

First protect correctness and stabilize load: identify the expensive query/workflow, check connection pool saturation and recent deploys, and reduce nonessential traffic or retries if needed. Inspect query plans and indexes, batch size, hot documents/keys, and write amplification. Scale the database only after understanding the bottleneck and consistency requirements.

### 63. What if one Kafka partition is overloaded?

Check whether the key distribution is skewed and whether one account or product generates disproportionate traffic. Adding partitions alone may not help if the same hot key must stay ordered. Consider splitting workload by a safe sub-key only if the business ordering requirement allows it, and monitor consumer lag after the change.

### 64. A deployment raises p99 latency from 80 ms to 2 seconds. What is your sequence?

Compare the new version with the previous one, break down latency by endpoint and dependency, and inspect thread/connection pools, GC, queue lag, and error/retry rates. If customer impact is high and the regression is release-linked, roll back through the established deployment process while preserving traces and logs. Reproduce the issue before making a targeted fix.

### 65. What is the biggest design trade-off in this architecture?

The key trade-off is responsiveness versus workflow complexity. Asynchronous processing can return sooner and isolate slow downstream work, but it adds eventual consistency, duplicate delivery, status tracking, retries, and reconciliation. Synchronous processing is easier for callers to understand but couples latency and availability to every dependency in the request path.

