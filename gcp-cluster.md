# GCP Cluster

Rancher 2.x does not include a node driver for GCP, just a cluster driver for GKE, which should be sufficient for most needs. However, to build a Rancher Kubernetes cluster on GCP, we can use Terraform and the custom cluster setup.

## Terraform directory setup

First step is to set up a new terraform plan (directory) for the GCP cluster:

```
-rw-r--r-- 1 cfrank ewscom  341 Jan 21 06:10 data.tf
drwxr-xr-x 2 cfrank ewscom 4096 Jan 19 06:48 files
-rw-r--r-- 1 cfrank ewscom 2076 Feb 11 09:25 main.tf
-rw-r--r-- 1 cfrank ewscom  278 Feb  8 04:46 output.tf
-rw-r--r-- 1 cfrank ewscom  319 Jan 21 07:38 provider.tf
-rw-r--r-- 1 cfrank ewscom  364 Jan 19 15:08 terraform.tf
-rw-r--r-- 1 cfrank ewscom  509 Feb  9 07:56 variables.tf
```

In the following, we'll exame the individual files - you can find the complete sources on [GitHub](https://github.com/chfrank-cgn/Rancher/tree/master/gcp-cluster). Terraform is state-based, for state storage we'll use the Terraform cloud:

```
  backend "remote" {
    organization = "xxxxxxxxxxx"
    workspaces {
      name = "compute-prod-us-central"
    }
  }
```

## Provider

To set up a custom cluster, we need define two providers, first Google for the infrastructure:

```
provider "google" {
  project     = "xxxxxxx-xxxxxx-xxxxxx"
  credentials = file("xxxxxxxxxxxx.json")
  region      = "us-central1"
  zone        = "us-central1-c"
}
```

and second Rancher for Kubernetes:

```
provider "rancher2" {
  api_url = var.rancher-url
  token_key = var.rancher-token
  insecure = true
}
```

## Resources

The next definitions are for all the cluster resources.

### Random ID

To have a unique identifier for cluster and nodes, we use Random:

```
resource "random_id" "instance_id" {
 byte_length = 3
}
```

### Cluster

We define the Rancher cluster, using the name from above and set Kubernetes networking and version:

```
resource "rancher2_cluster" "cluster_gcp" {
  name         = "gcp-${random_id.instance_id.hex}"
  description  = "Terraform"

  rke_config {
    kubernetes_version = var.k8version
    ignore_docker_version = false
    network {
      plugin = "flannel"
    }
  }
}
```

## Output

To facilitate ssh access, we show the external IP addresses upon successful completion:

```
output "Public" {
  value = "${google_compute_instance.vm_gcp.*.network_interface.0.access_config.0.nat_ip}"
}
```

Result
------

After terraform init / plan / apply, the resulting Kubernetes cluster will look like this and will be fully available in Rancher:

```
NAME           STATUS   ROLES                      AGE   VERSION   INTERNAL-IP   OS-IMAGE
rke-1ebacc-0   Ready    controlplane,etcd,worker   43m   v1.15.9   10.240.0.80   Ubuntu 18.04.3 LTS
rke-1ebacc-1   Ready    controlplane,etcd,worker   43m   v1.15.9   10.240.0.78   Ubuntu 18.04.3 LTS
rke-1ebacc-2   Ready    controlplane,etcd,worker   43m   v1.15.9   10.240.0.81   Ubuntu 18.04.3 LTS
```

Happy Ranching!



*(Last update: 2/13/20, cf)*
