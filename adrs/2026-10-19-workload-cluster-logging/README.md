---
title: Keep workload logs on the cluster that produces them, and hosted API audit on the hub
---

**Status:** Proposed. Decided 2026-10-19, implemented in
https://github.com/schwesig/OA_oac-apps/pull/2 (open, not merged), recorded 2026-10-20. It may
move to Accepted once both open points below are decided and the implementation has
merged into CCI-MOC/oac-apps.

## Context

oac-apps has no logging stack. A search of the repository at CCI-MOC/oac-apps@457b659 for Loki,
ClusterLogForwarder, cluster-logging, Vector and OpenTelemetry finds nothing. We want application,
infrastructure and audit logs for the workload clusters, dev first and prod later, without
blocking logging for the hubs themselves later on.

Three facts shape the design.

**HyperShift splits the logs before we do.** A workload cluster's kube-apiserver,
openshift-apiserver, oauth-apiserver and oauth-openshift run in `clusters-<name>` on the infra
hub, as [hosted control planes][hcp] do. Their audit log is produced there, by an `audit-logs`
sidecar that tails `audit.log` to stdout. It does not exist on the workload cluster, whose nodes
only have auditd and OVN audit. The sidecar is not described in the Red Hat documentation; we
read it off `openshift/hypershift` on `release-4.22`.

**Read access is authorized where the store runs.** A LokiStack in `openshift-logging` tenancy
mode checks every read against the RBAC of [the cluster that hosts it][access]. Tenants are
projects (`project-<name>`) that exist only on their workload cluster. On the hub they have
nothing to be authorized against.

**Storage on the workload clusters is Pure only.** The one StorageClass oac-apps creates there is
`pure-fb-nfsv4`, FlashBlade NFS through Portworx `pure_file`. There is no FlashArray and no block
class. Object storage comes from FlashBlade buckets through COSI (`fb-bucket`), the same way
[ACM observability][mco] already gets its Thanos bucket.

