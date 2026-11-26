# Integration Design

*The system design track for the combined job search plan (SE / Integration primary, FDE optional). These are the design rounds that SE, Integration Engineer and FDE loops tend to use: integrating with customers and third parties, rather than "design Twitter". Each topic follows the usual format: a deep-topic note, a cheat sheet, and a multiple-choice self-test at the end of the note.*

| Week | Topic | Notes | Cheat Sheet | Status |
| ---- | ----- | ----- | ----------- | ------ |
| 1 | API Design: resources, errors, pagination, idempotency, versioning | [API Design](API%20Design/API%20Design.md) | [Cheat Sheet](API%20Design/Cheat%20Sheet.md) | Ready to study |
| 1 | HLD Delivery Framework for integration rounds | [Delivery Framework](Delivery%20Framework/Delivery%20Framework.md) | — | To write |
| 2 | Auth: OAuth2/OIDC, API keys, mTLS, service-to-service | [Auth](Auth/Auth.md) | — | To write |
| 3 | Async and events: webhooks, queues, idempotency, retries, DLQs | [Async and Events](Async%20and%20Events/Async%20and%20Events.md) | — | To write |
| 4 (light) | Data integration: batch vs streaming, CDC, schema evolution | [Data Integration](Data%20Integration/Data%20Integration.md) | — | To write |
| 5 | Reliability at the boundary: rate limits, backoff, circuit breakers, observability | [Reliability](Reliability/Reliability%20at%20the%20Boundary.md) | — | To write |

**Worked designs** live in [Problem Breakdowns](../Problem%20Breakdowns/Problem%20Breakdowns.md).

**Still useful from earlier work:** [Core Building Blocks](../Core%20Concepts/Core%20Building%20Blocks.md) (caching, consistency per feature, back-of-envelope numbers) and the algorithms section of the [Rate Limiter](../../LLD/Problem%20Breakdowns/Rate%20Limiter/Rate%20Limiter.md) LLD (Week 5).

**How to study each topic:** read the HelloInterview page → design the week's prompt out loud using the delivery framework → compare with a real company's API docs → write the note, cheat sheet and self-test.
