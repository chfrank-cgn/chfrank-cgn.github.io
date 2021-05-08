# Big Prometheus

With Rancher 2.5 comes the new v2 [rancher monitoring operator](https://rancher.com/docs/rancher/v2.x/en/monitoring-alerting/v2.5/); the app is based on the upstream [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart.

With the new app, it has become straightforward to configure Prometheus for a federated setup. In this example, we'll look at how to send the cluster metrics to a central Prometheus instance, the [Grafana Cloud](https://grafana.com/products/cloud/). Other options could be to send metrics to [Thanos](https://thanos.io/) or Cortex(https://cortexmetrics.io/) similarly.

We'll use Terraform to install and configure monitoring because it's easier to document, but you could do it in the new UI as well.

## Resource

In our plan file, we define monitoring as a new v2 app resource:

```

resource "rancher2_app_v2" "monitor_ec2" {
  cluster_id = rancher2_cluster.cluster_ec2.id
  name = "rancher-monitoring"
  namespace = "cattle-monitoring-system"
  repo_name = "rancher-charts"
  chart_name = "rancher-monitoring"
  chart_version = var.monchart
  values = templatefile("${path.module}/files/values.yaml", {})
}
```

## Values

To enable federation, we pass the following to the Helm chart:

```
prometheus:
  prometheusSpec:
    remoteWrite:
    - url: https://prometheus-us-central1.grafana.net/api/prom/push
      basicAuth:
        username: 
          name: remote-writer
          key: username
        password: 
          name: remote-writer
          key: password
```

## Credentials

The URL is pointing to the federation endpoint, and we'll need to create a secret containing the user credentials in the applications' namespace:

```
# Monitoring namespace
resource "rancher2_namespace" "promns_ec2" {
  name = "cattle-monitoring-system"
  project_id = data.rancher2_project.system.id
  description = "Terraform"
}
```

```
# Monitoring secret
resource "rancher2_secret_v2" "promsecret_ec2" {
  cluster_id = rancher2_cluster.cluster_ec2.id
  name = "remote-writer"
  namespace = "cattle-monitoring-system"
  type = "kubernetes.io/basic-auth"
  data = {
    username = var.prom-remote-user
    password = var.prom-remote-pass
  }
}
```

## Result

After terraform init / plan / apply, your cluster monitoring will be sending metrics data, and you can start defining your new dashboards on the central instance.

Depending on your configuration, you might need to include dependency or lifecycle settings in the resource definitions; you can find sample plan files for this monitoring setup on my [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/ec2-cluster-1). 

Happy Ranching!

*(Last update: 5/8/21, cf)*
