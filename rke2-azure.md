# Azure Cloud Provider

Since 2.7.X, [Rancher](https://rancher.com/) RKE2 has become the default Kubernetes version for provisioning all major hyper scalers and virtualization platforms.

Let's look at enabling a cloud provider using Terraform.

## Terraform Plan

I'm assuming that you have set up Terraform already. For the actual RKE2 cluster, we will be using the rancher2_cluster_v2 resource:

```
resource "rancher2_cluster_v2" "cluster_az" {
  name = "az-${random_id.instance_id.hex}"
  kubernetes_version = var.k8version
  annotations = {
    "field.cattle.io/description" = "Terraform"
  }
...
}
```

## rke_config

Assuming that you already have a working plan to stand up an RKE2 cluster, we'll focus on the cloud provider configuration in the rke_config section, as described in the RKE2 documentation [here](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/set-up-cloud-providers/azure#rke2-cluster-set-up-in-rancher). 

### machine_selector_config

First, we need to pass the Azure credentials to RKE2 in JSON format:

```
machine_selector_config {
  config = {
    cloud-provider-config = <<EOF
    { 
      "cloud": "AzurePublicCloud",
      "aadClientId": "${var.az-client-id}",
      "aadClientSecret": "${var.az-client-secret}",
      "subscriptionId": "${var.az-subscription-id}",
      "tenantId": "${var.az-tenant-id}",
      "resourceGroup": "${var.az-resource-group}",
      "location": "${var.az-region}",
      "subnetName": "${var.az-subnet}",
      "vnetName": "${var.az-vnet}",
      "securityGroupName": "${var.az-sec-group}",
      "primaryAvailabilitySetName": "${var.az-avset}",
      "cloudProviderBackoff": false,
      "useManagedIdentityExtension": false,
      "useInstanceMetadata": true
    }
    EOF
    cloud-provider-name = "azure"
  }
}
```

### machine_global_config

The next step is to pass an argument to the controller not to use Azure networking for the internal networks:

```
machine_global_config = <<EOF
  kube-controller-manager-arg:
    - '--configure-cloud-routes=false'
EOF
```

## Azure Disk/File

You can access most Azure cloud functions with the cloud provider, such as creating a Load Balancer service.

You can find the Azure Disk CSI driver [here](https://github.com/kubernetes-sigs/azuredisk-csi-driver#install-driver-on-a-kubernetes-cluster) and the Azure File CSI driver [here](https://github.com/kubernetes-sigs/azurefile-csi-driver#install-driver-on-a-kubernetes-cluster).

Both drivers will be looking for a secret azure-cloud-provider in the kube-system namespace that should contain the base64-encoded cloud provider config from above:

```
apiVersion: v1
data:
  cloud-config: BASE64-encoded JSON config
kind: Secret
metadata:
  name: azure-cloud-provider
  namespace: kube-system
type: Opaque
```

After installing the drivers with Helm, create the storage classes, and you're all set:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-disk
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: disk.csi.azure.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  storageaccounttype: Standard_LRS
  resourceGroup: az-cluster-2
  kind: Managed
```

## Troubleshooting

The best place for troubleshooting during plan execution is the output of the pod running Rancher - it provides detailed information on what Rancher is currently doing and complete error messages if something goes wrong.

You can find sample plan files for this RKE2 installation with Rancher on my [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/az-cluster-2).

Happy Ranching!

*(Last update: 5/21/23, cf)*

