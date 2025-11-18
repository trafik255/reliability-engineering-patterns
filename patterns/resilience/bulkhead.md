## Goal: 
Prevent a failure in one part of the system from sinking the entire ship.

## Problem

In distributed systems, shared resources often become choke points:
* Thread pools
* Connection pools
* Worker queues
* CPU/memory in a shared pod
* Database connections

When one workload saturates those shared resources, unrelated features begin to fail.
This triggers:

* System-wide latency spikes
* Full outage from a single noisy neighbor
* Retry storms amplifying the failure
* Cascading failures across downstream dependencies

You need isolation — not everything should be able to kill everything else.

## Forces

You must balance:
* Isolation vs. efficiency — dedicating resources prevents collapse but may reduce utilization
*  vs. fairness — heavily used features cannot starve critical ones
* Cost vs. safety — more bulkheads = more cost
* Operational visibility — operators need metrics for each isolated pool
* Graceful degradation — the system must fail locally without ripple effects

## Solution
A Bulkhead divides system resources into isolated compartments so that a failure in one does not propagate to others.

### Typical bulkhead forms:
1. Thread/Worker Pool Isolation
* Each subsystem gets its own worker pool.
* Slow or failing operations exhaust only their own threads.
2. Connection Pool Isolation
* Separate database pools per feature or service.
* Prevents a high-volume feature from exhausting all DB connections.
3. Queue Isolation
* Independent queues per job type or priority class.
* Prevents long-running work from blocking fast-path operations.
4. Physical/Deployment Isolation
* Separate pods, nodes, node groups, or autoscaling groups per service tier.
* Noisy neighbors can’t consume shared CPU/memory unexpectedly.

## How It Works
Imagine your service handles two types of work:
* High-volume analytics queries (non-essential)
* Low-volume clinical updates (mission-critical)
Without bulkheads:
* A surge in analytics saturates DB connections → clinical updates fail → full outage.

With bulkheads:
* Analytics has its own connection pool.
* Clinical updates have their own, smaller, protected pool.
* Analytics can fail without touching clinical updates.

## Implementation Examples
### 🔹 Kubernetes (Pod-Level Isolation)

#### Separate Deployments + HPAs:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: patient-updates
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: app
          resources:
            limits:
              cpu: 500m
              memory: 512Mi
```
```aiexclude
apiVersion: apps/v1
kind: Deployment
metadata:
  name: analytics
spec:
  replicas: 10
  template:
    spec:
      containers:
        - name: app
          resources:
            limits:
              cpu: 200m
              memory: 256Mi
```

Each gets its own:
* HPA
* resource limits
* scaling behavior
* failure domain
Analytics can spike → only analytics pods struggle.

### 🔹 AWS ECS (Service-Level Isolation)

#### Separate ECS services using different task definitions and their own autoscaling:
* patient-updates-service → 2 tasks, high CPU reservation
* analytics-worker-service → 6 tasks, low CPU, no impact on updates
Provisioned concurrency prevents starvation.

### 🔹 Database Connection Pool Isolation

Example (pseudo-config):
``` 
[connections]
clinical_pool.min=5
clinical_pool.max=20

analytics_pool.min=1
analytics_pool.max=5
```
Analytics can never exhaust the clinical pool.

### When to Use It

Use bulkheads when:
* One subsystem can become unpredictable or bursty
* A downstream dependency is fragile or rate-limited
* System components have different criticality levels
* You want graceful degradation instead of outages
* You want predictable performance under heavy load

### Tradeoffs
* Harder capacity planning (each bulkhead needs its limits)
* May increase cost due to underutilized dedicated resources
* Incorrect boundaries can cause false isolation
* Requires good observability to tune properly

### Related Patterns
* Circuit Breaker → Protects callers from repeatedly hitting a failing dependency
* Timeouts → Prevents slow calls from blocking workers
* Load Shedding → Drop non-critical requests under load
* Retries + Backoff → Works best when coupled with bulkheads

### Summary

Bulkheads are one of the most effective resilience tools you can implement.
They localize failure, protect critical workloads, and prevent your system from collapsing when a single feature misbehaves.

Container ships use bulkheads to keep water from spreading.
Distributed systems should do the same.