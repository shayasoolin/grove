# Job Support MVP

This document summarizes the MVP for adding finite job support to Grove.

## MVP Principles

- **Hierarchical evaluation.** Completion and failure are defined bottom-up: at the pod level first, then PodClique, PodCliqueScalingGroup, and PodCliqueSet. Each level evaluates its own phase solely from the observed state of its direct children.

- **Failure means completion is unreachable.** A resource is marked `Failed` when enough children have permanently failed that the completion target can no longer be met, regardless of how any remaining children resolve. Once `Failed`, a resource cannot subsequently become `Completed`.

- **Phases propagate upward.** When a child reaches `Completed` or `Failed`, its parent re-evaluates its own phase. A child failure that makes the parent's completion target unreachable marks the parent `Failed`, which triggers a restart attempt at the next level up if retry budget remains.

- **Job mode is signaled by `restartPolicy`.** A PodClique is in job mode when its pod template spec sets `restartPolicy: Never`. With `restartPolicy: Never`, pods reach a terminal phase on exit — `Succeeded` on code 0, `Failed` on non-zero — and Grove owns all completion and restart decisions. `restartPolicy: Always` keeps the PodClique in the regular long-running Grove model. `restartPolicy: OnFailure` is not permitted. A PCSG or PCS is in job mode when at least one of its children is in job mode.

- **No pod-level retry.** A PodClique does not retry failed pods. Any required pod failure immediately marks the PodClique `Failed`. The design targets tightly coupled workloads where all ranks must be present for the job to make progress: a single worker failure stalls all others at the collective barrier, so replacing one pod in isolation cannot recover the job. The meaningful recovery unit is the gang. Retry budget lives at the parent level (PCSG or PCS), which owns the gang restart scope.

- **Terminal states are persisted before cleanup.** Phase and conditions are written to status before any pods are deleted. When a resource reaches a terminal outcome, active pods are terminated to free their allocated resources. Terminal pods (`Succeeded`, `Failed`) are retained by default and remain available for `kubectl logs` until the workload is deleted.

## Minimal API

The MVP API adds a small set of fields directly under `spec`:

- `completedNames` narrows completion to specific named children.
- `maxRestarts` bounds parent-level gang retries.
- `maxRuntime` sets a hard wall-clock deadline.

The exact placement follows the resource where the field is configured, but the
fields are direct children of that resource's `spec`.

| Field | PCLQ | PCSG | PCS |
|---|---|---|---|
| `completedNames` | Not applicable. | Required PCLQ names inside each completed PCSG replica. If omitted, all job-mode children must complete. | Required child PCLQ/PCSG names inside each completed PCS replica. If omitted, all job-mode children must complete. |
| `maxRestarts` | Not supported. | Per-PCSG-replica restart budget. | Per-PCS-replica restart budget. |
| `maxRuntime` | PCLQ deadline. | PCSG deadline. | PCS deadline. |

When a PCS or PCSG evaluates children, non-job-mode PCLQs (`restartPolicy: Always`)
are excluded from completion calculation. A PCSG or PCS completes when all of its
job-mode children complete. For example, if a PCS has three child PCLQs — two in job
mode that complete successfully, and one in regular long-running mode — the PCS can
still be considered complete.

`maxRestarts` is intentionally absent from PCLQ. In MVP, a required pod failure
fails the PCLQ immediately. Retry budget lives at the parent level that owns the
gang restart scope: PCSG for PCLQs inside a PCSG, PCS for standalone PCLQs.

`maxRuntime` is measured from first start at the level where it is set and is not
reset by restarts. Exceeding it marks that whole resource `Failed` immediately
and permanently.

For the common all-ranks case, set `restartPolicy: Never` on the PCLQ pod template:

```yaml
spec:
  replicas: 1
  maxRestarts: 3
  maxRuntime: 24h
  template:
    cliques:
    - name: trainer
      spec:
        replicas: 8
        template:
          spec:
            restartPolicy: Never
```

For leader-driven completion, prefer named cliques and `completedNames`:

```yaml
spec:
  replicas: 1
  maxRuntime: 24h
  template:
    cliques:
    - name: leader
      spec:
        replicas: 1
        template:
          spec:
            restartPolicy: Never
    - name: worker
      spec:
        replicas: 7
        # restartPolicy: Never  # set this if worker failures should also fail the PCLQ
    podCliqueScalingGroups:
    - name: trainer
      cliqueNames: [leader, worker]
      completedNames: [leader]
      maxRestarts: 3
```

The worker PCLQ has two valid options. With `restartPolicy: Always` (default), the
worker PCLQ stays in regular long-running mode: Grove does not track worker pod
failures, and worker completion is not evaluated. The PCSG completes when the leader
PCLQ completes. With `restartPolicy: Never`, the worker PCLQ is also in job mode:
a worker pod failure fails the worker PCLQ, which fails the PCSG replica and
consumes one restart from the PCSG's budget. Use `restartPolicy: Never` on workers
when you want failures caught and retried; use `restartPolicy: Always` when workers
are disposable and only the leader's exit code matters.

## Gang Termination

`minAvailable` serves two distinct purposes in Grove: gang scheduling (gating pod
launch until the full gang can be placed simultaneously) and gang termination
(detecting when a running gang has lost enough members to warrant a restart). Gang
scheduling applies in both job mode and non-job mode. Gang termination differs:

**Non-job-mode resources** use `MinAvailableBreached` as the termination trigger. If a
PCSG replica or PCS replica remains below its availability threshold past
`terminationDelay`, Grove deletes and recreates the affected scope.

**Job-mode resources** use the `Failed` phase as the termination trigger.
`MinAvailableBreached` is not evaluated. There are two reasons for this:

- `minAvailable` semantics conflict with job-mode completion counting. Pods leaving
  the active set by completing successfully is expected in job mode; counting them
  against an availability threshold would produce spurious breach signals.
- Stuck-pending pods are covered by the failure cascade. If pods cannot be scheduled
  after a node eviction, the remaining running pods will encounter framework-level
  errors (rendezvous timeouts, broken collectives) and exit non-zero, driving the
  PCLQ to `Failed` through the normal path.

The `Failed` trigger fires immediately, without `terminationDelay`. Termination scope
is determined by where the failure is observed: a failed PCLQ inside a PCSG is handled
at the PCSG replica scope; a failed standalone PCLQ is handled at the PCS replica
scope. Any recreate consumes retry budget at the parent level.

## Cleanup Behavior

The MVP applies a single fixed cleanup policy — equivalent to `Running` — with no user choice. Active pods are deleted when a resource reaches a terminal state; terminal pods are retained:

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

- `restartPolicy: OnFailure` — not supported. Supporting it would require the
  controller to implement `maxRestarts` at the PCLQ level, analogous to `backoffLimit`
  in the Kubernetes Job API.
- PCLQ-level `maxRestarts`.
- K-of-N completion (`completions`) and index-based completion (`completedIndexes`).
  In the MVP, all job-mode children must complete. `completedIndexes` would additionally
  allow selecting specific pod indexes as eligible completers — useful when leader and
  workers share a single PCLQ, avoiding the need to split them into separate named
  cliques just to express a leader-driven success policy.
- Local single-pod replacement inside a running PodClique.
- Other pod cleanup policies such as `None` and `All`.
