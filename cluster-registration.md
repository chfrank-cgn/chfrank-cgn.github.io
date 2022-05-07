# Cluster Registration

Even though [Rancher](https://rancher.com/) offers node and cluster drivers for most cloud providers and hypervisors, it can sometimes be helpful to import existing clusters and register them.

In this article, we'll be using [Terraform](https://www.terraform.io/) to register a cluster with Rancher. For more details on Rancher's options for cluster registration, look at the Rancher [documentation](https://rancher.com/docs/rancher/v2.6/en/cluster-provisioning/registered-clusters/).

We will be covering both options: Registering an existing cluster in a hosted Kubernetes provider or using a generic import. For both approaches, we will be using Microsoft Azure.

## Terraform Provider

As a first step, we need to define the [Rancher2 provider](https://www.terraform.io/docs/providers/rancher2/index.html):

```
provider "rancher2" {
  api_url = var.rancher-url
  token_key = var.rancher-token
}
```

## Hosted Kubernetes Provider

To import a cluster from a hosted Kubernetes provider, we first need to define the credentials:

```
resource "rancher2_cloud_credential" "credential_az" {
  name = "Azure Credentials"
  azure_credential_config {
    client_id = var.az-client-id
    client_secret = var.az-client-secret
    subscription_id = var.az-subscription-id
  }
}
```

and then define the cluster itself, using the actual name of the cluster from hosted Kubernetes provider as input:

```
resource "rancher2_cluster" "cluster_az" {
  name         = "aks-${random_id.instance_id.hex}"
  description  = "Terraform"

  aks_config_v2 {
    cloud_credential_id = rancher2_cloud_credential.credential_az.id
    name = var.az-aks-cluster
    resource_group = var.az-resource-group
    resource_location = var.az-region
    imported = true
  }
}
```

After terraform plan/apply, the cluster will be registered and fully manageable within Rancher.

You can find the sample plan files for this import on my [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/aks-import-1).

## Generic Kubernetes

We need to take a couple of extra steps to use the generic procedure that does not rely on a hosted provider.

### Kubernetes Provider

First, we need to add Terraform's [Kubernetes provider](https://registry.terraform.io/providers/hashicorp/kubernetes/2.11.0) to our plan, using the Azure credentials as an example:

```
provider "kubernetes" {
  host = data.azurerm_kubernetes_cluster.cluster_aks.kube_config.0.host
  username = data.azurerm_kubernetes_cluster.cluster_aks.kube_config.0.username
  password = data.azurerm_kubernetes_cluster.cluster_aks.kube_config.0.password
  client_certificate = base64decode(data.azurerm_kubernetes_cluster.cluster_aks.kube_config.0.client_certificate)
  client_key = base64decode(data.azurerm_kubernetes_cluster.cluster_aks.kube_config.0.client_key)
  cluster_ca_certificate = base64decode(data.azurerm_kubernetes_cluster.cluster_aks.kube_config.0.cluster_ca_certificate)
}
```

### Rancher settings

Second, we need to retrieve a couple of settings from Rancher, which we will later need to complete the installation:

```
data "rancher2_setting" "server_version" {
    name = "server-version"
}
data "rancher2_setting" "install_uuid" {
    name = "install-uuid"
}
data "rancher2_setting" "server_url" {
    name = "server-url"
}
```

### Cluster & Registration Secret

We then define the cluster in Rancher:

```
resource "rancher2_cluster" "cluster_az" {
  name         = "aks-${random_id.instance_id.hex}"
  description  = "Terraform"
}
```

and prepare the registration secret for the cattle-cluster-agent:

```
resource "kubernetes_secret" "cattle_credentials_az" {
  metadata {
    name      = "cattle-credentials-${random_id.instance_id.hex}"
    namespace = "cattle-system"
  }
  data = {
    token = rancher2_cluster.cluster_az.cluster_registration_token.0.token
    url = var.rancher-url
  }
  type = "Opaque"
}
```

### Import

Now we're ready for the actual import. The GUI would present us with a Kubernetes manifest to apply; using [k2tf](https://github.com/sl1pm4t/k2tf) or a similar tool, we can convert the manifest to HCL and use it directly in Terraform.

To make the manifest universal, we substitute the data with the variables that we retrieved from Rancher earlier. You can find a converted manifest in main.tf in the sample plan files for this import on my [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/aks-import-2).

### Lifecycle protection

The cattle-cluster-agent will modify some of the resources during its startup, so we need to protect all the converted Kubernetes resources in the manifest from Terraform lifecycle change:

```
  lifecycle {
    ignore_changes = all
  }
```

After terraform plan/apply, the cluster will be registered within Rancher.

### Destruction

Currently, destroying the imported cluster from Terraform will fail due to a finalizer in the cattle-system namespace. Removing this finalizer through the Azure console will allow Terraform to finish; see [36450](https://github.com/rancher/rancher/issues/36450).

## Troubleshooting

The best place for troubleshooting during plan execution is the output of the pod running Rancher - it provides detailed information on what Rancher is currently doing and complete error messages if something goes wrong.

Happy Ranching!

*(Last update: 5/7/22, cf)*

