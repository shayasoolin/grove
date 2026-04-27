# Training Job Support — High Level Design

This document describes the high-level design for adding training job support to Grove. The goal is to extend the `PodCliqueSet` API and its controllers to support finite workloads that complete normally, as opposed to the current inference-oriented model which assumes pods run indefinitely.

## Training Mode

Training mode is signaled implicitly by the presence of any of `completions`, `maxRestarts`, or `maxRuntime` on a PodClique template. No separate mode flag is needed.

In training mode, Grove sets the Kubernetes pod-level `restartPolicy: Never` on all pods it creates, instead of its normal `Always` default. This gives Grove — not kubelet — full ownership of restart decisions. With `Always`, kubelet silently restarts a container in-place and the pod never reaches a terminal phase, making it impossible for the controller to observe failures cleanly. With `Never`, a pod reaches `phase=Failed` on any container exit with a non-zero code, which is the signal Grove acts on.

---

## Key Concepts and Design Decisions

### Per-replica maxRestarts

`maxRestarts` is a per-child budget, not a total count across the resource:

- At the **PCLQ** level: each pod index gets its own `maxRestarts` budget independently of other pods.
- At the **PCSG** and **PCS** level: each replica index gets its own `maxRestarts` budget independently of other replicas.

This makes `maxRestarts` scale-invariant. A total-count alternative was considered but rejected: it creates a counterintuitive relationship between replica count and failure tolerance where scaling up a PCSG makes each replica's effective budget smaller.

### Failure means "completion is no longer reachable"

A resource is marked **Failed** when enough children have permanently exhausted their `maxRestarts` budget that the completion criterion can no longer be met, regardless of how all remaining children resolve.

**Example:** a PodClique with `replicas=10` and `completions=8`. Once 3 pods have each exhausted their per-pod `maxRestarts` budget, only 7 pods remain capable of completing. Since 7 < 8, the PCLQ is marked Failed.

This definition was chosen over the simpler alternative of treating `maxRestarts` as an independent failure threshold. The simpler formulation — "when the total number of pod failures across the PCLQ reaches `maxRestarts`, the PCLQ is Failed" — has a correctness problem: it can produce **phase toggling**. Consider a PCLQ with `completions=2` and `maxRestarts=3` (total): if 3 pod failures occur and the PCLQ is marked Failed, but then 2 other pods complete successfully, the completion criterion is now satisfied. Under orthogonal criteria the resource would transition Failed → Completed, which is confusing for users and error-prone for controllers watching for terminal states. The "cannot reach completion" definition avoids this entirely: once a resource is marked Failed, it is provably impossible for it to become Completed, because failure is defined as the completion threshold being permanently out of reach.

### maxRuntime is an unconditional failure

When `maxRuntime` is exceeded, the resource is marked **Failed** immediately and permanently. No subsequent pod or replica completions can reverse this — the resource stays Failed even if pods that were still running at the deadline later exit with code 0.

This differs from the pod-exhaustion failure path, where there is a window in which both failure and completion might theoretically be reached concurrently. In that case, completion wins: the workload achieved its goal, and the failures were incidental. `maxRuntime` does not participate in this trade-off — it is a hard external deadline that is not subject to outcome negotiation. Once the wall clock has spoken, the resource is Failed.

### Restart count bookkeeping

Restart counts cannot be derived purely from child resources at reconcile time, for two structural reasons:

1. **PCLQ pods can be deleted before the count is read.** Failed pods are retained for log access, but users may delete them manually, or a future garbage collection policy may clean them up. The controller cannot depend on the continued presence of failed pod objects to reconstruct restart history.
2. **PCSG has no replica CRD.** A PCSG replica is a logical grouping defined by a replica index, not a Kubernetes resource. There is no child object to attach restart state to.

The solution is to store per-child restart counts in the resource's own status, as a `[]int32` array indexed by child index (`status.replicaRestartCounts[podIndex]` for PodClique, `status.replicaRestartCounts[replicaIndex]` for PCSG and PCS). This is consistent with how JobSet and the Kubeflow Training Operator handle the same problem.

**Boundary:** using the status to maintain bookkeeping counters that accumulate history is valid and standard Kubernetes practice. Using the status to make structural decisions — such as how many pods should currently exist — is not; those decisions must always be derived from the actual count of live child resources.

