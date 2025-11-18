## Timeout Pattern

Goal: Prevent slow or hung operations from consuming resources indefinitely and causing cascading failures.

### Problem
Distributed systems fail in two ways:
1. Fast failures (connection refused, immediate error)
2. Slow failures (hangs, long tail latency, partial timeouts)
Slow failures are more dangerous — they:
* Block worker threads
* Consume connection pool slots
* Trigger retry storms
* Amplify throughout the system
Without timeouts, a single slow dependency can take down the entire service.

### Forces
You must balance:
* Correctness vs. speed — some operations legitimately take longer
* Consistency — every call path must have a timeout
* Resource preservation — timeouts free workers early
* Observability — you need visibility into timeout rates

### Solution
Set a maximum allowable time for any external or internal call.
If the operation exceeds that time, it is aborted and the caller handles the failure.

### Where to apply timeouts
* HTTP API calls
* Database queries
* Message broker operations
* Internal RPC calls
* External third-party APIs
* Long-running tasks in workers
* Timeout → abort → retry (with backoff) or fallback.

## Implementation Examples
### HTTP Client Example
```aiexclude
response = requests.get(
    url,
    timeout=2.0  # seconds
)
```

### Database Query Example
```
SET statement_timeout = '1500ms';
``` 
### Kubernetes Liveness/Readiness
```aiexclude
livenessProbe:
  httpGet:
    path: /health
  timeoutSeconds: 2
  failureThreshold: 3

```
### When to Use It
Always. Every external call must have:
* A timeout
* A retry policy
* A circuit breaker
Timeouts prevent blocking behavior and limit tail latency.

### Tradeoffs
* If set too low → false failures
* If set too high → worker starvation during slowdowns
* Requires tuning per dependency

### Related Patterns
* Retry + Backoff
* Circuit Breaker
* Bulkhead
* Load Shedding

