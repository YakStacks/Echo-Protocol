# Echo Protocol Specification

**Version:** 0.1 (Draft)
**Author:** James Yarber (Yak Stacks)
**Prior art established:** 2026
**Status:** Specification draft — reference implementation in progress

---

## Abstract

Echo Protocol is an open coordination specification governing how distributed AI inference nodes discover each other, advertise capabilities, route requests, and synchronize software and model state. It is designed for community-owned AI infrastructure deployed at anchor institutions (libraries, schools, courthouses) under unreliable network conditions, where local-first operation is a hard requirement rather than a degraded mode.

The protocol is named for orca echolocation: each node broadcasts its presence and capability into the network and listens for others, building shared awareness of the cluster from distributed signals. No central coordinator is required for normal operation.

Echo Protocol is model-agnostic and vendor-agnostic by design. Routing decisions are based on declared capability flags, never on model identity or vendor.

## 1. Design Principles

1. **Local-first.** Every node fully serves its local users with no network connection. WAN connectivity is an enhancement, not a dependency.
2. **Capability over identity.** Nodes advertise what they can do, not what they are. Any model, any vendor, any runtime that satisfies a capability contract participates identically.
3. **Typed handoffs.** All inter-node task transfer uses Agent Handoff Protocol (AHP) typed packets. No raw-string escalation. Every handoff is validated, auditable, and recoverable.
4. **Graceful degradation.** Network partition is a normal operating condition, not a fault. Each partition continues independently; state reconciles on reconnection.
5. **Gated propagation.** Software and model updates propagate only after passing quality verification (e.g., Pappy-style acceptance gating), with rollback capability on failure.
6. **No extraction.** The protocol is an open commons. No subscription, license fee, or vendor relationship is required to implement or participate.

## 2. Terminology

| Term | Definition |
| --- | --- |
| **Node** | A compute appliance running an Echo-compatible runtime and serving inference locally |
| **Primary** | A higher-capability anchor node; typical escalation target within its region |
| **Satellite** | A lighter node serving a local population independently; escalates beyond-tier tasks |
| **Cluster** | The set of nodes that share peer knowledge and route among themselves |
| **Capability packet** | The structured JSON state advertisement broadcast by every node |
| **AHP packet** | A typed task-handoff unit as defined by the Agent Handoff Protocol |
| **Escalation** | Routing a task from a node that cannot satisfy its capability requirements to one that can |

## 3. Core Functions

### 3.1 Node Discovery

Nodes announce presence to the local subnet and to known cluster peers.

- Periodic broadcast on the local subnet for zero-configuration discovery of co-located nodes
- Peer-to-peer announcement to known cluster members for cross-site awareness
- Heartbeat with a 30-second TTL; a node missing consecutive heartbeats is marked unreachable and routing tables update accordingly
- New nodes joining the network are recognized automatically; no central registration step is required

### 3.2 Capability Advertising

Each node continuously broadcasts a structured capability packet describing its current state:

- Model tier and capability flags (see §4)
- Available memory headroom
- Active request count and queue depth
- Specialization flags (domain-tuned models, modalities)
- Connectivity status (WAN up/down, failover active)

Packets are emitted on state change and on a 60-second interval. Receivers treat the most recent packet as authoritative and expire packets older than the advertisement interval plus grace.

### 3.3 Load Routing

When a node is saturated, or a task declares capability requirements beyond the node's tier, Echo selects the best available node and executes an AHP-typed handoff.

Routing inputs, in order of weight:

1. **Capability match** — hard requirement; a node that cannot satisfy the task's declared capability flags is never selected
2. **Proximity** — prefer geographically/topologically nearer nodes to keep data local by default
3. **Load balance** — among equivalent candidates, prefer lower queue depth

Routing decisions MUST be based on declared capability, not model identity. Locking routing to any single model vendor is a non-conforming implementation.

### 3.4 Model and Software Sync

Updates to model weights, specialist models, and Echo runtime software distribute across the cluster automatically during configured off-peak windows.

- Differential sync to minimize WAN transfer on constrained links
- Verification-gated: only updates that pass the configured quality gate propagate
- Rollback: a node that fails post-update health checks reverts to its previous known-good state and reports the failure
- Sync is opportunistic and resumable; a partition during sync is not an error condition

## 4. Capability Packet Format

A capability packet is a JSON object. Required fields:

```json
{
  "echo_version": "0.1",
  "node_id": "ky-montgomery-lib-01",
  "node_tier": "primary",
  "timestamp": "2026-06-11T14:00:00Z",
  "ttl_seconds": 90,
  "capabilities": {
    "modalities": ["text"],
    "max_context_tokens": 128000,
    "capability_flags": ["general_text", "document_drafting", "long_context"],
    "specializations": []
  },
  "state": {
    "memory_headroom_gb": 32.5,
    "active_requests": 3,
    "queue_depth": 1,
    "accepting_escalations": true
  },
  "connectivity": {
    "wan": "up",
    "failover_active": false
  }
}
```

