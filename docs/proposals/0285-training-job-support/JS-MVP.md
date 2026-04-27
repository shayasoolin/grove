# Job Support MVP

This document summarizes the MVP for adding finite job support to Grove.

## Rationale

The MVP targets the most common distributed training shape: tightly coupled gangs.
In synchronous training modes such as DDP, FSDP, MPI, JAX multi-process training,
and most multi-worker TensorFlow setups, all required ranks must be present for
the workload to make progress. If one required pod fails, the useful recovery
action is to restart the gang from checkpoint, not to replace one pod in place.

Some jobs use a leader, chief, or launcher pod as the completion authority. In
those cases, success may be defined by a single pod or clique completing, while
workers are supporting processes. The MVP should support both common success
shapes:

- all required pods or components complete successfully;
- only selected named components complete successfully.

Failure handling remains gang-oriented in both cases: a required pod failure
fails its PodClique, and the parent restarts the appropriate gang scope if retry
budget remains.

## MVP Principles

- Job mode uses `restartPolicy: Never` for pods so Grove can observe terminal
  pod states and own restart decisions.
- A PodClique does not retry failed pods in MVP.
- Any required pod failure makes the PodClique `Failed`.
- Parent resources own retry budgets and restart scope.
- Terminal states are persisted in status before cleanup or parent deletion.
- Active pods are terminated when a resource reaches a terminal outcome.
- Terminal pods are retained by default on final completion or failure.

## Minimal API

The MVP API adds a small set of fields directly under `spec`:

- `completions` enables finite job behavior and defines how many children must
  complete.
- `completedNames` narrows completion to specific named children.
- `maxRestarts` bounds parent-level gang retries.
- `maxRuntime` sets a hard wall-clock deadline.

The exact placement follows the resource where the field is configured, but the
fields are direct children of that resource's `spec`.

| Field | PCLQ | PCSG | PCS |
|---|---|---|---|
| `completions` | Number of pods that must exit `0`. | Number of PCSG replicas that must complete. | Number of PCS replicas that must complete. |
| `completedNames` | Not applicable. | Required PCLQ names inside each completed PCSG replica. | Required child PCLQ/PCSG names inside each completed PCS replica. |
| `maxRestarts` | Not supported. | Per-PCSG-replica restart budget. | Per-PCS-replica restart budget. |
| `maxRuntime` | PCLQ deadline. | PCSG deadline. | PCS deadline. |

`completions` is the explicit success condition. It does not default to
`replicas`, because the existing Grove default is non-job-oriented and should
continue to run indefinitely unless the user opts in.

Whenever `completedNames` is used, `completions` must also be set on the same
resource. `completedNames` scopes which named children can satisfy completion;
`completions` defines how many replicas must complete. If `completedNames` is
omitted on a resource with `completions`, all children are required.

When a PCS or PCSG evaluates child PCLQs, PCLQs that are not in job mode are
excluded from completion calculation. A PCLQ is in job mode only when
`completions` is set on it. For example, if a PCS has three child PCLQs, two
complete successfully, and the third has no `completions` set and continues as a
regular long-running PCLQ with `restartPolicy: Always`, the PCS can still be
considered complete. In short: a resource completes when all of its job-mode
children complete.

`maxRestarts` is intentionally absent from PCLQ. In MVP, a required pod failure
fails the PCLQ immediately. Retry budget lives at the parent level that owns the
gang restart scope: PCSG for PCLQs inside a PCSG, PCS for standalone PCLQs.

`maxRuntime` is measured from first start at the level where it is set and is not
reset by restarts. Exceeding it marks that whole resource `Failed` immediately
and permanently.

For the common all-ranks case, set `completions` explicitly:

```yaml
spec:
  replicas: 1
  completions: 1
  maxRestarts: 3
  maxRuntime: 24h
  template:
    cliques:
    - name: trainer
      spec:
        replicas: 8
        completions: 8
```

For leader-driven completion, prefer named cliques and `completedNames`:

