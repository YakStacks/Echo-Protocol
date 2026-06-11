# Example: Satellite → Primary Escalation

A walkthrough of a conforming Echo Protocol escalation using AHP typed packets.

## Scenario

A patron at a satellite node (a rural community anchor) asks for help restructuring a 40-page grant application — a long-context task beyond the satellite's advertised `max_context_tokens`.

## Step 1 — Local capability check

The satellite's router compares the task's requirements against its own capability packet:

- Required: `long_context` flag, ≥100K context
- Satellite advertises: 32K context, no `long_context` flag
- Result: local execution not possible → escalation path

## Step 2 — Candidate selection

The satellite consults its current view of the cluster (capability packets received within TTL):

| Node | long_context | accepting_escalations | queue_depth | proximity |
| --- | --- | --- | --- | --- |
| ky-montgomery-lib-01 | yes | true | 1 | same county |
| ky-rowan-lib-01 | yes | true | 0 | adjacent county |

Capability match passes for both. Proximity weighting selects `ky-montgomery-lib-01`.

## Step 3 — AHP packet construction

The satellite constructs an AHP root packet (abridged):

```json
{
  "ahp_version": "1.0",
  "packet_id": "task-8f3a...",
  "origin_node": "ky-montgomery-sat-02",
  "owner_node": "ky-montgomery-lib-01",
  "objective": "Restructure grant application draft per attached outline",
  "constraints": {
    "data_locality": "in-cluster-only",
    "max_duration_seconds": 300
  },
  "required_capabilities": ["long_context", "document_drafting"],
  "expected_output": "Restructured document, markdown",
  "lifecycle_state": "ready"
}
```

No raw conversation dump. The packet carries the current job, its constraints, and its expected output — nothing else.

## Step 4 — Execution and lifecycle visibility

The Primary validates the packet against its own policy, accepts ownership, and transitions the packet `ready → running`. The satellite observes lifecycle state throughout. On completion, the result returns through the packet outcome (`running → completed`), not as freeform text in a side channel.

## Step 5 — Audit record

Both nodes retain the packet lifecycle record: who created it, who owned it, what was required, what was produced, and every state transition with timestamps. For a node deployed as public infrastructure, this is the accountability trail.

## Failure variant

If the Primary goes unreachable mid-task, the packet transitions to `blocked` at the originator. The satellite's policy may re-route to `ky-rowan-lib-01` (constructing a new child packet with the same objective) or queue the task until the Primary returns. Either way, nothing is lost and nothing is ambiguous — the user is told the request is queued, and local service continues uninterrupted for everyone else.