---

## Terminal States

The approach is bottom-up: define completion and failure semantics for a PodClique first, then for a PodCliqueScalingGroup, then for the PodCliqueSet.

### PodClique

A PodClique's terminal state is determined by the exit states of its pods. `completions` defaults to the total replica count when not specified (all pods must complete).

| Policy param | Effect on Completed | Effect on Failed |
|---|---|---|
| *(default)* | All pods exit with code 0 | Any pod failure makes completion unreachable |
| `completions` | At least `completions` pods exit with code 0 | Enough pods have permanently failed that fewer than `completions` can still complete |
| `completions` + `completedIndexes` | At least `completions` pods in the eligible index set exit with code 0 | Enough eligible pods have permanently failed that fewer than `completions` eligible pods can still complete |
| `maxRestarts` | — | Per-pod budget; a pod that exhausts it is permanently dead and counts toward unreachability |
| `maxRuntime` | — | Wall-clock time since first pod running exceeds `maxRuntime` (unconditional, cannot be overridden by subsequent completions) |

`completedIndexes` restricts which pod indexes are eligible to count toward `completions`. For example, `completions=3, completedIndexes=0-2` requires exactly pods 0, 1, and 2 to complete successfully. A failed pod outside the eligible set does not affect the failure criterion unless its failure makes it impossible to reach `completions` eligible completions.

`maxRuntime` is a wall-clock deadline measured from when the first pod starts running. It is not reset on restarts.

`maxRestarts: 0` means no replacement is created on failure — a pod's first failure immediately marks it permanently dead.

---

### PodCliqueScalingGroup

PCSG evaluation is two-level:

1. **Replica state** — a PCSG replica is Completed when its constituent PodCliques have completed. If `completedNames` is set, only the named PodCliques must complete for the replica to be considered Completed; if `completedNames` is empty, all PodCliques within the replica must complete.
2. **PCSG state** — derived by counting how many replicas have Completed or permanently Failed.

`completions` defaults to the total replica count when not specified.

| Policy param | Effect on Completed | Effect on Failed |
|---|---|---|
| *(default)* | All replicas Completed | Any replica permanently Failed makes completion unreachable |
| `completedNames` | Scopes which PodCliques within a replica must complete for that replica to count as Completed | — |
| `completions` | At least `completions` replicas Completed | Enough replicas have permanently failed that fewer than `completions` can still complete |
| `completions` + `completedIndexes` | At least `completions` replicas in the eligible index set Completed | Enough eligible replicas have permanently failed that fewer than `completions` eligible replicas can still complete |
| `maxRestarts` | — | Per-replica budget; a replica that exhausts it is permanently dead and counts toward unreachability |
| `maxRuntime` | — | Wall-clock time since first replica running exceeds `maxRuntime` (unconditional) |

`completedIndexes` restricts which replica indexes are eligible to count toward `completions`. For example, a PCSG with 10 replicas, `completions=5`, and `completedIndexes=0-4` requires replicas 0–4 to complete; replicas 5–9 are excluded from counting.

---

### PodCliqueSet

PCS follows the same two-level evaluation pattern as PCSG:

1. **Replica state** — a PCS replica is Completed when its constituent PodCliques and PodCliqueScalingGroups have completed. If `completedNames` is set, only the named constituents must complete.
2. **PCS state** — derived by counting how many replicas have Completed or permanently Failed.

The same parameters apply at the PCS level — `completions`, `completedIndexes`, `completedNames`, `maxRestarts`, and `maxRuntime` — with identical semantics to PCSG.

---

## Failure Handling

### Pod failures within a PodClique

When a pod fails (`pod.status.phase == Failed`), Grove does not delete it immediately. The failed pod is retained so its logs remain accessible via `kubectl logs`. Instead, a replacement pod is created alongside it. This counts as one restart against that pod's per-index budget in `status.replicaRestartCounts`.

Pods in `PodFailed` and `PodSucceeded` phases are excluded from the active replica count — only Running and Pending pods count toward the desired replica total. Without this exclusion, a retained failed pod would make the count appear satisfied and no replacement would be created.

All terminal-phase pods (both succeeded and failed) are cleaned up together when the PodClique itself reaches a terminal state.

`maxRestarts: 0` means no replacement is ever created — the first failure immediately marks that pod as permanently dead.

