---
title: "Technical proposal: Loki logging decisions for OAC clusters"
---

**Status:** Proposed. Nothing in this proposal is decided yet. It is tracked as MOC-issues 633,
written 2026-10-21. The first implementation, in https://github.com/schwesig/OA_oac-apps/pull/2
(open, not merged), is described here as the current state, not as a decision. This document
moves to Accepted when the team has reviewed it and each of the seven decisions below records the
option chosen.

## Context

oac-apps has no logging stack today. The first implementation,
https://github.com/schwesig/OA_oac-apps/pull/2, runs one LokiStack on each logging-enabled
workload cluster. It puts the chunks in a Pure FlashBlade S3 bucket through COSI and the PVCs on
FlashBlade NFS. A ClusterLogForwarder collects application logs, infrastructure logs and node
audit, and the Logging console plugin lets project members read their own namespaces' logs. The
hosted control plane API audit is collected on the hub. Retention is 15 / 15 / 90 days, and dev
runs size `1x.demo`. Two of those choices depart from Red Hat support statements, and the rest
were made without the team looking at the alternatives. This proposal lays out each decision so
the team can make it on purpose.

Versions in use: OpenShift Container Platform 4.22, Red Hat OpenShift Logging 6.6 (channel
`stable-6.6`), Red Hat Advanced Cluster Management 2.17.

The facts that constrain every option:

- **Hosted control planes split the logs.** A workload cluster's kube-apiserver,
  openshift-apiserver, oauth-apiserver and oauth-openshift run in `clusters-<name>` on the infra
  hub, as [hosted control planes][hcp] do. Their audit log only exists there, written to stdout
  by an `audit-logs` sidecar. Red Hat does not document that sidecar; we read it in
  `openshift/hypershift` on `release-4.22`. The workload cluster's own nodes only have auditd and
  OVN audit.
