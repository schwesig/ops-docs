---
title: "Dashboard: Project overview"
---

This document describes one dashboard as it exists in oac-apps
[`457b659`][main], 2026-10-20. It records why the dashboard exists and why
each metric is on it. The dashboard strategy (which tool for which audience,
naming, ownership) is the subject of a separate ADR,
`adrs/<date>-dashboard-strategy`, which is still pending.

## Question and audience

**Question:** how much CPU and memory do the pods in my project use?

**Audience:** the members of one project, in that project's own Grafana. They
see only their own namespace. Operators who want the cluster-wide view use the
console dashboards instead (see the
[MCO capacity dashboard](../2026-10-20-dashboard-mco-capacity/) and
https://github.com/CCI-MOC/ops-docs/pull/37, "Perses stays").

**Why it was created:** it is the dashboard that ships with the per-project
Grafana, added in CCI-MOC/oac-apps#14 (merged 2026-09-28). That PR gives the
reasons for Grafana itself:

> Grafana is widely known and was already used in MOC 1.0, so existing
> dashboards onboard easy and old users get the same look and feel.
>
> Second reason is separation. Perses is locked to port 9091 and cannot
> separate tenants today, so startups sharing a workload cluster would see each
> other's metrics.

Why a starter dashboard is a chart resource rather than something each project
builds, from https://github.com/CCI-MOC/ops-docs/pull/37: "the same dashboard
definition shows a different project its own data without being edited. Write
a panel once, every project that opts in gets it". Why the first two panels are
CPU and memory, and not something else, is not recorded.

## Tool and location

| | |
|---|---|
| Tool | Grafana 11.3.0, one instance per project, run by the community Grafana operator (`grafana.integreatly.org/v1beta1`) |
| Resource | `GrafanaDashboard` `project-overview` in `project-<name>`, matched to the instance by the label `dashboards: project-<name>` |
| Source | [`charts/user-projects/templates/grafana.yaml`][grafana], last part of the file |
| Switch | Per project, `grafana.enabled: true` in the `user-projects` values. Off by default |
| Deployed to | Managed clusters, through `all-managed-clusters` ([`all-managed-clusters.yaml`][managedappset]). Today only `project-gavin-test` on `oac-dev-workload0` sets the switch ([values][devvalues]). No prod project does |
| UID | `project-<first 16 hex chars of sha256(name)>-ovw`, fixed at 28 characters, because Grafana caps a UID at 40 characters and the readable form would break at 24 characters of project name (#14) |
| Default time range | 6 hours |
| Tests | [`grafana_test.yaml`][test] |

Viewers cannot save changes. From the chart: "Dashboards belong in git as
GrafanaDashboard resources, because Grafana OSS cannot grant 'may save
dashboards' without also granting 'may create datasources'". So this dashboard
is changed only through oac-apps. CCI-MOC/oac-apps#150 (open) proposes an opt-in
editable Grafana. It leaves this dashboard as it is.

## Data source

One `GrafanaDatasource` named `thanos`, the only datasource the instance has.
It points at
`https://thanos-querier.openshift-monitoring.svc.cluster.local:9092`, the
tenancy port. It authenticates as the project's `grafana-sa` service account,
which has `view` in the project only, and adds `namespace=project-<name>` to
every query.

Red Hat documents port 9092 as providing "access to the `/api/v1/query`,
`/api/v1/query_range/`, `/api/v1/labels`, `/api/v1/label/*/values`, and
`/api/v1/series` endpoints restricted to a given project. Granting access
requires binding a user to the view cluster role in the project"
([OCP 4.22, Accessing metrics, services][cmo-svc]). The full proxy chain behind
9092, and why the namespace cannot be overridden, is in
https://github.com/CCI-MOC/ops-docs/pull/37 and not repeated here.

**History depth.** This Grafana stores no metrics of its own. Both metrics live
in the cluster's platform Prometheus, so the history is that instance's
retention. On both hubs' managed clusters it is set to 15 days on a volume
(`platform.enabled: true` in the hub-wide `user-workload-monitoring` values,
CCI-MOC/oac-apps#131 and #134). Without that volume the platform Prometheus
mounts `emptyDir`, and the panels show only the age of the pod. CCI-MOC/oac-apps#132
(open) adds a note about this to the chart values.

## Metrics

| Panel | PromQL (rendered for project `<name>`) | Metric source | Needs the MCO allowlist? | Why it is there |
|---|---|---|---|---|
| CPU usage by pod | `sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="project-<name>", container!=""}[5m]))` | cAdvisor in the kubelet, scraped by the platform Prometheus in `openshift-monitoring` | No | CPU per pod in the project. The choice of this metric is not recorded |
| Memory usage by pod | `sum by (pod) (container_memory_working_set_bytes{namespace="project-<name>", container!=""})` | Same | No | Memory per pod in the project. The choice of this metric is not recorded |

In the chart, the template inserts the project name in place of `<name>`. The
legend shows one series per pod.

Two details in the queries are deliberate:

- **The namespace is named in the query**, even though the datasource already
  forces it. From the commit that added it (CCI-MOC/oac-apps@0b135b3,
  2026-09-28): "The dashboard queries name the namespace as well rather than
  relying on the port to inject it, so they stay scoped if the datasource is
  ever repointed or the JSON gets copied."
- **`container!=""`** drops the series that cAdvisor reports for the pod as a
  whole, so each pod is summed from its containers only and not counted twice.
  The reason for the filter is not recorded in oac-apps. This is what the
  filter does.

The dashboard queries the cluster's local `thanos-querier` and not MCO, so the
MCO allowlist (https://github.com/CCI-MOC/ops-docs/pull/36) does not affect it.
Worth knowing anyway: `container_cpu_usage_seconds_total` is one of the
metrics #36 lists as measured locally but not collected at the hub. A fleet
version of this dashboard would need it on the allowlist.

## Thresholds

None. No thresholds, colouring or alerts. Alerting is "phase 2" in #14.
The project's own quota (`ResourceQuota`, default `requests.cpu: 10`,
`limits.memory: 40Gi`, [`user-projects/values.yaml`][values]) would be the
natural reference line, but it is not drawn.

## Dependencies

- **Grafana operator** (`charts/grafana-operator`, community catalog, channel
  `v5`) on the cluster. It is only in the community catalog, which means "no
  official support and no Red Hat certification" ([chart values][opvalues]).
- **Network access** from the project namespace to `thanos-querier` on 9092, and
  from the operator namespace to Grafana on 3000, through
  `restrict-tenant-networks` (CCI-MOC/oac-apps#31).
- **Platform Prometheus volume** on the cluster for any history beyond the age
  of the pod (see Data source).
- **MCO allowlist:** not needed.

## Owner

Not recorded in oac-apps. The dashboard was added in CCI-MOC/oac-apps#14 with
the per-project Grafana.

## Open points

1. **Panel choice.** Why CPU and memory, and whether projects need more (GPU,
   storage, network), is not recorded. GPU is a likely next question:
   `requests.nvidia.com/gpu` is part of the project quota, and the DCGM exporter
   runs on GPU clusters.
2. **Rate window.** The CPU panel uses `[5m]`. CCI-MOC/oac-apps#149 found that a
   fixed `[5m]` draws drops that are not there once the step passes 5 minutes,
   for the MCO capacity dashboard. With a 6 hour default range this panel is
   less exposed, but a long range would hit the same effect. #149 does not touch
   this file.
3. **Owner.** No owner is named. The pending strategy ADR is expected to decide
   who owns dashboards.
4. **Only on dev so far.** No prod project uses it yet. The tenancy filter was
   to be "measured with a real token" on `gavin-test` first (CCI-MOC/oac-apps@9c08828).
   Whether that test happened is not recorded.

## See also

- `adrs/<date>-dashboard-strategy` (pending): which tool for which audience,
  dashboards as code, naming and ownership.
- https://github.com/CCI-MOC/ops-docs/pull/37: per-project Grafana ADR, the
  decision this dashboard depends on.
- https://github.com/CCI-MOC/ops-docs/pull/36: MCO metrics allowlist ADR.
- [Dashboard: MCO capacity and growth](../2026-10-20-dashboard-mco-capacity/),
  the other dashboard in oac-apps.
- CCI-MOC/oac-apps#14: the PR that added the Grafana and this dashboard.
- CCI-MOC/oac-apps#132, #150: open follow-ups on the project Grafana.

[main]: https://github.com/CCI-MOC/oac-apps/tree/457b659283f3f8da57bb44454628170058625a44
[grafana]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/charts/user-projects/templates/grafana.yaml
[values]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/charts/user-projects/values.yaml
[test]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/charts/user-projects/tests/unit/grafana_test.yaml
[devvalues]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/values/oac-dev-infra/oac-dev-workload0/user-projects.yaml
[managedappset]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/applicationsets/templates/managed/all-managed-clusters.yaml
[opvalues]: https://github.com/CCI-MOC/oac-apps/blob/457b659283f3f8da57bb44454628170058625a44/charts/grafana-operator/values.yaml
[cmo-svc]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/accessing_metrics/index#cmo-services-resources_accessing-monitoring-apis-by-using-the-cli
