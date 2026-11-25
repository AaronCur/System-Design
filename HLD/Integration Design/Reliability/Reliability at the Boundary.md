# Integration Design: Reliability at the Boundary

*Week 5. Rate limits, backoff with jitter, circuit breakers, timeouts, observability and distributed tracing. Prompt to troubleshoot out loud: "A customer's API calls started failing with 429s and timeouts after their launch."*

---

## Outline (to fill in)

1. Timeouts: every outbound call needs one
2. Retries: backoff with jitter, retry budgets, what not to retry
3. Rate limiting: token bucket vs sliding window (see the Rate Limiter LLD algorithms section), per-customer limits, rate-limit headers
4. Circuit breakers and bulkheads (Resilience4j)
5. Graceful degradation and fallbacks
6. Observability: logs, metrics, traces; correlation and request IDs
7. Distributed tracing with OpenTelemetry
8. Troubleshooting walkthrough: impact first, reproduce, isolate, mitigate, root cause, keep the customer updated
9. Level expectations

## Notes

## Suggested Resources

- HelloInterview: [Distributed Rate Limiter breakdown](https://hellointerview.com/learn/system-design/problem-breakdowns/distributed-rate-limiter); Networking Essentials
- Resilience4j docs (circuit breaker, retry)
- OpenTelemetry: "What is tracing?" overview
- Cross-reference: `LLD/Problem Breakdowns/Rate Limiter/Rate Limiter.md` section 5

## Self-Test — Multiple Choice