### PCLQ failures within a PCSG

When a PodClique replica is marked Failed, the PCSG deletes it to free resources and creates a new one in its place. This counts as one restart against that replica's per-index budget in `status.replicaRestartCounts`.

Unlike pod-level failures, the failed PCLQ is deleted rather than retained. The constituent pods will have their own logs accessible until the PCLQ deletion cascades and removes them.

### Constituent failures within a PCS

When a constituent PCLQ or PCSG within a PCS fails, the PCS treats this as a gang-level failure: all PCLQs and PCSGs belonging to that PCS replica are deleted and recreated together. This counts as one restart against that PCS replica's per-index budget in `status.replicaRestartCounts`.

### Failure cascading

Failure propagates upward through the hierarchy. A pod exhausting its per-index restart budget contributes to its PCLQ's failure evaluation. A PCLQ failure causes its parent PCSG or PCS to consume from its own per-replica restart budget. Each level maintains its own independent budget, and each level's failure criterion ("cannot reach completion") is evaluated independently.

---

## Notes

### Synchronous training and the restartScope deferral

An earlier design included a `restartScope` field at the PodClique level with two values: `Local` (replace only the failed pod, leave others running) and `Global` (restart all pods when any one fails). `restartScope` has been deferred from the initial implementation for two reasons.

**A clean workaround exists for the primary use case.** For synchronous training workloads such as AllReduce and FSDP, a single worker failure stalls all others, making a full restart the only sensible response. This can be expressed without `restartScope: Global` by setting `completions=replicas` and `maxRestarts=0` on the PodClique. Any single pod failure immediately exhausts that pod's budget, makes the PCLQ unable to reach its completions target (since completion requires all pods and one is now permanently dead), and marks the PCLQ Failed. The parent PCSG or PCS then handles the failed PCLQ by deleting and recreating it as a whole — achieving the same effect as a global restart, through the normal cascading failure path.

**`restartScope: Global` at the PodClique level is redundant with parent-level handling.** If a PodClique restarts all its pods on a single failure (`restartScope: Global`), and the parent PCSG also restarts failed PCLQ replicas, both mechanisms respond to the same underlying failure event. This creates ambiguity about which layer is responsible and can lead to redundant concurrent restart attempts. Removing `restartScope` from the initial implementation avoids this overlap entirely and keeps the failure-handling responsibility clearly at a single level of the hierarchy.

### Gang scheduling and gang termination in training mode

**Gang scheduling** remains essential in training mode. Distributed training frameworks (AllReduce, FSDP, parameter server) typically block at a rendezvous barrier during startup, so a partial gang — where only some pods are running — cannot make progress. The existing `minAvailable` gang scheduling constraint applies unchanged: all pods wait behind scheduling gates until the full gang can be placed simultaneously, both on initial launch and after each restart.

**Gang termination** in inference mode is triggered by the `MinAvailableBreached` condition: when `readyOrStartingPods < minAvailable` for longer than `terminationDelay` (default 4 hours), the affected unit is deleted and recreated. The termination scope is determined by where in the hierarchy the breach is observed:

| Trigger | Scope terminated | Condition |
|---|---|---|
| PCLQ `minAvailable` breached | Nothing — sets condition only | — |
| PCSG replica has a breached PCLQ and PCSG still meets its own `minAvailable` | That single PCSG replica's PodCliques | after `terminationDelay` |
| PCSG's own `minAvailable` breached, or standalone PCLQ breached | Entire PCS replica (all PodCliques, all groups) | after `terminationDelay` |

A PCLQ breach alone never directly triggers deletion — it propagates upward. The PCSG evaluates whether its own `minAvailable` is still met: if yes, it restarts only the affected replica in isolation; if not, it escalates to the PCS, which terminates the entire PCS replica.

In training mode, gang termination is extended with a **second trigger**: a resource entering the `Failed` phase. Rather than introducing a separate restart mechanism for training failures, the `Failed` phase is treated as an additional signal that feeds into the same termination machinery, using the same scope and escalation semantics.

The two triggers cover complementary failure scenarios:

