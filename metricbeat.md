# RKE2 and Metricbeat/Filebeat

Rancher bundles a powerful monitoring suite (Prometheus) and a robust log forwarding option (Banzai Logging). However, if you're already invested in Elasticsearch and want to use Kibana as your observability interface, [Elastic](https://www.elastic.co/) offers two lightweight shippers that work well with RKE and RKE2.

## Metricbeat

To send metrics to your Elasticsearch cluster, you can download [Metricbeat](https://www.elastic.co/beats/metricbeat) from Elastic.

Download the [manifest](https://www.elastic.co/guide/en/beats/metricbeat/current/running-on-kubernetes.html) that matches your Elasticsearch installation:

```
wget https://raw.githubusercontent.com/elastic/beats/7.3/deploy/kubernetes/metricbeat-kubernetes.yaml
```

Customize the login information for your Elasticsearch installation:

```
- name: ELASTICSEARCH_HOST
  value: elasticsearch
- name: ELASTICSEARCH_PORT
  value: "9200"
- name: ELASTICSEARCH_USERNAME
  value: elastic
- name: ELASTICSEARCH_PASSWORD
  value: changeme
```

Metricbeat relies on [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) for some of its data. On RKE2, you'll need to install it:

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-community/kube-state-metrics
```

Alternatively, you could add the prometheus-community catalog to your Rancher Marketplace and install it from there.

Once kube-state-metrics is running, you can start Metricbeat and begin configuring your Kibana dashboards:

```
kubectl create -f metricbeat-kubernetes.yaml
```

## Filebeat

To forward container logs to your Elasticsearch cluster, you can download [Filebeat](https://www.elastic.co/beats/filebeat) from Elastic.

Download the [manifest](https://www.elastic.co/guide/en/beats/filebeat/current/running-on-kubernetes.html) that matches your Elasticsearch installation:

```
wget https://raw.githubusercontent.com/elastic/beats/7.3/deploy/kubernetes/filebeat-kubernetes.yaml
```

Customize the login information for your Elasticsearch installation:

```
- name: ELASTICSEARCH_HOST
  value: elasticsearch
- name: ELASTICSEARCH_PORT
  value: "9200"
- name: ELASTICSEARCH_USERNAME
  value: elastic
- name: ELASTICSEARCH_PASSWORD
  value: changeme
```

For RKE2, you might want to modify the mount point for the container logs:

```
volumes:
- name: varlibdockercontainers
  hostPath:
  path: /var/log/pods
```

In the Elasticsearch output configuration, you can add indices, patterns, and templates to match your installation.

Also, if your Elasticsearch installation uses self-signed certificates, you might want to add an option there:

```
output.elasticsearch:
  ssl.verification_mode: "none"
```

Once Filebeat is configured, you can start it and begin configuring your Kibana dashboards:

```
kubectl create -f filebeat-kubernetes.yaml
```

## Tolerations

Both Metricbeat and Filebeat will deploy as DaemonSet on all worker nodes. If you have a separate control plane, you might want to adjust the tolerations in the deployment files.


Happy Measuring!

*(Last update: 3/21/22, cf)*