Field semantics:

- `node_id` — stable, human-auditable identifier; uniqueness is the deployer's responsibility within a cluster
- `node_tier` — `primary` | `satellite`; informational, never a routing key on its own
- `capability_flags` — the routing contract; an open, extensible vocabulary (see §4.1)
- `accepting_escalations` — explicit consent bit; a node under maintenance can remain online for local users while opting out of cluster routing
- Model name and vendor are deliberately absent from required fields. Implementations MAY include an informational `model_info` object, but routers MUST NOT key on it.

### 4.1 Capability Flag Vocabulary

The base vocabulary is open and extensible. Initial flags:

`general_text`, `document_drafting`, `long_context`, `vision`, `audio_transcription`, `code_assistance`, `structured_output`, plus namespaced specialization flags of the form `specialist:<domain>` (e.g., `specialist:ky-agriculture`, `specialist:ky-statutes`).

Unknown flags MUST be ignored, not rejected, to allow forward-compatible extension.

## 5. AHP Integration

All inter-node task handoffs use Agent Handoff Protocol typed packets ([github.com/junkyard22/AHP](https://github.com/junkyard22/AHP)).

When a Satellite escalates a task to a Primary, it does not pass a raw text string. It constructs a structured AHP packet containing:

- Task objective and constraints
- Required capability flags (matched against §4 advertisements)
- User context and conversation history scoped to the task
- Expected output and lifecycle state

The receiving node has a complete, validated task description. Every escalation is therefore typed, validated, auditable, and recoverable — a documented event, not a fire-and-forget string. The resulting audit trail is the compliance and accountability record required for public infrastructure deployment.

AHP lifecycle states (`ready`, `running`, `waiting`, `completed`, `failed`, `blocked`, `cancelled`) are visible to the originating node throughout. A handoff whose owning node becomes unreachable transitions to `blocked` and is eligible for re-routing under the originator's policy.

## 6. Graceful Degradation

Echo Protocol treats unreliable connectivity as the design baseline, because that is the reality of rural deployment.

| Condition | Behavior |
| --- | --- |
| Primary offline | Satellites continue independent operation; escalation queue builds locally and drains when the Primary returns |
| Satellite offline | Primary absorbs the satellite's geographic area; load increases but no outage occurs |
| Full WAN outage | All nodes serve their local networks from local models; no capability is lost for physically present users |
| Partial cluster partition | Each partition operates independently; state reconciles when connectivity restores |

Reconciliation is last-writer-wins on capability state (it is ephemeral by design) and resumable-transfer on sync state. Task state reconciliation follows AHP lifecycle rules.

## 7. Security Considerations

Echo assumes a zero-trust posture: a node is not trusted because it is on the network.

- **Authentication:** cluster membership requires a shared cluster credential or per-node keys established at provisioning; implementations SHOULD support mutual TLS between peers
- **Authorization:** `accepting_escalations` and capability flags are advertisements, not entitlements; receivers validate inbound AHP packets against their own policy before execution
- **Audit:** every escalation produces a durable AHP lifecycle record on both sides of the handoff
- **Injection containment:** task payloads are data, not instructions to the Echo runtime; the routing layer never executes payload content
- **Blast radius:** compromise of one node does not grant cluster-wide control; there is no central coordinator to capture

A fuller threat model is maintained in `docs/THREAT-MODEL.md` (in progress).

## 8. Conformance

An implementation is Echo-conforming if it:

1. Emits valid capability packets per §4 on the required cadence
2. Routes on capability flags, never on model identity
3. Performs all inter-node task transfer via AHP typed packets
4. Continues full local service under WAN outage
5. Gates propagated updates behind a verification step with rollback

## 9. Prior Art Statement

Echo Protocol was conceived and developed in 2026 as an original architectural contribution within the Pod Network community AI infrastructure initiative. This specification is published as a defensive publication to establish prior art and prevent third parties from obtaining exclusive rights to the concept.

The novelty is not service discovery, heartbeats, or load balancing — all long-established in distributed systems. The novelty is the combination applied to community-owned AI inference infrastructure: capability-flag routing deliberately decoupled from model identity, typed and auditable agent task handoff as the only inter-node transfer mechanism, verification-gated model propagation with rollback, and local-first operation as a conformance requirement rather than a failure mode.

---

*Echo Protocol — the node speaks, the network listens, the work finds the right place.*