- **Reads are authorized by the cluster that hosts the store.** In `openshift-logging` tenancy
  mode a LokiStack checks every read against [its own cluster's RBAC][access]. Project members
  are namespaces on their workload cluster and have no access on the hub.
- **Only the infra hubs have block storage.** `oac-dev-infra` and `oac-prod-infra` run LVMS
  (`lvms-vg0`) on local SSDs, and hosted control plane etcd already uses that class. Every
  workload cluster, and `nist-dev-infra`, has only FlashBlade NFS (`pure-fb-nfsv4` through PX-CSI
  `pure_file`; `nist-dev-infra` defaults to an NFS-over-TLS class). There is no FlashArray, so no
  Pure block class is possible.
- **Object storage comes from FlashBlade buckets through COSI** (`fb-bucket`), the way ACM
  observability already gets its Thanos bucket. ACM observability also runs its Thanos PVCs on
  `pure-fb-nfsv4` (`charts/acm-observability/templates/multiclusterobservability.yaml` in
  oac-apps).
- **Every ingester holds up to about two hours of logs that are not yet flushed.** That is
  `max_chunk_age: 2h` in the Loki configuration the operator renders
  ([`openshift/loki` `release-6.6`][lokicfg]). The data sits in the write-ahead log on the
  ingester PVC. With one LokiStack per cluster, each cluster has its own two-hour window.

## Decision

The decision is pending. This proposal asks the team for seven decisions: six are below, and the
seventh is the answers to the questions under Open points. Each one has options, trade-offs and
a recommendation, and the recommendations are proposals only. Some of the decisions depend on
each other: the answer to decision 2 changes decisions 1 and 5.

### 1. Storage for LokiStack volumes

The chunks go to object storage whatever we choose here. This decision is about the PVCs for the
write-ahead log, the index and the compactor.

OpenShift Container Platform 4.22 [says][nfs]: "Using NFS storage as a persistent volume or
through NAS, such as Gluster, can corrupt the data. Therefore, NFS is not supported for
Elasticsearch storage and LokiStack log store in OpenShift Container Platform Logging." Logging
6.6 [requires][sc] "a StorageClass that supports dynamic provisioning of block storage
(ReadWriteOnce)".

| Option | For | Against |
|---|---|---|
| **FlashBlade NFS** (`pure-fb-nfsv4`, as implemented) | Exists on every cluster. "Pure for all observability", the same as ACM observability | Not supported by Red Hat, with a data-corruption warning. With `1x.demo` there is one ingester and no replication, so a damaged PVC loses up to two hours of that cluster's logs |
| **LVMS `lvms-vg0` on the infra hubs** | Supported block storage. Already in use. Moving the hub's stack is a one-value change | Only on the hubs. LVMS [does not replicate data between nodes][lvms], so it relies on Loki's own replication factor 2, which `1x.demo` does not have. Shares the SSDs that the hosted control plane docs ask to be [dedicated to hosted etcd][etcd] |
| **Block storage per workload cluster** (LVMS on the workers' local disks, or a FlashArray) | Keeps everything local and supportable | Does not exist. LVMS inside a hosted cluster is not covered by the hosted control plane docs we read, and Loki would compete with tenant workloads for those disks |

**Recommendation:** move the hub's LokiStack to `lvms-vg0` now. It holds the only copy of the
hosted API audit. On the workload clusters, run NFS for dev only, written down as an accepted
unsupported configuration, and require block storage, or the central option in decision 2,
before prod.

### 2. Placement

| Option | For | Against |
|---|---|---|
| **A. One LokiStack per workload cluster** (as implemented) | The only option that gives project members their own logs in the console (Observe → Logs): reads are checked against the workload cluster's RBAC, where the projects are. A failure loses only that cluster's logs. No cross-cluster plumbing | N stacks on tenant compute, N two-hour windows, all on NFS (decision 1). The dev workload clusters' object storage path has to be fixed first (https://github.com/schwesig/OA_oac-apps/issues/1) |
| **B. One central LokiStack per hub**, with forwarder-only workload clusters | Supported block storage (`lvms-vg0`). One window, replicated at `1x.pico` and above. Reuses the hub bucket that already works. The Logging 6.6 planning guide lists ["Forward logs to central cluster with LokiStack"][scenarios] | **Project members lose self-service.** The console plugin only reaches a LokiStack [in its own cluster][coo], and the hub checks reads by namespace name only, so `project-x` on two clusters cannot be told apart. The store is admin-only. The `lokiStack` output [only reaches an in-cluster stack][outputs], so each workload cluster needs three [`loki` outputs][loki] (one per tenant) to the hub gateway Route, a per-cluster writer token issued on the hub, and a cluster label on every stream from day one. Loki's WAL on the hub's etcd SSDs. Collectors on every cluster lose their target together during a hub or network outage |
| **C. Hybrid**: the hub's stack on `lvms-vg0` now, and the workload clusters per cluster as in A | The hub part becomes supportable at once. Project members keep their view. Can be staged, and an operator copy of infrastructure and audit logs to the hub can be added later | The workload stacks stay on NFS until block storage exists (decision 1) |

ACM 2.17 is no shortcut. Its [observability guide][mcoa] documents the multicluster observability
add-on for metrics only and does not mention Loki or ClusterLogForwarder. The upstream `logs`
field in `release-2.17` is undocumented, and the Red Hat feature request for multicluster Loki
through ACM (OBSDA-393) is still in progress.

**Recommendation:** C, the hybrid, if project members' own logs in the console stay a
requirement. If the team can drop or defer that requirement, B, after the network, hub capacity
and Red Hat KB questions in Open points have answers.

### 3. Retention per tenant

Logging 6.6 [says][retention]: "The maximum supported retention period is 30 days." Retention is
set per tenant (application, infrastructure, audit), with stream rules for finer control. The
implementation sets 15 / 15 / 90 days, plus a 90-day stream rule for the hosted API audit, which
arrives in the hub's application tenant.

| Option | For | Against |
|---|---|---|
| **15 / 15 / 90** (as implemented) | One place to search audit for 90 days | Audit is three times the supported maximum, and Loki is not immutable: Logging is ["not a Security Information and Event Monitoring (SIEM) system and does not provide secure storage for audit logs"][siem] |
| **15 / 15 / 30** | Within support. Loki stays the fast search tool for recent logs | Anything older than 30 days needs decision 4 |
| **30 / 30 / 30** | Within support, the most history in the console | Twice the bucket for application and infrastructure logs, for little troubleshooting value |

**Recommendation:** 15 / 15 / 30 days, with the hosted API audit stream rule at 30 days as well,
and anything longer handled by decision 4.

### 4. Long-term audit retention beyond 30 days

Logging 6.6's [storage design][short-term] says: "For long-term storage, compliance archives, or
queries spanning more than 30 days, forward logs to an external log management platform". It
has a GA [`s3` output for S3-compatible storage][s3], with the documented use case "immutable
long-term retention of audit logs" ([output reference][outputs]).

The proposed shape is a second ClusterLogForwarder output of type `s3`, fed by the audit inputs
(node audit on the workload clusters, hosted API audit on the hub), writing to a FlashBlade
bucket with S3 Object Lock in the same realm. The output writes gzipped JSON objects of up to
10 MB or five minutes each. The points to decide:

- **Retention period.** 90 days, a year, or longer. That is for the compliance owner to name.
- **Lock mode.** Compliance mode cannot be shortened by anyone, including the array admin, so a
  wrong value is permanent. Governance mode lets privileged users delete early.
- **Provisioning.** pure-cosi `v0.1.1` cannot create a locked bucket: it ignores BucketClass
  parameters, deletes buckets with their claims, and its access key includes delete and read.
  The archive bucket and its keys would be provisioned outside the cluster, with the storage
  team's Ansible, and the key delivered through AWS Secrets Manager and an ExternalSecret.
- **Access.** A put-only writer key in the cluster. A read-only auditor key that stays off the
  cluster. Lock settings administered by the storage team only.
- **Offline retrieval.** There is no query UI. Audits use an offline procedure: list the
  objects, download, filter (`jq`, DuckDB), or replay them into a scratch Loki for a large
  investigation. That procedure has to be written and tested once.
- **Front-end loss window.** Without `deliveryMode: AtLeastOnce` the collector buffers in memory,
  and the docs warn: ["Buffered data is lost on process termination or failure"][tuning]. With
  it, a disk buffer survives restarts and may duplicate records. Events not yet written out when a node
  fails can still be lost. Whether that window is acceptable is a compliance call.
- **One failure domain.** The same FlashBlade holds the Loki chunks and the archive. A second
  copy needs FlashBlade replication combined with Object Lock.

Other options: a SIEM (Splunk, Elasticsearch, an external Loki), which does not exist today and
complements an archive rather than replacing it; or a self-run Vector or OpenTelemetry Collector
in front of S3, which only buys date-partitioned keys and adds a component we would run.

**Recommendation:** the `s3` output to a per-realm Object Lock bucket, in governance mode on dev
to rehearse and compliance mode for the compliant environment, with the retention period from
the compliance owner. Start with the dev hub's hosted API audit, where the S3 endpoint already
works.

### 5. Sizing

| Size | Ingest per day | Replication | CPU and memory requests | PVCs |
|---|---|---|---|---|
| `1x.demo` | "Demo use only" | none | none | 40Gi |
| `1x.pico` | 50 GB | 2 | 7 vCPU, 17Gi | 590Gi |
| `1x.extra-small` | 100 GB | 2 | 14 vCPU, 31Gi | 590Gi |
| `1x.small` | 500 GB | 2 | not checked | 590Gi |

Ingest and requests are from the Logging 6.6 [sizing table][sizing]. The PVC split is from the
operator's [`sizes.go` on `release-6.6`][sizes]. `1x.demo` is one ingester with 10Gi of data and
10Gi of WAL, plus a 10Gi compactor and a 10Gi index gateway. Every production size is three
ingesters with 10Gi of data and 150Gi of WAL each, two 50Gi index gateways and a 10Gi compactor.
Adding the ruler (two more 160Gi volumes) makes 910Gi. For production, Logging 6.6 names
["1x.extra-small, 1x.small, or 1x.medium"][prodsize] as the supported sizes.

The bucket comes on top: about daily compressed ingest times retention days. Nothing is measured
yet, because no cluster has logging switched on.

What that means in storage:

- **Per cluster (option A or C):** 40Gi with `1x.demo`, 590Gi with any production size. Three
  dev workload clusters and the dev hub at a production size come to about 2.4Ti, all on NFS
  except the hub's.
- **Central (option B):** 590Gi per hub on `lvms-vg0`, about 160 to 220Gi per hub node, next to
  hosted etcd. Free space on the hub volume groups is not verified.

**Recommendation:** `1x.demo` on dev only, and switch it on for about a week to measure ingest
per tenant and bucket growth. Size each production stack from those numbers, at `1x.extra-small`
or larger, so that it runs with replication.

### 6. Separation of dev, prod and compliant environments

Never one LokiStack across dev, prod and the compliant environment. A comment on MOC-issues 605
puts it as: prod/staging logging and dev logging "can't be centralized together when we have
compliant clusters". Whatever placement wins in decision 2, the boundary is the environment:
`oac-dev-infra` and its workload clusters, `oac-prod-infra` and its workload clusters, and
`nist-dev-infra` on its own. Each has its own buckets and, for decision 4, its own archive and
keys. `nist-dev-infra` has no block storage, logging, COSI overlay or object storage proxy in
oac-apps today. Its logging design is a separate piece of work that depends on the compliance
answers.

**Recommendation:** record this as a standing rule.

## Consequences

If the recommendations are adopted:

- The hub's stack, the only copy of the hosted API audit, moves to supported storage. The
  workload stacks run knowingly unsupported on NFS in dev, and prod waits for block storage or
  for a decision to centralize.
- Project members keep their logs in the workload cluster's console. An admin investigating one
  incident uses two consoles: the workload cluster's for application, infrastructure and node
  logs, and the hub's for API audit.
- Loki holds 30 days at most. Audit beyond that lives in a locked bucket that is not searchable
  in the console, so the offline retrieval procedure becomes part of operating the platform.
- Work outside oac-apps: the storage team provisions the archive buckets and keys and asks Pure
  Technical Services to enable per-bucket SafeMode.
- The hosted API audit path relies on an undocumented sidecar, so it has to be rechecked after
  every hosted control plane minor upgrade.
- The implementation in https://github.com/schwesig/OA_oac-apps/pull/2 needs value changes for
  storage class, retention and size. Its own defects (LokiStack `adminGroups`, S3
  `forcepathstyle`) are tracked there.

## Open points

These are the questions for the storage, network and compliance owners. Several of the decisions
above wait on them.

**Storage**

1. How much space is free in `vg0` on each hub node, and can Loki get its own device or device
   class so that it does not share the hosted etcd SSDs?
2. Do the workload clusters' workers have spare local disks, and is a FlashArray planned
   anywhere?
3. Which Purity//FB version runs on the array? Object Lock needs 4.1 or later. Can per-bucket
   Object SafeMode be enabled, and per realm?
4. Can the realm service account that pure-cosi uses change Object Lock or retention settings on
   a bucket whose retention lock is ratcheted?
5. Does lifecycle expiry wait until an object's lock has run out, or is it rejected? What is the
   archive's capacity budget, and what happens at the quota?
6. Is a second copy of the archive required?

**Network**

7. Can every workload cluster's node network reach the hub's default ingress on port 443, and how
   much log volume is acceptable on that path? This only matters for option B or for an
   operator copy.
8. Can every cluster reach its realm's S3 endpoint for the archive, or does one central audit
   realm need new routes?

**Compliance**

9. Which control set applies, and which retention period for audit logs?
10. Is compliance-mode WORM required, or is governance mode acceptable?
11. Which audit sources are in scope: node auditd and OVN, the hosted API audit, and the API audit
    of standalone clusters such as `nist-dev-infra`?
12. Who may read the archive, and must reads be logged?
13. Is an offline retrieval procedure acceptable for audits?
14. Is a short loss window at the front end acceptable, or must every audit event be delivered?
15. Is a SIEM required eventually, or are retention and tamper resistance enough?

**Platform**

16. Someone with a Red Hat portal login should read KB 7017738 on forwarding to a LokiStack in
    another cluster. It is the only Red Hat procedure for option B.
17. How much CPU and memory headroom do the compact dev hub and the prod hub have?

## See also

- MOC-issues 633, which tracks this proposal, and MOC-issues 605, on centralized logging for
  compliant clusters.
- https://github.com/schwesig/OA_oac-apps/pull/2, the first implementation, and
  https://github.com/schwesig/OA_oac-apps/issues/1, the object storage prerequisites on the dev
  workload clusters.
- The per-project metrics ADR proposed upstream in the CCI-MOC ops-docs repository, pull request 37
  (not yet merged), which makes the same separation by namespace RBAC for metrics.
- Red Hat OpenShift Logging 6.6, OpenShift Container Platform 4.22 and ACM 2.17 documentation,
  linked below.

[hcp]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/hosted_control_planes/index#hcp-bm-hc_hcp-deploy-bm
[access]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/secure/index#logging-loki-log-access_control-access-to-loki-logs
[lokicfg]: https://github.com/openshift/loki/blob/release-6.6/operator/internal/manifests/internal/config/loki-config.yaml
[nfs]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/scalability_and_performance/index#recommended-configurable-storage-technology_persistent-storage
[sc]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/install/index#verifying-default-storageclass_verify-cluster-prerequisites
[lvms]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/storage/index#lvms-about-lvm-storage-installation_logical-volume-manager-storage
[etcd]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/hosted_control_planes/index#hcp-bm-prepare_hcp-deploy-bm
[scenarios]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/plan/index#logging-deployment-scenarios_plan-your-logging-installation
[coo]: https://github.com/rhobs/observability-operator/blob/release-coo-1.5/pkg/controllers/uiplugin/logging.go
[outputs]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/configure/index#clf-outputs-reference_configure-outputs
[loki]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/integrate/index#cluster-logging-collector-log-forward-loki_forward-to-third-party-systems
[mcoa]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/observability/index#enable-mcoa
[retention]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/optimize/index#logging-loki-retention_enhanced-reliability-performance-for-loki
[siem]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/configure/index#adding-audit-log-collection_configure-inputs
[short-term]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/discover/index#short-term-storage-design_loki-log-store
[s3]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/integrate/index#forward-to-s3-compatible-object-storage
[tuning]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/optimize/index#tuning-outputs_fine-tune-log-forwarding
[sizing]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/plan/index#loki-sizing_size-your-lokistack-deployment
[sizes]: https://github.com/openshift/loki/blob/release-6.6/operator/internal/manifests/internal/sizes.go
[prodsize]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/configure/index#logging-create-loki-cr-console_create-the-lokistack
