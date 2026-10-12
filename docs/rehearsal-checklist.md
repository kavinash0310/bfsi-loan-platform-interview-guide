# Personal answer and rehearsal checklist

Use this page to turn the architecture notes into truthful answers about your own work. Fill it in privately before interviews; do not add employer-specific or customer data to a public repository.

## My contribution

- **Services I personally changed:** [name only the services you worked in]
- **Feature or workflow I implemented:** [one concrete change]
- **My part of the design:** [API, validation, mapping, persistence, Kafka, concurrency, testing, operations]
- **How I tested it:** [unit, integration, contract, manual, staging]
- **How we knew it worked:** [observable result; avoid invented numbers]
- **One trade-off I can explain:** [latency vs complexity, consistency vs availability, etc.]

## Details to confirm before presenting architecture as fact

- Which data store is authoritative for the loan account and payment ledger?
- Which products use synchronous creation and which use store-and-forward?
- What exactly does the API response guarantee on the asynchronous path?
- How are Kafka offsets committed, and what delivery guarantee was configured?
- Which topics and keys can I safely name? Do not publish internal topic names.
- Are retries handled by a scheduler, a Kafka retry/DLQ policy, or both for this flow?
- What idempotency key is used at each boundary: API, database, Kafka consumer, and payment provider?
- Is there an outbox, reconciliation job, or explicit compensation for cross-service failures?
- What lock implementation is behind the annotation, and is it shared across service instances?
- Which OAuth/JWT flow did I personally implement or support? Who issues and validates the token?
- What deployment, CI/CD, monitoring, and alerting tools did I personally use?
- Which parts of the wider service bundle did I not work on directly?

## 60-second answer framework

1. **Context:** What business capability did the platform support?
2. **Architecture:** Which service owned the workflow and which dependencies did it call?
3. **Flow:** Walk through one request from validation to persisted state to downstream result.
4. **Failure handling:** Explain one retry, duplicate, or status-update case.
5. **My contribution:** State exactly what you implemented and how you verified it.

## STAR production-story template

- **Situation:** What failed, and who or what was affected?
- **Task:** What part were you responsible for?
- **Action:** What evidence did you inspect, what root cause did you find, and what change did you make?
- **Result:** How did you verify recovery, and what prevention did you add?

Use a real event from your experience. Keep customer identifiers, account numbers, internal URLs, screenshots, logs, and confidential business details out of the story and this repository.

