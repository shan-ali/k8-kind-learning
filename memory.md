# memory <!-- omit from toc -->

Running log for this repo: decisions made and why, plus concepts covered along the way. Update as the project grows.

## Table of Contents <!-- omit from toc -->
- [Goal \& Working Style](#goal--working-style)
- [Decision Log](#decision-log)
- [Concepts Learned](#concepts-learned)

## Goal & Working Style

Learning Kubernetes hands-on using kind, working through a basic postgres deployment as the vehicle and continuing to "proof" it (harden it against realistic failure modes) rather than just getting it running once.

Working style: guided, one step at a time. Explain the concept and what needs to change, then I write the YAML myself. No complete/ready-to-paste manifests, no dumping every fix at once — surface one issue, let me act on it, then move to the next. Reviewing config after I've written it is welcome.

## Decision Log

- **Deployment, not StatefulSet, for postgres (initial choice)** — started with a Deployment for a single postgres pod. Simpler object, fine for a single non-HA instance.
- **Added a PVC** — original setup had no volume at all; postgres was writing to the pod's ephemeral container filesystem, so any pod delete/restart wiped the database. Added `postgres-pvc` (`ReadWriteOnce`, 1Gi, default `standard` StorageClass) and wired it into the Deployment via `volumes` + `volumeMounts` at `/var/lib/postgresql/data`. Verified: created data, deleted the pod, data survived.
- **`strategy.type: Recreate`** — default Deployment rollout is `RollingUpdate`, which starts the new pod before killing the old one. On a single-node kind cluster both pods land on the same node and can both mount the same RWO volume at once. Tested this directly: a `RollingUpdate` rollout produced a new pod that had to run crash recovery (`database system was not properly shut down`) because the old postgres process hadn't cleanly shut down before the new one grabbed the data directory. Switched to `Recreate` (old pod fully terminates before the new one is created) and re-verified: clean `database system was shut down` log line, no recovery needed.
- **StatefulSet conversion — decided, in progress.** No real data in this environment, so the migration cost that made this "wait" is moot; doing it purely to learn the idiom (the `Recreate` fix already closed the race condition that would have otherwise motivated it). Scaffold started at `apps/postgres-deployment-statefulset.yaml` (currently just a copy of the Deployment — not yet converted to `kind: StatefulSet`). Plan, documented in [postgres.md § Using a StatefulSet](docs/postgres/postgres.md#using-a-statefulset):
  - Standalone PVC + `volumes` block → `volumeClaimTemplates` (auto-creates one PVC per replica, named `<template-name>-<statefulset-name>-<ordinal>`).
  - Add a second, headless Service (`clusterIP: None`) referenced via `serviceName` — existing NodePort Service stays as-is, both select `app: postgres`.
  - Drop `strategy: Recreate` entirely — StatefulSet has no `Recreate` option (only `RollingUpdate`/`OnDelete`); at `replicas: 1`, `RollingUpdate`'s ordering rule reproduces the same guarantee. Still need to verify this by testing a rollout restart and checking for a clean shutdown log line, same as was done for `Recreate` on the Deployment.
  - This project only ever needs `replicas: 1` — a StatefulSet does not give Postgres replication/HA on its own; bumping `replicas` with the vanilla image just produces N independent empty databases, not a cluster (needs an operator like CloudNativePG for that).
- **Password currently in plaintext** in the Deployment manifest (`POSTGRES_PASSWORD`). Not yet moved to a Secret.

## Concepts Learned

- **PV vs PVC** — a PersistentVolume is the actual storage (the parking spot); a PersistentVolumeClaim is a request for storage (the ticket). Pods only ever reference the PVC, never the PV directly — that indirection is what lets the same pod spec work regardless of the underlying storage backend.
- **`WaitForFirstConsumer`** — this cluster's default StorageClass won't bind a PVC to a volume until a pod actually claims it. A PVC sitting `Pending` with nothing consuming it is expected, not broken.
- **Pod vs container networking** — containers within a pod share one network namespace (one IP, one port space), same as multiple processes on one host. There's no "routing to a container" step inside a pod; whichever process is bound to a port receives traffic there. `containerPort` in YAML is mostly documentation — it doesn't create a routing rule by itself.
- **Service port fields** — `targetPort` (on the Service) routes to the Pod's IP on that port number; it has to match a `containerPort` a container is actually listening on. `nodePort` is opened on every node in the cluster, which is why nodePort values must be unique cluster-wide (a physical one-listener-per-port-per-host constraint, not a selector issue). `selector` is a separate concern — it's how the Service decides which pods' IPs to route to, matched by labels (can match multiple pods, not "the pod").
- **kind node = container, not two layers** — unlike Pod → container (a real two-object nesting), a kind "Node" and its backing Docker container are the same object viewed from two vocabularies (Kubernetes calls it a Node, Docker calls it a container). Two nodes in a kind config means two separate Docker containers. Consequently `nodePort` (k8s) and kind's `containerPort` (the target of the `hostPort` mapping) are the same port in the same network namespace — not two ports needing translation.
- **`kubectl rollout restart`** — patches a restart-timestamp annotation onto the pod template, which changes its hash and triggers the same rollout mechanism as any other template edit, following whatever `strategy` is configured.
- **`RollingUpdate` vs `Recreate`** — RollingUpdate creates the new pod before killing the old one (zero downtime, but risks two pods touching the same RWO volume on a single-node cluster). Recreate tears down the old pod fully before creating the new one (brief downtime, but no overlap) — the safer default for a single-replica stateful workload.
- **StatefulSet vs Deployment** — stable pod identity (`<name>-<ordinal>`, e.g. `postgres-0`) that persists across restarts/scaling, vs. a Deployment's random hash suffix. Pods are created/scaled one at a time in order by default (`podManagementPolicy: OrderedReady`) — each must be Running and Ready before the next starts; scale-down reverses the order.
- **StatefulSet requires a headless governing Service** — set via `serviceName` on the StatefulSet, pointing at a Service with `clusterIP: None`. This doesn't load-balance; it makes each pod resolvable individually via DNS (`postgres-0.<serviceName>.<namespace>.svc.cluster.local`) — that per-pod addressability is the actual point of the object, not just naming cosmetics. A StatefulSet's headless Service and a regular Service (e.g. for NodePort) can coexist, both selecting the same pods for different jobs.
- **StatefulSet has no `Recreate` strategy** — `updateStrategy.type` only accepts `RollingUpdate` (default) or `OnDelete`. At `replicas: 1`, `RollingUpdate`'s own ordering guarantee (old pod fully terminated before the new one is created) reproduces what `Recreate` gives a Deployment — via ordering semantics, not an explicit setting.
- **PVCs from `volumeClaimTemplates` aren't deleted automatically** on scale-down or StatefulSet deletion (same orphan-by-default behavior as a standalone PVC). Since Kubernetes 1.27, `persistentVolumeClaimRetentionPolicy` can opt into automatic deletion on scale-down/whenScaled or delete/whenDeleted.
