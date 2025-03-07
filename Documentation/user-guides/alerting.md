<br>
<div class="alert alert-info" role="alert">
    <i class="fa fa-exclamation-triangle"></i><b> Note:</b> Starting with v0.39.0, Prometheus Operator requires use of Kubernetes v1.16.x and up.<br><br>
</div>

# Alerting

This guide assumes you have a basic understanding of the `Prometheus` resource and have read the [getting started guide](getting-started.md).

The Prometheus Operator introduces an Alertmanager resource, which allows users to declaratively describe an Alertmanager cluster. To successfully deploy an Alertmanager cluster, it is important to understand the contract between Prometheus and Alertmanager.

The Alertmanager may be used to:

* Deduplicate alerts fired by Prometheus
* Silence alerts
* Route and send grouped notifications via providers (PagerDuty, OpsGenie, ...)

Prometheus' configuration also includes "rule files", which contain the [alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/). When an alerting rule triggers it fires that alert against *all* Alertmanager instances, on *every* rule evaluation interval. The Alertmanager instances communicate to each other which notifications have already been sent out. For more information on this system design, see the [High Availability scheme description](../high-availability.md).

The Prometheus Operator also introduces an AlertmanagerConfig resource, which allows users to declaratively describe Alertmanager configurations. The AlertmanagerConfig resource is currently v1alpha1, testing and feedback are welcome.

First, create an example Alertmanager cluster, with three instances.

```yaml mdox-exec="cat example/user-guides/alerting/alertmanager-example.yaml"
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  name: example
spec:
  replicas: 3
```

The Alertmanager instances will not be able to start up, unless a valid configuration is given. An config file Secret will be composed by taking an optional base config file Secret specified through the `configSecret` field in the Alertmanager resource Spec, and merging that with any AlertmanagerConfig resources that get matched by using the `alertmanagerConfigSelector` and `alertmanagerConfigNamespaceSelector` selectors from the `Alertmanager` resource.

