# Circuit Breaker Pattern

## Problem
Distributed systems fail in unpredictable ways.  
When a downstream service becomes slow or unstable, upstream callers keep sending traffic, making the outage worse. This creates:

- Cascading failures  
- Thread/connection exhaustion  
- Long tail latencies  
- Full-system meltdowns

## Forces
You must balance:

- **Fail fast vs. retrying** — too many retries overwhelm downstreams; too few reduce reliability.  
- **Statefulness** — the circuit must track failures/timeouts without becoming a bottleneck.  
- **Graceful degradation** — clients need fallback behavior when the circuit is open.  
- **Visibility** — operators need metrics and logs showing breaker state.

## Solution
A circuit breaker wraps a risky call and transitions through three states:

1. **Closed**  
   - Traffic flows normally.  
   - Failures/timeouts are tracked.  
   - If failures exceed a threshold → move to **Open**.

2. **Open**  
   - Immediately block new calls (“fail fast”).  
   - Protects downstream from overload.  
   - Returns fallback responses or errors without calling the dependency.

3. **Half-Open**  
   - After a cooldown, allow a small number of test calls.  
   - If successful → transition to **Closed**.  
   - If failures persist → return to **Open**.

This feedback loop prevents cascading failure across distributed systems.

## Reference Implementation (Python)

```python
import time
import requests

class CircuitBreaker:
    def __init__(self, fail_threshold=5, reset_timeout=30, half_open_probe=1):
        self.fail_threshold = fail_threshold
        self.reset_timeout = reset_timeout
        self.half_open_probe = half_open_probe
        
        self.fail_count = 0
        self.state = "CLOSED"
        self.next_attempt_at = 0

    def call(self, fn, *args, **kwargs):
        now = time.time()

        # Move to HALF-OPEN?
        if self.state == "OPEN" and now >= self.next_attempt_at:
            self.state = "HALF_OPEN"
            self.fail_count = 0

        # OPEN state → fail fast
        if self.state == "OPEN":
            raise Exception("Circuit Open: failing fast")

        try:
            result = fn(*args, **kwargs)

            # Success in HALF-OPEN → close
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"

            self.fail_count = 0
            return result

        except Exception:
            self.fail_count += 1

            if self.fail_count >= self.fail_threshold:
                self.state = "OPEN"
                self.next_attempt_at = now + self.reset_timeout

            raise
  ```` 
# Example usage
````breaker = CircuitBreaker()

def fetch_user():
    return requests.get("https://api.example.com/user").json()

try:
    data = breaker.call(fetch_user)
except Exception:
    # Fallback logic
    data = {"user": None, "fallback": True}
````
# Kubernetes Example (Envoy / Istio)
```apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: users-service
spec:
  host: users
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 5
      interval: 1s
      baseEjectionTime: 30s
      maxEjectionPercent: 80
```` 
# Terraform Example (API Gateway Throttling as Circuit Breaker)
```resource "aws_api_gateway_method_settings" "circuit_breaker" {
  rest_api_id = aws_api_gateway_rest_api.api.id
  stage_name  = aws_api_gateway_stage.stage.stage_name
  method_path = "*/*"

  settings {
    throttling_burst_limit = 100
    throttling_rate_limit  = 50
    metrics_enabled        = true
    logging_level          = "ERROR"
  }
}
```

## Failure Modes
* Threshold too low → circuit flaps
* Threshold too high → breaker opens too late
* No fallback paths → users see hard failures
* No observability → teams don’t understand breaker state
* Single global breaker → becomes a bottleneck or SPOF

## Operational Playbook
* Track metrics:
  * circuit_open_total 
  * circuit_half_open_total 
  * circuit_failures_total
* Alert when:
    * Breaker remains open > 2× reset timeout 
    * Half-open probes keep failing
* Recommended defaults:
  * Threshold: 5–10 failures 
  * Reset timeout: 30–60s
* Always implement fallback responses:
  * Cached data 
  * Degraded API response 
  * Alternate backend

## Diagram
```   
    [Closed]
      |
      | failures >= threshold
      v
    [Open] --- (timeout) ---> [Half-Open]
      ^                        |
      |----- success <---------|
``` 

