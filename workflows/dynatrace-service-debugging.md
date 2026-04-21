# Dynatrace Service Debugging

Read `../workflows/dynatrace-investigation.md` first, then use this playbook when the question is a direct debugging question rather than a formal incident.

## Workflow

1. Identify the failing workload shape.
- HTTP or API request path
- async worker, projection, or consumer path
- scheduled job or batch execution
- downstream dependency failure
- If the workload shape is unclear, use topology, logs, and operation names to classify it before deep searching.

2. Define the symptom precisely.
- Capture the exact failing endpoint, operation, exception, status code, or user-visible symptom.
- Anchor the first search on the narrowest known combination of:
  - entity
  - time window
  - operation name
  - exception type
  - identifier

3. Follow the workload-specific drill-down.
- HTTP or API:
  - check failing endpoints or operations
  - compare status classes and top exception types
  - use spans to identify the first failing downstream hop
- Worker, consumer, or projection:
  - check failure spikes, retry noise, poison or repeat processing patterns, and backlog or lag symptoms
  - use logs and spans to isolate the failing stage
- Scheduled job or batch:
  - check whether runs started, completed, took longer than normal, or stopped entirely
  - look for partial progress markers and terminal exceptions
- Downstream dependency:
  - look for one dependency signature repeated across multiple callers, endpoints, or operations
  - confirm whether the service is failing because of its own code or because a shared downstream is timing out, rejecting, or returning bad data

4. Stop when the first credible failing hop is identified.
- Prefer identifying the earliest confirmed failing boundary over collecting every downstream symptom.
- If the root cause remains unclear, say which hop is last known good and which hop is first known bad.