For more information on configuring Alertmanager, see the Prometheus [Alerting Configuration document](https://prometheus.io/docs/alerting/configuration/).

## AlertmanagerConfig Resource

The following example configuration creates an AlertmanagerConfig resource that sends notifications to a non-existent `wechat` receiver:

```yaml mdox-exec="cat example/user-guides/alerting/alertmanager-config-example.yaml"
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: config-example
  labels:
    alertmanagerConfig: example
spec:
  route:
    groupBy: ['job']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 12h
    receiver: 'wechat-example'
  receivers:
  - name: 'wechat-example'
    wechatConfigs:
    - apiURL: 'http://wechatserver:8080/'
      corpID: 'wechat-corpid'
      apiSecret:
        name: 'wechat-config'
        key: 'apiSecret'

---
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: wechat-config
data:
  apiSecret: d2VjaGF0LXNlY3JldAo=
```

Save the above AlertmanagerConfig in a file called `alertmanager-config.yaml` and create a resource from it using `kubectl`.

```bash
$ kubectl create -f alertmanager-config.yaml
```

The `alertmanagerConfigSelector` field in the Alertmanager resource Spec needs to be specified so that the operator can select such AlertmanagerConfig resources. In the previous example, the label `alertmanagerConfig: example` is added, so the Alertmanager instance should be updated, adding the `alertmanagerConfigSelector`:

```yaml mdox-exec="cat example/user-guides/alerting/alertmanager-selector-example.yaml"
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  name: example
spec:
  replicas: 3
  alertmanagerConfigSelector:
    matchLabels:
      alertmanagerConfig: example
```

## Manually Managed Secret

The following example configuration sends notifications against to a `webhook`:

```yaml mdox-exec="cat example/user-guides/alerting/alertmanager.yaml"
global:
  resolve_timeout: 5m
route:
  group_by: ['job']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'webhook'
receivers:
- name: 'webhook'
  webhook_configs:
  - url: 'http://alertmanagerwh:30500/'
```

Save the above Alertmanager config in a file called `alertmanager.yaml` and create a secret from it using `kubectl`

Alertmanager instances require the secret resource naming to follow the format
`alertmanager-{ALERTMANAGER_NAME}`. In the previous example, the name of the Alertmanager is `example`, so the secret name must be `alertmanager-example`, and the name of the config file `alertmanager.yaml`. Also, the name of the secret can be set through the field `configSecret` in Alertmanager configuration, if you desire to use a different one.

```bash
$ kubectl create secret generic alertmanager-example --from-file=alertmanager.yaml
```

Note that Alertmanager configurations can use templates (`.tmpl` files), which can be added on the secret along with the `alertmanager.yaml` config file. For example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-example
data:
  alertmanager.yaml: {BASE64_CONFIG}
  template_1.tmpl: {BASE64_TEMPLATE_1}
  template_2.tmpl: {BASE64_TEMPLATE_2}
  ...
```

Templates will be placed on the same path as the configuration. To load the templates, the configuration (`alertmanager.yaml`) should point to them:

```yaml
templates:
- '*.tmpl'
```

## Expose Alertmanager

Once the operator merges the optional manually specified Secret with any selected `AlertmanagerConfig` resources, a new configuration Secret is created with the name `alertmanager-<Alertmanager name>-generated`, and is mounted into Alertmanager Pods created through the Alertmanager object.

To be able to view the web UI, expose it through a Service. A simple way to do this is to use a Service of type `NodePort`.

```yaml mdox-exec="cat example/user-guides/alerting/alertmanager-example-service.yaml"
apiVersion: v1
kind: Service
metadata:
  name: alertmanager-example
spec:
  type: NodePort
  ports:
  - name: web
    nodePort: 30903
    port: 9093
    protocol: TCP
    targetPort: web
  selector:
    alertmanager: example
```

Once created it allows the web UI to be accessible via a Node's IP and the port `30903`.

## Fire Alerts

This Alertmanager cluster is now fully functional and highly available, but no alerts are fired against it. Create Prometheus instances to fire alerts to the Alertmanagers.

```yaml mdox-exec="cat example/user-guides/alerting/prometheus-example.yaml"
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: example
spec:
  replicas: 2
  alerting:
    alertmanagers:
    - namespace: default
      name: alertmanager-example
      port: web
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  ruleSelector:
    matchLabels:
      role: alert-rules
      prometheus: example
```

The above configuration specifies a `Prometheus` that finds all of the Alertmanagers behind the `Service` created with `alertmanager-example-service.yaml`. The `alertmanagers` `name` and `port` fields should match those of the `Service` to allow this to occur.

### Rule Selection

Prometheus rule files are held in `PrometheusRule` custom resources. Use the label selector field `ruleSelector` in the Prometheus object to define the rule files that you want to be mounted into Prometheus.

By default, only `PrometheusRule` custom resources in the same namespace as the `Prometheus` custom resource are discovered.

This can be further controlled with the `ruleNamespaceSelector` field, which is a [`metav1.LabelSelector`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#labelselector-v1-meta).

To discover from all namespaces, pass an empty dict (`ruleNamespaceSelector: {}`).

To discover from all namespaces with a certain label, use the `matchLabels` field:

```yaml mdox-exec="cat example/user-guides/alerting/prometheus-example-rule-namespace-selector.yaml"
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: example
spec:
  replicas: 2
  alerting:
    alertmanagers:
    - namespace: default
      name: alertmanager-example
      port: web
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  ruleSelector:
    matchLabels:
      role: alert-rules
      prometheus: example
  ruleNamespaceSelector:
    matchLabels:
      team: frontend
```

This will discover `PrometheusRule` custom resources from all namespaces with a `team=frontend` label.

In case you want to select individual namespace by their name, you can use the `kubernetes.io/metadata.name` label, which gets populated automatically with the [`NamespaceDefaultLabelName`](https://kubernetes.io/docs/reference/labels-annotations-taints/#kubernetes-io-metadata-name) feature gate.

### `PrometheusRule` labelling

The best practice is to label the `PrometheusRule`s containing rule files with `role: alert-rules` as well as the name of the Prometheus object, `prometheus: example` in this case.

```yaml mdox-exec="cat example/user-guides/alerting/prometheus-example-rules.yaml"
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  creationTimestamp: null
  labels:
    prometheus: example
    role: alert-rules
  name: prometheus-example-rules
spec:
  groups:
  - name: ./example.rules
    rules:
    - alert: ExampleAlert
      expr: vector(1)
```

The example `PrometheusRule` always immediately triggers an alert, which is only for demonstration purposes. To validate that everything is working properly have a look at each of the Prometheus web UIs.

Use kubectl's proxy functionality to view the web UI without a Service.

Run:

```bash
kubectl proxy --port=8001
```

Then the web UI of each Prometheus instance can be viewed, they both have a firing alert called `ExampleAlert`, as defined in the loaded alerting rules.

* http://localhost:8001/api/v1/proxy/namespaces/default/pods/prometheus-example-0:9090/alerts
* http://localhost:8001/api/v1/proxy/namespaces/default/pods/prometheus-example-1:9090/alerts

Looking at the status page for "Runtime & Build Information" on the Prometheus web UI shows the discovered and active Alertmanagers that the Prometheus instance will fire alerts against.

* http://localhost:8001/api/v1/proxy/namespaces/default/pods/prometheus-example-0:9090/status
* http://localhost:8001/api/v1/proxy/namespaces/default/pods/prometheus-example-1:9090/status

These show three discovered Alertmanagers.

Heading to the Alertmanager web UI now shows one active alert, although all Prometheus instances are firing it. [Configuring the Alertmanager](https://prometheus.io/docs/alerting/configuration/) further allows custom alert routing, grouping and notification mechanisms.
