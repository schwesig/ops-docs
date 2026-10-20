---
title: "How to create alerts on metric values and on log contents"
# Render as plain Markdown: the alert examples contain Prometheus {{ }} templates that Liquid would parse.
templateEngineOverride: md
---

This document describes how an operator on the OAC clusters creates an alert
on a metric value and an alert on log contents. It covers what oac-apps
deploys at [`1260a86`][main] (2026-10-20) and the logging change that is
still open in [schwesig/OA_oac-apps#2][logging-pr]. For each path it gives a
minimal YAML example, where the rule should live as code, how to check that it
fires, and where the alert goes.

> [!IMPORTANT]
> Alerts are evaluated today, but they are not delivered to anyone. oac-apps
> configures no Alertmanager receivers on any cluster, and log alerts cannot be
> evaluated yet. See [Where alerts end up today](#where-alerts-end-up-today)
> and [Open points](#open-points).

## Which path applies where

OAC has two infra hubs (`oac-dev-infra`, `oac-prod-infra`). Each runs ACM and
ACM Observability (MCO) and hosts the control planes of its workload clusters
(`oac-dev-workload0`, `oac-dev-workload1`, `oac-prod-workload0`, ...).

| You want to alert on | Rule object | Where it is evaluated | Applies on OAC to |
|---|---|---|---|
| A platform metric of one cluster (nodes, operators, kube-state-metrics) | `AlertingRule` (`monitoring.openshift.io/v1`) in `openshift-monitoring` | That cluster's platform Prometheus | Every cluster, hubs and workload clusters |
| A metric of a project on a workload cluster | `PrometheusRule` (`monitoring.coreos.com/v1`) in the project namespace | User workload monitoring (Thanos Ruler) on that cluster | Workload clusters only. User workload monitoring is not enabled on the hubs |
| A metric across the fleet | Entry in the `thanos-ruler-custom-rules` ConfigMap on the hub | MCO's `observability-thanos-rule` on the hub | Both hubs, for every cluster that sends metrics to MCO |
| Log contents | `AlertingRule` (`loki.grafana.com/v1`) | The LokiStack ruler | **Nowhere yet.** Logging is not on `main`, and the open logging change does not enable the ruler |

Where each component comes from in oac-apps:

- **User workload monitoring** is switched on by the `user-workload-monitoring`
  chart ([`enable.yaml`][uwm-enable]), which the `all-managed-clusters`
  ApplicationSet sends to every managed cluster
  ([`all-managed-clusters.yaml`][managedappset]). It is not in
  `hub-components` ([`hub-components.yaml`][hubappset]), so the hubs run
  platform monitoring only.
- **MCO** is deployed to both hubs by the `acm-observability` chart
  ([`multiclusterobservability.yaml`][mco]). It runs three Thanos rule
  replicas and three Alertmanager replicas.
- **Logging** (Loki Operator, `cluster-logging` chart, LokiStack
  `openshift-logging/logging-loki`) exists only on the branch of
  [schwesig/OA_oac-apps#2][logging-pr]. The design is recorded in the
  workload cluster logging ADR ([schwesig/ops-docs#1][logging-adr]).

## Alert on a metric value

### Where the rule lives as code

oac-apps contains no alerting rule today. A search of [`1260a86`][main] for
`PrometheusRule`, `AlertingRule` and `custom_rules` finds nothing. ArgoCD
syncs every chart with `selfHeal: true`, so a rule created by hand with
`oc apply` stays until someone removes it, but it is not reviewed and is not
reproduced on a new cluster. Put rules in the chart that deploys the thing
they watch:

- A platform or project rule for one cluster: a template in the chart that
  owns the component, so it reaches the same clusters as the component.
- A fleet rule: the `acm-observability` chart, next to the
  `observability-metrics-custom-allowlist` ConfigMap
  ([`metrics-custom-allowlist.yaml`][allowlist]).

No chart holds rules yet, so the first rule sets the pattern. See open point 4.

### Platform metric on one cluster

Red Hat documents `AlertingRule`, not a `PrometheusRule` in
`openshift-monitoring`, for new platform rules: "You must create the
AlertingRule object in the openshift-monitoring namespace. Otherwise, the
alerting rule is not accepted." It requires `cluster-admin`
([OCP 4.22, Creating new alerting rules][ocp-platform-rule]).

```yaml
apiVersion: monitoring.openshift.io/v1
kind: AlertingRule
metadata:
  name: oac-node-filesystem
  namespace: openshift-monitoring
spec:
  groups:
    - name: oac-node
      rules:
        - alert: OACNodeRootFilesystemAlmostFull
          expr: |
            node_filesystem_avail_bytes{mountpoint="/"}
              / node_filesystem_size_bytes{mountpoint="/"} < 0.10
          for: 15m
          labels:
            severity: warning
          annotations:
            message: "{{ $labels.instance }} has less than 10% free on /."
```

To change a rule that ships with the platform, do not copy it. Red Hat uses
`AlertRelabelConfig` for that
([OCP 4.22, Modifying core platform alerting rules][ocp-platform-modify]).

### Project metric on a workload cluster

Project rules go in the project namespace, `project-<name>`, which the
`user-projects` chart creates. Red Hat's prerequisites are that user workload
monitoring is enabled and that the author is "a cluster administrator or [...]
a user that has the monitoring-rules-edit cluster role for the project"
([OCP 4.22, Creating alerting rules for user-defined projects][ocp-uwm-rule]).
The `user-projects` chart binds the plain `admin`, `edit` or `view` role to
the project groups and does not bind `monitoring-rules-edit`
([`rbac.yaml`][projects-rbac]). The OCP 4.22 monitoring guides do not say
whether those roles include it, so a project member may need it bound before they can create rules
themselves ([monitoring cluster roles][ocp-uwm-roles]). Through oac-apps the
rule is applied by ArgoCD, so this only matters for rules created by hand.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: web-unavailable
  namespace: project-example
spec:
  groups:
    - name: project-example
      rules:
        - alert: WebDeploymentUnavailable
          expr: kube_deployment_status_replicas_available{deployment="web"} == 0
          for: 5m
          labels:
            severity: warning
          annotations:
            message: "Deployment web in project-example has no available replicas."
```

The rule only sees series from its own namespace. To make one rule apply to
every project, Red Hat documents cross-project rules, configured with a list
of namespaces in `user-workload-monitoring-config`
([OCP 4.22, Creating cross-project alerting rules][ocp-uwm-cross]). oac-apps
sets no such namespace ([`tuning.yaml`][uwm-tuning]).

A namespace labelled `openshift.io/user-monitoring=false` is excluded from user
workload monitoring and its rules are ignored
([OCP 4.22, Excluding a user-defined project from monitoring][ocp-uwm-exclude]).
oac-apps sets that label nowhere.

### Metric across the fleet, on the hub

MCO evaluates hub rules against the fleet data. Red Hat:
"Create custom rules for the observability installation by adding Prometheus
recording rules and alerting rules to the observability resource" and "When
you update your custom rules, observability-thanos-rule pods restart
automatically" ([ACM 2.17, Creating custom rules][acm-rules]).

The 2.17 guide shows only the `data` part of the ConfigMap. Its name,
`thanos-ruler-custom-rules`, and the namespace
`open-cluster-management-observability` come from the operator source for
release 2.17 ([`config.go`][mco-config]).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: thanos-ruler-custom-rules
  namespace: open-cluster-management-observability
data:
  custom_rules.yaml: |
    groups:
      - name: oac-fleet
        rules:
          - alert: OACClusterNotSendingMetrics
            expr: |
              absent_over_time(cluster:cpu_usage_cores:sum{cluster="oac-dev-workload0"}[30m])
            for: 15m
            labels:
              cluster: oac-dev-workload0
              severity: warning
            annotations:
              summary: "oac-dev-workload0 has sent no metrics to MCO for 30 minutes."
              description: "Check the observability add-on on oac-dev-workload0."
```

The hub ruler only sees what the managed clusters send. MCO collects the names
in its allowlist and nothing else
([ACM 2.17, Default metrics][acm-default-metrics]), plus the custom allowlist
in oac-apps, which today adds three DCGM GPU metrics
([`values.yaml`][mco-values]). A fleet rule on a metric that is in neither list
evaluates against nothing and never fires. The allowlist strategy is the
subject of [CCI-MOC/ops-docs#36][adr-allowlist].

For a fleet alert on a project metric, Red Hat recommends evaluating the rule
on the managed cluster instead. It forwards user workload alerts if the
`PrometheusRule` carries the label
`openshift.io/prometheus-rule-evaluation-scope: leaf-prometheus`, which keeps
the rule away from the Thanos Ruler
([ACM 2.17, Forwarding your alerts][acm-forward]).
With this label the rule is evaluated only by the user workload Prometheus, so
it can only use metrics that the project itself exposes through a
`ServiceMonitor` or `PodMonitor`, not platform metrics such as
kube-state-metrics. The project example above would never fire with the label.

### Check that a metric alert fires

1. Check that the rule was accepted:
   `oc get alertingrule.monitoring.openshift.io -n openshift-monitoring`,
   `oc get prometheusrule -n project-example`, or, on the hub,
   `oc -n open-cluster-management-observability get configmap thanos-ruler-custom-rules`.
2. To test the path end to end, add a temporary rule with `expr: vector(1)`
   and `for: 1m`, the pattern Red Hat uses for its example rule
   ([OCP 4.22, Creating new alerting rules][ocp-platform-rule]). Remove it
   afterwards.
3. On a single cluster, open Observe > Alerting in the console. "The three main
   pages in the Alerting UI in this perspective are the Alerts, Silences, and
   Alerting rules pages"
   ([OCP 4.22, Accessing the alerting UI][ocp-alerting-ui]). The rule appears on
   Alerting rules, and the alert on Alerts once `for` has passed.
4. On the hub, open MCO's Grafana, go to Explore and query `ALERTS`. "The alert
   is only available in Grafana if you created the alert"
   ([ACM 2.17, Creating custom rules][acm-rules],
   [Exploring data with Grafana][acm-grafana]).

## Alert on log contents

> [!WARNING]
> Not possible on OAC today. `main` deploys no logging stack. The open logging
> change [schwesig/OA_oac-apps#2][logging-pr] deploys a LokiStack, but its
> template has no `spec.rules` section ([`lokistack.yaml`][lokistack]), so the
> ruler that evaluates log alerts is not enabled. The steps below are what it
> takes once logging is deployed. See open point 2.

Log alerts are LogQL expressions in an `AlertingRule` of the Loki Operator. Red
Hat: "LokiStack log-based alerts and recorded metrics are triggered by providing
LogQL [...] expressions to the ruler component"
([Logging 6.6, Log-based alerting and recording rules for Loki][log-about]).

### Enable the ruler

An administrator enables the ruler in the LokiStack. `selector` matches labels
on the `AlertingRule` objects and `namespaceSelector` matches labels on their
namespaces. If `namespaceSelector` is "undefined, the system uses only the rules
in the same namespace as the LokiStack"
([Logging 6.6, Configure Loki ruler][log-ruler]). Red Hat gives no label name;
the one below is an example. In oac-apps this belongs in the `cluster-logging`
chart's [`lokistack.yaml`][lokistack].

```yaml
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: logging-loki
  namespace: openshift-logging
spec:
  # ...
  rules:
    enabled: true
    selector:
      matchLabels:
        massopen.cloud/log-alerts: "true"
    namespaceSelector:
      matchLabels:
        massopen.cloud/log-alerts: "true"
```

### Write the rule

Which namespace an `AlertingRule` may live in depends on its tenant
([Logging 6.6, Create a log-based alerting rule with Loki][log-rule]):

| `tenantID` | Valid namespaces |
|---|---|
| `audit` | `openshift-logging` |
| `infrastructure` | `openshift-*`, `kube-*`, `default` |
| `application` | All other namespaces, for example `project-<name>` |

For the application tenant, "The value of kubernetes_namespace_name must match
the metadata.namespace value." `severity` (`critical`, `warning` or `info`),
`summary` and `description` are mandatory. The namespace must also carry the
`namespaceSelector` label from the LokiStack.

```yaml
apiVersion: loki.grafana.com/v1
kind: AlertingRule
metadata:
  name: web-errors
  namespace: project-example
  labels:
    massopen.cloud/log-alerts: "true"
spec:
  tenantID: application
  groups:
    - name: project-example-web
      rules:
        - alert: WebLogsManyErrors
          expr: |
            sum(count_over_time({kubernetes_namespace_name="project-example", kubernetes_container_name="web"} |= "ERROR" [5m])) > 10
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "More than 10 ERROR lines in 5 minutes from web in project-example."
            description: "Check Observe > Logs for project-example."
```

Which logs a cluster has decides which tenant can alert. On the workload
clusters the change collects application, infrastructure and node audit
(auditd, OVN) logs. On `oac-dev-infra` it collects only the hosted control
plane API audit, from `clusters-<name>`, which arrives in the **application**
tenant of the hub ([`values.yaml`][logging-values],
[`cluster-logging.yaml`][logging-hub-values]). An alert on a workload
cluster's API audit therefore runs on the hub, as an application rule in
`clusters-<name>`.

Project members who should write their own log alerts need an
`alertingrules.loki.grafana.com-v1-*` role bound in their namespace. The
`cluster-logging` chart grants read access to logs only
([Logging 6.6, Authorize LokiStack rules RBAC permissions][log-rbac],
[`project-log-reader.yaml`][log-reader]).

### Check that a log alert fires

1. Check that the rule was accepted: `oc get alertingrule.loki.grafana.com -n project-example`.
   The webhook rejects an invalid interval, `for`, LogQL expression or a
   duplicate group name ([Logging 6.6, Create a log-based alerting rule with Loki][log-rule]).
2. Run the expression in Observe > Logs first. It should return the lines you
   expect.
3. Write a matching line, for example
   `oc -n project-example exec deploy/web -- sh -c 'echo ERROR test > /proc/1/fd/1'`,
   enough times to cross the threshold.
4. Look in Observe > Alerting
   ([Logging 6.6, Accessing the Alerting UI][log-alerting-ui]). Log alerts go
   to the same Alertmanager as metric alerts (next section).

## Where alerts end up today

| Source | First Alertmanager | Forwarded to | Receivers configured in oac-apps |
|---|---|---|---|
| Platform rules on a workload cluster | `alertmanager-main` in `openshift-monitoring` | The hub's MCO Alertmanager | None |
| Project rules on a workload cluster | `alertmanager-main` (no separate user workload Alertmanager) | The hub, if MCO forwards user workload alerts (open point 3) | None |
| Platform rules on a hub | `alertmanager-main` on the hub | Not determined (open point 3) | None |
| Fleet rules (`thanos-ruler-custom-rules`) | MCO's `observability-alertmanager` on the hub | | None |
| Log rules (once enabled) | `alertmanager-main` of the cluster with the LokiStack | As platform or project rules above | None |

The details:

- **No user workload Alertmanager.** oac-apps sets neither `alertmanager` in
  `user-workload-monitoring-config` nor `enableUserAlertmanagerConfig` in
  `cluster-monitoring-config` ([`tuning.yaml`][uwm-tuning],
  [`enable.yaml`][uwm-enable]). "If you set this value to false or if the key
  is omitted, user-defined alerts are routed to the default platform
  Alertmanager instance"
  ([OCP 4.22, Enabling a separate Alertmanager instance][ocp-uwm-am]). Without
  `enableUserAlertmanagerConfig`, `AlertmanagerConfig` objects in projects are
  not used ([OCP 4.22, Enabling the platform Alertmanager instance for
  user-defined alert routing][ocp-uwm-am-platform]). Log alerts follow the same
  rule: they go to `openshift-monitoring` unless a user workload Alertmanager is
  enabled ([Logging 6.6][log-about]).
- **Platform Alertmanager has no receivers.** "Alertmanager does not send
  notifications by default"
  ([OCP 4.22, Configuring alert notifications][ocp-notify]). oac-apps does not
  manage the `alertmanager-main` secret on any cluster, and the console shows
  the `AlertmanagerReceiverNotConfigured` alert where none is set
  ([Configuring alert routing in the console][ocp-notify-console]).
- **Forwarding to the hub.** "After you enable observability, alerts from your
  OpenShift Container Platform managed clusters are automatically sent to
  Alertmanager on your hub cluster"
  ([ACM 2.17, Forwarding your alerts][acm-forward]). oac-apps sets neither
  `mco-disable-alerting` nor `mco-disable-uwl-alerting` on the
  `MultiClusterObservability` resource ([`multiclusterobservability.yaml`][mco]),
  so forwarding is on by default
  ([Disabling alert forwarding][acm-forward-off],
  [Disabling user workload alert forwarding][acm-forward-uwl-off]). Whether it
  works on OAC has not been checked (open point 3).
- **Hub Alertmanager has no receivers either.** Receivers are added by
  overriding "the alertmanager-config secret in the
  open-cluster-management-observability namespace"
  ([ACM 2.17, Configuring alerts for observability][acm-am-config]). oac-apps
  does not manage that secret. Silences go through `amtool` in the
  `observability-alertmanager-0` pod
  ([ACM 2.17, Silencing alerts][acm-silence]).
- **Hub alerts in the console.** The COO Monitoring UI plugin can show the MCO
  Alertmanager in the console through `spec.monitoring.acm`
  ([COO, Installing the Monitoring UI plugin][coo-acm]). oac-apps leaves it off,
  `uiPlugin.acm.enabled: false`, "until MCO is running"
  ([`values.yaml`][coo-values]). Until it is on, fleet alerts are visible in
  MCO's Grafana only.

## Open points

1. **No receivers.** Neither the platform Alertmanagers nor the MCO
   Alertmanager have a receiver in oac-apps, so no alert reaches a person or a
   ticket system. Someone has to decide where alerts go (mail, Slack, a pager)
   and which Alertmanager owns that routing: per cluster, or centrally on the
   hub through MCO forwarding. The secret then belongs in oac-apps through
   External Secrets, like the other credentials.
2. **Log alerts need the ruler.** The LokiStack in
   [schwesig/OA_oac-apps#2][logging-pr] has no `spec.rules`. Enabling it needs a
   label convention for `selector` and `namespaceSelector`, ruler capacity on
   the `1x.demo` size used on dev, and a decision on whether project members
   get `alertingrules.loki.grafana.com-v1-edit`. It should be added to that
   change or to a follow-up once logging is enabled.
3. **Alert forwarding is not verified, and may be overwritten.** MCO enables
   forwarding by editing `cluster-monitoring-config` (and, for user workload
   alerts, `user-workload-monitoring-config`) on each managed cluster. The
   ACM 2.17 guide says these updates are reverted when forwarding is turned off
   ([Disabling alert forwarding][acm-forward-off]). The `user-workload-monitoring`
   chart writes the same `config.yaml` key in both ConfigMaps with server-side
   apply and `selfHeal: true` ([`enable.yaml`][uwm-enable],
   [`tuning.yaml`][uwm-tuning]). If ArgoCD reverts MCO's additions, forwarding
   stops without an error. Check on a workload cluster whether the forwarding
   settings survive a sync, and whether alerts from a workload cluster appear
   in the hub Alertmanager. It is also not recorded whether the hubs forward
   their own alerts.
4. **Where rules live.** No chart holds alerting rules yet. The first rule
   decides whether rules sit in the chart of the component they watch, as
   suggested above, or in one dedicated chart per path.
5. **Project rule permissions.** The `user-projects` chart does not bind
   `monitoring-rules-edit`. Whether project members should create their own
   metric and log alerts, or only through oac-apps, is not recorded.

## See also

- [CCI-MOC/ops-docs#36][adr-allowlist]: MCO metrics allowlist ADR. Decides
  which metrics a fleet rule can see.
- [schwesig/ops-docs#1][logging-adr]: workload cluster logging ADR (Proposed).
- [schwesig/OA_oac-apps#2][logging-pr]: the logging implementation (open).
- [OCP 4.22, Managing alerts][ocp-alerting-ui]: silences, alert details,
  rule management.

[main]: https://github.com/CCI-MOC/oac-apps/tree/1260a865c69c1851a75b781df53ae40ca41e6be1
[hubappset]: https://github.com/CCI-MOC/oac-apps/blob/1260a865c69c1851a75b781df53ae40ca41e6be1/applicationsets/templates/hub/hub-components.yaml
[managedappset]: https://github.com/CCI-MOC/oac-apps/blob/1260a865c69c1851a75b781df53ae40ca41e6be1/applicationsets/templates/managed/all-managed-clusters.yaml
[uwm-enable]: https://github.com/CCI-MOC/oac-apps/blob/1260a865c69c1851a75b781df53ae40ca41e6be1/charts/user-workload-monitoring/templates/enable.yaml
[uwm-tuning]: https://github.com/CCI-MOC/oac-apps/blob/1260a865c69c1851a75b781df53ae40ca41e6be1/charts/user-workload-monitoring/templates/tuning.yaml
[mco]: https://github.com/CCI-MOC/oac-apps/blob/1260a865c69c1851a75b781df53ae40ca41e6be1/charts/acm-observability/templates/multiclusterobservability.yaml
[mco-values]: https://github.com/CCI-MOC/oac-apps/blob/1260a865c69c1851a75b781df53ae40ca41e6be1/charts/acm-observability/values.yaml
[allowlist]: https://github.com/CCI-MOC/oac-apps/blob/1260a865c69c1851a75b781df53ae40ca41e6be1/charts/acm-observability/templates/metrics-custom-allowlist.yaml
[coo-values]: https://github.com/CCI-MOC/oac-apps/blob/1260a865c69c1851a75b781df53ae40ca41e6be1/charts/cluster-observability-operator/values.yaml
[projects-rbac]: https://github.com/CCI-MOC/oac-apps/blob/1260a865c69c1851a75b781df53ae40ca41e6be1/charts/user-projects/templates/rbac.yaml
[logging-pr]: https://github.com/schwesig/OA_oac-apps/pull/2
[lokistack]: https://github.com/schwesig/OA_oac-apps/blob/76449c660613fbf9e1cb4185eaf3a2fde9dd1cb4/charts/cluster-logging/templates/lokistack.yaml
[logging-values]: https://github.com/schwesig/OA_oac-apps/blob/76449c660613fbf9e1cb4185eaf3a2fde9dd1cb4/charts/cluster-logging/values.yaml
[logging-hub-values]: https://github.com/schwesig/OA_oac-apps/blob/76449c660613fbf9e1cb4185eaf3a2fde9dd1cb4/values/oac-dev-infra/local-cluster/cluster-logging.yaml
[log-reader]: https://github.com/schwesig/OA_oac-apps/blob/76449c660613fbf9e1cb4185eaf3a2fde9dd1cb4/charts/cluster-logging/templates/project-log-reader.yaml
[logging-adr]: https://github.com/schwesig/ops-docs/pull/1
[adr-allowlist]: https://github.com/CCI-MOC/ops-docs/pull/36
[mco-config]: https://github.com/stolostron/multicluster-observability-operator/blob/45aa96dde952598cae0e601148b139b51ad5fb99/operators/multiclusterobservability/pkg/config/config.go#L97
[ocp-platform-rule]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/managing_alerts/index#creating-new-alerting-rules_managing-alerts-as-an-administrator
[ocp-platform-modify]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/managing_alerts/index#modifying-core-platform-alerting-rules_managing-alerts-as-an-administrator
[ocp-uwm-rule]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/managing_alerts/index#creating-alerting-rules-for-user-defined-projects_managing-alerts-as-a-developer
[ocp-uwm-cross]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/managing_alerts/index#creating-cross-project-alerting-rules-for-user-defined-projects_managing-alerts-as-a-developer
[ocp-alerting-ui]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/managing_alerts/index#monitoring-accessing-the-alerting-ui_managing-alerts-as-an-administrator
[ocp-uwm-roles]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/configuring_user_workload_monitoring/index#granting-users-permission-to-monitor-user-defined-projects_preparing-to-configure-the-monitoring-stack-uwm
[ocp-uwm-exclude]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/configuring_user_workload_monitoring/index#excluding-a-user-defined-project-from-monitoring_preparing-to-configure-the-monitoring-stack-uwm
[ocp-uwm-am]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/configuring_user_workload_monitoring/index#enabling-a-separate-alertmanager-instance-for-user-defined-alert-routing_preparing-to-configure-the-monitoring-stack-uwm
[ocp-uwm-am-platform]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/configuring_user_workload_monitoring/index#enabling-the-platform-alertmanager-instance-for-user-defined-alert-routing_preparing-to-configure-the-monitoring-stack-uwm
[ocp-notify]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/configuring_core_platform_monitoring/index#configuring-alert-notifications_configuring-alerts-and-notifications
[ocp-notify-console]: https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.22/html-single/configuring_core_platform_monitoring/index#configuring-alert-routing-console_configuring-alerts-and-notifications
[log-about]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/configure/index#loki-about-alerting-recording-rules_configure-custom-logging-alerts
[log-ruler]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/configure/index#configuring-logging-loki-ruler_configure-custom-logging-alerts
[log-rbac]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/configure/index#loki-rbac-rules-permissions_configure-custom-logging-alerts
[log-rule]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/configure/index#logging-enabling-loki-alerts_configure-custom-logging-alerts
[log-alerting-ui]: https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html-single/observe/index#monitoring-accessing-the-alerting-ui_default-logging-alerts
[acm-rules]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/observability/index#creating-custom-rules
[acm-default-metrics]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/observability/index#default_metrics
[acm-forward]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/observability/index#obs-forward-alerts
[acm-forward-off]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/observability/index#obs-disable-alert-forwarding
[acm-forward-uwl-off]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/observability/index#disable-uw-alert-forwarding
[acm-am-config]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/observability/index#configure-obs-alerts
[acm-silence]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/observability/index#silence-alerts
[acm-grafana]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17/html-single/observability/index#view-explore-data-grafana
[coo-acm]: https://docs.redhat.com/en/documentation/red_hat_openshift_cluster_observability_operator/1-latest/html-single/ui_plugins_for_red_hat_openshift_cluster_observability_operator/index#coo-monitoring-ui-plugin-install_monitoring-ui-plugin