```yaml
spec:
  replicas: 1
  completions: 1
  maxRuntime: 24h
  template:
    cliques:
    - name: leader
      spec:
        replicas: 1
        completions: 1
    - name: worker
      spec:
        replicas: 7
    podCliqueScalingGroups:
    - name: trainer
      cliqueNames: [leader, worker]
      completions: 1
      completedNames: [leader]
      maxRestarts: 3
```

### Deferred: Index-Based Completion

`completedIndexes` was considered for homogeneous PCLQs where pod index `0`
acts as the leader. Rank or process `0` is often special in training frameworks,
but exposing pod indexes as completion API is low-level and not very semantic.

The Grove-native alternative is the leader-driven example above: split the
workload into named PodCliques and use `completedNames` at the parent level.

Pros:

- The API says what the user means: completion depends on `leader`, not index
  `0`.
- It uses existing Grove concepts: named cliques and PCSG replicas.
- It supports different leader and worker pod specs when needed.
- It keeps the MVP API smaller by avoiding index-selection semantics.

Cons:

- Users must map global training ranks themselves. For example, leader rank is
  `0`, worker rank is `1 + GROVE_PCLQ_POD_INDEX`.
- Users may duplicate pod spec fields if leader and workers run the same image
  with mostly the same configuration.
- Discovery is slightly more explicit: workers construct the leader address from
  `GROVE_PCSG_NAME`, `GROVE_PCSG_INDEX`, and the `leader` clique name.

MVP direction: prefer this named-clique pattern and remove or defer
`completedIndexes` unless a concrete use case requires homogeneous pods with
index-based completion.

## Restart Scope

PCLQ-level pod retry is intentionally excluded from the MVP.

When a pod fails:

1. PodClique marks itself `Failed`.
2. If the PodClique belongs to a PCSG, the PCSG handles the failed PCSG replica.
3. If the PodClique is standalone, the PCS handles the failed PCS replica.

This keeps retry accounting at the same level as gang restart scope.

## Gang Termination

Job failures should feed into Grove's existing gang-termination machinery rather
than introduce a separate restart path.

Today, gang termination is driven by `MinAvailableBreached`: if a PCSG replica
or PCS replica remains below its availability threshold past `terminationDelay`,
Grove deletes and recreates the affected gang scope.

In job mode, a child resource entering `Failed` is an additional trigger for the
same machinery:

- `MinAvailableBreached` still handles availability loss where pods are alive,
  pending, or missing but no terminal pod failure has been observed.
- `Failed` handles deliberate job failure, such as a required pod exiting
  non-zero or a completion target becoming unreachable.

The `Failed` trigger is immediate and does not wait for `terminationDelay`.
Termination scope stays unchanged: a failed PCLQ inside a PCSG is handled at the
PCSG replica scope; a failed standalone PCLQ is handled at the PCS replica scope.
Any recreate consumes retry budget at the parent level.

## Cleanup Behavior

No advanced cleanup policy is part of the MVP.

Default cleanup:

- When a PCLQ succeeds or fails, delete active pods in that PCLQ: `Pending`,
  `Running`, and other non-terminal pods.
- When a PCSG or PCS reaches a terminal state, delete active pods in its child
  scope as well. This covers leader-driven completion, where the leader PCLQ may
  finish while worker PCLQ pods are still running.
- Retain terminal pods by default: `Succeeded` and `Failed`.
- Persist phase and conditions before deleting active pods.

During a retry, the parent deletes the failed child resource or replica scope and
recreates it. That cascading delete may remove terminal pods from the failed
attempt. This is acceptable for the MVP; durable historical logs should come
from the cluster logging stack.

On final success or failure with no retry, terminal pods remain available for
`kubectl logs` and inspection until the workload is deleted by the user or a
future TTL policy.

## Out Of Scope

- PCLQ-level `maxRestarts`.
- `completedIndexes` / index-based completion selection.
- Local single-pod replacement inside a running PodClique.
- Elastic membership changes.
- Advanced pod cleanup policy such as `None`, `Running`, or `All`.
- TTL-after-finished cleanup.
- Framework-specific failure policies.
