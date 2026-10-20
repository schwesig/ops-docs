---
title: Pick the dashboard tool by audience, keep every dashboard in git, and document each one
---

**Status:** Proposed. The first dashboard surface was merged on 2026-09-04 (CCI-MOC/oac-apps#7), recorded 2026-10-20.

## Context

We build dashboards for the OAC clusters in three tools, and so far every one of them lives in
oac-apps. What is not written down anywhere is the reasoning: why a dashboard exists, why it sits in
the tool it sits in, and why it is fed by the metrics it is fed by. The reasons are spread over PR
descriptions, commit messages and YAML comments, and a few are not recorded at all.

What runs today, read off oac-apps at `457b659` (2026-10-20):

| Dashboard | Tool and console route | Defined in | Datasource | Rendered on |
|---|---|---|---|---|
| `mco-capacity` | Perses, Observe > Dashboards (Perses) | [`perses-dashboard-mco-capacity.yaml`][perses-cap] | the Perses datasource COO creates, `thanos-querier` port 9091 | hub and every managed cluster |
| `mco-capacity` | Console, Observe > Dashboards | [`console-dashboard-mco-capacity.yaml`][console-cap] | the console's built-in query path, not configured in the chart | hub and every managed cluster |
| `project-overview` | Grafana, one instance per project, own route | [`user-projects/templates/grafana.yaml`][grafana] | `thanos-querier` port 9092, scoped to the project | projects that opt in, today one on dev |

Next to these, there are dashboards we use but did not write: the ones the OpenShift console ships
under Observe > Dashboards, the ones COO ships for Perses (`accelerators-dashboard` and
`apm-dashboard` are named in CCI-MOC/oac-apps#13 and CCI-MOC/oac-apps#149), and the Grafana that
ACM MultiClusterObservability runs on the hub with its own dashboards.

How each piece got there:

- **Perses in the console.** COO came first as an operator only (CCI-MOC/oac-apps#5, merged
  2026-09-02). Its Monitoring UI plugin with Perses followed in CCI-MOC/oac-apps#7, merged
  2026-09-04. Every panel then showed Forbidden until `cluster-monitoring-view` was bound to the
  operator group in CCI-MOC/oac-apps#9, because Perses queries the `prometheuses/api` subresource
  and `cluster-reader` does not grant it. CCI-MOC/oac-apps#13 put the same chart on the managed
  clusters, because the datasource COO creates points at the local `thanos-querier` and on the hub
  shows only hub metrics.
- **Grafana per project.** CCI-MOC/oac-apps#14, merged 2026-09-28. Why projects get Grafana and not
  Perses is the subject of the [per-project metric separation ADR][adr37]. In short, COO points the
  Perses datasource at port 9091, which does not filter by namespace, and the CRD has no field to
  change that. That ADR also records why Perses stays for the cluster-wide view.
- **The MCO capacity dashboard.** CCI-MOC/oac-apps#139, merged 2026-10-20. It is in Perses and not
  in MCO's own Grafana because that Grafana has one datasource, the `rbac-query-proxy`, which serves
  fleet data and returns the hub's own TSDB metrics empty. A second datasource cannot be added
  either: the `grafana-datasources` secret belongs to the `MultiClusterObservability` resource and
  the CRD has no field for one. It ships on both console routes "because the console serves two
  dashboard lists".

So the three tools did not come from one plan, each was the answer to the problem in front of it
at the time. They still fall into a pattern, and the pattern is worth stating before the next
dashboard gets added somewhere by habit.

### What the tools can and cannot do

- **Perses through COO** is documented for this use. The COO 1-latest guide describes
  [Perses dashboards][coo-perses] as Kubernetes resources and says: "Because dashboards are YAML
  resources, you can store them in version control, track changes, and roll back to previous
  versions." The documented [global Thanos Querier datasource][coo-global-ds] points at port 9091,
  which "provides access to all the Thanos Querier endpoints" according to the
  [OCP 4.22 monitoring stack reference][ocp-ports]. That is right for an operator and wrong for a
  project.
- **Port 9092** "provides access to the `/api/v1/query`, `/api/v1/query_range/`, `/api/v1/labels`,
  `/api/v1/label/*/values`, and `/api/v1/series` endpoints restricted to a given project"
  ([same reference][ocp-ports]). That is what the project Grafana uses.
- **The console's older Observe > Dashboards list** is [documented for viewing][ocp-dashboards].
  How we add a dashboard to it, a ConfigMap in `openshift-config-managed` labelled
  `console.openshift.io/dashboard: "true"`, does not appear in the OCP 4.22 monitoring stack guides
  we checked, nor in the COO or ACM 2.17 guides. It works, but we rely on behaviour and not on a
  documented interface.
- **MCO's Grafana** takes custom dashboards as ConfigMaps labelled `grafana-custom-dashboard` in
  `open-cluster-management-observability`. The ACM 2.17 guide says
  ["wrapping your dashboard layout into a config map ensures reproducibility and persistence across environments"][acm-cm].
  It only sees what the MCO allowlist brings to the hub, see the [MCO allowlist ADR][adr36].
- **ACM 2.17 also has Perses dashboards for the fleet**, but only with the multicluster
  observability add-on, which is [Technology Preview][acm-perses-enable]. We run the classic
  collector and not the add-on (measured in the [MCO allowlist ADR][adr36]), so this is not
  available to us today.

## Decision

**We choose the dashboard tool by who reads it and what they may see, we keep every dashboard we
own as a resource in an oac-apps chart, and we give every such dashboard a short document in
ops-docs that says why it exists and where its metrics come from.** A dashboard gets an ADR of its
own only when building it forces a decision this ADR does not already cover.

### Tool by audience

| Reader | What they may see | Tool | Datasource |
|---|---|---|---|
| Operators of one cluster, including the hub's own services | the whole cluster | Perses via the COO Monitoring UI plugin | `thanos-querier` 9091 |
| Members of one project | their own namespace | the project's Grafana, as decided in the [per-project metric separation ADR][adr37] | `thanos-querier` 9092 |
| Operators looking across the fleet | every managed cluster, limited to what MCO collects | MCO's Grafana on the hub | the `rbac-query-proxy` that MCO configures |

The console's older Observe > Dashboards list is a **second route for an operator dashboard**, not
a separate tool. We ship a ConfigMap copy there only when somebody reads that list and not the
Perses one, and the Perses version stays the reference. The capacity dashboard is the one case
today.

### Datasource follows the audience

- A dashboard uses the datasource that matches its reader, never a broader one. A project
  dashboard on 9091, or a second datasource next to the project one, defeats the separation; the
  project Grafana is set up so that a Viewer cannot add one.
- A metric has to exist where the datasource reads. A local panel reads the cluster's own
  Prometheus and user workload monitoring. A fleet panel reads only what the MCO allowlist sends to
  the hub. A metric that is missing on a fleet dashboard is an allowlist question first
  ([MCO allowlist ADR][adr36]).
- On the hub, metrics that ACM scrapes carry an `acm_` prefix while their labels keep the real
  values, so `acm_prometheus_tsdb_head_series{job="observability-thanos-receive-default"}` is the
  correct pairing. A dashboard that depends on such a pairing has a `helm-unittest` that asserts
  it, as `capacity_dashboard_test.yaml` does.

### Dashboards as code

- Every dashboard we own is a resource in an oac-apps chart and reaches the cluster through Argo
  CD: `PersesDashboard`, `GrafanaDashboard`, or a ConfigMap for the console and for MCO's Grafana.
- It lives in the chart that owns the instance it runs in. Operator dashboards go next to the
  `UIPlugin` in `cluster-observability-operator`, project dashboards in `user-projects`, fleet
  dashboards in `acm-observability`.
- Each one sits behind a values switch, so it can be turned off for clusters where its data does
  not exist.
- A dashboard built in a UI is not a source of truth. Whatever should be kept moves into git. The
  editable project Grafana proposed in CCI-MOC/oac-apps#150 is the one exception: what a project
  saves there belongs to the project and not to us.

### Naming

- One kebab-case name per dashboard, used for the resource name, the uid where the tool has one,
  and the ops-docs document: `mco-capacity`, `project-overview`.
- Grafana caps a uid at 40 characters, so a uid that includes a project name is hashed, as
  `project-overview` does.
- A dashboard defined in its own template file names that file `<route>-dashboard-<name>.yaml`,
  as in `perses-dashboard-mco-capacity.yaml` and `console-dashboard-mco-capacity.yaml`. The
  existing exception is `project-overview`, which is defined inline in
  `charts/user-projects/templates/grafana.yaml`.
- The display title says what the dashboard shows, in sentence case: "MCO capacity and growth".
- Copies of one dashboard on different routes share the name.

### Ownership and documentation

- Every dashboard we own gets one document in ops-docs, `docs/<date>-dashboard-<name>/README.md`.
  It covers:
  - the question the dashboard answers and for whom
  - the tool, the console route and where it lives in oac-apps
  - the datasource
  - each metric with its PromQL, and why it is there and where it comes from
  - thresholds, if any
  - dependencies, such as an MCO allowlist entry
  - the owner
  - related ADRs
- The PR that adds or changes a dashboard updates its document. Where the reason for a panel or a
  metric is not on record, the document says "not recorded" instead of making one up.
- The owner is a person or team named in the document, and is who a reader asks when a panel looks
  wrong.
- Dashboards that ship with a product, from the console, COO or ACM, are not documented one by one.
  This ADR names them as present and leaves them alone.

### Alternatives considered

**One tool for every audience.** Grafana everywhere would put the operator view outside the
console and would add `grafana-operator`, a community operator, to the hub as well. Perses
everywhere cannot separate projects, as the [per-project metric separation ADR][adr37] shows. MCO's
Grafana for everything cannot reach the hub's own TSDB metrics, as CCI-MOC/oac-apps#139 found. We
would reconsider if the COO Perses datasource became configurable per namespace, which would let
Perses serve projects too.

**One ADR per dashboard.** That gives each dashboard a reviewed record, but most dashboards decide
nothing. They apply the choices above to a set of metrics, and an ADR for each would bury the few
dashboards that do decide something. A short document per dashboard records the why without the
ADR process. We would reconsider if those documents start to carry decisions, for example a new
datasource or a change in what gets collected.

**Console ConfigMaps as the main format.** It is the oldest route and needs no operator, but it is
not documented as an extension point in the guides we checked, and it lacks features the Perses
route has. The rate-interval fix in CCI-MOC/oac-apps#149 uses `$__rate_interval` in Perses and has
to fall back to a fixed `30m` window in the ConfigMap. We would reconsider if Red Hat documents the
ConfigMap route and Perses stops being the console's dashboard surface.

**Dashboards built in the UI, without git.** This is the fastest way to a panel, but it leaves no
review, no history and no record of why. The project Grafana runs without a volume and loses such
dashboards on restart anyway. We would reconsider for a project that wants to own its dashboards,
which is what CCI-MOC/oac-apps#150 proposes.

## Consequences

- **The why gets a fixed place.** Somebody looking at a panel can find out why it exists, which
  metric feeds it and where that metric comes from, without reading PR history. This applies to the
  dashboards we write ourselves, not to the ones products ship.
- **Every dashboard change becomes two changes.** One goes to oac-apps and one to ops-docs, and the
  two can drift. The document records intent rather than copying the YAML, so it only needs editing
  when the intent or the metrics change.
- **Two copies of one dashboard on two console routes are maintained twice.** CCI-MOC/oac-apps#149
  shows they already differ, in the rate window. Keeping the ConfigMap copy for a reader who needs
  it is a cost we accept knowingly.
- **We keep three dashboard tools.** Each is there for a different reader, and each is one more
  thing to upgrade and explain. The [per-project metric separation ADR][adr37] already accepts this
  for Perses and Grafana. This ADR adds MCO's Grafana to the list for the fleet view, which we run
  anyway as part of MCO.
- **Retroactive documents will show gaps.** For the existing dashboards, some reasons are not
  recorded: why these four capacity panels, and why CPU and memory in `project-overview`. The
  documents say so instead of filling in a guess.

## Open points

1. **No owner is recorded for any existing dashboard.** Both were written by one person. Who
   owns them as a team is undecided.
2. **The capacity dashboard renders on clusters without MCO.** The values comment says it should be
   off there, but the defaults are `true` and no file under `values/` overrides them. The chart runs
   on every managed cluster since CCI-MOC/oac-apps#13, so its panels are probably empty on the
   workload clusters. This has not been checked on a cluster yet, and the fix belongs in oac-apps.
3. **The console ConfigMap route has no documented support** in the guides we checked. Whether to
   keep shipping copies there, or to rely on Perses alone, needs a check of what the console team
   says about the older list.
4. **Thresholds.** No dashboard we own defines any. Whether thresholds belong in panels, or only in
   `PrometheusRule` alerts with dashboards showing raw values, is not decided.
5. **Fleet dashboards.** We have none of our own yet. If we move to the multicluster observability
   add-on, ACM 2.17 offers [Perses dashboards for the fleet][acm-perses-view] (Technology Preview
   today), and the fleet row of the table above should be decided again then. The
   [MCO allowlist ADR][adr36] already lists that move as a reason to revisit collection.
6. **The per-dashboard documents are pending.** `mco-capacity` and `project-overview` get theirs in
   a separate change, at `docs/<date>-dashboard-<name>/README.md`.

## See also

- [Per-project metric separation ADR][adr37]: Grafana per project on port 9092, and why Perses stays
  for the cluster-wide view
- [MCO allowlist ADR][adr36]: what reaches the hub at all, and so what a fleet dashboard can show
- CCI-MOC/oac-apps#7, #9, #13: Perses in the console, the role it needs, and why it also runs on the
  managed clusters
- CCI-MOC/oac-apps#14: the per-project Grafana and `project-overview`
- CCI-MOC/oac-apps#139 and #149: the MCO capacity dashboard, and the rate-window fix still open
- CCI-MOC/oac-apps#150: the editable project Grafana, still open
- COO 1-latest, [Perses dashboards][coo-perses] and [the global Thanos Querier datasource][coo-global-ds]
- Monitoring stack for Red Hat OpenShift 4.22, [ports and access roles of `thanos-querier`][ocp-ports]
  and [reviewing monitoring dashboards][ocp-dashboards]
- ACM 2.17 observability, [Grafana dashboards from a config map][acm-cm] and
  [Perses dashboards with the add-on][acm-perses-view]

[perses-cap]: https://github.com/CCI-MOC/oac-apps/blob/457b659/charts/cluster-observability-operator/templates/perses-dashboard-mco-capacity.yaml
[console-cap]: https://github.com/CCI-MOC/oac-apps/blob/457b659/charts/cluster-observability-operator/templates/console-dashboard-mco-capacity.yaml
[grafana]: https://github.com/CCI-MOC/oac-apps/blob/457b659/charts/user-projects/templates/grafana.yaml
[adr36]: https://github.com/CCI-MOC/ops-docs/pull/36
[adr37]: https://github.com/CCI-MOC/ops-docs/pull/37
[coo-perses]: https://docs.redhat.com/en/documentation/red_hat_openshift_cluster_observability_operator/1-latest/html-single/ui_plugins_for_red_hat_openshift_cluster_observability_operator/index#coo-about-perses-dashboard_perses-dashboard
[coo-global-ds]: https://docs.redhat.com/en/documentation/red_hat_openshift_cluster_observability_operator/1-latest/html-single/ui_plugins_for_red_hat_openshift_cluster_observability_operator/index#coo-perses-dashboards-datasources-global-thanos_perses-dashboard
[ocp-ports]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/accessing_metrics/index#cmo-services-resources_accessing-monitoring-apis-by-using-the-cli
[ocp-dashboards]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/accessing_metrics/index#reviewing-monitoring-dashboards-admin_accessing-metrics-as-an-administrator
[acm-cm]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/observability/index#design-dashboards-configmaps
[acm-perses-enable]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/observability/index#enable-mcoa-perses
[acm-perses-view]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/observability/index#view-perses-dashboards
