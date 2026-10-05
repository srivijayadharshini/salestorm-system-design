# Reliability and Failure Handling

## 1. Retry

Temporary failures may be retried a limited number of times.

The system should use backoff between retry attempts to avoid
overloading the failing service.

Retries must be limited and must not continue forever.

## 2. Timeout

Every call to a downstream service has a timeout.

If the downstream service does not respond within the allowed
time, the request stops waiting.

A timeout does not automatically mean that an operation failed.
For payment operations, the system must check the payment status
before attempting another payment.

## 3. Circuit Breaker

If a downstream service repeatedly fails, the circuit breaker
opens.

While the circuit is open, requests are temporarily stopped.

After a recovery period, the system allows a test request.

If the downstream service has recovered, the circuit closes and
normal traffic resumes.

## 4. Dead Letter Queue

Messages that repeatedly fail processing are moved to a Dead Letter
Queue (DLQ).

The DLQ allows failed messages to be investigated and safely
reprocessed later.

## 5. Reliability Strategy

Temporary failure:
→ Retry

Slow response:
→ Timeout

Repeated downstream failure:
→ Circuit Breaker

Repeated message processing failure:
→ Dead Letter Queue

Payment timeout:
→ Check payment status before retrying payment.