The per-project metrics ADR (https://github.com/CCI-MOC/ops-docs/pull/37) made the same call for
metrics: separate tenants by the workload cluster's own namespace RBAC.

## Decision

**Each logging-enabled workload cluster keeps its own logs, and the hub keeps the hosted API
audit it produces.**

On every workload cluster labelled `massopen.cloud/feature-logging` (Placement
`logging-clusters`):

- a LokiStack `openshift-logging/logging-loki`, with chunks in a COSI bucket on Pure FlashBlade
  S3 and its PVCs on `pure-fb-nfsv4`;
- a ClusterLogForwarder `collector` for application logs, infrastructure logs, and node audit
  (`auditd` and `ovn`);
- the `logging` [UIPlugin][uiplugin], which adds Observe → Logs to the console.

On the hub, a LokiStack from the same chart collects the `audit-logs` containers of
`clusters-<name>`. Prod is not labelled yet.

The supporting choices:

- **Tenants read their own projects' application logs.** A ClusterRole with the rule of
  `cluster-logging-application-view` aggregates into `view` and `edit`, so project members can
  read [their namespaces][ns] and nothing else. They never get the infrastructure or audit
  roles. `open-accelerator-admins` gets all three view roles. The implementing PR checked this
  with SubjectAccessReviews on a live cluster.
- **Hosted API audit stays on the hub.** It is stored where it is produced. There is no
  cross-cluster log traffic, no gateway on the public guest ingress and no token federation. Its
  readers are platform admins, who have hub access anyway. On the hub these logs land in the
  [application tenant][inputs], because `clusters-*` is not an infrastructure namespace. That is
  acceptable while only hub admins have access to those namespaces.
- **Retention is 15 / 15 / 90 days** for application, infrastructure and audit, set per tenant
  with [stream-based retention][retention]. A stream rule keeps the hub's `audit-logs` containers
  for 90 days, or they would expire as application logs after 15. See open point 2.
- **The LokiStack size has no default.** Dev runs `1x.demo`, which Red Hat labels ["Demo use
  only"][sizing], because the stack runs on tenant compute and dev-workload1 has two nodes. We
  size prod from about a week of measured ingest and bucket growth on dev, not from a guess.
- **Pure for all observability storage**: FlashBlade NFS for the Loki PVCs, a FlashBlade bucket
  for the chunks. See open point 1.
- **Operators on pinned channel `stable-6.6` with `installPlanApproval: Automatic`**, the same
  policy as every other operator in oac-apps. With Automatic approval an update [starts as soon
  as][operator] a new version is in the selected channel; pinning the channel keeps that within
  6.6.

### Alternatives considered

**B. One central LokiStack per hub**, with forwarders on the workload clusters. It is cheaper,
one stack per hub, and the hubs already have their object storage. It gives tenants nothing:
reads are authorized against the hub's RBAC, where tenants have no projects. It also needs a
gateway Route the workload nodes can reach, a hub token on every workload cluster, and a cluster
label on every stream from day one, because labels added later do not fix data already ingested.
The chart can still do B as a values change. We would reconsider it if tenant self-service stops
being a requirement.

**C. An external store** (a MOC-wide Loki, Splunk, a SIEM, S3). Nobody owns one today, and it
gives no view in the console. It remains the likely answer for long-term or compliance audit
(open point 2), as an additional output rather than a replacement.

**H2. Forward the hub's API audit into each workload cluster's LokiStack.** It would put all of a
cluster's logs in one console, at the cost of a gateway Route on the public guest ingress and
token plumbing, for readers who already have hub access.

**The documented HTTP receiver for hosted control plane audit.** Logging 6.6 [documents][configure]
a ClusterLogForwarder HTTP receiver (`kubeAPIAudit`) for hosted control planes, fed by
`HostedCluster.spec.auditWebhook`. That field is immutable in the HyperShift API, so using it means
recreating the existing hosted clusters. We would reconsider it for new clusters.

## Consequences

- Each LokiStack runs on tenant compute. On dev that is the demo size; prod will cost more, by an
  amount we will know after measuring.
- An admin looking into one incident uses two consoles: the workload cluster's for application,
  infrastructure and node logs, the hub's for API audit. A Grafana with several Loki datasources
  could join them later.
- Logging on the hub is a values change on the hub's slice. Central aggregation is additive, an
  extra output on the existing forwarders, and leaves local data in place.
- Dev workload clusters need their object storage path fixed first: dev-workload1 has no COSI
  overlay, and `object-storage-proxy` does not render for either dev workload cluster. That work
  is tracked separately.
- The bucket is deleted with its BucketClaim (`deletionPolicy: Delete`), so the claim and the
  `openshift-logging` Namespace carry `Prune=false,Delete=false`.
- The hosted API audit path relies on an undocumented sidecar. We recheck it after every hosted
  control plane minor upgrade.
- The project log role also renders on the hub, where `view` on a `clusters-<name>` namespace
  would grant that cluster's API audit. Nobody holds such a binding today. If non-admin namespace
  bindings are ever added on the hub, the role has to be switched off there.

## Open points

Points 1 and 2 each depart from a Red Hat support statement. We record the question here and do not
decide it. If a point is resolved by changing the configuration, the Decision changes before
acceptance and the rejected variant moves to Alternatives considered. If it is resolved by keeping
the configuration, Consequences gets a paragraph that names it as an accepted unsupported
configuration and quotes the statement.

1. **Loki PVCs on FlashBlade NFS.** OpenShift Container Platform 4.22 [says][nfs]: "Using NFS
   storage as a persistent volume or through NAS, such as Gluster, can corrupt the data.
   Therefore, NFS is not supported for Elasticsearch storage and LokiStack log store in OpenShift
   Container Platform Logging." The PVCs hold the WAL, the index cache and the compactor's
   working space; the chunks are in object storage either way. Logging 6.6 also
   [asks for][sc] "a StorageClass that supports dynamic provisioning of block storage
   (ReadWriteOnce)".

   The workload clusters have no block class and no FlashArray to provide one. The dev hub has
   LVMS (`lvms-vg0`), but the implementation uses `pure-fb-nfsv4` there too. ACM observability
   already runs its Thanos PVCs on `pure-fb-nfsv4` on the hubs, so the platform carries this
   pattern for metrics today. The options:

   - (a) add a block StorageClass for the Loki PVCs on the workload clusters (and use `lvms-vg0`
     on the hub), keeping the FlashBlade bucket for the chunks;
   - (b) keep NFS and record it as an accepted unsupported configuration, for dev only or for
     both tiers.

2. **90-day audit retention.** Logging 6.6 [says][retention]: "The maximum supported retention
   period is 30 days." Its [storage design][short-term] adds: "For long-term storage, compliance
   archives, or queries spanning more than 30 days, forward logs to an external log management
   platform". The options:

   - (a) keep audit in Loki for at most 30 days, and add an external copy through a second
     forwarder output if 90 days is a requirement;
   - (b) keep 90 days in Loki and record it as an accepted unsupported configuration.

   Which one fits depends on whether 90 days is a compliance requirement or a preference. If it
   is a requirement, alternative C comes back for audit.

3. Implementation defects that do not change the decision, such as the LokiStack `adminGroups`
   and the S3 `forcepathstyle` setting, are tracked in the implementing PR.

## See also

- https://github.com/schwesig/OA_oac-apps/pull/2, which implements this. Replace it with the
  upstream PR once there is one.
- https://github.com/CCI-MOC/ops-docs/pull/37, the same separation by namespace RBAC for metrics.
- https://github.com/CCI-MOC/ops-docs/pull/36, on central metric collection through ACM.
- Red Hat OpenShift Logging 6.6 and OpenShift Container Platform 4.22 documentation, linked
  below.

[hcp]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/hosted_control_planes/index#hcp-bm-hc_hcp-deploy-bm
[access]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/secure/index#logging-loki-log-access_control-access-to-loki-logs
[ns]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/secure/index#namespaced-access
[mco]: https://github.com/CCI-MOC/oac-apps/blob/457b659/charts/acm-observability/templates/multiclusterobservability.yaml
[uiplugin]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/install/index#enabling-the-logs-tab_enable-the-logs-tab-in-the-web-console
[inputs]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/configure/index#clf-inputs-reference_configure-inputs
[retention]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/optimize/index#logging-loki-retention_enhanced-reliability-performance-for-loki
[sizing]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/plan/index#loki-sizing_size-your-lokistack-deployment
[operator]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/install/index#install-the-loki-operator-cli_install-the-loki-operator
[configure]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/configure/index
[nfs]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/scalability_and_performance/index#recommended-configurable-storage-technology_persistent-storage
[sc]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/install/index#verifying-default-storageclass_verify-cluster-prerequisites
[short-term]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/discover/index#short-term-storage-design_loki-log-store
