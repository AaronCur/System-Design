# Integration Design: Async and Events

*Week 3. Webhooks, queues, idempotency keys, retries with backoff, dead-letter queues and ordering. Prompt to design out loud: "Deliver webhooks reliably to thousands of customers."*

---

## Outline (to fill in)

1. Sync vs async: when to return 202 and finish the work later
2. Webhooks as an outbound API: payload design, event IDs, thin vs fat events
3. Delivery guarantees: at-most-once, at-least-once, "effectively once"
4. Consumer-side idempotency: dedupe on event ID
5. Retries with exponential backoff and jitter; giving up
6. Dead-letter queues and replay tooling
7. Ordering: why global ordering is a trap; per-key ordering
8. Signing payloads (HMAC) and verifying them
9. Queues: RabbitMQ vs Kafka (link to your own RabbitMQ integration work)
10. Transactional outbox: publishing events without losing them
11. Worked example: webhook delivery platform
12. Level expectations

## Notes

## Suggested Resources

- HelloInterview: Key Technologies (Kafka); Common Patterns (long-running tasks)
- Stripe's webhook docs and best practices
- Your RabbitMQ integration in Spring Boot as a worked example

## Self-Test — Multiple Choice