- **`MinAvailableBreached`** — handles cases where pods are alive but insufficient: a replacement pod stuck in `Pending` after a node failure never reaches `PodFailed` and cannot drive the failure cascade on its own. The `MinAvailableBreached` condition fires regardless, and after `terminationDelay` the gang is terminated and restarted.
- **`Failed` phase** — handles cases where the failure cascade has run to completion: enough pods have exhausted their `maxRestarts` budget that the `completions` target is permanently unreachable. This trigger is immediate — no `terminationDelay` applies, since the `Failed` determination is already deliberate.

In training mode, both triggers consume one restart from the affected unit's `replicaRestartCounts` budget at the parent level when the unit is recreated. This connects gang termination to the `maxRestarts` budget, ensuring that restarts from either trigger are counted and bounded.

**Standalone PCLQ restart granularity.** For PCSGs, the escalation ladder naturally provides per-replica restart granularity: a failing PCSG replica is restarted in isolation without disturbing other replicas. Standalone PCLQs within a PCS (those not owned by a PCSG) do not have this middle layer. A PCLQ breach propagates directly to the PCS, which restarts the entire PCS replica. This means that if a standalone PCLQ needs to be restarted as a whole — for example, because its leader pod failed and the PCLQ cannot make progress — the only path under the current model is a full PCS replica restart. This is correct behavior but has a wider blast radius than necessary. The straightforward workaround is to wrap the standalone PCLQ in a PCSG — even a PCSG with a single PCLQ — which restores per-replica restart granularity without any API or semantic changes.

---

## User Guide

### Overview

Training mode is activated on any PodClique whose template sets at least one of `completions`, `maxRestarts`, or `maxRuntime`. In training mode, Grove sets the Kubernetes pod-level `restartPolicy: Never` on all pods it creates.

### API Fields

All fields below apply at the PCLQ, PCSG, and PCS levels.

| Field | Behavior |
|---|---|
| `completions` | Number of children that must succeed for the resource to reach Completed. Defaults to the total child count. |
| `completedIndexes` | Restricts which child indexes are eligible to count toward `completions` (pod indexes at PCLQ level; replica indexes at PCSG/PCS level). Children outside this set are ignored for both completion and failure evaluation. |
| `completedNames` | Restricts which named children within a replica must complete for that replica to count as Completed. Unnamed children are not required to complete. |
| `maxRestarts` | Per-child restart budget: per pod index at the PCLQ level, per replica index at PCSG/PCS level. A child that exhausts its budget is permanently dead. `maxRestarts: 0` means no retries — the first failure permanently kills the child. |
| `maxRuntime` | Wall-clock deadline from when the first child starts running. Exceeding it marks the resource Failed immediately and permanently. Not reset on restarts. |

### Phase Semantics

**Completed:** at least `completions` eligible children have exited with code 0.

**Failed:** enough children have permanently exhausted their `maxRestarts` budget that fewer than `completions` eligible children remain capable of completing — or `maxRuntime` has been exceeded. A resource that reaches Failed cannot subsequently become Completed.

### Failure and Restart Behavior

When a **pod** fails, a replacement is created alongside it (the failed pod is retained for log access). This consumes one restart from that pod index's budget. Pods in terminal phases (`PodFailed`, `PodSucceeded`) are excluded from the active replica count, so a retained failed pod does not prevent a replacement from being created. Terminal-phase pods are cleaned up when the PCLQ reaches a terminal state.

When a **PCLQ** fails, its parent deletes it and creates a replacement PCLQ. This consumes one restart from that replica index's budget at the parent level. The failed PCLQ is not retained.

When a **PCLQ or PCSG** within a PCS fails, the PCS treats it as a gang-level failure: all constituents in that PCS replica are deleted and recreated together. This consumes one restart from that PCS replica's budget.

### Failure Cascade

Failure propagates bottom-up. Each level maintains its own independent `maxRestarts` budget and evaluates its own completion reachability independently:

- **Pod → PCLQ**: a pod exhausting its budget is permanently dead. When enough pods are permanently dead that the PCLQ cannot reach `completions`, the PCLQ is marked Failed.
- **PCLQ → PCSG**: a failed PCLQ replica counts against its PCSG replica's budget. When enough PCSG replicas are permanently Failed that the PCSG cannot reach `completions`, the PCSG is marked Failed.
- **PCLQ/PCSG → PCS**: a failed constituent triggers deletion and recreation of the full PCS replica, consuming one restart from that replica's PCS-level budget.
