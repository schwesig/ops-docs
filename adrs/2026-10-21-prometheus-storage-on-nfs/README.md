---
title: Run the Prometheus TSDB on FlashBlade NFS, knowingly
---

## Context

Every Prometheus on OAC/MOC 2.0 that keeps its data keeps it on NFS. There is no block storage class
on these clusters, and Prometheus itself prints a warning about that on every start. This ADR records
the choice rather than leaving it as a surprise for whoever reads the log next.

The warning, from `prometheus-k8s-0` on both hubs:

```
level=WARN msg="This filesystem is generally not recommended for Prometheus and may lead to data
corruption and data loss. Some NFS implementations may work if they are POSIX-compliant. Please
consult the documentation and carefully consider your NFS implementation" fs_type=NFS_SUPER_MAGIC
```

### What the clusters actually offer

Read off both hubs on 2026-10-21:

| Class | Provisioner | Mode | What it is |
|---|---|---|---|
| `pure-fb-nfsv4` (default) | `pxd.portworx.com` | Immediate | FlashBlade NFS through Portworx |
| `lvms-vg0` | `topolvm.io` | WaitForFirstConsumer | node-local LVM, `ext4` |

So the choice is not between NFS and block. It is between NFS and a node-local volume, and ACM's own
documentation says not to use local volumes for observability, because the data dies with the node
rather than with the pod. A real block class would have to be provisioned first, and nobody has.

The dev hub also still carries a third class, `pure-fb-nfsv4-no-snapshot`, 12 days old against the
default's 8. Its parameters are **identical** to the default's, including
`pure_fb_snapshot_directory_enabled: "false"`. It is a leftover from before that switch moved into
the default class and should be removed, prod never had it anyway.

### Why the snapshot directory matters

FlashBlade puts a read-only `.snapshot` directory into every export. Anything that walks or locks its
whole data directory on start trips over it, and that is what kept `thanos-receive` down **273
restarts**. The fix was the parameter above, which the default class now carries.

That history is the reason this ADR exists. The same substrate that broke one TSDB component now
carries four Prometheus instances, so "it works" needs to be written down with what was measured,
not assumed from the absence of crashes.

## Decision

**Prometheus and the user workload Prometheus keep their TSDB on `pure-fb-nfsv4`, the default class,
with no storage class named in our charts.** Leaving it unset is deliberate: the cluster's default is
the only sensible answer today, and naming it in a values file would have to be changed on every
cluster the day a block class apears.

Measured after enabling persistence on both hubs, 2026-10-21:

| | platform | user workload |
|---|---|---|
| volumes per hub | 2 × 32Gi Bound | 2 × 20Gi Bound |
| retention | `15d`, cap `25GiB` | `15d`, cap `18GiB` |
| access mode | ReadWriteOnce | ReadWriteOnce |
| mount options | `nfsvers=4.1`, `tcp` | `nfsvers=4.1`, `tcp` |
| pods after rollout | 6/6, 0 restarts | 6/6, 0 restarts |

The TSDB opened cleanly on every one of them. WAL replay took 14.5s on the dev hub and 22ms on prod,
with no lock error and no `resource busy`. The three workload clusters have been on the same class
since 2026-10-09 and 10-12 with 32Gi each.

Two properties of the mount are worth naming, because they are what makes the locking work:
`nfsvers=4.1` rather than v3, and no `nolock` in the options. NFSv4 does its locking in the protocol
instead of through a separate lock daemon. A pod mounting such an export reports
`local_lock=none`, which is what we want to see: locks go to the server, not to a local file that
another node cannot see.

### What we did not pick

**`lvms-vg0`.** Node-local, so a node going away takes the history with it, and ACM says not to use
it for observability. It would remove the NFS question entirely and replace it with a worse one.

**A real block class.** The right answer, and not available. Whoever provisions one should expect to
revisit this ADR rather than extend it.

**Keeping `emptyDir` and relying on the hub's MCO.** This was the state until 2026-10-21 and it is
not a neutral option. The hub's own metrics were lost on every pod restart, and MCO does not backfill:
the collector pushes the current value every 300s and gives up after about 40s of retries, so a gap
in the sender is a permanent gap in the receiver.

## Consequences

**Upstream does not support this, and that does not change because it works for us.** Prometheus'
own documentation is explicit about NFS, and the warning is printed on every start. If a TSDB does
corrupt, the first question from upstream will be the filesystem, and the answer will be NFS. We
carry that risk ourselves.

**ReadWriteOnce plus NFS has a rollout cost.** A single writer per volume means the pod has to go
before its replacement can mount, so every change that touches the stateful set is a short outage per
replica rather than a rolling overlap. In the measured rollouts that was about 90 seconds per
replica, taken one at a time. An NFS mount can also stall shutdown for up to 15 minutes if the server
stops answering, which turns the same step into something much longer.

**Thanos is a separate question and is not settled by this.** `thanos-receive`, `store` and `compact`
are the components upstream explicitly wants on block storage, and they are also the ones that
already failed here. Nothing in this ADR says they should run on NFS; it covers the Prometheus TSDB
only.

**The retention caps are what protect the volume, not the window.** `retentionSize` sits below the
volume size on purpose, `25GiB` against 32Gi and `18GiB` against 20Gi, so Prometheus drops the oldest
blocks before the filesystem fills. Watch `prometheus_tsdb_size_retentions_total`: if it climbs, the
cap is biting before the 15 days are up and that instance wants a bigger volume rather than a longer
window.

**The sizing rests on one measurement.** 2,901 bytes per series and day, taken from platform metrics
on `oac-dev-workload1` over 40 hours. That put the largest cluster, `oac-dev-workload0` at 489,601
series, at 19.8 GiB for 15 days, which is where 32Gi comes from. One cluster over fewer than two
days is a thin basis for a fleet-wide number.

## Open points

1. **`pure-fb-nfsv4-no-snapshot` on the dev hub is dead weight** and should be deleted, now that its
   parameters match the default class exactly. Removing a storage class needs a check that nothing
   references it first.
2. **No volume has been through a node failure or a FlashBlade outage.** Every measurement so far is
   a planned rollout. How a locked TSDB behaves when the server stops answering mid-write is the
   thing we do not know, and it is the thing the upstream warning is about.
3. **Nothing alerts on a Prometheus running without a volume.** Both hubs sat on `emptyDir` for
   months with Argo green throughout. A check that a persistent Prometheus actually has a
   `persistentVolumeClaim` would have caught it, and would catch the next cluster too.
4. **The 2,901 bytes per series and day wants a second measurement**, from a different cluster and a
   longer window, before more volumes are sized from it.
5. **Whether a block class is coming is unanswered.** If it is, the sizing and the access mode here
   are provisional. If it is not, this ADR is the long-term state and deserves the node-failure test
   in point 2.

## See also

- The ADR on per-project metric separation, for what reads these volumes and why the retention
  matters to a project rather than only to an operator
- CCI-MOC/oac-apps#130, which added the FlashBlade snapshot-directory switch after the
  `thanos-receive` restarts
- CCI-MOC/oac-apps#161, which gave the two hubs their volumes, and
  CCI-MOC/MOC-issues#630 for why the first sync after it did not take effect
- Prometheus' storage documentation, for the upstream position on NFS
