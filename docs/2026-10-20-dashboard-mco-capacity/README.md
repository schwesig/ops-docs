---
title: "Dashboard: MCO capacity and growth"
---

This document describes one dashboard as it exists in oac-apps
[`457b659`][main], 2026-10-20. It records why the dashboard exists and why
each metric is on it. The dashboard strategy (which tool for which audience,
naming, ownership) is the subject of a separate ADR,
`adrs/<date>-dashboard-strategy`, which is still pending.

## Question and audience

**Question:** how fast does the hub's ACM Observability (MCO) store grow, and
is it losing data on the way in?

**Audience:** hub operators. The dashboard queries hub-local metrics and needs
`cluster-monitoring-view`, which the `oac-simple-rbac` chart binds to the admin
group ([`allow-monitoring-view.yaml`][rbac]). It is not meant for projects.

**Why it was created**, from CCI-MOC/oac-apps#139 (merged 2026-10-20):

> Since MCO retains data now, how fast it grows is a question we can answer,
> and until now I measured it by hand each time. This puts it on a dashboard.

MCO started retaining data once Thanos receive ran on both hubs
(CCI-MOC/oac-apps#136, #137; CCI-MOC/MOC-issues#611).

## Tool and location

One dashboard with two renderings. Both carry the same four panels and the
same queries. They exist because the console has two dashboard lists.

| Rendering | Resource | Namespace | Where it shows up | Source |
|---|---|---|---|---|
| Perses | `PersesDashboard` `mco-capacity` (`perses.dev/v1alpha2`) | `openshift-cluster-observability-operator` | Observe > Dashboards (Perses), added by the COO Monitoring UI plugin | [`perses-dashboard-mco-capacity.yaml`][perses] |
| Console | ConfigMap `dashboard-mco-capacity`, label `console.openshift.io/dashboard: "true"` | `openshift-config-managed` | The older Observe > Dashboards list | [`console-dashboard-mco-capacity.yaml`][console] |

- **Chart:** `charts/cluster-observability-operator`. The values
  `capacityDashboard.perses.enabled` and `capacityDashboard.console.enabled`
  switch each rendering, and both default to `true` ([`values.yaml`][values]).
- **Deployed to:** every cluster that runs the chart. That is both hubs, through
  `hub-components`, and every managed cluster, through `all-managed-clusters`
  ([`hub-components.yaml`][hubappset], [`all-managed-clusters.yaml`][managedappset]).
  See open point 1.
- **Default time range:** 7 days.
- **Tests:** [`capacity_dashboard_test.yaml`][test] checks that both renderings
  appear by default, that the console label and namespace are set, and that the
  `acm_` prefix sits on the metric name and not on the `job` label.

Red Hat documents the Perses route: "You can create Perses dashboards by
defining PersesDashboard custom resources by enabling dashboard-as-code
practices and GitOps workflows" ([COO UI plugins, Perses dashboards][coo-crd]).
The ConfigMap route in `openshift-config-managed` is not described in the
OCP 4.22 monitoring guides we checked. The chart relies on observed console
behaviour (open point 3).

## Data source

The hub's own platform monitoring stack: `thanos-querier` in
`openshift-monitoring`. Not MCO's Thanos, and not the fleet data.

Why not MCO's own Grafana, from #139:

> That one has a single datasource, the rbac-query-proxy, and it serves fleet
> data. The tsdb metrics in these panels sit in the hub's own prometheus and
> come back empty over the proxy.
>
> A second datasource is not possible either: the `grafana-datasources` secret
> is owned by the `MultiClusterObservability` resource, `mco-operator` is the
> only manager on it, and the CRD has no field to declare one.

The COO documentation shows how a Perses datasource reaches "the platform
Thanos Querier instance run by the Cluster Monitoring Operator", on port 9091
([COO, global Thanos datasource][coo-ds]). Port 9091 "provides access to all
the Thanos Querier endpoints" ([OCP 4.22, Accessing metrics, services][cmo-svc]),
which is what a fleet-operator dashboard wants.

**The `acm_` prefix.** The metrics carry an `acm_` prefix on their name, but
the `job` label keeps the unprefixed value
`observability-thanos-receive-default`. From the chart comment: "ACM renames
the metrics it scrapes, so the name carries acm_ while the job label keeps the
real name." This is read off the cluster. We did not find it in the ACM 2.17
observability guide.

## Metrics

All four panels look at Thanos receive, the MCO component that ingests what the
managed clusters send. Two panels show growth and two show storage.

| Panel | PromQL (as in [`457b659`][main]) | Metric source | Needs the MCO allowlist? | Why it is there |
|---|---|---|---|---|
| Series held by receive | `sum(acm_prometheus_tsdb_head_series{job="observability-thanos-receive-default"})` | The TSDB of the Thanos receive pods in `open-cluster-management-observability`, scraped by the hub's platform Prometheus and renamed with `acm_` | No | How many active series receive holds, the size of the store. It is the first row of the hand measurements in #139 (dev hub 607,960, prod hub 342,205 on 2026-10-20) |
| Samples written per second | `sum(rate(acm_prometheus_tsdb_head_samples_appended_total{job="observability-thanos-receive-default"}[5m]))` | Same as above | No | Ingest rate, the second number from #139 (dev 6,546/s, prod 3,549/s) |
| Disk used per receive replica | `kubelet_volume_stats_used_bytes{namespace="open-cluster-management-observability",persistentvolumeclaim=~"data-observability-thanos-receive.*"}` | kubelet volume stats for the receive PVCs, a platform metric on the hub. One series per PVC | No | Shows when a receive volume fills up. A per-panel reason is not recorded beyond the "capacity and growth" purpose |
| Rejected samples, out of order or out of bounds | `sum(rate(acm_prometheus_tsdb_out_of_order_samples_total{job="observability-thanos-receive-default"}[5m]))` and `sum(rate(acm_prometheus_tsdb_out_of_bound_samples_total{job="observability-thanos-receive-default"}[5m]))` | Same TSDB as the first two panels | No | Data loss on ingest. Chart comment: "Receive has no out-of-order window, so anything counted here is data that never lands." |

The dashboard does not read these metrics through MCO. It queries the hub's
own `thanos-querier`, so whether the MCO allowlist names them makes no
difference here, and the allowlist decision in
https://github.com/CCI-MOC/ops-docs/pull/36 does not affect this dashboard.

The `[5m]` rate windows are under review: CCI-MOC/oac-apps#149 (open) found
that over a 14 day range the step grows past 5 minutes and the samples panel
"draws drops that are not in the data". It proposes `$__rate_interval` for
Perses and a fixed `30m` window for the console list, "because that one does
not resolve the variable". Update this table when #149 merges.

## Thresholds

None. The dashboard has no thresholds, colouring or linked alerts. The numbers
measured on 2026-10-20 in #139 are a reference point, not a limit:

| | dev hub | prod hub |
|---|---|---|
| series | 607,960 | 342,205 |
| samples/s | 6,546 | 3,549 |
| metric names | 223 | 177 |

Any value above zero in the rejected-samples panel means data that never lands.

## Dependencies

- **Cluster Observability Operator** with the Monitoring UI plugin and Perses
  enabled (`uiPlugin.perses.enabled: true`), for the Perses rendering
  ([`uiplugin.yaml`][uiplugin]).
- **ACM Observability (MCO) with Thanos receive running** on the cluster
  (`charts/acm-observability`, CCI-MOC/oac-apps#119). Without it the panels are
  empty.
- **`cluster-monitoring-view`** for the reader, or "the dashboards render but
  every panel says Forbidden" ([`allow-monitoring-view.yaml`][rbac],
  CCI-MOC/oac-apps#9).
- **MCO allowlist:** not needed (see Metrics).

## Owner

Not recorded in oac-apps. Both renderings were added in CCI-MOC/oac-apps#139.

## Open points

1. **Rendered on clusters without MCO.** The values comment says the dashboard
   is "Off for a cluster without MCO, where the panels stay empty". But both
   switches default to `true`, and no values file in oac-apps turns them off. So
   the managed clusters also get the dashboard, with empty panels. Either
   default it to off and enable it in the hub values, or record that empty
   panels on managed clusters are accepted.
2. **History depth on the hub.** The dashboard opens on 7 days and #149 talks
   about 14. The data comes from the hub's platform Prometheus. oac-apps sets no
   `cluster-monitoring-config` on the hubs (`user-workload-monitoring` is not in
   `hub-components`). So the hub's retention and whether it has a volume are not
   recorded. Without a volume, the history ends at the age of the pod.
3. **Console ConfigMap route is undocumented.** We found no OCP 4.22 Red Hat
   documentation for dashboards loaded from ConfigMaps in
   `openshift-config-managed`. It works today, but a console upgrade could
   change it without notice.
4. **Owner.** No owner is named. The pending strategy ADR is expected to decide
   who owns dashboards.

## See also

- `adrs/<date>-dashboard-strategy` (pending): which tool for which audience,
  dashboards as code, naming and ownership.
- https://github.com/CCI-MOC/ops-docs/pull/36: MCO metrics allowlist ADR.
- https://github.com/CCI-MOC/ops-docs/pull/37: per-project Grafana ADR,
  section "Perses stays", on why Perses is the tool for the operator view.
- [Dashboard: Project overview](../2026-10-20-dashboard-project-overview/), the
  other dashboard in oac-apps.
- CCI-MOC/oac-apps#139: the PR that added the dashboard, with screenshots of
  both renderings.
- CCI-MOC/oac-apps#149: rate window fix (open).

[main]: https://github.com/CCI-MOC/oac-apps/tree/457b659283f3f8da57bb44454628170058625a44
[perses]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/charts/cluster-observability-operator/templates/perses-dashboard-mco-capacity.yaml
[console]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/charts/cluster-observability-operator/templates/console-dashboard-mco-capacity.yaml
[values]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/charts/cluster-observability-operator/values.yaml
[uiplugin]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/charts/cluster-observability-operator/templates/uiplugin.yaml
[test]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/charts/cluster-observability-operator/tests/unit/capacity_dashboard_test.yaml
[hubappset]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/applicationsets/templates/hub/hub-components.yaml
[managedappset]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/applicationsets/templates/managed/all-managed-clusters.yaml
[rbac]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/charts/oac-simple-rbac/templates/allow-monitoring-view.yaml
[coo-crd]: https://docs.redhat.com/en/documentation/red_hat_openshift_cluster_observability_operator/1-latest/html-single/ui_plugins_for_red_hat_openshift_cluster_observability_operator/index#coo-perses-dashboards-create-crd_perses-dashboard
[coo-ds]: https://docs.redhat.com/en/documentation/red_hat_openshift_cluster_observability_operator/1-latest/html-single/ui_plugins_for_red_hat_openshift_cluster_observability_operator/index#coo-perses-dashboards-datasources-global-thanos_perses-dashboard
[cmo-svc]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/accessing_metrics/index#cmo-services-resources_accessing-monitoring-apis-by-using-the-cli
