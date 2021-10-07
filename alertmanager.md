# Alertmanager

Beginning with Rancher 2.5, there is the new [Rancher Monitoring Application](https://rancher.com/docs/rancher/v2.6/en/monitoring-alerting/); this v2 app is based on the upstream [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart and includes Alertmanager.

Configuration of Alertmanager is covered in detail in the [Prometheus documentation](https://prometheus.io/docs/alerting/latest/configuration/); we'll not cover the actual configuration items here, but look at two of the available options to configure it in Rancher instead.

## values.yaml

Our first option would be to pass the configuration values during installation.

To do this, in our plan file, we define a values.yaml file with the desired configuration:

```
# Cluster monitoring
resource "rancher2_app_v2" "monitor_ec2" {
  ...
  values = templatefile("${path.module}/files/values.yaml", {})
}
```

In the configuration file, we then pass the Alertmanager configuration to the app's Helm chart, in this example for a simple notification in Slack:

```
alertmanager:
  config:
    receivers:
      - name: slack-receiver
        slack_configs:
          - send_resolved: true
            api_url: >-
              https://hooks.slack.com/services/xxx/xxx
            channel: '#ec2-cluster-1'
      - name: 'null'
    route:
      group_by: ['job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 1h
      receiver: 'slack-receiver'
      routes:
      - match:
          alertname: Watchdog
        receiver: 'null'
        continue: true
      - match: {}
        match_re:
          alertname: '.*'
        receiver: slack-receiver
```

It might not be entirely helpful to have continue set to true for the Watchdog alert, but it can help troubleshoot receivers and routes.

Right after terraform init / plan / apply, you should receive the first notification in Slack.

The defined routes and receivers will show up in the Rancher GUI.

As always, you can find the plan files on my [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/ec2-cluster-1).

## Custom Resource Definition

Our second, more dynamic option would be to use a custom resource definition.

The syntax is slightly different from the syntax in the values.yaml file, it is described in the documentation for the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator/blob/master/README.md).

In this example, we will send EMail messages instead of Slack notifications:

```
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: default-receiver
  namespace: cattle-monitoring-system
  labels:
    alertmanagerConfig: default-receiver
spec:
  receivers:
    - name: 'default-receiver'
      emailConfigs:
        - to: 'user.name@company.com'
          sendResolved: true
          smarthost: 'smtprelay.company.com:25'
          from: 'rancher@company.com'
          requireTLS: no
  route:
    groupBy: ['node']
    groupWait: 30s
    groupInterval: 5m
    receiver: 'default-receiver'
    repeatInterval: 3h
    matchers:
      - name: alertname
        regex: true
        value: '.*'
```
Apply the CRD with kubectl, and the operator will perform the changes; to check whether the configuration was successful, you can navigate to Alertmanager in the Rancher GUI in check the Status tab.

Happy Alerting!

*(Last update: 10/7/21, cf)*
